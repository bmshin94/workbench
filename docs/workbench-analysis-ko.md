# Workbench 전수조사 분석 정리 (한국어)

> BullMQ 대시보드 `Workbench` 저장소를 전수조사하고, 설치/사용법·정체성·수익화까지 정리한 문서.
> 작성일: 2026-09-27

## 🔗 관련 링크

| 구분 | URL |
| --- | --- |
| 원본 저장소 (upstream) | https://github.com/pontusab/workbench |
| 포크 저장소 (이 저장소) | https://github.com/bmshin94/workbench |
| 공식 사이트 | https://getworkbench.dev |
| 공식 문서 | https://getworkbench.dev/docs |
| bull-board 비교글 | https://getworkbench.dev/blog/workbench-vs-bull-board |
| 표준 Docker 이미지 | `ghcr.io/pontusab/workbench-standalone` |
| BullMQ 공식 문서 | https://docs.bullmq.io/ |
| 관련 프로젝트 (제작자) | https://github.com/pontusab/hyper · https://midday.ai |

---

## 1. 이게 뭐하는 프로젝트인가

**Workbench = BullMQ(Node.js 작업 큐)용 오픈소스 대시보드.** `bull-board`의 현대적 drop-in 대체품이며,
기존 백엔드 앱에 **라우트 한 줄만 마운트**하면 동작한다.

```
라이선스   MIT (상업적 이용·수정·재배포·클로즈드소스화 허용, 저작권 고지 의무)
버전       0.9.1 (CLI 0.5.0 / MCP 0.5.1)
코드 규모   TS/TSX 약 34,441줄
패키지 관리 bun 1.3.11 + turbo + biome
요구사항   Node 18+ (또는 Bun 1.1+), Redis
```

### 핵심 아키텍처 — "코어 1개 + 얇은 어댑터 N개"

`@getworkbench/core`가 웹 표준 `Request`/`Response`(Hono 기반)로 모든 로직을 처리하고,
프레임워크 어댑터는 형식 변환만 담당한다. 그래서 어댑터 1개가 파일 1개, 약 30줄이다.

```ts
// packages/hono/src/index.ts — 어댑터 전체가 이게 끝
export function workbench(options: WorkbenchOptions | Queue[]): Hono {
  const core = new WorkbenchCore(options);
  return buildWorkbenchApp(core);
}
```

### 저장소 구조 전수조사

#### `packages/` — 배포되는 16개 npm 패키지

| 패키지 | 역할 |
| --- | --- |
| **`core`** | 심장부. QueueManager + REST API 라우터 + React UI 전부 (87개 파일) |
| `hono` `express` `fastify` `koa` `nestjs` `adonis` `elysia` `bun` `h3` `next` `nuxt` `astro` `tanstack-start` | **프레임워크 어댑터 13개** (각 파일 1개, ~30줄) |
| `cli` | `npx @getworkbench/cli init` — 프레임워크 자동 감지 + 코드 주입 |
| `mcp` | MCP 서버 (Cursor/Claude Desktop/Zed/Continue), 도구 23개 |

#### `packages/core` 내부

```
src/core/
  queue-manager.ts      약 2,300줄. Redis/BullMQ 실제 조작 담당
  workbench.ts          설정 정규화 + 큐 자동 발견
  discover.ts           Redis에서 `<prefix>:*:meta` 스캔해 큐 자동 탐지
  alert-manager.ts      BullMQ QueueEvents 구독 → 알림 발사
  alert-destinations.ts Slack/Discord/Webhook 페이로드 생성 + URL 검증
  redis-alert-store.ts  알림 설정 Redis 영속화 (`{prefix}:workbench:alerts:*`)
  types.ts              전체 타입 정의
src/api/handlers.ts     REST 라우트 39개 (746줄)
src/server/             basic-auth, basePath 자동감지, 정적 파일 서빙
src/ui/                 React 19 대시보드 (페이지 10개)
```

**UI 페이지 10개**: `overview` · `queue` · `job` · `runs` · `metrics` · `schedulers` · `flows` + `flow`(DAG) · `alerts` · `test`

#### `apps/` — 실행 가능한 앱 3개

| 앱 | 정체 |
| --- | --- |
| `standalone` | Bun 서버 + Dockerfile → GHCR 도커 이미지 |
| `desktop` | **Tauri 2 macOS 앱.** Rust 코어 + Bun 사이드카(127.0.0.1 루프백 전용) + React UI, 자동 업데이트, 키체인 비밀 저장 |
| `web` | Next.js 16 마케팅 사이트 + Fumadocs 문서 |

#### `examples/` + `scripts/`

13개 프레임워크별 동작 예제(`with-express`, `with-hono`, `with-next` 등)와
`scripts/smoke.ts`가 Redis를 띄워 **13개 예제 전부 E2E 스모크 테스트**를 돌린다.

### 기능 요약 (REST 엔드포인트 39개 기준)

| 분류 | 기능 |
| --- | --- |
| 조회 | 큐 목록/카운트, 잡 상세(payload·attempts·스택트레이스), 로그, 실행 이력, 자유 텍스트 검색, 태그 필터 |
| 지표 | p50/p95 지연시간, 처리량 차트, 24시간 활동 버킷, 최장 실행 잡 |
| 조작 | 재시도 / 삭제 / 즉시승격(promote), 큐 일시정지·재개, 대량 재시도·삭제·승격, 완료·실패 잡 청소 |
| 스케줄러 | 반복 스케줄러 목록, **스케줄을 깨지 않고 지금 1회 실행** |
| Flows | FlowProducer DAG 시각화 (`@xyflow/react` + `dagre`), 플로우 생성 |
| 알림 | Grafana 스타일 Contact Point + Rule 모델 |

**알림 트리거 5종** (`packages/core/src/core/types.ts:60`)
`job_failed` · `job_stalled` · `retries_exhausted` · `failed_backlog` · `no_workers_with_backlog`
심각도 3단계(critical/warning/info), 큐·잡이름 필터, 쿨다운 지원.
컨택포인트 프리셋: **Slack / Discord / 일반 Webhook** (0.9.1에서 Discord 추가)

### 언제 쓰는가

BullMQ로 백그라운드 작업(이메일 발송, 이미지 리사이즈, PDF 생성, 결제 웹훅, 크롤링,
정기 배치, LLM 배치 추론 등)을 돌리는 상황에서 **"내 잡이 어디서 왜 죽었는지"** 를 눈으로 확인할 때.

- 장애 대응: 실패 스택트레이스 확인 → 수정 후 대량 재시도
- 배포 직후 모니터링: p95 급등, 실패율 추이 확인
- 운영 알림: "실패 50건 넘으면 Discord로 통보"
- 배치 검증: 새벽 정산 스케줄러를 낮에 1회만 실행해 확인
- 파이프라인 디버깅: 업로드 → 변환 → 썸네일 → 알림 DAG 중 끊긴 지점 추적

### 도움이 되는 지점

1. **즉시 실전 투입 가능한 도구** — CLI 한 줄로 관리자 대시보드 확보 (직접 만들면 2~3주)
2. **아키텍처 교과서** — "코어 1개 + 얇은 어댑터 N개", 웹표준 fetch 핸들러로 프레임워크 독립성 확보
3. **MIT 라이선스** — 포크·리브랜딩해 사내 툴/상용 제품에 투입 가능
4. **모던 스택 레퍼런스** — React 19, Tailwind 4, TanStack Router/Query, Hono, Bun, Turbo, Biome, Tauri 2, MCP SDK
5. **오픈소스 기여 입구** — 어댑터 1개가 30줄 수준이라 신규 프레임워크 어댑터 PR 난이도가 낮다

---

## 2. 쉬운 설명 — "햄버거 가게 주방" 비유

| 비유 | 실제 개념 |
| --- | --- |
| 주문서 1장 | **Job** (일거리 하나) |
| 주문서 꼬챙이 | **Queue** (일거리 줄) |
| 요리사 | **Worker** (일을 실제로 처리) |
| 주문서 보관함 | **Redis** (큐가 저장되는 곳) |
| 주방 시스템 자체 | **BullMQ** (Node.js 큐 라이브러리) |
| **주방 모니터 화면** | **Workbench** |

**큐를 쓰는 이유**: 회원가입 시 환영 이메일 발송이 5초 걸린다고 사용자를 5초 기다리게 할 수 없다.
그래서 "이메일 보내기" 주문서만 큐에 꽂고 사용자에겐 즉시 완료 응답을 준다.

**문제**: 그 큐는 Redis 안에 있어서 사람 눈에 보이지 않는다.
→ 몇 건 쌓였는지, 어떤 게 실패했는지, 워커가 다 죽었는데 대기만 쌓이는지 알 수 없다.

**해결**: Workbench가 그 주방을 웹 화면으로 보여준다.

| 주방 비유 | Workbench 화면 |
| --- | --- |
| 주문 현황판 | Overview (대기/처리중/완료/실패 개수) |
| 타버린 주문 영수증 | Job 상세 (실패 원인 에러 전문) |
| "이거 다시 만들어" | Retry 버튼 |
| "타버린 거 전부 다시" | Bulk Retry |
| 평균 조리시간 그래프 | Metrics (p50/p95, 처리량) |
| 매일 새벽 정산 예약 | Schedulers |
| "지금 한번 돌려봐" | Run now (예약 유지) |
| 세트메뉴 조립 순서도 | Flows (DAG) |
| "타이머 울리면 알림" | Alerts (Slack/Discord) |

### 설치가 쉬운 이유

별도 서버를 띄우지 않고 앱에 페이지 하나 추가하는 것과 같다.

```ts
app.use("/jobs", workbench({ queues: [emailQueue] }));
```

→ 서버 추가 비용 없음, 별도 배포 없음, **기존 앱의 인증/권한을 그대로 상속**(보안상 유리).

### 3가지 배포 형태

1. **내 앱에 마운트** — 라우트 한 줄 (권장)
2. **Docker 컨테이너로 분리** — `workbench-standalone` 이미지
3. **macOS 데스크톱 앱** — Redis 주소만 입력, 큐를 자동 발견(`discover.ts`)

### 반전 매력 — MCP

`packages/mcp`가 AI용 출입구다. Cursor/Claude에 연결하면 채팅으로 큐를 조작할 수 있다.

```
사용자: "email-send 큐 왜 막혔어?"
AI: workbench_get_metrics → "14:02에 p95 급등, 실패 143건. SMTP 타임아웃."
사용자: "최근 1시간 실패한 거 다 재시도해줘"
AI: workbench_list_runs + workbench_bulk_retry → "143건 재시도 완료"
```

읽기 도구엔 `readOnlyHint`, 쓰기 도구엔 `destructiveHint`가 붙어 클라이언트가 사전 확인을 띄운다.

### 레고 비유로 본 폴더 구조

```
core        = 레고 본체 (실제 기능 전부, 약 90%)
어댑터 13개  = 콘센트 어댑터 (프레임워크별 플러그 변환기, 각 30줄)
cli         = 자동 조립 로봇
mcp         = AI 리모컨
standalone  = 통째로 도커 박스
desktop     = macOS 앱 포장
web         = 홍보 웹사이트
examples    = 실제로 돌아가는 조립 설명서 13개
```

---

## 3. 질문 7개 정밀 답변

### ① 설치 및 사용법

**방법 A — CLI 자동 설치 (권장)**

```bash
npx @getworkbench/cli init
```

`packages/cli/src/lib/framework-detect.ts`가 프레임워크를 자동 감지 → 패키지 설치 →
마운트 코드 주입(Next.js는 라우트 파일 생성) → `.env.example` 작성 → Redis용 `docker-compose.yml` 제안.

**방법 B — 수동 (Express 예시)**

```bash
npm i @getworkbench/express bullmq
```

```ts
import express from "express";
import { Queue } from "bullmq";
import { workbench } from "@getworkbench/express";

const app = express();
const emailQueue = new Queue("email", { connection: { url: process.env.REDIS_URL! } });

app.use("/jobs", workbench({
  queues: [emailQueue],
  auth: { username: "admin", password: process.env.WB_PASS! }, // 운영 필수
  readonly: false,
  tags: ["userId", "tenantId"],   // job.data에서 필터 태그 추출
}));
app.listen(3000);
```

→ `http://localhost:3000/jobs` 접속.

> 주의: `koa`, `elysia`, `next`, `tanstack-start`, `astro`, `nuxt`, `h3` 어댑터는 **`basePath` 필수**
> (프레임워크가 마운트 prefix를 알려주지 않아 스스로 감지 불가).

**방법 C — Docker (앱 무수정)**

```bash
docker run --rm -p 3000:3000 \
  -e REDIS_URL=redis://host.docker.internal:6379 \
  -e QUEUE_NAMES=email,image \
  -e AUTH_USERNAME=admin -e AUTH_PASSWORD=secret \
  -e READONLY=false \
  ghcr.io/pontusab/workbench-standalone:latest
```

환경변수: `REDIS_URL` · `QUEUE_NAMES` · `PORT` · `BASE_PATH` · `AUTH_USERNAME` / `AUTH_PASSWORD` ·
`TITLE` · `LOGO_URL` · `READONLY` · `TAGS`. 헬스체크는 `GET /healthcheck`.

**방법 D — macOS 데스크톱 앱** — Redis URL만 입력, 큐 자동 발견, 배포할 인프라 0개.

**이 저장소 자체 개발 모드**

```bash
bun i                        # bun 1.3.11
bun run build
bun run typecheck
docker compose up -d redis   # Redis 먼저
bun run smoke                # 13개 예제 E2E 스모크
bun run dev                  # turbo 병렬 개발
```

버전 요구사항: Node 18+ (Elysia/Bun 어댑터는 Bun 1.1+), TS 4.x~5.x —
단 **`@getworkbench/hono`는 TS 5.0+ 필수**(Hono 4의 `.d.ts`가 `const` 타입 파라미터 사용).

### ② 플러그인인가, 스킬인가, MCP인가

**넷 중 하나가 아니라 여러 형태를 동시에 갖는다.**

| 형태 | 해당 | 설명 |
| --- | --- | --- |
| npm 라이브러리 | ✅ **주력** | 정체성의 약 90%. `@getworkbench/*` 16개 패키지 |
| 프레임워크 플러그인/미들웨어 | ✅ | Express 미들웨어, Fastify 플러그인, NestJS 모듈, Nuxt 서버 라우트 등 정식 플러그인 형태 |
| MCP 서버 | ✅ **일부** | `packages/mcp` 하나. 전체의 약 3%, 선택 기능 |
| Claude Skill | ❌ | `SKILL.md` 등이 없다. Skill은 프롬프트 지침 묶음이고 이건 실행 코드 |
| 독립 앱 | ✅ | Docker 이미지 + Tauri macOS 앱 |

정확한 표현: **"BullMQ 대시보드 라이브러리이고, 프레임워크 플러그인 형태로 마운트되며, MCP 서버를 부가 기능으로 제공하는 프로젝트."**

MCP는 `@modelcontextprotocol/sdk ^1.29.0` + **stdio** 방식의 얇은 프록시다.

```
Cursor / Claude Desktop / Zed / Continue ──stdio──▶ workbench-mcp ──HTTP──▶ Workbench 대시보드 ──▶ Redis
```

대시보드의 기존 auth와 `readonly` 플래그가 권한의 single source of truth이며,
MCP 자체는 별도 권한 체계를 두지 않는다.

**MCP 도구 23개** (`packages/mcp/src/tools.ts`)

- 조회(13): `get_overview` `list_queues` `get_quick_counts` `get_metrics` `get_activity` `list_jobs` `list_runs` `get_job` `search_jobs` `list_schedulers` `list_flows` `get_flow` `list_tag_values`
- 조작(10): `retry_job` `remove_job` `promote_job` `pause_queue` `resume_queue` `run_scheduler_now` `enqueue_job` `clean_jobs` `bulk_retry` `bulk_delete`

(모두 `workbench_` 접두사)

### ③ API 토큰이 필요한가

**LLM API 키(Anthropic/OpenAI 등)는 전혀 필요 없다.** 코드베이스에 LLM 호출이 없고,
MCP는 사용자의 에디터에 있는 모델을 사용하는 구조다.

| 항목 | 필수 | 설명 |
| --- | --- | --- |
| `REDIS_URL` | ✅ 필수 | 유일한 진짜 필수값. 로컬은 `redis://localhost:6379` |
| Basic Auth (id/pw) | ⚠️ 운영 강력권장 | 없으면 큐 조작 화면이 무인증 공개된다 |
| `WORKBENCH_TOKEN` | 선택 | MCP용 Bearer 토큰. 토큰 인식 리버스 프록시 뒤에서 사용 |
| Slack Webhook URL | 선택 | 알림용. `hooks.slack.com/services/...` 형식 검증 |
| Discord Webhook URL | 선택 | 알림용. `discord.com/api/webhooks/...` 형식 검증 |
| Tauri 서명 키 | 선택 | macOS 앱을 직접 빌드·배포할 때만 |

보안 설계상 좋은 점: 웹훅 URL은 서버에만 저장하고 API 응답에서는 마스킹한다(`toPublicContactPoint`).
`readonly: true`를 주면 모든 쓰기 동작이 차단되므로, 감독 없는 AI 에이전트를 붙일 때 안전장치로 쓸 수 있다.

### ④ 왜 GitHub에서 유명한가

> 참고: 본 분석 세션은 포크 저장소만 접근 가능했으므로 **원본 저장소의 스타 수는 직접 확인하지 않았다.**
> 아래는 코드·문서·CHANGELOG에서 읽어낸 근거 기반 분석이다.

1. **제작자 신뢰도** — `pontusab` = Pontus Abrahamsson, **Midday**(midday.ai) 창업자.
   `MIGRATION.md`가 결정적 증거로, *midday/apps/worker에 벤더링해 쓰던 workbench를 npm 패키지로 교체하는 diff*가 담겨 있다.
   → **실제 프로덕션 내부 도구를 떼어내 오픈소스화한 것.**
2. **명확한 빈틈 공략** — BullMQ 대시보드는 사실상 `bull-board` 독점이었다. Workbench는 정면으로
   drop-in 대체를 선언하고 README에 패키지 대조표(`@bull-board/express` → `@getworkbench/express`)를 실었으며,
   CLI 한 줄로 교체하게 해 마이그레이션 마찰을 제거했다.
3. **어댑터 물량** — bull-board 7개 대비 **13개**(AdonisJS, Next, Nuxt, Astro, TanStack Start, Bun, h3 추가).
   "내 프레임워크가 목록에 있다"가 곧 스타 동기다.
4. **npm SEO / 마케팅** — CHANGELOG 0.7.1에 *"npm discoverability"* 항목이 명시돼 있다.
   전 패키지에 `bull-board`, `bull-board-alternative`, `@bull-board` 키워드를 심고,
   비교 블로그 글과 `llms.txt`까지 준비해 AI 검색까지 겨냥했다.
5. **AI 시대 타이밍** — MCP 서버를 기본 제공하는 운영 도구는 희소하다.
   "에디터 채팅으로 프로덕션 큐 고치기"는 그 자체로 화제성이 있다.
6. **관리 품질 신호** — MIT · Keep a Changelog 규격 CHANGELOG · CONTRIBUTING · SECURITY ·
   이슈/PR 템플릿 · CI(lint + typecheck + build) · 13개 예제 E2E 스모크 ·
   도커 이미지 자동 퍼블리시 · 데스크톱 릴리스 워크플로.
7. **비주얼** — `hero.png` 히어로 이미지, 다크모드 UI, DAG 그래프, `cmdk` 커맨드 팔레트.

### ⑤ 로컬 에이전트 구축에 도움이 되는가

**레벨 1 — 지금 바로 도구로 사용**
로컬 에이전트가 백그라운드 작업(크롤링, 파일 변환, LLM 배치 추론)을 BullMQ로 돌린다면,
MCP를 연결하는 것만으로 에이전트가 자기 작업 큐를 관측·복구할 수 있다(자기 관측성 확보).

```json
{ "mcpServers": { "workbench": {
  "command": "npx", "args": ["-y", "@getworkbench/mcp"],
  "env": { "WORKBENCH_URL": "http://localhost:3000/jobs" }
}}}
```

> 무인 에이전트에는 대시보드를 `readonly: true`로 띄운다. 쓰기 도구가 403을 받고 명확한 에러를 반환한다.

**레벨 2 — MCP 서버 작성법 학습**
`packages/mcp`는 1,059줄짜리 완결된 MCP 구현 예제다.

- `index.ts` (61줄) 서버 부팅 + stdio 연결
- `client.ts` (247줄) HTTP 호출 + 인증 + 에러 변환
- `tools.ts` (751줄) 도구 23개 정의 (Zod 스키마 + 힌트)

배울 패턴: 도구 네이밍 접두사로 충돌 방지 · 읽기/쓰기 힌트 분리 · Zod 입력 검증 ·
에러를 LLM이 읽을 수 있는 문장으로 번역.

**레벨 3 — 에이전트 인프라의 뼈대로 활용**
로컬 AI 에이전트의 핵심 난제는 "긴 작업을 유실 없이 돌리기"다. BullMQ + Workbench 조합이 이를 해결한다.

| 에이전트 고민 | BullMQ의 해답 | Workbench가 보여주는 것 |
| --- | --- | --- |
| 작업 중간 크래시 | 재시도 + 백오프 | 실패 잡 + 스택트레이스 |
| 다단계 파이프라인 | FlowProducer DAG | Flows 그래프 시각화 |
| 동시 실행 제어 | concurrency / rate limit | 워커 수, 처리량 차트 |
| 정기 실행 | Repeatable job | Schedulers + 즉시 1회 실행 |
| 비용 폭주 감시 | QueueEvents | Alerts → Slack/Discord |

즉 **"두뇌는 LLM, 신경계는 BullMQ, 눈은 Workbench"** 구조를 만들 수 있다.

> 한계: 이것은 에이전트 프레임워크가 아니다. 계획·추론·툴 콜링은 제공하지 않는 **큐 관측/조작 레이어**다.

### ⑥ 수익화 아이디어

→ 4장에서 상세히 다룬다.

### ⑦ React나 PHP로 만들 수 있는가

**React — 이미 React다.** `packages/core/src/ui/` 전체가 React 19 기반.

```
React 19 + TypeScript
TanStack Router (라우팅) + TanStack Query (서버 상태)
Tailwind CSS 4 + Radix UI + shadcn 스타일 (components/ui/ 20개)
Recharts (차트) / @xyflow/react + dagre (DAG 자동 레이아웃)
framer-motion (애니메이션) / cmdk (커맨드 팔레트) / sonner (토스트)
lucide-react (아이콘) / date-fns
```

가능한 작업:

- **UI 커스터마이징** — `src/ui/pages/` 수정. 로고/테마/한국어화 가능(`title`, `logo` 옵션도 존재)
- **페이지 추가** — `router.tsx`에 라우트 추가
- **UI만 임포트** — `@getworkbench/core/ui`로 export되어 있어 **기존 React 앱 안에 컴포넌트로 임베드 가능**
  (`apps/desktop`이 실제로 이 방식을 사용)
- **프론트만 새로 구현** — REST 39개가 정리되어 있어 가능

**PHP — 갈래를 나눠야 한다.**

| 목표 | 가능 | 난이도 | 설명 |
| --- | --- | --- | --- |
| A. PHP에서 Workbench API 호출 | ✅ | ⭐ | Laravel `Http::get('.../api/overview')` — 단순 REST 호출 |
| B. Laravel 안에 대시보드 임베드 | ⚠️ 부분 | ⭐⭐ | 코어가 Node 전용. Node 프로세스를 사이드카로 띄우고 PHP가 리버스 프록시 |
| C. PHP 어댑터 패키지 제작 | ❌ | — | 어댑터는 Node 코어를 감싸는 구조라 PHP에서 불가 |
| D. PHP로 완전 재구현 | 🔶 | ⭐⭐⭐⭐⭐ | 아래 주의사항 참고 |

**D의 난점**: Workbench는 단순히 Redis를 읽는 게 아니라 **BullMQ의 데이터 규약**을 읽는다.
BullMQ는 원자성을 위해 상태 전이를 **Lua 스크립트**로 처리하고,
`bull:<queue>:wait` / `:active` / `:failed` / `:meta` / `:events` 등 키 구조와 ZSET·LIST·HASH 조합을 쓴다.
PHP 재구현은 이를 전부 역공학해야 하고, BullMQ 버전 업 때마다 깨질 위험이 있다.

```
읽기(카운트, 잡 조회)      → predis로 가능, 중간 난이도
쓰기(retry/promote/clean)  → Lua 원자성 재현 필요, 매우 어려움
```

**현실적 권고**: PHP 생태계에는 이미 동일 포지션의 완성품이 있다 —
**Laravel Horizon**(Redis 큐 공식 대시보드), **Laravel Pulse**.
"PHP로 Workbench 만들기"보다 **"Workbench의 설계를 배워 Horizon을 커스터마이징"** 하는 편이 효율적이다.

성장 루트로는 React 쪽이 최적이다: ① UI 한국어화 & 테마 → ② 페이지 추가 → ③ 어댑터 기여(약 30줄).

---

## 4. 수익화 아이디어 상세

> **라이선스 체크**: MIT는 상업적 이용·수정·재배포·클로즈드소스화를 모두 허용하되,
> **배포물에 원본 저작권 표시와 MIT 전문을 포함**해야 한다.
> "Workbench" 이름/로고는 상표 이슈가 있으므로 재판매 시 **반드시 리브랜딩**하고, 원저작자 사칭은 금지.

### 티어 1 — 혼자서도 가능 (초기비용 거의 0)

#### 1. SaaS 관제센터 (Queue Observability Cloud) — 잠재력 최상

여러 Redis / 여러 환경을 한 화면에 모아보는 멀티테넌트 SaaS.

```
대시보드
 ├─ 고객사 A ─ prod Redis (에이전트가 메트릭 push)
 ├─ 고객사 A ─ staging Redis
 └─ 고객사 B ─ prod Redis
+ 90일 히스토리 보관 (Workbench는 Redis에 있는 것만 본다 = 휘발성)
+ 팀 권한(RBAC), SSO, 감사 로그, 온콜 연동(PagerDuty)
```

오픈소스가 의도적으로 하지 않는 영역(장기 히스토리, 멀티테넌시, 팀 권한)이 곧 기업이 지불하는 지점이다.
가격: Free(큐 3개) / Team $29~49/월 / Business $199~/월 / Enterprise 협의.
난이도 ⭐⭐⭐⭐ · 수익 잠재력 최상.

#### 2. 프리미엄 엔터프라이즈 플러그인 (Open-core) — 가장 현실적

| 유료 기능 | 지불 이유 |
| --- | --- |
| SSO/SAML + RBAC | 현재 Basic Auth 1개뿐. 엔터프라이즈 필수 |
| 감사 로그 | "누가 결제 잡 30건을 삭제했는가" — 금융/의료는 법적 요구 |
| 장기 메트릭 보관 | Postgres/ClickHouse 적재, 90일+ 추이 |
| AI 에러 트리아지 | 실패 스택트레이스 자동 클러스터링 + 원인 요약 |
| 고급 알림 | 이상 탐지, 온콜 스케줄, 에스컬레이션 |
| PII 마스킹 | job.data 민감정보 자동 마스킹 (GDPR) |

가격: 개발자당 연 $99~299, 또는 사이트 라이선스 $2,000~10,000/년.
참고 선례: Sidekiq Pro. 난이도 ⭐⭐⭐ · 수익 잠재력 최상.

#### 3. 한국 시장 특화 버전 — 최적의 시작점

글로벌 오픈소스의 가장 큰 빈틈은 한국화다.

- 완전 한국어 UI + 한국어 문서/블로그
- **카카오톡 알림톡 / 네이버웍스 / 잔디(JANDI) / 두레이** 컨택포인트 추가 (현재 Slack/Discord만)
- 네이버클라우드 / 카카오클라우드 / NHN클라우드 배포 가이드
- 한국 시간대·공휴일 기준 스케줄러 프리셋

수익 경로: 무료 배포로 인지도 확보 → **구축 컨설팅 + 유지보수 계약**.
난이도 ⭐⭐ · 수익 잠재력 높음.

#### 4. 기술 콘텐츠 / 교육 — 시작 속도 최상

- 인프런·유데미 강의: "Node.js 백그라운드 작업 완전정복 — BullMQ + Workbench + MCP" (₩55,000~99,000)
- 유료 뉴스레터/전자책: "큐 아키텍처 실전" (Gumroad $29)
- 유튜브: "AI가 내 프로덕션 큐를 고쳐준다 (MCP 실습)"
- 기업 세미나: 반일 워크숍 200~500만원

난이도 ⭐ · 수익 잠재력 중상.

### 티어 2 — 서비스 / 수주형

#### 5. 큐 마이그레이션 & 구축 대행

```
기본 패키지 (₩150만~300만)    : 설치, 어댑터 연동, 알림 세팅, 문서화
심화 패키지 (₩500만~1,000만)  : 큐 아키텍처 재설계, DAG 파이프라인, 부하 튜닝
유지보수   (월 ₩50만~150만)   : 온콜 대응, 업그레이드, 리포트
```

설득 포인트: "실패 잡 방치로 인한 매출 손실 vs 구축비" ROI 계산.
난이도 ⭐⭐ · 수익 높음(즉시 현금화).

#### 6. 관리형 BullMQ 호스팅

Redis + 워커 + 대시보드를 원클릭 배포 상품으로 제공. 프로비저닝 자동화, 오토스케일, 백업, 모니터링 포함.
가격 $19~199/월. 난이도 ⭐⭐⭐⭐⭐(인프라 운영 부담) · 수익 중상.

#### 7. 화이트라벨 리셀링

SaaS 기업이 자기 고객에게 보여줄 관리자 화면으로 리브랜딩 임베드.
MIT라서 합법(저작권 고지 준수 전제)이고, `title`/`logo` 옵션이 이미 있어 리브랜딩이 쉽다.
가격: 초기 구축 ₩500만 + 연 라이선스 ₩300만. 난이도 ⭐⭐⭐ · 수익 중상.

### 티어 3 — AI / MCP 특화 (블루오션)

#### 8. AI 큐 닥터 (자율 운영 에이전트) — 차별화 1위

MCP를 활용해 큐를 24시간 자율 감시·복구하는 AI 에이전트를 SaaS로 제공.

```
1. 실패 급증 감지 → 2. 스택트레이스 자동 클러스터링
3. 원인 분류 (일시적 네트워크 / 코드 버그 / 외부 API 장애)
4. 일시적이면 자동 재시도 (허용된 권한 범위 내에서만)
5. 코드 버그면 GitHub Issue 자동 생성 + 의심 커밋 링크
6. Slack/카카오로 처리 리포트 발송
```

2026년 기준 "관측"은 레드오션이지만 **"자율 복구"는 블루오션**이다.
MCP 도구 23개가 이미 액션 레이어를 제공한다.
가격: 감시 큐당 $99~499/월(사람 온콜 인건비 대비 저렴하게 인식됨).
난이도 ⭐⭐⭐⭐ · 수익 잠재력 최상.

#### 9. MCP 도구 마켓 / 템플릿 판매

`packages/mcp` 패턴을 템플릿화해 "내 SaaS를 MCP로 여는 스타터킷" 판매.
Zod 스키마 자동생성, 인증, readonly 가드, 에러 번역 포함. 가격 $49~199.
난이도 ⭐⭐ · 수익 중 · 부수효과로 개발자 브랜딩 상승.

#### 10. 큐 비용 최적화 리포트

LLM 배치 추론을 큐로 돌리는 기업 대상. 잡별 토큰·비용 태깅, 낭비 패턴 탐지
(무한 재시도로 인한 API 비용 폭주 등), 절감 리포트 제공.
가격: 절감액의 10~20% 성과보수 또는 $199/월. 난이도 ⭐⭐⭐ · 수익 높음.

### 추천 로드맵

```
1~2개월차  : 아이디어 3(한국화) + 4(콘텐츠)
             한국어 UI PR + 카카오 알림톡 컨택포인트 오픈소스 공개
             "BullMQ 완전정복" 블로그 시리즈로 인지도 축적
             기대: 콘텐츠 소액 수익 + 포트폴리오 확보

3~6개월차  : 아이디어 5(구축 대행)
             한국화 결과물이 곧 영업 자료. 2~3개사 수주
             기대: ₩500만~2,000만 (즉시 현금화)

6~12개월차 : 아이디어 8(AI 큐 닥터) + 2(엔터프라이즈 플러그인)
             대행에서 얻은 현장 데이터로 자율복구 에이전트 설계, MRR 모델 진입
             기대: MRR $2,000~10,000
```

근거: ① 한국화는 난이도가 가장 낮고 차별화가 가장 크다 ② 대행은 현금과 고객 인사이트를 동시에 준다
③ SaaS는 그 인사이트 없이 만들면 실패 확률이 매우 높다.
즉 **오픈소스 기여 → 신뢰 → 서비스 → 제품** 순서.

### 리스크와 대응

| 리스크 | 대응 |
| --- | --- |
| 원저작자가 유료 클라우드를 직접 출시 | 한국 시장 특화로 포지션 차별화 |
| BullMQ 생태계 규모 한계 (Node 한정) | Celery(Python) / Sidekiq(Ruby)까지 확장 가능한 설계 |
| MIT라서 경쟁자도 동일하게 복제 가능 | 실질 방어선은 도메인 전문성 + 고객 관계 |
| Redis만 보므로 히스토리가 휘발됨 | 그 자체가 유료 기능(장기 보관)의 기회 |

---

## 5. 참고 — 근거 파일 경로

| 주장 | 근거 위치 |
| --- | --- |
| 어댑터가 얇은 래퍼 | `packages/hono/src/index.ts:29` |
| REST 엔드포인트 39개 | `packages/core/src/api/handlers.ts` |
| 알림 트리거 5종 / 심각도 3단계 | `packages/core/src/core/types.ts:60` |
| Slack/Discord 웹훅 URL 검증 | `packages/core/src/core/alert-destinations.ts` |
| 큐 자동 발견(Redis 스캔) | `packages/core/src/core/discover.ts` |
| MCP 도구 23개 | `packages/mcp/src/tools.ts` |
| MCP 아키텍처 / 보안 주의 | `packages/mcp/README.md` |
| Tauri 데스크톱 아키텍처 | `apps/desktop/README.md` |
| Docker 환경변수 목록 | `apps/standalone/README.md` |
| npm SEO 전략 명시 | `CHANGELOG.md` (0.7.1 항목) |
| Midday 내부 도구 출신 증거 | `MIGRATION.md` |
| CI 구성(lint/typecheck/build) | `.github/workflows/ci.yml` |
