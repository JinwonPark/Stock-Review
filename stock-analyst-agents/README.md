# Stock Analyst Agents - Orchestration Package

증권사 리서치센터 워크플로우를 모사한 Claude Code 서브에이전트 패키지.
오케스트레이터 1개 + 분석 전문 서브에이전트 5개 + 종합 자문 1개 = 총 7개 에이전트.

## 구성

```
stock-analyst-agents/
├── stock-analyst-orchestrator.md   # 메인 오케스트레이터 (purple)
├── company-overview-analyst.md     # 기업 개요 분석 (blue)
├── financial-analyst.md            # 재무 분석 + 밸류에이션·DCF (green)
├── industry-analyst.md             # 산업 분석 (orange)
├── momentum-analyst.md             # 모멘텀 분석 (yellow)
├── risk-analyst.md                 # 리스크 분석 (red)
└── investment-advisor.md           # 종합 자문 + 스코어카드 + 트레이딩 전략 (cyan)
```

## 오케스트레이션 구조

```
┌────────────────────────────────────────────────────────────────────────┐
│            stock-analyst-orchestrator (메인 - 조율 전담)                │
│                                                                          │
│  Phase 1: 분석 위임 (5개 병렬)                                          │
│  Phase 2: 정합성 검증                                                    │
│  Phase 3: investment-advisor 호출 (5개 결과 전달)                       │
│  Phase 4: 최종 리포트 포맷팅                                             │
└──────────────┬───────────────────────────────────┬─────────────────────┘
               │                                   │
       [Phase 1: 병렬 위임]              [Phase 3: 종합 자문]
               │                                   │
   ┌───────────┼───────────┬───────────┐           ▼
   ▼           ▼           ▼           ▼           ┌──────────────────┐
┌──────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐    │ investment-      │
│ 기업  │ │  재무    │ │  산업    │ │  모멘텀  │    │ advisor          │
│ 개요  │ │  분석    │ │  분석    │ │  분석    │    │                  │
└──────┘ └──────────┘ └──────────┘ └──────────┘    │ - 5점 스코어카드 │
   ┌──────────┐                                    │ - 시나리오 목표가│
   │  리스크  │                                    │ - 투자의견       │
   │  분석    │                                    │ - 트레이딩 전략  │
   └──────────┘                                    └──────────────────┘
```

## 설치

### 옵션 1: 프로젝트 단위 (해당 프로젝트에서만 사용)

```bash
mkdir -p .claude/agents
cp stock-analyst-agents/*.md .claude/agents/
```

### 옵션 2: 유저 단위 (모든 프로젝트에서 사용)

```bash
mkdir -p ~/.claude/agents
cp stock-analyst-agents/*.md ~/.claude/agents/
```

설치 후 Claude Code를 재시작해야 에이전트가 로드됨.

## 사용법

### 방법 1: 자동 위임 (PROACTIVELY)

오케스트레이터의 `description`에 `PROACTIVELY use` 키워드가 포함되어 있어,
종목 분석 요청 시 Claude가 자동으로 위임함.

```
삼성전자 종목 분석해줘.
NVIDIA 추천픽으로 적합한지 평가해줘.
SK하이닉스랑 마이크론 둘 다 분석해서 비교해줘.
```

### 방법 2: 명시적 호출 (@-mention)

```
@stock-analyst-orchestrator 삼성전자 분석해줘
```

### 방법 3: 메인 세션 자체로 실행

전체 세션을 오케스트레이터로 운영하고 싶을 때:

```bash
claude --agent stock-analyst-orchestrator
```

### 단일 측면만 분석하고 싶을 때

특정 서브에이전트만 직접 호출 가능:

```
@financial-analyst 카카오 재무 상태만 빠르게 봐줘
@risk-analyst 테슬라의 핵심 리스크만 정리해줘
@investment-advisor [5개 분석 결과 전달] 이거 기반으로 최종 의견 줘
```

## 분석 프레임워크 (오케스트레이터가 자동 수행)

### Phase 1: 병렬 분석 (5개 서브에이전트 동시 실행)
1. **기업 개요** — 사업 모델, 매출 구성, 경쟁 우위 (Moat), 지배구조
2. **재무 분석** — 손익·재무상태·현금흐름, 수익성·효율성, **DCF 모델·SOTP·시나리오별 목표가**
3. **산업 분석** — 시장 규모·성장률, 사이클 위치, 5-Forces, 규제
4. **모멘텀 분석** — 실적 모멘텀, 가격 모멘텀, 수급 모멘텀
5. **리스크 요인** — 기업·산업·거시·규제·ESG·이벤트 리스크

### Phase 2: 정합성 검증
- 누락된 핵심 데이터 보강
- 데이터 시점 일관성 확인

### Phase 3: 종합 자문 (investment-advisor)
- **5-Pillar 스코어카드** (각 5점 × 5 = 25점 만점)
- **시나리오 분석** + 확률 가중 기대수익률
- **최종 투자의견** (Strong BUY / BUY / HOLD / Reduce / SELL)
- **트레이딩 전략** (분할 매수·익절·손절·트레일링 스탑·포지션 사이징)
- **추천 무효화 트리거** + 핵심 모니터링 지표

### Phase 4: 최종 리포트 통합

## 도구 권한 요약

| 에이전트 | 도구 | Write |
|---|---|---|
| stock-analyst-orchestrator | `Agent(6개)`, Read, WebSearch, WebFetch, Glob | ✓ |
| company-overview-analyst | WebSearch, WebFetch, Read, Grep, Glob | ✗ |
| financial-analyst | WebSearch, WebFetch, Read, Grep, Glob | ✗ |
| industry-analyst | WebSearch, WebFetch, Read, Grep, Glob | ✗ |
| momentum-analyst | WebSearch, WebFetch, Read, Grep, Glob | ✗ |
| risk-analyst | WebSearch, WebFetch, Read, Grep, Glob | ✗ |
| investment-advisor | WebSearch, WebFetch, Read, Grep, Glob | ✗ |

분석 서브에이전트와 advisor 모두 읽기·웹 조회 전용. 파일 수정 권한 없음.
오케스트레이터만 `Write` 권한 보유 (최종 리포트 저장용).

## 모델 설정

전체 에이전트가 **Claude Opus 4.7** (`claude-opus-4-7`)로 통일 설정됨.
- 오케스트레이터: 워크플로우 조율 + 최종 포맷팅
- 분석 서브에이전트: 깊이 있는 정성·정량 분석
- investment-advisor: 5개 분석 결과 종합 판단 (복잡한 다중 추론)

**비용 최적화가 필요한 경우** 각 .md 파일의 `model:` 필드를 다음 중 하나로 변경:
- `sonnet` — 균형 잡힌 성능/비용 (단순 데이터 조회 용도)
- `haiku` — 빠르고 저렴
- `inherit` — 메인 세션 모델 따름

**권장:** investment-advisor와 financial-analyst는 절대 다운그레이드하지 말 것.
종합 판단과 밸류에이션 모델링은 모델 격차가 가장 크게 드러나는 영역.

## 역할 분리 설계 (왜 advisor를 별도로 두는가?)

오케스트레이터에 종합 판단까지 모두 맡기는 단순 구조 대비 장점:

1. **단일 책임 원칙** — 오케스트레이터는 조율만, advisor는 판단만
2. **컨텍스트 깊이** — advisor가 독립 컨텍스트에서 5개 분석 결과에 풀 컨텍스트 집중
3. **정량 점수화** — 5-Pillar 스코어카드를 한 에이전트가 책임지면 종목 간 비교 가능
4. **튜닝 용이성** — 추천 로직(보수/공격, 절대수익/상대수익)을 advisor만 수정하면 됨
5. **재사용성** — 외부에서 작성된 5개 분석 결과를 받아 advisor만 단독 호출 가능

## 커스터마이징

각 .md 파일의 YAML frontmatter와 본문을 자유롭게 수정하여 사용 환경에 맞출 수 있음.

- **미국 종목 중심**이면 한국거래소 관련 부분 → SEC EDGAR / 13F 등으로 교체
- **특정 산업** (바이오, 반도체 등) 전문화 시 산업 특화 KPI 추가
- **영문 리포트** 필요하면 시스템 프롬프트 언어 변경
- **퀀트 전략** 추가 시 advisor에 팩터 점수(밸류·퀄리티·모멘텀) 항목 추가

## Disclaimer

본 에이전트가 생성하는 분석은 정보 제공 목적이며, 실제 투자 결정은 투자자 본인의 책임이다.
