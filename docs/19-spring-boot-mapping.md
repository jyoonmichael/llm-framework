# 19. 참조 구현: Spring Boot

> 설계 박스 하나 = 패키지 하나. 같은 코드베이스를 **프로파일**로 api / scheduler / worker 역할 배포.

## 패키지 구조

```
com.example.agentflow
├── design                        ① 설계
│   ├── registry   ToolRegistry, ToolSpec(sideEffect, approval), PromptRegistry
│   └── policy     Policy, PolicyDefaults, PolicyEnforced, PolicyResolver
├── definition                    ② 정의
│   ├── dsl        WorkflowDocument, NodeDef, EdgeDef, DslParser
│   ├── compile    WorkflowValidator, WorkflowCompiler, CompileException
│   └── ir         WorkflowIr, NodeIr(Task/Agent/Human/Map), EdgeIr, IrRepository, GraphHasher
├── runtime                       ③ 실행
│   ├── run        RunController, RunManager, Run, RunStatus, RunRepository
│   ├── schedule   Scheduler, EdgeEvaluator, JoinTracker, MapWindow, RetryPolicy
│   ├── queue      WorkQueue, WorkItem, DelayQueue
│   ├── worker     WorkerLoop, LeaseService, Lease, HeartbeatTask, LeaseSweeper
│   ├── executor   NodeExecutor, TaskExecutor, AgentExecutor, HumanExecutor, NodeResult
│   ├── state      StateStore, CheckpointRepository, IterationJournal, Outbox, OutboxRelay
│   ├── budget     BudgetGuard (reserve / settle / release)
│   └── expr       ExpressionEvaluator (CEL)
├── integration                   ④ 연동
│   ├── llm        LlmGateway, LogicalModelRouter, LlmResponse(modelUsed, degraded, usage)
│   ├── tool       McpGateway, ConnectorInvoker, CredentialVault
│   └── hitl       HitlService, HumanTask, ResumeTokenService, InboxController
├── common                        ⑤ 공통
│   ├── guard      GuardrailChain, InjectionFence, MaskingService
│   ├── observe    TraceSupport
│   └── security   RbacService
└── extension                     확장
    ├── realtime   SseController, RunEventSubscriber, EventLogReplayer      (E1)
    ├── planning   PlannerExecutor, ChildRunSpawner                          (E4)
    └── memory     MemoryService, MemoryExtractor, MemoryRecall              (E5)
```

## 배포 프로파일

| 프로파일 | 활성 빈 | 확장 기준 |
|---|---|---|
| `api` | RunController, RunManager, InboxController, WorkflowCompiler, SseController | 요청 수 · SSE 연결 수 |
| `scheduler` | Scheduler, OutboxRelay, LeaseSweeper, DelayQueue, MemoryExtractor | run_id 파티션 수 |
| `worker` | WorkerLoop, 실행기, Gateway, BudgetGuard | 큐 깊이 · LLM 속도 제한 |

## 기술 매핑

| 설계 | Spring 기술 |
|---|---|
| Run API · Inbox · SSE | Spring Web (`SseEmitter`) |
| State store · Outbox · 펜싱 | JdbcTemplate 조건부 UPDATE, `@Transactional` |
| Queue · 이벤트 | Spring AMQP / Kafka, 또는 PostgreSQL `SKIP LOCKED` |
| Heartbeat · Outbox 릴레이 · 임대 청소 | `@Scheduled` |
| LLM · MCP | Spring AI (ChatClient, MCP client) |
| Circuit breaker · 재시도 | Resilience4j |
| Trace | Micrometer Tracing + OpenTelemetry |

---

## 핵심 코드 스켈레톤

### IR · 노드 결과 (문서와 동일한 7종)

```java
public sealed interface NodeIr permits TaskNodeIr, AgentNodeIr, HumanNodeIr, MapNodeIr {
    String id(); List<String> reads(); List<String> writes();
}
public record AgentNodeIr(String id, List<String> reads, List<String> writes,
        PromptRef prompt, List<ToolRef> tools, String logicalModel,
        int maxIter, long maxTokens, JsonSchema outputSchema, MemorySpec memory) implements NodeIr {}
public record MapNodeIr(String id, List<String> reads, List<String> writes,
        String itemsRef, String body, int maxConcurrency, int batchSize,
        FailurePolicy failurePolicy, String reducer) implements NodeIr {}

public sealed interface NodeResult {
    record Completed(Map<String, Object> writes) implements NodeResult {}
    record Suspended(String reason, ResumeSpec resume) implements NodeResult {}
    record RetryableFailure(String code) implements NodeResult {}
    record PermanentFailure(String code) implements NodeResult {}
    record LimitExceeded(String reason) implements NodeResult {}
    record Blocked(String policyId) implements NodeResult {}
    record Cancelled() implements NodeResult {}
}
```

### Compiler (Policy 적용 순서)

```java
@Service
public class WorkflowCompiler {
    public WorkflowIr compile(String yaml) {
        WorkflowDocument doc = parser.parse(yaml);                 // ① 문법
        List<ValidationError> errors = validator.validate(doc);    // ② 의미 (참조 · reads/writes · 탈출 조건 · reducer)
        if (!errors.isEmpty()) throw new CompileException(errors);
        Policy policy = policyResolver.current();
        Map<String, NodeIr> nodes = doc.nodes().stream()
            .map(n -> policy.applyDefaults(n))                     // 기본값
            .map(n -> resolveRefs(n))                              // 프롬프트 해시 · Registry 메타
            .map(n -> policy.enforce(n))                           // 강제 규칙 (덮어쓰기 불가)
            .collect(toMap(NodeIr::id, identity()));
        List<EdgeIr> edges = policy.injectApprovals(withImplicitEdges(doc.edges(), nodes), nodes);
        return new WorkflowIr(doc.workflow(), doc.version(), hasher.hash(nodes, edges),
                              doc.entry(), nodes, edges);
    }
}
```

### Run manager

```java
@Service
public class RunManager {
    @Transactional
    public Run start(StartRunCommand cmd) {
        return runs.findByIdempotencyKey(cmd.idempotencyKey())                 // 멱등 생성
            .orElseGet(() -> {
                WorkflowIr ir = irRepository.latestDeployed(cmd.workflowId());  // 버전 고정
                Run run = runs.save(Run.created(ir, cmd, cmd.parentRunId()));
                stateStore.saveInitial(run.id(), cmd.input());
                outbox.add(new RunStartedEvent(run.id()));                      // 같은 트랜잭션
                return run;
            });
    }

    @Transactional
    public void resume(String runId, ResumeCommand cmd) {
        resumeTokens.consume(cmd.token(), runId);                              // 1회용
        if (runs.transition(runId, WAITING, RUNNING, cmd.expectedVersion()) == 0)
            throw new RunStateConflict(runId);                                 // CAS
        outbox.add(new ResumeRequestedEvent(runId, cmd.nodeId(), cmd.decision()));
    }

    @Transactional
    public void cancel(String runId) {
        runs.markCancelRequested(runId);                                       // 협조적 취소
        runs.findChildren(runId).forEach(child -> cancel(child.id()));         // 자식 연쇄 취소
        outbox.add(new RunCancelRequestedEvent(runId));
    }
}
```

### Worker (임대 · 펜싱 · ack 순서)

```java
@Component @Profile("worker")
public class WorkerLoop {
    public void handle(WorkItem item, Acknowledgement ack) {
        Lease lease = leaseService.acquire(item.runId(), item.nodeId(), item.itemIndex()); // token +1
        if (lease == null || stateStore.alreadyCommitted(item.dedupeKey())) { ack.ack(); return; }
        heartbeat.start(lease);
        try {
            NodeIr node = irRepository.node(item);
            ExecutionContext ctx = ExecutionContext.of(item, lease, stateStore.load(item.runId()));
            NodeResult result = executors.get(node).execute(node, ctx);
            stateStore.commitWithFence(item, lease.token(), result);           // 펜싱 + Checkpoint + Outbox
            ack.ack();                                                         // 저장 "후" ack
        } catch (FencedOutException e) {
            ack.ack();                                                         // 낡은 Worker: 결과 폐기
        } finally {
            heartbeat.stop(lease);
        }
    }
}
```

### State store (한 트랜잭션)

```java
@Repository
public class StateStore {
    @Transactional
    public void commitWithFence(WorkItem item, long token, NodeResult result) {
        int ok = jdbc.update("""
            UPDATE node_lease SET status = 'DONE'
            WHERE run_id = ? AND node_id = ? AND item_index = ? AND token = ?""",
            item.runId(), item.nodeId(), item.itemIndexOrMinus1(), token);
        if (ok == 0) throw new FencedOutException(item, token);
        checkpoints.append(item, result);
        if (result instanceof NodeResult.Suspended s) {
            runs.markWaiting(item.runId());
            budget.releaseReservations(item.runId(), item.nodeId());           // 중단 시 예약 해제
        }
        outbox.add(NodeEvent.of(item, result));                               // HITL 요청 포함
    }
}
```

### Agent executor

```java
@Component
public class AgentExecutor implements NodeExecutor<AgentNodeIr> {
    public NodeResult execute(AgentNodeIr node, ExecutionContext ctx) {
        Conversation conv = contextBuilder.build(node, ctx);                  // reads + 프롬프트 + Tool 명세
        memoryRecall.injectIfConfigured(node, ctx, conv);                     // E5 (조회 결과는 journal 기록)
        journal.replayInto(conv, ctx);                                        // 반복 기록 재생 (재청구 없음)
        while (true) {
            Optional<NodeResult> stop = loopGuard.check(node, ctx, conv);     // 반복 · 데드라인 · 취소 · 루프 감지
            if (stop.isPresent()) return stop.get();

            Reservation r = budget.reserve(ctx, estimate(conv, node));        // 예약 (실패 시 LimitExceeded)
            LlmResponse res;
            try { res = llm.chat(node.logicalModel(), guardrails.input(conv), ctx); }
            finally { budget.settle(r, /* actual */ lastUsageOrZero()); }    // 정산 · 차액 반환

            if (res.isFinalAnswer()) {
                return outputValidator.validate(node, res)
                    .<NodeResult>map(out -> new NodeResult.Completed(pick(out, node.writes())))
                    .orElseGet(() -> repairOnce(node, conv, res));
            }
            for (ToolCall call : res.toolCalls()) {
                if (!node.allows(call.tool())) { conv.addToolError(call, "not allowed"); continue; }
                if (tools.requiresApproval(call.tool()))
                    return new NodeResult.Suspended("tool_approval", ResumeSpec.of(call));
                conv.addToolResult(call, injectionFence.wrap(mcp.invoke(call, ctx)));
            }
            journal.append(ctx, conv.lastTurn(), res.usage());               // 펜싱 토큰 포함 기록
        }
    }
}
```

### LLM gateway

```java
@Service
public class LlmGateway {
    public LlmResponse chat(String logicalModel, Conversation conv, ExecutionContext ctx) {
        for (Endpoint ep : router.candidates(logicalModel, ctx.dataClass(), conv.requires())) {
            if (breakers.isOpen(ep)) continue;
            try {
                return client(ep).call(conv).withModelUsed(ep.name())
                       .withDegraded(!ep.tier().equals(router.primaryTier(logicalModel)));
            } catch (SafetyRefusalException e) {
                throw e;                                                      // 폴백 금지 → Blocked
            } catch (TransientLlmException e) {
                breakers.recordFailure(ep);                                   // 다음 후보
            }
        }
        throw new NoAvailableModelException(logicalModel);
    }
}
```

## 요청 하나가 클래스를 지나가는 순서
1. `RunController.start` → `RunManager.start` (멱등 · 버전 고정 · Outbox)
2. `OutboxRelay` → `Scheduler` → `WorkQueue.enqueue`
3. `WorkerLoop.handle` → `LeaseService.acquire` (펜싱 토큰)
4. 실행기 → `LlmGateway` · `McpGateway` (Agent는 `BudgetGuard` 예약 · 정산)
5. `StateStore.commitWithFence` (펜싱 + Checkpoint + Outbox) → ack
6. `OutboxRelay` → `Scheduler` → 다음 노드 … (`SseController`는 이벤트 구독)
7. HITL: `HitlService` → `InboxController` → `RunManager.resume` → `Scheduler`
