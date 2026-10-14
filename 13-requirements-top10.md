# 13. 예상 요구사항 Top 10

| 순위 | 예상 요구사항 | 담당 블록 | 핵심 장치 |
|---|---|---|---|
| 1 | 사람의 승인 · 검토 단계, 수일 대기 | Human executor, HITL | 중단·재개, Outbox, 타임아웃·에스컬레이션, 재개 경쟁 |
| 2 | 장애 시 진행 중 작업 유실 없이 이어서 수행 | Queue, Worker, State store | 임대·재전달, Checkpoint, 반복 기록, 펜싱, 멱등키 |
| 3 | 현업이 코딩 없이 정의 · 수정 | Designer, Compiler, IR | DSL→IR, 배포 전 검증, 정책 상속 |
| 4 | Agent의 무한 반복 · 비용 폭증 · 권한 남용 통제 | Agent executor | 루프 가드, 예산, 최소 권한, 승인 필요 Tool |
| 5 | 여러 Agent 협업 · 상호 검토 | Scheduler, State store | 공유 State, Router 종료 조건, 병렬 분기·합류 |
| 6 | 레거시 · 외부 시스템 다양하게 연계 | MCP GW, Registry | 커넥터 등록, 인증 위임, 쓰기는 Task, 업무 결과 vs 오류 |
| 7 | 판단 근거 · 이력 추적 및 재현 | State store, Observability | Trace 계층, Checkpoint 이력, 버전 고정, Replay/fork |
| 8 | 민감정보 · 프롬프트 공격 대응 | Guardrail, Gateway | 마스킹, 데이터 등급 라우팅, 간접 인젝션 격리 |
| 9 | 운영 중 변경이 진행 건에 영향 없음 | IR store, Run manager | 불변 버전, 버전 고정, graph_hash, 안전 지점 마이그레이션 |
| 10 | 대량 동시 실행 | Scheduler, Queue, Worker, LLM GW | Stateless Worker, 큐 분리, run_id 파티션, 호출 한도, 역압 |

번외: 비용 통제(Budget guard), LLM 교체 · 종속 탈피(논리 모델), 품질 평가 · 테스트(골든셋 회귀)

## 해석 포인트
- 2번: 인프라 장애 vs 논리적 Agent 실패
- 4번: 실행 중 통제(루프·예산) vs 권한 통제(최소 권한·승인)
- 7번: 기록 재생(같은 결과 보기) vs fork 재실행(다시 돌려보기)

## 공유 메커니즘 (실제로 깊게 익힐 것은 4개)

| 뿌리 메커니즘 | 해당 요구사항 |
|---|---|
| 중단 · 재개 (Checkpoint) | 1, 2, 7 |
| Agent 통제 (실행기 검사 지점) | 4, 8 |
| DSL → IR | 3, 9 |
| 공유 State · 이벤트 기반 스케줄링 | 5, 10 |
