# Agent System

이 시스템의 핵심은 **여러 AI 모델을 페르소나로 배치한 토론 엔진**과, **자기 가설을 검증해 다음 토론에 반영하는 학습 루프**다.

---

## 멀티모델 5인 토론

### 왜 단일 모델이 아닌가

단일 모델로 여러 페르소나를 흉내내면 톤만 다를 뿐 사고 패턴은 같아진다. 모델마다 강점이 다르다는 점을 이용해, **모델 자체를 페르소나에 매핑**했다.

### 페르소나 구성

| 페르소나 | 책임 | 모델 |
|---------|------|------|
| Tech Analyst | RS·Phase·브레드스 기반 기술적 분석 | xAI Grok 4.3 |
| Macro Economist | 금리/유동성/매크로 지표 해석 | Claude Opus |
| Industry Analyst | 업종/공급망/병목 분석 | Claude Opus |
| Sentiment Analyst | 뉴스/심리/내러티브 추적 | OpenAI Codex CLI (gpt-5.5 계열) |
| Geopolitics | 지정학·정책 국면 해석 | Google Gemini 3.1 Pro Preview |
| Moderator | 3R 종합 + thesis 구조화 | Claude Opus |

**5개 토론 페르소나 + 1개 Moderator = 4개 lineage (Anthropic·xAI·Google·OpenAI)** — 단일 모델 합의 회피 + 도메인별 강점 매칭.

### 토론 구조

```mermaid
flowchart TB
    R1["<b>Round 1 · 독립 의견</b><br/>5 페르소나 각자 별도 컨텍스트<br/>서로의 답을 보지 않음<br/>정량 데이터 + few-shot(과거 학습) 주입"]
    R2["<b>Round 2 · 상호 반박/보강</b><br/>Round 1 결과를 전체 페르소나에 노출<br/>동의 · 반박 · 추가 근거 요청"]
    R3["<b>Round 3 · Moderator 합의</b><br/>합의점/대립점 정리<br/>검증 가능한 thesis 구조화<br/>narrative_chain 갱신"]

    R1 --> R2 --> R3

    classDef round fill:#1f2937,stroke:#60a5fa,color:#f9fafb
    class R1,R2,R3 round
```

토론 결과는 `debate_sessions` 테이블에 라운드별로 저장된다. 모든 출력이 보존되므로 사후에 어떤 페르소나가 어떤 시점에 무엇을 말했는지 추적 가능.

---

## Thesis: 검증 가능한 예측

토론의 산출물은 의견이 아니라 **검증 가능한 예측**이다.

각 thesis는 다음을 가진다:

- 명시적 가설 ("AI 인프라 사이클 진입, 반도체 장비 업종 1~2분기 내 RS 70 돌파")
- 검증 시점 (특정 날짜 또는 트리거 조건)
- 검증 메트릭 (RS 임계값, 가격 변동, 실적 지표)
- 상태: `ACTIVE` / `HYPOTHESIS` / `CONFIRMED` / `INVALIDATED` / `EXPIRED`
- 연결된 국면 FK (geopolitical_regimes, policy_regimes) — 어떤 매크로 전제 위에서 만든 가설인지 추적
- 수혜 종목 매핑 (narrative_chains.beneficiaryTickers)

### 자동 검증

`thesisVerifier`가 매일 ACTIVE thesis를 스캔해 검증 조건을 평가한다.

- 만료일 도래 + 조건 미충족 → `INVALIDATED` 또는 `EXPIRED`
- 조건 충족 → `CONFIRMED`
- HOLD 임계 초과 stale thesis 자동 만료
- 2일 연속 grace period 적용으로 단기 노이즈에 의한 조기 청산 방지

검증 결과는 단순한 라벨이 아니다. **다음 토론의 few-shot 입력**이 된다.

---

## Learning Loop

검증된 thesis와 패턴은 시스템의 장기 기억으로 누적된다.

```mermaid
flowchart TB
    D["<b>Debate</b><br/>thesis 생성 (ACTIVE)"]
    V["<b>thesisVerifier</b><br/>시간 경과 후<br/>ACTIVE → CONFIRMED / INVALIDATED"]
    P["<b>promote-learnings (ETL job)</b><br/>반복 적중 → agent_learnings<br/>반복 실패 → failure_patterns<br/>최대 50개 유지 (오래된 것 강등)"]
    F["<b>Few-shot 주입</b><br/>다음 토론 호출 시<br/>페르소나 프롬프트에 학습 사례 첨부"]

    D --> V --> P --> F
    F -. 다음 라운드 .-> D

    classDef step fill:#1f2937,stroke:#34d399,color:#f9fafb
    class D,V,P,F step
```

이 루프 덕에 시스템은 시간이 지날수록 **자신이 어떤 종류의 가설에 강하고 약한지** 학습한다. 새로 만든 thesis가 과거 실패 패턴과 매칭되면 토론 단계에서 자동으로 경고가 발생한다.

---

## Deep Research — 외부 증거로 내러티브 검증

토론은 "AI 인프라 사이클의 병목은 HBM 메모리" 같은 **내러티브 체인**(megatrend → bottleneck → 수혜주)을 만든다. 하지만 이건 모델 내부 지식에 기반한 가설일 뿐이다. 병목이 지금도 유효한지, 누가 진짜 수혜를 보는지는 **외부 실시간 증거**로 확인해야 한다.

이 역할을 토론 페르소나와 분리된 독립 에이전트 `chain-researcher`(Sonnet)가 맡는다.

### 토론 페르소나와 무엇이 다른가

| | 5인 토론 페르소나 | chain-researcher |
|---|---|---|
| 입력 | DB 정량 데이터 + few-shot | 병목 체인 1개 + 보유 데이터(어닝콜/뉴스) |
| 도구 | 없음 (내부 추론) | WebSearch · WebFetch (외부 증거) |
| 출력 | 의견 → 합의 → thesis | 구조화 JSON (verdict + 출처 + 수혜주 후보) |
| 호출 | debate 엔진 | `run-chain-researcher.ts` 잡이 CLI로 호출 |

### 무엇을 산출하는가

체인 하나를 받아 외부 증거를 수집하고 세 가지를 판정한다.

1. **병목 지속성** — `PERSISTS` / `WEAKENING` / `RESOLVED`. 증산 발표·대체 공급망 등 해소 신호를 근거로 판정.
2. **수혜주 발굴·확장** — 현재 병목에 직접 노출된 기업만 후보로 발굴 (다음 병목 수혜는 제외하고 텍스트로만 서술).
3. **다음 병목 예측** — 이 병목이 풀리면 어디로 이동할지.

모든 주장에는 출처 URL + 발행 타임스탬프가 강제된다. Bloomberg·WSJ·SEC 공시는 confidence 상향, 소셜/익명 소스만 있으면 confidence 0.3 미만으로 기록되어 후속 잡이 자동 기각한다.

### 3층 분리 — 증거 / 스테이징 / 상태

리서치 결과를 체인 본체에 바로 덮어쓰지 않는다. 알파 귀속을 사후 측정할 수 있도록 증거를 보존한다.

```mermaid
flowchart TB
    R["<b>chain-researcher</b><br/>외부 웹 증거 수집<br/>verdict · 수혜주 후보 · 출처 JSON"]
    RUN["<b>research_runs</b> (증거층)<br/>append-only · 출처/confidence 보존<br/>덮어쓰기 없음"]
    CAND["<b>beneficiary_candidates</b> (스테이징)<br/>PROPOSED → ACCEPTED / REJECTED<br/>row 단위 보존 (6개월 후 알파 귀속)"]
    CHAIN["<b>narrative_chains</b> (상태)<br/>verdict 스트릭 → status 전이"]

    R --> RUN
    RUN -->|apply 잡| CAND
    RUN -->|evaluate 잡| CHAIN

    classDef step fill:#1f2937,stroke:#34d399,color:#f9fafb
    class R,RUN,CAND,CHAIN step
```

### 생애주기 전이 — verdict 히스테리시스

단발 verdict로 체인 상태를 바꾸지 않는다. 노이즈에 흔들리지 않도록 **연속 판정**을 요구하는 히스테리시스를 적용한다.

- `evaluate-chain-lifecycle` 잡이 미평가 run을 오래된 순서로 fold
- 예: `WEAKENING` 2회 연속 → 체인 `ACTIVE` → `RESOLVING` 전이
- **LLM 호출 없는 결정론적 룰** — 같은 입력은 같은 결과
- 각 run은 `lifecycleEvaluatedAt` 마커로 정확히 1회만 스트릭에 반영 (재실행 안전, idempotent)

토론이 가설을 세우고 → 딥리서치가 외부 증거로 검증하고 → 검증 결과가 다시 토론의 few-shot으로 돌아온다. 학습 루프가 가격·실적이라는 *시장 결과*로 닫힌다면, 딥리서치 루프는 *외부 사실*로 닫힌다.

---

## QA Gate — 발행 전 3축 채점

리포트는 작성되었다고 바로 발행되지 않는다. 3축 채점을 통과해야 한다.

| 축 | 채점 방식 | 측정 대상 |
|----|---------|---------|
| 성과 축 | 코드 (정량) | 과거 추천의 누적 PnL, 적중률, drawdown |
| 분석 품질 축 | 코드 (정량) | 데이터 일관성, thesis 검증 비율, 모집단 sanity check |
| 리포트 품질 축 | LLM (페르소나) | 논리 흐름, 근거 명시, 톤, 모순 검출 |

### 임계값 정책

- `daily_qa_reports.severity`: `ok` / `warn` / `block`
- `block` 시 발행 차단 + GitHub Discussions Alert 자동 게시
- mismatches는 JSONB로 저장되어 어디서 어떤 사실이 어긋났는지 보존

### 페르소나 피드백 루프

리포트 품질 축의 페르소나 피드백은 **다음 주간 에이전트의 시스템 프롬프트에 자동 주입**된다. 같은 실수를 두 번 하지 않게 하는 가장 단순하지만 강력한 메커니즘.

예: "지난주 리포트에서 섹터 RS 변동을 일간 단위로 봤다가 노이즈로 판정됐다. 4주 변화로 보라" 같은 메모가 다음 주에 자동으로 첨부된다.

---

## 매니저 + 에이전트 조직 체계

엔지니어링 작업 자체도 에이전트 체계로 운영된다.

```mermaid
flowchart TB
    CEO(["CEO (사용자)"])
    M["Manager"]
    MP["mission-planner<br/>미션 기획"]
    PR["pr-manager<br/>PR 생애주기"]
    SA["strategic-aide<br/>매일 04:00<br/>전략 브리핑"]
    AP["analyst-po"]
    PP["portfolio-po"]
    OP["ops-po"]
    BP["backoffice-po"]
    BE["backend-engineer"]
    FE["frontend-engineer"]

    CEO --> M
    M --> MP
    M --> PR
    M --> SA
    M -.PO 위임.-> AP
    M -.PO 위임.-> PP
    M -.PO 위임.-> OP
    M -.PO 위임.-> BP
    AP --> BE
    AP --> FE
    PP --> BE
    OP --> BE
    BP --> FE

    classDef direct fill:#1f2937,stroke:#fbbf24,color:#f9fafb
    classDef po fill:#1f2937,stroke:#a78bfa,color:#f9fafb
    classDef exec fill:#1f2937,stroke:#34d399,color:#f9fafb
    classDef boss fill:#1f2937,stroke:#f87171,color:#f9fafb
    class CEO,M boss
    class MP,PR,SA direct
    class AP,PP,OP,BP po
    class BE,FE exec
```

### 동작 원리

1. CEO가 미션 부여 → 매니저는 즉시 코딩하지 않음
2. `mission-planner`에 위임 → 기획서 생성 (기획서 없이 구현 금지)
3. 매니저가 기획서 검증 (불필요한 제약, 타이밍 충돌, 근거 없는 숫자 체크)
4. PO 위임 또는 매니저 직접 디스패치 결정
5. 실행 에이전트 디스패치 (독립 작업은 병렬)
6. `code-reviewer` 실행, CRITICAL/HIGH 수정
7. `pr-manager`에 위임해 PR 생성 (직접 `gh pr create` 금지)
8. CEO 명시적 지시 시에만 머지

이 체계 자체가 코드 베이스에 prompt 파일로 정의되어 있고, Claude Code CLI가 매 세션에서 자동 로드한다.

총 specialized agent 20종. 토론 페르소나 5종 + 직속 3종 + PO 4종 + 실행팀 + 딥리서치(chain-researcher) + QA·data viz 등.

---

## 어필 포인트 요약

- **모델 다양성을 페르소나로 활용** — 단일 모델 합의가 아닌, 서로 다른 모델의 강점을 페르소나로 배치
- **자기 검증** — thesis 자동 검증으로 시스템이 자신의 적중률을 정량 추적
- **외부 사실 검증** — 독립 딥리서치 에이전트가 외부 웹 증거로 내러티브를 검증하고 출처와 함께 보존
- **자기 학습** — 검증 결과가 다음 토론 few-shot으로 자동 주입
- **자기 품질 통제** — QA gate가 발행을 차단하고, 페르소나 피드백이 프롬프트를 개선
- **자기 운영** — 엔지니어링 작업까지 에이전트 조직으로 운영 (다음 문서 참조)
