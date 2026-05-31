# SETUP 가이드 (v4)

## 1. Claude Code에 에이전트 설치

### 프로젝트 단위 설치 (해당 프로젝트에서만 사용)

```bash
mkdir -p .claude/agents
cp stock-analyst-agents/*.md .claude/agents/
```

### 유저 단위 설치 (모든 프로젝트에서 사용)

```bash
mkdir -p ~/.claude/agents
cp stock-analyst-agents/*.md ~/.claude/agents/
```

설치 후 Claude Code 재시작 필요.

### 실행 (⚠️ 중요)

오케스트레이터는 **반드시 메인 세션 에이전트로 실행**해야 합니다.
([공식 문서](https://code.claude.com/docs/en/sub-agents): 서브에이전트는 다른 서브에이전트를 spawn할 수 없음)

```bash
# 권장
claude --agent stock-analyst-orchestrator
```

또는 프로젝트 기본값:

```bash
# .claude/settings.json
{ "agent": "stock-analyst-orchestrator" }
```

위 방법에서만 `Agent(...)` tool이 활성화되어 19개 서브에이전트 호출 가능.

---

## 2. 입력 파일 준비

v4는 사용자 입력 파일을 활용합니다. 작업 디렉토리(Claude Code 실행 위치)에 다음 파일들을 준비:

### portfolio.json (필수 — Mode A에서 portfolio-analyst 사용 시)

```bash
# 템플릿 복사
cp stock-analyst-agents/templates/portfolio.json.example portfolio.json

# 편집 (본인 보유 종목·평균 매수가·현금 등)
nano portfolio.json  # 또는 본인이 선호하는 에디터
```

**핵심 필드:**
- `holdings`: 보유 종목 리스트 (티커, 평균매수가, 비중)
- `cash.krw`, `cash.usd`: 현금 잔고
- `total_value_krw`: 총 자산 (원화 환산)
- `target_weights`: 목표 비중 (리밸런싱 시 사용)
- `risk_settings.risk_tolerance_per_trade_pct`: 거래당 리스크 허용 (기본 1%)
- `user_profile.risk_appetite`: conservative / balanced / aggressive

### tax-settings.json (권장)

```bash
cp stock-analyst-agents/templates/tax-settings.json.example tax-settings.json
```

**핵심 필드:**
- `trading_brokerage.KR_*` / `US_*`: 본인 증권사 수수료
- `fx_spread.USD_KRW_*`: 환전 수수료 (증권사별 0.3~0.7%)
- `major_shareholder`: 대주주 여부 (기본 false)
- `tax_year`: 적용 연도 (세율은 매년 변경 가능)

### market-watchlist.json (선택)

```bash
cp stock-analyst-agents/templates/market-watchlist.json.example market-watchlist.json
```

보유는 안 하지만 매수 검토 중인 종목 + 거시 알림.

### notifications/preferences.json (선택, v6 신규)

카톡 알림 (Mode G) 사용 시:

```bash
mkdir -p notifications
cp stock-analyst-agents/templates/notification-preferences.json.example \
   notifications/preferences.json
```

**핵심 필드:**
- `channel.kakaotalk.enabled`: 카톡 알림 켜기/끄기
- `triggers.*.enabled`: 각 트리거별 알림 활성화
- `quiet_hours`: 야간 시간대 (Critical만 발송)
- `duplicate_prevention.window_hours`: 중복 발송 방지 시간 (기본 24시간)
- `rate_limit`: 시간당/일별 최대 알림 수

**PlayMCP 연결 필요:** claude.ai 설정 > Connectors에서 PlayMCP 활성화.

### 파일 부재 시 동작
- **portfolio.json 없음**: portfolio-analyst·position-sizer·rebalancer 단계 스킵 (Mode A의 종목 분석은 정상 작동)
- **tax-settings.json 없음**: 한국 거주자 표준값 사용
- **market-watchlist.json 없음**: 영향 없음

---

## 3. 디렉토리 구조 (사용 시)

작업 디렉토리는 다음과 같이 구성됩니다:

```
your-workspace/
├── portfolio.json              ← 본인 정보로 작성
├── tax-settings.json           ← 본인 정보로 작성
├── market-watchlist.json       ← (선택)
│
├── .claude/
│   └── agents/                 ← 에이전트 설치 위치
│       ├── stock-analyst-orchestrator.md
│       └── ... (20개)
│
└── reports/                    ← 자동 생성
    ├── 005930_recommendations.jsonl
    ├── AAPL_recommendations.jsonl
    └── ...
```

---

## 4. 첫 실행 시나리오

### 시나리오 A: 신규 종목 분석

```bash
cd your-workspace
claude --agent stock-analyst-orchestrator

# Claude Code 세션 시작
> 삼성전자 분석해줘

# 10-Phase 자동 실행
# 결과 출력 + reports/005930_recommendations.jsonl 자동 생성
```

### 시나리오 B: 보유 종목 점검 (정기)

```bash
> 내 보유 종목 점검해줘
> 이번 주 액션 정리

# position-monitor + reanalysis-trigger 호출
# 액션 우선순위 리스트 출력
```

### 시나리오 C: 모닝 브리핑

```bash
> 오늘 아침 브리핑

# daily-briefing 호출
# 5분 안에 읽는 시장+보유종목 요약
```

### 시나리오 D: 분기 리밸런싱

```bash
> 포트폴리오 리밸런싱

# portfolio-rebalancer 호출
# 세금·비용 반영 거래 리스트 출력
```

### 시나리오 E: 단일 측면 분석

```bash
> @financial-analyst 카카오 재무만 분석
> @position-sizer 위 advisor 결과로 비중 계산
> @tax-cost-calculator AAPL 100주 매도 시 세후 손익
```

### 시나리오 F: 백테스팅 (v5 신규)

```bash
> 시스템 성과 평가
> 백테스팅 돌려줘
> 추천 정확도 확인

# 3단계 자동 실행:
# 1. backtest-collector → reports/*.jsonl + 실제 가격 매칭
# 2. backtest-analyzer → 6개 지표 + 세그먼트 분석
# 3. backtest-optimizer → 개선 권고 (선택)
```

**조건:**
- 추천 데이터 30+ 누적 필요 (통계적 유의성)
- 12개월 경과 추천이 있어야 풀 평가 가능
- 첫 의미 있는 결과는 사용 시작 후 약 1년 후

### 시나리오 G: 카톡 알림 (v6 신규)

**자동 트리거 (백그라운드 동작):**
- 손절선 도달, 무효화 트리거 발동, 목표가 도달, 신규 Strong BUY 등
- Mode A/B/C/F 실행 중 조건 충족 시 자동으로 kakao-notifier 호출

**수동 요청:**
```bash
> 카톡으로 보내줘
> 손절선 도달하면 카톡 알림 켜줘
> 오늘 모닝 브리핑 카톡으로 보내줘
> 알림 설정 확인
```

**선행 조건:**
1. PlayMCP 연결 (claude.ai 설정 > Connectors > PlayMCP 활성화)
2. 알림 설정 파일 작성:
   ```bash
   cp stock-analyst-agents/templates/notification-preferences.json.example \
      notifications/preferences.json
   ```
   본인 설정으로 편집 (어떤 트리거를 켤지 등).

**제약:**
- PlayMCP `KakaotalkChat-MemoChat` 도구는 **메시지당 최대 200자**
- 200자 초과 시 kakao-notifier가 자동 분할 ("(1/3)", "(2/3)" 표시)
- 분할 메시지 간 2초 간격
- 동일 종목·트리거 24시간 내 중복 발송 방지

---

## 5. 권장 워크플로우 (월간)

```
[항상 백그라운드 — v6 신규]
└─ kakao-notifier       → 손절선·트리거·목표가 발동 시 자동 카톡 알림

[매일 아침]
└─ daily-briefing       → 시장+보유종목 점검 (선택: 카톡으로 받기)

[주 1회]
└─ position-monitor     → 보유 종목 손절·트리거 점검

[월 1회]
├─ reanalysis-trigger   → 재분석 필요 종목 식별
└─ 식별된 종목 풀 재분석

[분기 1회]
├─ portfolio-rebalancer → 비중 점검·리밸런싱
└─ 모든 보유 종목 정기 재분석

[6개월 / 1년 1회 — v5]
└─ 백테스팅 사이클 (Mode F)
   ├─ backtest-collector  → 실제 결과 매칭
   ├─ backtest-analyzer   → 시스템 정확도 측정
   └─ backtest-optimizer  → 개선 권고 (필요시)

[신규 매수 검토 시]
├─ Mode A 풀 분석 (10 Phases — v6에서 6개 분석가 + ESG 통합)
└─ entry-timing-checker → 진입 시점 확인

[매수·매도 직전]
├─ tax-cost-calculator  → 세후 수익률 확인
└─ order-book-analyst   → 주문 실행 권고
```

---

## 6. 자주 발생하는 이슈

### "Agent tool not available" 또는 spawn 실패
- 원인: 오케스트레이터를 서브에이전트로 호출 (자동 위임 경로)
- 해결: `claude --agent stock-analyst-orchestrator`로 메인 세션 실행

### "portfolio.json not found"
- 원인: 작업 디렉토리에 파일 없음
- 해결: 템플릿 복사 후 편집 (위 2번 참고)

### 추천 기록(jsonl)이 누적되지 않음
- 원인: 오케스트레이터의 Bash/Edit 권한 누락 또는 작업 디렉토리 권한 부족
- 해결: 작업 디렉토리에 쓰기 권한 확인, `reports/` 디렉토리 수동 생성 시도

### 분석 시간이 너무 오래 걸림
- v4는 20개 에이전트 협업으로 v2 대비 시간이 더 걸림 (정상)
- 평균 풀 분석 시간: 5~15분
- 단축 옵션: Mode E로 필요한 측면만 호출

---

## 7. 라이선스

본 시스템은 정보 제공 목적으로 제공됩니다. 본 시스템의 출력은 **투자 자문이 아니며**, 모든 매매 결정과 결과는 사용자 본인 책임입니다.
