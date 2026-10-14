# 1. 전체 구성도 (논리 아키텍처)

> "무엇이 어디에 있나" — 프레임워크 기능 범위 전체를 보여준다

```mermaid
flowchart TB
  subgraph L1["① 설계 레이어"]
    D["Designer<br/>GUI · YAML"]
    R["Registry<br/>Tool · 커넥터 · 프롬프트"]
    P["조직 정책<br/>예산 · 보안 · 재시도"]
  end
  subgraph L2["② 정의 레이어"]
    C["Compiler<br/>검증 · 확정"]
    IR[("IR store<br/>불변 버전")]
  end
  subgraph L3["③ 실행 레이어"]
    RM["Run manager<br/>생명주기"]
    S["Scheduler<br/>다음 노드 결정"]
    Q[["Queue<br/>유실 없는 전달"]]
    W["Worker<br/>Task · Agent · Human 실행기"]
    ST[("State store<br/>Checkpoint · 이력")]
  end
  subgraph L4["④ 연동 레이어"]
    LLM["LLM gateway"]
    MCP["MCP gateway"]
    H["HITL service"]
  end
  subgraph L5["⑤ 공통 관리"]
    X["Guardrail · Observability · Security · Evaluation"]
  end
  L1 --> L2 --> L3 --> L4
```

## 컴포넌트 역할 (한 줄)

| 레이어 | 컴포넌트 | 역할 | 키워드 |
|---|---|---|---|
| 설계 | Designer | GUI · YAML로 워크플로 정의 작성 | 선언적 |
| 설계 | Registry | Tool · 커넥터 · 프롬프트 등록 (메타: side_effect, approval) | 부품 |
| 설계 | 조직 정책 | 예산 · 보안 · 재시도 기본값 | 상속 |
| 정의 | Compiler | 검증하고 빈칸 채워 IR로 확정 | 배포 전 검증 |
| 정의 | IR store | 확정된 정의를 버전별 보관 | 불변 |
| 실행 | Run manager | 실행 건 생성 · 취소 · 재개 · 조회 | 생명주기 |
| 실행 | Scheduler | 완료 이벤트 받아 Edge 평가, 다음 노드 결정 | 결정 |
| 실행 | Queue | 결정된 작업을 Worker에 유실 없이 전달 | 전달 |
| 실행 | Worker | 노드 실행 (상태 없음, 수평 확장) | stateless |
| 실행 | State store | 모든 State · Checkpoint · 이력 | 장애 복구 |
| 연동 | LLM gateway | 모델 라우팅 · 폴백 · 호출 한도 | 통제 지점 |
| 연동 | MCP gateway | Tool 인증 위임 · 권한 · 승인 가로채기 | 통제 지점 |
| 연동 | HITL service | 승인 요청 · 알림 · 타임아웃 · 재개 요청 | 분리 |
| 공통 | Guardrail · Security | 입출력 · 권한 검사, 마스킹 | 보호 |
| 공통 | Observability · Evaluation | Trace · 비용 · 품질 평가 | 관측 |

## 설계 원칙 (장표 2용)

1. **통제와 자율성의 균형**: 결정론 뼈대 + 제한된 Agent 구간
2. **정의와 실행의 분리**: DSL → IR 컴파일, 엔진은 IR만 실행
3. **제어 평면과 실행 평면 분리**: 상태는 State store에, Worker는 stateless
4. **단일 통제 지점**: 모든 모델 · Tool 호출은 Gateway 경유
5. **런타임 비종속**: 엔진 어댑터로 LangGraph / Temporal 등 교체 가능
