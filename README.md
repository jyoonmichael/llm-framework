# AI Agent Workflow 프레임워크 설계

> 주제: AI Agent의 동작을 Workflow로 정의하고 실행하기 위한 프레임워크 설계

## 핵심 논지

**흐름의 뼈대는 결정론으로, 판단이 필요한 구간만 Agent로, 그 Agent는 엔진이 통제한다.**

- 포지셔닝: 검증된 실행 모델(LangGraph 방식: State · Node · Edge · Checkpoint)을 코어로, 선언적 정의 · 거버넌스 · 통제 · 운영 계층을 더한 엔터프라이즈 프레임워크
- 차별점: 업무마다 코드를 짜는 방식 → 정의(DSL)와 실행(엔진)을 분리해 설정만으로 새 워크플로 생성

## 설계 불변식 (모든 문서가 지키는 규칙)

| # | 불변식 | 지켜지는 곳 |
|---|---|---|
| I1 | 실행 중인 Run의 그래프는 바뀌지 않는다 (동적 계획은 자식 Run으로) | Run manager, IR store |
| I2 | 모든 상태는 State store에만 있다 (Worker · Scheduler · SSE gateway는 stateless) | 실행 레이어 전체 |
| I3 | 상태 변경과 이벤트 발행은 한 트랜잭션이다 (Transactional Outbox) | State store |
| I4 | 내부 쓰기는 펜싱 토큰, 외부 쓰기는 멱등키로 보호한다 | Worker, Task executor |
| I5 | 모든 모델 · Tool 호출은 Gateway를 거친다 | LLM GW, MCP GW |
| I6 | 비결정 결과(LLM 응답, Tool 결과, 기억 조회, 사람 결정)는 기록하고 재생 시 재사용한다 | 반복 기록, Checkpoint |
| I7 | Compiler 검증을 통과한 정의만 실행된다 (사람이 만든 것이든 Agent가 만든 것이든) | Compiler |
| I8 | Agent의 권한 · 예산은 IR이 정한 범위를 넘을 수 없다 (자식 Run 포함) | Agent executor, Policy |
| I9 | Policy의 강제 규칙은 워크플로가 덮어쓸 수 없다 | Compiler |
| I10 | 병렬은 정의(무엇을 병렬로)와 용량(얼마나 동시에)을 분리한다 | DSL, Scheduler, 운영 |

## 요구사항 요약

**필수 (12)**: 혼합 실행 · 제어 흐름 · 외부 연계 · 버전 무영향 · 노코드 정의 · HITL · Agent 통제(토큰 포함) · 멀티 Agent · 추적 · 재현 · 보안 · 장애 시 무손실 재개 · 가용성 · 확장성
**옵션 (5)**: 실시간 진행 확인 · 모델 교체 대응 · 품질 사전 검증 · 동적 계획 · 기억 · 지식

→ 상세: [00 요구사항 · 정합성 검증](docs/00-requirements.md)

## 목차

### 기본 장표 (미리 손에 익힐 것)
- [01 전체 구성도](docs/01-layered-architecture.md)
- [02 컴포넌트 상호작용도](docs/02-component-interaction.md)
- [03 주요 처리흐름](docs/03-main-process-flow.md)

### 필수 요구사항 설계
- [04 정의: DSL → IR · Policy](docs/04-definition-dsl-ir.md)
- [05 제어 흐름 · 실행 루프 · State](docs/05-control-flow-state.md)
- [06 Run manager](docs/06-run-manager.md)
- [07 Scheduler · Queue](docs/07-scheduler-queue.md)
- [08 실행기 3종](docs/08-executors.md)
- [09 HITL](docs/09-hitl.md)
- [10 장애 시 무손실 재개 (펜싱 · 멱등)](docs/10-durability-failover.md)
- [11 가용성 · 확장성 · 병렬 처리](docs/11-availability-scalability.md)
- [12 Agent 통제 · 토큰 예산](docs/12-agent-control-budget.md)
- [13 추적 · 재현 · 보안](docs/13-observability-security.md)
- [14 LLM gateway](docs/14-llm-gateway.md)

### 옵션 요구사항 (필수 설계에 터치)
- [15 실시간 진행 확인 (SSE)](docs/15-option-realtime.md)
- [16 동적 계획](docs/16-option-dynamic-planning.md)
- [17 기억 · 지식](docs/17-option-memory-knowledge.md)
- [18 확장 지점 · 품질 검증](docs/18-extension-points.md)

### 구현 · 참고
- [19 Spring Boot 패키지 · 클래스](docs/19-spring-boot-mapping.md)
- [20 참고 오픈소스 · 링크](docs/20-references.md)

## 시험 운영

### 시간 배분 (3.5시간)

| 단계 | 시간 | 산출물 |
|---|---|---|
| 요구사항 분석 | 20분 | 요구사항 → 필수/옵션 분류 → 대응 문서 매핑 |
| 설계 원칙 + 구성도 | 40분 | 원칙(불변식 중 4~5개), 01 · 02 |
| 상세기능 | 35분 | 컴포넌트 역할, 실행기 비교표 |
| 처리흐름 | 40분 | 03 정상 흐름 + 예외 흐름 |
| 주요 메커니즘 | 55분 | 요구사항 연관 순 4~5장 (시간 되면 멈춤) |
| 검토 · 확장 방향 | 20분 | 요구사항 추적표, 18 확장 지점 |

### 장표 목차 템플릿
1. 요구사항 분석 (필수/옵션, 해석)
2. 설계 원칙 (불변식)
3. 전체 구성도 → 4. 상호작용도 → 5. 정의 방식 → 6. 상세기능 → 7. 처리흐름
8~12. 주요 메커니즘
13. 비기능 대응 (가용성 · 확장성 · 보안)
14. 확장 방향 (옵션 확장 지점)

### 메커니즘 장표 4단계 공식
1. **해석**: 모호하면 유형을 나눠 명시
2. **위치**: 기본 구성도의 어느 박스가 담당하나
3. **시나리오**: 정상 → 장애/예외 → 복구를 시퀀스로
4. **장치 정리**: 핵심 장치 + 한 줄 설명, 재사용한 블록 명시

### 색 규칙
보라 = 판단 · 실행 주체 / 청록 = 외부 연동 관문 / 회색 = 저장소 · 인프라
