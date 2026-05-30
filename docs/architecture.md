# Architecture

## 설계 원칙

1. **도메인 경계 분리** — 시그널 발굴 / PF 결정 / 자동화 운영 / 대시보드 / 외부 노출을 서로 다른 패키지로 격리.
2. **shared 단방향 의존** — 모든 패키지는 `shared`에만 의존한다. analyst가 portfolio를 직접 호출하지 않는다.
3. **DB가 인터페이스** — 도메인 간 통신은 함수 호출이 아니라 DB 테이블이다. 한 도메인이 쓰고, 다른 도메인이 읽는다. 이 구조 덕에 어느 도메인이든 독립적으로 cron으로 재실행할 수 있다.
4. **재현성** — 모든 파생 지표와 리포트는 입력 데이터만 있으면 재생성 가능해야 한다. 학습 결과는 별도 테이블에 누적.

---

## 모노레포 레이아웃

```mermaid
flowchart LR
    subgraph Domains
        A["analyst<br/>시그널 발굴<br/>jobs · debate · agent · fundamental"]
        P["portfolio<br/>PF 결정<br/>tier · exit · sizing"]
        O["ops<br/>자동화 운영<br/>etl · reports · qa<br/>issue-processor · pr-reviewer"]
        B["backoffice<br/>운영자 대시보드 (Next.js)<br/>KPI · 매매일지 · 5트랙 국면"]
        C["b2c<br/>외부 노출 (Next.js)<br/>레이어 A 가공 데이터"]
    end
    S["shared<br/>Drizzle schema · DB client · 도메인 타입"]

    A --> S
    P --> S
    O --> S
    B --> S
    C --> S

    classDef domain fill:#1f2937,stroke:#60a5fa,color:#f9fafb
    classDef shared fill:#1f2937,stroke:#fbbf24,color:#f9fafb
    class A,P,O,B,C domain
    class S shared
```

모든 도메인은 `shared`에만 단방향 의존한다. 도메인 간 직접 호출은 없고, 통신은 DB 테이블을 매개로 한다.

---

## 데이터 플로우

```mermaid
flowchart TB
    EXT["External Sources<br/>FMP Ultimate · FRED · Brave Search · RSS · GitHub"]
    L1["<b>Layer 1 · Market Raw</b><br/>symbols · daily_prices · daily_ma · index_prices<br/>quarterly_financials · earning_call_transcripts<br/>stock_news · news_archive · credit_indicators"]
    L2["<b>Layer 2 · Derived Signals</b><br/>stock_phases · sector_rs_daily · industry_rs_daily<br/>theme_rs_daily · market_breadth_daily<br/>fundamental_scores (SEPA)"]
    L3["<b>Layer 3 · Analysis & Synthesis</b><br/>debate_sessions · theses · narrative_chains<br/>narrative_chain_research_runs · beneficiary_candidates<br/>market_regimes · meta_regimes<br/>geopolitical_regimes · policy_regimes<br/>tracked_stocks · portfolio_positions"]
    L4["<b>Layer 4 · Learning</b><br/>agent_learnings · failure_patterns<br/>recommendation_factors · signal_log"]
    L5["<b>Layer 5 · Reports & QA</b><br/>daily_reports · stock_analysis_reports<br/>weekly_qa_reports · daily_qa_reports"]
    OUT["Discord · Backoffice · B2C"]

    EXT -->|ETL| L1
    L1 -->|Derived| L2
    L2 -->|Agent| L3
    L3 -->|Verify & Learn| L4
    L4 -. few-shot 주입 .-> L3
    L3 -->|Deliver| L5
    L5 --> OUT

    classDef raw fill:#1f2937,stroke:#60a5fa,color:#f9fafb
    classDef derived fill:#1f2937,stroke:#a78bfa,color:#f9fafb
    classDef agent fill:#1f2937,stroke:#34d399,color:#f9fafb
    classDef learn fill:#1f2937,stroke:#fbbf24,color:#f9fafb
    classDef out fill:#1f2937,stroke:#f87171,color:#f9fafb
    class EXT,L1 raw
    class L2 derived
    class L3 agent
    class L4 learn
    class L5,OUT out
```

---

## 도메인 책임

### Analyst

시장에서 알파 후보를 발굴한다. 직접 매수/매도 결정을 내리지 않는다.

- **Phase 판정** — Stan Weinstein 4단계 모델로 종목별 사이클 단계를 매일 판정
- **RS 계산** — 종목/섹터/업종/테마 4계층의 상대강도를 가중 평균(단기 0.5 + 중기 0.3 + 장기 0.2)으로 계산
- **SEPA 스코어링** — Mark Minervini의 Specific Entry Point Analysis 기반 펀더멘탈 등급(S/A/B/C/F)
- **5인 토론** — 5개 페르소나가 RS 귀납 → 공급망 연역 → 수요-공급-병목 서사 → thesis 구조화
- **narrative chain** — megatrend → bottleneck → beneficiary 종목으로 이어지는 인과 사슬 추적
- **딥리서치** — 독립 에이전트(chain-researcher)가 외부 웹 증거로 병목 지속성 판정·수혜주 발굴, 증거를 append-only 층에 보존하고 verdict 히스테리시스로 체인 생애주기를 전이

### Portfolio (PM)

Analyst의 시그널을 받아 모델 포트폴리오 편입/청산을 결정한다.

- 다중 게이트(기술적 + 펀더멘탈 + 직전 상태)를 통과한 종목을 featured로 격상
- 격상 게이트가 깨지면 강등, Phase Exit 조건 충족 시 청산
- 동시 보유 한도 (총 N / 섹터 N / 업종 N) 강제
- 현재 L1 룰 기반. L2 LLM 결정 단계로 진화 예정.

### Operations

도메인 간 cron 오케스트레이션과 외부 인터페이스를 담당한다.

- ETL: FMP/FRED/뉴스/어닝콜 데이터 수집
- 리포트 발행 (일간/주간/기업/QA)
- QA 게이트 3축 채점
- Issue Processor: GitHub Issue를 받아 Claude Code CLI로 PR 생성
- PR Reviewer: 열린 PR Strategic + Code 자동 리뷰

### Backoffice

Next.js 기반 운영자 대시보드. control-tower 구버전을 모노레포로 흡수했다.

- 알파 KPI: 누적 PnL vs SPY, Sharpe, 적중률
- 매매일지 UI
- 5트랙 국면 시각화 (시장 레짐 / 섹터 사이클 / 메가 사이클 / 지정학 / 정책)
- 운영 헬스 대시보드

### B2C

외부 사용자에게 노출하는 서비스. 레이어 A만 가동.

- 레이어 A: 레짐, 섹터/업종 RS, 내러티브 가공 데이터 (개별 종목 거명 없음)
- 레이어 B: 종목 알파 노출 — 누적 PnL/Sharpe 선행 조건 충족 시 가동
- FMP raw 데이터(가격 OHLCV / 재무제표 raw / 어닝콜 원문)는 외부 노출 금지, 우리 가공물(RS 순위, 등급, 에이전트 콘텐츠)만 노출

---

## DB 구조 요약

81개 테이블을 7개 군으로 분류한다.

| 군 | 대표 테이블 | 책임 |
|----|-----------|------|
| 시장 원본 | daily_prices, symbols, quarterly_financials | ETL 수집 데이터 |
| 파생 지표 | stock_phases, sector_rs_daily, fundamental_scores | ETL 계산 결과 |
| 분석·추천 | tracked_stocks, portfolio_positions, signal_log | 에이전트 산출물 |
| 토론·학습 | debate_sessions, theses, agent_learnings, narrative_chains, narrative_chain_research_runs | 토론 + 딥리서치 + 학습 |
| 리포트·QA | daily_reports, weekly_qa_reports | 발행물 + 품질 검증 |
| 기업 데이터 | company_profiles, earning_call_transcripts, eps_surprises | FMP 확장 |
| 패턴 분석 | sector_phase_events, sector_lag_patterns | 시차 패턴 |

스키마는 Drizzle ORM으로 정의되어 있으며, `db push` 기반의 단방향 마이그레이션 흐름을 사용한다.

---

## 비기능 결정 메모

- **Suspense + ErrorBoundary** — Next.js 대시보드의 데이터 페치는 모두 Suspense 경계로 감싼다. 로딩/에러 상태를 컴포넌트마다 수동 관리하지 않는다.
- **불변성** — 모든 도메인 객체는 `Readonly<T>` 또는 `as const` 사용. 변경은 새 객체 생성.
- **Discriminated Union** — 상태 모델링은 boolean flag 대신 union 타입. 예: `ThesisStatus = 'ACTIVE' | 'HYPOTHESIS' | 'CONFIRMED' | 'INVALIDATED' | 'EXPIRED'`
- **Branded type** — symbol, accountId 같은 도메인 primitive는 brand로 컴파일 타임에 구분.
