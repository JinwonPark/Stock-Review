# Stock Analyst Multi-Agent System (v6)

Claude Code 기반 다중 에이전트 주식 분석·매매 의사결정·시스템 자가 평가·실시간 알림 시스템.
**증권사 리서치팀 + 포트폴리오 매니저 + 트레이딩 데스크 + 리스크 관리 + 세무 회계 + ESG 평가 + 시스템 감사 + 알림 데스크**를 1인 투자자에게 제공.

---

## 시스템 구조 (26 Agents)

```
                    ┌──────────────────────────────┐
                    │  stock-analyst-orchestrator  │  ← 메인 (purple)
                    └──────────────┬───────────────┘
                                   │
        ┌──────────────────────────┼──────────────────────────┐
        │                          │                          │
   [Phase 1: 컨텍스트]      [Phase 2: 분석 6개]      [Phase 6~10: 매매 지원]
        │                          │                          │
   ┌────┴─────┐    ┌─────────────┴──────────────┐    ┌──────┴──────┐
   │          │    │                            │    │             │
   market-regime  company-overview              ...  position-sizer
   portfolio-analyst financial-analyst          ...  tax-cost-calculator
                    industry-analyst                  hedge-advisor
                    momentum-analyst                  fx-hedge-analyst
                    risk-analyst                      entry-timing-checker
                    esg-analyst (v6 신규)             order-book-analyst
                          │                          
                          ▼                          
                  [Phase 4~5: 종합]
                  investment-advisor (cyan)
                  devils-advocate (pink)
                          │
                          ▼
                  [Phase 8: 기록]
                  reports/[티커]_recommendations.jsonl

  + 별도 워크플로우:
    - position-monitor       (보유 종목 점검)
    - reanalysis-trigger     (재분석 트리거 식별)
    - portfolio-rebalancer   (정기 리밸런싱)
    - daily-briefing         (일일 모닝 브리핑)

  + Mode F: 백테스팅 (v5)
    backtest-collector → backtest-analyzer → backtest-optimizer
    (시스템 자가 평가 + 개선 권고)

  + Mode G: 카톡 알림 (v6 신규)
    kakao-notifier → PlayMCP:KakaotalkChat-MemoChat
    (손절선·목표가·트리거 발동 시 자동 알림)

  + 시스템 인프라:
    - realtime-data-fetcher  (실시간 시세·지표 통합 게이트웨이, v6 신규)
```

---

## 에이전트 목록

### 메인 (1)
| 에이전트 | 색상 | 역할 |
|---|---|---|
| stock-analyst-orchestrator | purple | 19개 서브에이전트를 지휘하는 메인 리서치 디렉터 |

### 분석 (6) — 핵심 종목 분석 (v6에서 esg 추가)
| 에이전트 | 색상 | 역할 |
|---|---|---|
| company-overview-analyst | blue | 기업 개요·사업 모델·경쟁 우위 |
| financial-analyst | green | 재무 분석·밸류에이션·시나리오 목표가 |
| industry-analyst | orange | 산업 분석·경쟁 환경·규제 |
| momentum-analyst | yellow | 컨센서스·기술적 흐름·수급 |
| risk-analyst | red | 회사·산업·매크로·ESG 리스크 |
| **esg-analyst** (v6) | **red** | **환경(10) + 사회(10) + 거버넌스(5) = 25점 ESG 평가, risk-analyst와 통합** |

### 종합 (2) — 추천 결정
| 에이전트 | 색상 | 역할 |
|---|---|---|
| investment-advisor | cyan | 5-Pillar 스코어카드 + 시나리오 + 최종 의견 + 트레이딩 전략 |
| devils-advocate | pink | 추천 적대적 검증 + 비판 점수 (0~35) |

### 시장·포트폴리오 컨텍스트 (3) — v4 신규
| 에이전트 | 색상 | 역할 |
|---|---|---|
| market-regime-analyst | blue | 시장 국면 판단 (Bull/Bear/Sideways/HighVol) + 매매 권고 조정 |
| portfolio-analyst | blue | 보유 종목 vs 신규 매수 적합도 (집중도·상관관계·분산) |
| daily-briefing | blue | 일일 모닝 브리핑 (시장+보유종목+이벤트) |

### 정량 매매 계산 (4) — v4 신규
| 에이전트 | 색상 | 역할 |
|---|---|---|
| position-sizer | green | Kelly Criterion + ATR + 변동성 타게팅 통합 사이징 |
| tax-cost-calculator | green | 거래 비용·세금·세후 수익률 (한국/미국 주식) |
| hedge-advisor | green | 옵션·인버스 ETF 헷지 검토 (큰 포지션만) |
| fx-hedge-analyst | green | 환헷지 분석 (미국 주식 전용) |

### 추적·재평가 (3) — v4 신규
| 에이전트 | 색상 | 역할 |
|---|---|---|
| position-monitor | orange | 보유 종목 손절선·트리거·목표가 도달 모니터링 |
| reanalysis-trigger | orange | 시간·가격·이벤트 기반 재분석 필요 종목 식별 |
| portfolio-rebalancer | orange | 정기 리밸런싱 + 세제 효율적 거래 리스트 |

### 타이밍·실행 (2) — v4 신규
| 에이전트 | 색상 | 역할 |
|---|---|---|
| entry-timing-checker | red | 단기 과열 점검 + 이벤트 캘린더 + Go/Wait 판정 |
| order-book-analyst | red | 호가창 분석 + 주문 유형 권고 (실시간 데이터 가용 시) |

### 백테스팅·시스템 자가 평가 (3) — v5 신규
| 에이전트 | 색상 | 역할 |
|---|---|---|
| backtest-collector | yellow | reports/*.jsonl + 실제 가격 매칭 → backtest_results.jsonl 생성 |
| backtest-analyzer | yellow | 6개 핵심 지표 (Win Rate, Sharpe, Profit Factor 등) + 세그먼트 분석 + 통계적 함정 점검 |
| backtest-optimizer | yellow | 발견된 약점 기반 시스템 가중치·임계값 조정 권고 (과적합 방지 검증 포함) |

### 시스템 인프라 (1) — v6 신규
| 에이전트 | 색상 | 역할 |
|---|---|---|
| realtime-data-fetcher | blue | 실시간 시세·기술 지표·호가·FX 통합 데이터 게이트웨이 (MCP 자동 라우팅 + 표준 스키마) |

### 알림 (1) — v6 신규
| 에이전트 | 색상 | 역할 |
|---|---|---|
| kakao-notifier | orange | PlayMCP `KakaotalkChat-MemoChat`을 통한 카톡 알림 (200자 자동 분할 + 우선순위 + 중복 방지) |

---

## v2 대비 v6 핵심 변화

| 영역 | v2 | v4 | v5 | v6 |
|---|---|---|---|---|
| 에이전트 수 | 8 | 20 | 23 | **26** |
| 워크플로우 모드 | 1 | 5 (A~E) | 6 (A~F) | **7 (A~G)** |
| 분석 영역 | 종목+추천 | + 시장/포트폴리오/매매/추적 | + 시스템 자가평가 | + **ESG/실시간데이터/카톡알림** |
| 분석가 수 (Phase 2) | 5 | 5 | 5 | **6 (ESG 추가)** |
| 시스템 학습 | 없음 | JSONL 기록만 | 백테스팅 | 동일 |
| 실시간 데이터 | WebSearch만 | 동일 | 동일 | **MCP 통합 게이트웨이** |
| 외부 알림 | 없음 | 없음 | 없음 | **카톡 (PlayMCP)** |

---

## 사용법

### ⚠️ 중요: 오케스트레이터 실행 방식

**Claude Code의 서브에이전트는 다른 서브에이전트를 spawn할 수 없습니다.**
([공식 문서](https://code.claude.com/docs/en/sub-agents) 명시)

오케스트레이터는 **반드시 "메인 세션 에이전트"**로 실행해야 합니다.

```bash
# 권장 실행 방법
claude --agent stock-analyst-orchestrator

# 또는 프로젝트 기본값
# .claude/settings.json:
{ "agent": "stock-analyst-orchestrator" }
```

### Mode A: 신규 종목 풀 분석 (10 Phases)

```bash
claude --agent stock-analyst-orchestrator

> 삼성전자 분석해줘
> NVDA 추천픽으로 적합한지 평가해줘
```

Phase 0~9 자동 실행 → 20개 에이전트 협업 → 통합 리포트 + JSONL 기록.

### Mode B: 보유 종목 점검

```bash
> 내 종목 점검해줘
> 보유 종목 리뷰
```

position-monitor + reanalysis-trigger 호출 → 액션 우선순위 리스트.

### Mode C: 일일 브리핑

```bash
> 오늘 아침 브리핑
> 모닝 점검
```

daily-briefing 호출 → 5분 안에 읽는 모닝 리포트.

### Mode D: 리밸런싱

```bash
> 포트폴리오 리밸런싱
> 비중 조정
```

portfolio-rebalancer 호출 → 세제 효율적 거래 리스트.

### Mode E: 단일 에이전트 직접 호출

```bash
@financial-analyst 카카오 재무만 분석해줘
@risk-analyst 테슬라의 핵심 리스크만 정리해줘
@position-sizer [advisor 결과] 비중 계산해줘
@tax-cost-calculator AAPL 100주 매도 시 세후 수익 계산
```

### Mode F: 백테스팅 (v5 신규)

```bash
> 시스템 성과 평가해줘
> 백테스팅 돌려줘
> 추천 정확도 확인
> 분기 시스템 리뷰
```

3단계 자동 실행:
1. **backtest-collector** — reports/*.jsonl + 실제 가격 매칭
2. **backtest-analyzer** — 6개 지표 + 세그먼트 분석
3. **backtest-optimizer** — 개선 권고 (선택)

⚠️ **첫 의미 있는 결과는 추천 누적 12개월 + 30+ 추천 후** 가능.

### Mode G: 카톡 알림 (v6 신규)

**자동 트리거:** 다른 Mode 실행 중 손절선·트리거·목표가 발동 시 자동 호출.

**수동 요청:**
```bash
> 카톡으로 보내줘
> 손절선 도달하면 카톡 알림 켜줘
> 오늘 브리핑 카톡으로 보내줘
> 알림 설정 확인
```

**필수 조건:**
- PlayMCP 연결 (claude.ai 설정에서 카카오톡 MCP 활성화)
- `notifications/preferences.json` 설정 (`templates/notification-preferences.json.example` 복사 후 편집)

**제약:**
- PlayMCP `KakaotalkChat-MemoChat` 도구는 **메시지당 최대 200자**
- kakao-notifier가 자동 분할 처리 (예: "(1/3)", "(2/3)")
- 동일 종목·트리거 24시간 내 중복 발송 방지

---

## 입력 파일

### portfolio.json (필수 — Mode A에서 portfolio-analyst 사용 시)

```bash
cp templates/portfolio.json.example portfolio.json
# 본인 정보로 수정
```

내용: 보유 종목·평균 매수가·현재 비중·현금 잔고·목표 비중·리스크 설정.

### tax-settings.json (권장)

```bash
cp templates/tax-settings.json.example tax-settings.json
```

내용: 거주국·증권사 수수료·세율 (한국 기본값 내장).

### market-watchlist.json (선택)

```bash
cp templates/market-watchlist.json.example market-watchlist.json
```

내용: 보유는 안 하지만 모니터링 중인 종목 + 거시 알림.

### 파일 없을 때 동작
- portfolio.json 없음 → portfolio-analyst, position-sizer 일부, rebalancer 스킵
- tax-settings.json 없음 → 한국 거주자 표준값 사용
- market-watchlist.json 없음 → 영향 없음

---

## 도구 권한

| 에이전트 그룹 | 도구 | Write |
|---|---|---|
| stock-analyst-orchestrator | `Agent(25개)`, Read, Write, Edit, Bash, WebSearch, WebFetch, Glob | ✓ |
| 분석 6개 + advisor + devils-advocate | WebSearch, WebFetch, Read, Grep, Glob | ✗ |
| portfolio-analyst, position-monitor 등 (파일 읽기 필요) | Read, Bash, WebSearch, WebFetch, Grep, Glob | ✗ |
| market-regime, entry-timing, order-book, hedge, fx-hedge, **esg** (v6) | WebSearch, WebFetch, Read, Grep, Glob | ✗ |
| position-sizer, tax-cost-calculator | Read, Bash, WebSearch, Grep, Glob | ✗ |
| backtest-collector (v5) | WebSearch, WebFetch, Read, Bash, Grep, Glob | ✗ |
| backtest-analyzer (v5) | Read, Bash, WebSearch, Grep, Glob | ✗ |
| backtest-optimizer (v5) | Read, Bash, Grep, Glob | ✗ |
| **realtime-data-fetcher** (v6) | WebSearch, WebFetch, Read, Bash, Grep, Glob | ✗ |
| **kakao-notifier** (v6) | WebSearch, Read, Bash, Grep, Glob (+ PlayMCP MCP 도구) | ✗ |

오케스트레이터만 `Write/Edit/Bash` 권한 보유 (JSONL append + 최종 리포트 저장).
백테스팅·인프라 에이전트들은 `Bash` 권한이 있지만 `Write/Edit` 없음.

### 버전 호환성

- **Claude Code 2.1.63 이상 권장.** 이 버전에서 `Task` 도구가 `Agent`로 이름 변경.
- 구버전에서는 `Agent(...)`를 `Task(...)`로 자동 alias 처리.
- **fork mode (선택):** `CLAUDE_CODE_FORK_SUBAGENT=1` 환경변수 (v2.1.117+).

---

## 핵심 설계 원칙

1. **다중 출처 교차 검증** — 모든 분석가는 2개 이상 출처 인용. 단일 출처 데이터는 명시.
2. **시간 일관성** — 모든 분석은 동일 시점 데이터 기준.
3. **시나리오 기반** — Bull / Base / Bear 확률 가중 목표가.
4. **devils-advocate 통과 의무** — advisor 추천은 반드시 비판 검증 후 확정.
5. **정량 우선** — "느낌"이 아닌 수학적 근거 (Kelly, ATR, HHI).
6. **포트폴리오 컨텍스트** — 종목 자체보다 포트폴리오 적합성 중시.
7. **세후 실수익** — 명목이 아닌 세후 기준.
8. **추적·재평가** — 매수 후 방치 방지.

---

## 절대 경계 (시스템이 하지 않는 것)

- ❌ **자동 매매 실행** (한국 자본시장법 + 책임 소재 문제)
- ❌ **실시간 시세 자동 수집** (실시간 MCP 통합 시 가능)
- ❌ **세무 자문** (정확한 세무는 세무사 영역)
- ❌ **투자 결과 보증** (모든 결정과 책임은 사용자)

---

## 디렉토리 구조

```
stock-analyst-agents/
├── README.md
├── SETUP.md
├── ROADMAP.md
├── .gitignore
│
├── stock-analyst-orchestrator.md              ← 메인 (Mode A~G)
│
├── (분석 6개 — v6에서 ESG 추가)
├── company-overview-analyst.md
├── financial-analyst.md
├── industry-analyst.md
├── momentum-analyst.md
├── risk-analyst.md
├── esg-analyst.md                             ← v6 신규
│
├── (종합 2개)
├── investment-advisor.md
├── devils-advocate.md
│
├── (시장·포트폴리오 컨텍스트 3개 — v4)
├── market-regime-analyst.md
├── portfolio-analyst.md
├── daily-briefing.md
│
├── (정량 매매 계산 4개 — v4)
├── position-sizer.md
├── tax-cost-calculator.md
├── hedge-advisor.md
├── fx-hedge-analyst.md
│
├── (추적·재평가 3개 — v4)
├── position-monitor.md
├── reanalysis-trigger.md
├── portfolio-rebalancer.md
│
├── (타이밍·실행 2개 — v4)
├── entry-timing-checker.md
├── order-book-analyst.md
│
├── (백테스팅 3개 — v5)
├── backtest-collector.md
├── backtest-analyzer.md
├── backtest-optimizer.md
│
├── (인프라 1개 — v6 신규)
├── realtime-data-fetcher.md
│
├── (알림 1개 — v6 신규)
├── kakao-notifier.md
│
├── templates/
│   ├── portfolio.json.example
│   ├── tax-settings.json.example
│   ├── market-watchlist.json.example
│   └── notification-preferences.json.example  ← v6 신규
│
├── reports/                                   ← 자동 생성 (추천 기록)
│   ├── 005930_recommendations.jsonl
│   └── ...
│
├── backtests/                                 ← 자동 생성 (v5)
│   └── backtest_results.jsonl
│
└── notifications/                             ← 자동 생성 (v6)
    ├── preferences.json
    └── sent.jsonl
```

---

## 향후 확장 (Roadmap)

상세 내용은 [ROADMAP.md](./ROADMAP.md) 참고.

**완료 (v4):** 매매 의사결정 보조 12개 에이전트 통합.

**미래 검토:**
- 실시간 시세 MCP 통합 (Alpha Vantage, Polygon.io, KIS 등)
- 백테스팅 자동화 (reports/*.jsonl 활용)
- 옵션 가격 데이터 통합 (정밀 헷지 분석)

---

## License

본 시스템은 정보 제공 목적으로 제공된다. 본 시스템의 출력은 **투자 자문이 아니며**, 모든 매매 결정과 결과는 사용자 본인 책임이다.
