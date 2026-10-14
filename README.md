# AI Agent Workflow 프레임워크 설계 노트

> 주제: AI Agent의 동작을 Workflow로 정의하고 실행하기 위한 프레임워크 설계

## 핵심 논지 (한 문장)

**흐름의 뼈대는 결정론으로, 판단이 필요한 구간만 Agent로, 그 Agent는 엔진이 통제한다.**

- 포지셔닝: 검증된 실행 모델(LangGraph 방식)을 코어로 채택하고, 그 위에 선언적 정의 · 거버넌스 · 통제 · 운영 계층을 얹은 엔터프라이즈 프레임워크
- 차별점: 개별 업무마다 코드를 짜는 방식 → 정의(DSL)와 실행(엔진)을 분리해 설정만으로 새 워크플로 생성

## 목차

### 기본 장표 (미리 손에 익힐 것)
1. [전체 구성도 (레이어)](slides/01-layered-architecture.md)
2. [컴포넌트 상호작용도](slides/02-component-interaction.md)
3. [주요 처리흐름](slides/03-main-process-flow.md)

### 정의 · 실행 핵심
4. [Workflow 정의: DSL → IR](slides/04-definition-dsl-ir.md)
5. [실행 루프와 State](slides/05-execution-loop-state.md)
6. [Run manager](slides/06-run-manager.md)
7. [Scheduler와 Queue](slides/07-scheduler-queue.md)
8. [실행기 3종 (Task · Agent · Human)](slides/08-executors.md)

### 세부 메커니즘 (요구사항에 따라 꺼내 쓸 것)
9. [HITL](slides/09-hitl.md)
10. [LLM gateway 라우팅 · 폴백](slides/10-llm-gateway.md)
11. [Observability · Budget guard](slides/11-observability.md)
12. [실행 장애 시 작업 인계](slides/12-failover.md)

### 시험 운영
13. [예상 요구사항 Top 10](slides/13-requirements-top10.md)
14. [참고 오픈소스 · 링크](slides/14-references.md)

## 시간 배분 (3.5시간)

| 단계 | 시간 | 산출물 |
|---|---|---|
| 요구사항 분석 | 20분 | 요구사항 → 설계 이슈 → 대응 블록 매핑표 |
| 설계 원칙 + 구성도 | 40분 | 원칙 3~4개, 구성도 (01, 02) |
| 상세기능 | 35분 | 컴포넌트 역할 · 실행기 비교표 |
| 처리흐름 | 40분 | 정상 흐름 (03) + 예외 흐름 |
| 주요 메커니즘 | 55분 | 요구사항 연관도 순 4~5개 (중단 규칙: 시간 되면 멈춤) |
| 검토 · 기대효과 | 20분 | 요구사항 추적표 |

## 장표 목차 템플릿

1. 요구사항 분석 및 설계 이슈
2. 설계 원칙 (통제 vs 자율성 균형)
3. 전체 구성도
4. 컴포넌트 상호작용도
5. Workflow 정의 방식 (DSL → IR)
6. 상세기능 (컴포넌트 역할 · 실행기 비교)
7. 주요 처리흐름
8~12. 주요 메커니즘 (요구사항별)
13. 비기능 요건 대응
14. 기대효과 및 확장 방향

## 메커니즘 장표 4단계 공식

1. **해석**: 요구사항이 모호하면 유형을 나눠 명시
2. **위치**: 기본 구성도의 어느 박스가 담당하는지
3. **시나리오**: 정상 → 장애/예외 → 복구 순서를 시퀀스로
4. **장치 정리**: 핵심 장치 이름 + 한 줄 설명, 재사용한 블록 명시

## 색 규칙

- 보라: 판단·실행 주체 (Compiler, Run manager, Scheduler, Worker)
- 청록: 외부 연동 관문 (LLM GW, MCP GW, HITL)
- 회색: 저장소·인프라 (IR store, State store, Queue)
