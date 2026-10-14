# 20. 관련 오픈소스

## 우리 설계와의 대응

| 우리 설계 | n8n Queue Mode | LangGraph Server | Dify |
|---|---|---|---|
| DSL 문서 | 워크플로 JSON | 코드 + langgraph.json | Dify DSL (YAML) |
| Compiler 검증 | | | Graph Validator |
| Run manager | Main instance | API Server | API Server |
| Scheduler | Main instance | | GraphEngine |
| Queue | Redis (Bull) | Redis Streams | Celery |
| Worker | Worker 인스턴스 | Worker | Worker pool |
| State store | PostgreSQL | PostgreSQL (Checkpoint) | Variable pool + DB |
| HITL | | interrupt | Human Input 노드 |

**본 설계의 추가 요소**: Compiler · IR · Policy 기반 거버넌스, Agent 실행기 내부 통제(루프 가드 · 토큰 예약/정산 · 반복 기록), 설계 불변식으로 보장되는 정합성

## 구조 참고
- n8n Queue mode: https://docs.n8n.io/hosting/scaling/queue-mode/
- Langrove (오픈소스 LangGraph Server): https://pypi.org/project/langrove/
- Dify: https://pyshine.com/2026/04/20/Dify-Open-Source-LLM-App-Development-Platform/
- 실행 엔진 3종 비교 (BPM · Dify · Agent loop): https://www.besthub.dev/articles/how-three-workflow-engines-run-bpm-token-engine-dify-graph-scheduler-agent-loop-fc55dd152314

## 메커니즘 참고
- LangGraph Durable execution: https://docs.langchain.com/oss/python/langgraph/durable-execution
- MS Agent Framework 체크포인트 (graph signature hash): https://ravichaganti.com/blog/visualizing-and-checkpointing-workflows-in-microsoft-agent-framework/
- MS Agent Framework HITL: https://techcommunity.microsoft.com/blog/azure-ai-foundry-blog/-/4460342
- Pydantic AI + Temporal: https://pydantic.dev/docs/ai/integrations/durable_execution/temporal/
- OTel GenAI Agent Span: https://opentelemetry.io/docs/specs/semconv/gen-ai/gen-ai-agent-spans

## 패턴 분류
- Google ADK: LlmAgent vs Workflow agents (Sequential · Parallel · Loop)
- Anthropic "Building effective agents": prompt chaining, routing, parallelization, orchestrator-workers, evaluator-optimizer
- 워크플로 패턴(Workflow Patterns): 순차, 병렬 분기 · 합류, 배타 선택, 다중 인스턴스, 반복
