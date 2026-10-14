# 03. 주요 처리흐름

> Task 조회 → Agent 판단 → 사람 승인 → 쓰기 → 완료

```mermaid
sequenceDiagram
  participant T as Trigger
  participant RM as Run manager
  participant SQ as Scheduler+Queue
  participant W as Worker
  participant GW as Gateway (LLM·MCP)
  participant H as HITL
  T->>RM: ① 실행 요청 (멱등키)
  RM->>SQ: ② IR 로드 · Run 시작
  SQ->>W: ③ Task 노드 투입
  W->>GW: ④ 조회 (MCP)
  W-->>SQ: ⑤ 완료 이벤트 (Checkpoint 저장)
  SQ->>W: ⑥ Agent 노드 투입
  loop ReAct 반복 (루프 가드 · 토큰 예약/정산)
    W->>GW: ⑦ LLM 호출 · Tool 호출
  end
  W-->>H: ⑧ 승인 요청 (Run → WAITING, Worker 반환)
  Note over H: 사람 결정 (수 분 ~ 수 일)
  H-->>RM: ⑨ 승인 → 재개 요청
  RM->>SQ: ⑩ 재개
  SQ->>W: ⑪ 쓰기 Task (멱등키)
  RM-->>T: ⑫ 완료 통지
```

| 구간 | 단계 | 설명 |
|---|---|---|
| 시작 | ①~② | 멱등 생성, 버전 고정, 초기 Checkpoint |
| Task 노드 | ③~⑤ | 결정론 조회, 완료 이벤트로 다음 결정 |
| Agent 노드 | ⑥~⑦ | 허용 Tool 범위에서 반복, 매 반복 가드 · 예산 검사 |
| 승인 | ⑧~⑩ | 원자적 중단 → 사람 결정 → Checkpoint에서 재개 |
| 완료 | ⑪~⑫ | 쓰기는 결정론 Task + 멱등키 |

**강조점**: ⑧과 ⑨ 사이가 며칠이어도 Worker는 아무것도 붙잡고 있지 않다.
