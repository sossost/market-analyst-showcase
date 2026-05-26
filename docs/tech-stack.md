# Tech Stack

기술 선택은 모두 한 명이 전 도메인을 관리할 수 있도록 **인지 부하 최소화** 기준으로 했다.

---

## 런타임 / 언어

### Node.js 20 (ESM) + TypeScript strict

- 백엔드 cron 잡 / Next.js 대시보드 / B2C 서비스 모두 동일 런타임 → 컨텍스트 스위칭 최소
- ESM 통일로 dual-package hazard 회피
- TypeScript strict + `noUncheckedIndexedAccess` — 런타임에 발견될 버그를 컴파일 타임에 잡는다
- Branded type / Discriminated union으로 도메인 primitive 구분

### 트레이드오프

- Python을 안 쓴 이유: 머신러닝 학습 파이프라인이 없다. 추론은 모두 외부 LLM API로 위임하므로 Python 생태계 의존이 불필요.
- Bun/Deno를 안 쓴 이유: Drizzle ORM / Next.js 안정성이 Node에서 가장 검증되어 있다.

---

## 데이터 / ORM

### PostgreSQL (Supabase 호스팅) + Drizzle ORM

- 37개 테이블, 시계열 + 관계형 데이터 혼합
- Supabase의 PostgREST는 backoffice/B2C 프론트엔드용. 무거운 ETL은 Drizzle로 직접 SQL.
- 마이그레이션은 `drizzle-kit push` 기반 단방향 흐름 (롤백 케이스가 거의 없는 분석 시스템)

### 왜 Prisma가 아닌가

- 쿼리 최적화 시 raw SQL 자유도가 더 높다
- 스키마 단일 source of truth (`src/db/schema/`)가 TS 파일이라 IDE 친화
- 마이그레이션 파일 관리 부담이 적다

### 왜 Mongo가 아닌가

- 시계열 join / 윈도우 함수가 빈번하다 (RS 계산, Phase 추적, brigade 분석)
- PostgreSQL의 JSONB로 유연성도 충분히 확보 (theses, narrative_chains 등에 사용)

---

## AI 모델 / SDK

### 다중 SDK 동시 사용

| SDK / CLI | 용도 |
|-----------|------|
| `@anthropic-ai/sdk` | Claude Opus/Sonnet/Haiku 직접 호출 (Macro·Industry·Moderator, QA, 분석) |
| Claude Code CLI | Issue Processor / PR Reviewer (코드 작성/리뷰 컨텍스트) |
| OpenAI Codex CLI | Sentiment 페르소나 (gpt-5.5 계열) |
| OpenAI SDK (xAI baseURL) | Tech 페르소나 (Grok 4.3 — xAI는 OpenAI 호환 API) |
| `@google/generative-ai` | Geopolitics 페르소나 (Gemini 3.1 Pro Preview) |

### 모델 선택 원칙

| 작업 | 모델 | 이유 |
|------|------|------|
| Macro / Industry / Moderator | Claude Opus | 깊은 추론, 긴 컨텍스트 |
| Tech | xAI Grok 4.3 | agentic tool calling 강점, 환각 적음, 1M context |
| Geopolitics | Google Gemini 3.1 Pro Preview | reasoning + 검색 통합, 다국어 |
| Sentiment | OpenAI gpt-5.5 (Codex CLI) | 언어 직감 + 다양성 확보 |
| 짧은 분류/정리 | Claude Haiku | 비용·속도 |
| 코드 작성/리뷰 | Claude Sonnet (Code CLI) | 코드 품질 |

비용 최적화가 목표가 아니다. **모델 다양성(4 lineage)**으로 토론 품질을 확보하는 것이 목표.

---

## 프론트엔드

### Next.js (App Router) + React 19 + Suspense

- backoffice / b2c 모두 Next.js
- Server Component 우선, 인터랙션 부분만 Client Component
- 데이터 페치는 모두 Suspense 경계 + ErrorBoundary로 감싸 로딩/에러 상태를 컴포넌트마다 수동 관리하지 않음

### 스타일링

- CSS variables + 모듈형 스타일
- 한국 금융 컨벤션: **상승 = 빨강 / 하락 = 파랑** (적녹색약 호환 + 한국 표준)
- 글로벌 오버라이드 없음. 컴포넌트별 격리.

### 차트

- Recharts (정형 차트)
- D3 (Gantt, 5트랙 국면 시각화 같은 커스텀)

---

## 테스트

### Vitest

- 단위 테스트 80% 이상 라인 커버리지
- 통합 테스트는 실제 DB 인스턴스(테스트 schema)에서 실행
- TDD 워크플로 — RED → GREEN → REFACTOR

### 무엇을 테스트하지 않는가

- LLM 출력 자체는 단위 테스트하지 않음 (비결정적)
- LLM **주변의 컨텍스트 빌드 / 결과 파싱 / 후처리**만 테스트
- E2E는 critical user flow (매매일지, 리포트 발행)만 Playwright로 커버

---

## 운영 인프라

### macOS launchd

- 일 17회 cron 안정 가동, 1년 이상 누적 가동 무중단
- plist 파일은 `scripts/launchd/` 디렉터리에 git 관리
- 배포는 `launchctl unload && launchctl load`

### GitHub Actions

- 사용처는 PR CI (lint + build + test)만
- 배포는 맥미니 자체 cron으로 처리

### Discord

- 운영 알림 단일 채널 정책. 채널 5개로 도배되는 것 회피.
- webhook 기반, 모든 cron이 시작/완료/실패 시 통보

### Tailscale

- 맥미니 원격 SSH 접근
- 외부 노출 없이 안전한 운영 채널 확보

---

## 외부 데이터 소스

| 소스 | 용도 | 라이선스 정책 |
|------|------|--------------|
| FMP Ultimate | 가격, 재무, 어닝콜, 추정치 | raw 외부 노출 금지, 가공물(RS/등급/콘텐츠)만 노출 |
| FRED | 매크로 지표 (HY OAS, 신용 스프레드 등) | 공공 |
| Brave Search | 매크로 뉴스 |  |
| RSS (CNBC/MarketWatch) | 뉴스 보강 |  |
| Yahoo Finance | 보조 (ad-hoc) |  |

---

## 종합 결정 메모

### "왜 모노레포인가"

- 5개 도메인이 같은 schema(drizzle)에 의존
- 같은 타입을 5개 패키지에서 import해야 함
- yarn workspaces로 충분히 가벼움 (Turborepo/Nx 도입 안 함 — 도구 학습 비용 회피)

### "왜 마이크로서비스가 아닌가"

- 1인 운영 시스템에는 네트워크 hop만큼의 비용 가치가 없다
- DB가 도메인 간 인터페이스 역할을 충분히 한다
- 트래픽 압박이 없으므로 수직 확장으로 충분

### "왜 자체 호스팅(맥미니)인가"

- 클라우드 비용보다 디버깅 단순성이 더 중요한 단계
- 모든 컴포넌트가 같은 머신에 있으면 로그 추적이 단순하다
- 머신 1대로 충분히 처리되는 워크로드

이 모든 선택은 **재검토 대상**이다. 트래픽이 늘거나 팀이 커지면 바뀐다.
