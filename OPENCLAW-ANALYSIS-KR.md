# OpenClaw 전수조사 & 수익화 분석 정리 🦞

> 작성일: 2026-09-29
> 분석 대상 레포: **https://github.com/bmshin94/openclaw**
> 원본 업스트림: **https://github.com/openclaw/openclaw**
> 작성 브랜치: `claude/keen-heisenberg-6f5q0d`

---

## 📑 목차

1. [먼저 짚고 갈 것 — CLAUDE.md 요약문 정정](#1-먼저-짚고-갈-것--claudemd-요약문-정정)
2. [프로젝트 정체 & 기본 정보](#2-프로젝트-정체--기본-정보)
3. [아키텍처](#3-아키텍처)
4. [폴더별 전수조사 결과](#4-폴더별-전수조사-결과)
5. [확장 3단 구조 — 툴 / 스킬 / 플러그인](#5-확장-3단-구조--툴--스킬--플러그인)
6. [설치 및 사용법](#6-설치-및-사용법)
7. [플러그인? 스킬? MCP?](#7-플러그인-스킬-mcp)
8. [API 토큰이 필요한가?](#8-api-토큰이-필요한가)
9. [왜 깃허브에서 유명한가](#9-왜-깃허브에서-유명한가)
10. [로컬 에이전트 구축에 도움이 되는가](#10-로컬-에이전트-구축에-도움이-되는가)
11. [React / PHP 로 만들 수 있는가](#11-react--php-로-만들-수-있는가)
12. [수익화 아이디어 상세](#12-수익화-아이디어-상세)
13. [실행 로드맵](#13-실행-로드맵)
14. [참고 문서 링크 모음](#14-참고-문서-링크-모음)

---

## 1. 먼저 짚고 갈 것 — CLAUDE.md 요약문 정정

레포 루트 `CLAUDE.md` 상단에 자동 생성된 요약문이 있으나 **실제 코드베이스와 일치하지 않는 부분**이 있다.

| 자동 생성 요약문 주장 | 실제 확인 결과 |
|---|---|
| "39만 스타 자랑" | 레포 내 근거 없음. **검증 불가 / 과장 가능성 높음** |
| "직접 클릭하고 실행하는 만능 실행 에이전트" | 컴퓨터 유즈는 **일부 기능**. 본질은 멀티채널 AI 게이트웨이 |
| "복잡한 설정 없이" | `VISION.md` 는 **"terminal-first by design"** 을 명시. 설정은 명시적/수동이 설계 의도 |

**공식 설명 (`package.json`):**

> "Multi-channel AI gateway with extensible messaging integrations"

**README.md 첫 문장:**

> "OpenClaw is an open-source AI assistant that runs on your own computer and meets you in the channels you already use: Discord, iMessage, Slack, Teams, Telegram, WhatsApp, and 20+ more"

---

## 2. 프로젝트 정체 & 기본 정보

### 한 줄 정의

> **내 PC에 상주하는 "AI 비서 본체 서버". 원래 쓰던 메신저(카톡·텔레그램·디스코드·슬랙 등)로 말을 걸면 내 컴퓨터에서 실제 작업을 수행한다.**

### 기본 신원

| 항목 | 값 |
|---|---|
| 패키지명 | `openclaw` (전역 CLI) |
| 버전 | `2026.9.4` (날짜 기반 버저닝) |
| 라이선스 | **MIT** (상업 이용 가능) |
| 저작권자 | OpenClaw Foundation (독립 501(c)(3) 비영리) |
| 주 언어 | TypeScript (ESM, strict) + Swift/Kotlin(앱) + Rust(crates) |
| 런타임 | Node.js 24.16+ 또는 26.1+ (26 권장) |
| 상태 저장소 | **SQLite** (JSON/JSONL 사이드카 저장 금지가 내부 규칙) |
| DB 접근 | Kysely (raw SQL 은 스키마/마이그레이션/부트스트랩 한정) |
| 이름 변천사 | Warelay → Clawdbot → Moltbot → **OpenClaw** |
| 마스코트 | Molty (우주 랍스터 🦞) |
| 원작자 | Peter Steinberger (steipete) + 커뮤니티 |
| 후원사 | Amazon, OpenAI, Red Hat, NVIDIA, Vercel, GitHub, Convex, 미시간대 등 |
| 유료 티어 | **없음** ("no paid tier, hosted service, or token") |

### 규모 실측치

```
TypeScript/TSX 파일   : 34,794 개
테스트 파일(*.test.ts): 16,202 개
문서(.md)             :  1,307 개
플러그인(extensions/) :    167 개
번들 스킬(skills/)    :     52 개
src/ 총 라인 수       : 약 604 만 줄
```

---

## 3. 아키텍처

`docs/concepts/architecture.md` 기준.

```
   ┌─────────────── 사용자가 쓰는 메신저 ───────────────┐
   │ WhatsApp  Telegram  Slack  Discord  iMessage      │
   │ Teams  Signal  Matrix  IRC  LINE  Google Chat ... │
   └────────────────────────┬──────────────────────────┘
                            │  Channels (= 플러그인)
                            ▼
        ╔═══════════════════════════════════════════╗
        ║   GATEWAY  (로컬 데몬 · 단 하나만 존재)    ║
        ║   WebSocket @ 127.0.0.1:18789 (기본)      ║
        ║ ───────────────────────────────────────── ║
        ║  · 모든 메신저 연결 독점 소유              ║
        ║  · 세션 / 메모리 / 권한 / 승인 정책        ║
        ║  · 에이전트 루프 + 툴 디스패치             ║
        ║  · SQLite 상태 저장                        ║
        ║  · HTTP 서버 (/__openclaw__/canvas, /a2ui) ║
        ╚═══╤═══════════════╤═══════════════╤════════╝
            │               │               │
   ┌────────▼──────┐  ┌─────▼──────┐  ┌────▼─────────────┐
   │ 조작 클라이언트│  │ 모델 공급자 │  │ Nodes (디바이스) │
   │ · Control UI  │  │ Anthropic   │  │ macOS / iOS      │
   │   (Lit, 웹)   │  │ OpenAI      │  │ Android / Win    │
   │ · CLI         │  │ Google      │  │ Linux            │
   │ · TUI         │  │ Ollama(로컬)│  │ → camera.*       │
   │ · MCP 클라이언트│ │ ... 60+     │  │   screen.record  │
   │ · 자동화      │  │             │  │   location.get   │
   └───────────────┘  └────────────┘  └──────────────────┘
```

### 프로토콜 요약

- 전송: WebSocket, JSON 텍스트 프레임
- **첫 프레임은 반드시 `connect`**
- 요청: `{type:"req", id, method, params}` → `{type:"res", id, ok, payload|error}`
- 이벤트: `{type:"event", event, payload, seq?, stateVersion?}`
- `send` / `agent` 등 **부수효과 메서드는 멱등키(idempotency key) 필수**
- 모든 connect 는 `connect.challenge` 논스에 서명 (payload v3 는 platform/deviceFamily 도 바인딩)
- 노드는 `role: "node"` + caps/commands/permissions 를 connect 에 포함
- 새 디바이스 ID 는 **페어링 승인** 필요 → 이후 device token 발급
- 로컬 루프백은 자동 승인 가능, **비로컬은 항상 명시 승인**

### 프라이버시 기본값

- 기본적으로 OpenClaw 자체가 외부로 보내는 것은 **하루 1회 버전 체크뿐**
- 익명 기능 통계는 **옵트인** (기본 미선택)
- `update.checkOnStart: false` 로 둘 다 차단
- 상태 / 메모리 / 자격증명은 **전부 사용자 하드웨어**

---

## 4. 폴더별 전수조사 결과

| 폴더 / 파일 | 정체 | 비고 |
|---|---|---|
| `src/` | **코어 본체** (79개 서브디렉터리) | `gateway/` `agents/` `channels/` `sessions/` `memory/` `mcp/` `cli/` `commands/` `tui/` `skills/` `cron/` `security/` `secrets/` `context-engine/` `routing/` `fleet/` `claws/` 등 |
| `extensions/` | **플러그인 167개** | 채널(discord, slack, telegram, whatsapp, imessage, matrix, line, feishu, msteams, signal, irc, nostr...) / 모델공급자(anthropic, openai, google, ollama, groq, deepseek, mistral, xai, bedrock, vertex...) / 기능(browser, canvas, memory-*, webhooks, vault, policy, diffs, crabbox, workboard, qa-lab) |
| `skills/` | **번들 스킬 52개** (`SKILL.md`) | 1password, notion, obsidian, spotify-player, trello, weather, github, gh-issues, apple-notes, apple-reminders, peekaboo(스크린샷), tmux, summarize, visualize, diagram-maker, clawhub, coding-agent 등 |
| `packages/` | **재사용 라이브러리 25개** | ⭐`gateway-client`(WS 참조 클라이언트) ⭐`gateway-protocol`(프로토콜 타입) ⭐`plugin-sdk` `sdk` `agent-core` `llm-core` `media-core` `net-policy` `model-catalog-core` `retry` `tool-call-repair` `workboard-contract` |
| `apps/` | **네이티브 앱** | `macos`(Swift) `ios` `android`(Kotlin) `windows` `linux` `mobile` `shared` `macos-mlx-tts` `swabble` |
| `ui/` | **Control UI (웹 대시보드)** | ⚠️ **React 아님 → Lit(Web Components) + Vite**. CodeMirror, novnc, ghostty-web, markdown-it, mermaid 탑재 |
| `crates/` | **Rust 코드** | `openclaw-gateway-client`, `openclaw-node-host` |
| `docs/` | **문서 1,307개** | `concepts/` `gateway/` `channels/` `providers/` `tools/` `cli/` `platforms/` `plugins/` `reference/` `help/` `security/` |
| `scripts/` | 개발 / 릴리스 자동화 | 빌드, 테스트 러너, 린트 레인, 릴리스, `check-changed.mjs`, `run-vitest.mjs` |
| `test/`, `qa/` | 통합 · E2E · QA 랩 | `qa-lab`, `qa-channel` 시나리오 |
| `custodian-skills/` | 유지보수용 에이전트 스킬 | `add-model-provider` `configure-channel` `diagnose-gateway` `cloud-image-bake` |
| `.agents/skills/` | **레포 개발용 내부 스킬** | `.claude/skills` 가 심볼릭 링크로 연결. Claude Code 가 이 레포 작업할 때 쓰는 워크플로 |
| `security/` | 보안 정책 / 시크릿 스캐닝 | `SECURITY.md` 가 36KB |
| `config/` | 린트 / 포맷 / 타입체크 설정 | oxlint, oxfmt, stylelint, swiftlint, markdownlint |
| `Dockerfile`(25KB) `docker-compose.yml` `fly.toml` `render.yaml` `deploy/` | **배포 경로** | Docker / Fly.io / Render 지원. 이미지 `ghcr.io/openclaw/openclaw:latest` |
| `appcast.xml`(334KB) | macOS 앱 자동 업데이트 피드 (Sparkle) | |
| `taxonomy.yaml`(719KB) | 프로젝트 전체 분류 체계 | |
| `.crabbox.yaml` | **Crabbox = 샌드박스 격리 설정** | 신뢰 못 하는 코드 실행용 |
| `AGENTS.md`(22KB) | ⭐ **AI 에이전트용 코드베이스 규칙서** | `CLAUDE.md` / `GEMINI.md` 가 대응 |
| `VISION.md` | 제품 범위 / 로드맵 / 머지 거절 목록 | |
| `pnpm-workspace.yaml` | pnpm 모노레포 정의 | **`npm install` 미지원** |

---

## 5. 확장 3단 구조 — 툴 / 스킬 / 플러그인

`docs/tools/index.md` 기준.

| 구분 | 정의 | 비유 | 예시 |
|---|---|---|---|
| **Tool** | AI 가 호출하는 타입 있는 함수. 모델에 function definition 으로 전달 | 손과 도구 | `exec` `read` `write` `edit` `apply_patch` `browser` `web_search` `web_fetch` `message` `ask_user` `secrets` `image_generate` |
| **Skill** | `SKILL.md` 지침 팩. 에이전트 프롬프트에 주입 | 업무 매뉴얼 | "노션 정리법", "1password 사용법" |
| **Plugin** | 코드 + 자격증명 + 라이프사이클 + 매니페스트 | 설비 교체 | 채널 추가, 모델 공급자 추가, 훅 추가 |

### 내장 툴 카테고리

| 분류 | 툴 |
|---|---|
| 런타임 | `exec` `process` `terminal` `code_execution` |
| 파일 | `read` `write` `edit` `apply_patch` |
| 웹 | `web_search` `x_search` `web_fetch` |
| 사람 입력 | `ask_user` `secrets` |
| 브라우저 | `browser` (CDP 기반) |
| 미디어 | `image_generate` `music` `video` `tts` `pdf` |
| 멀티에이전트 | `subagents` `swarm` `agent_send` |
| 디바이스(노드) | `camera.*` `screen.record` `location.get` `canvas.*` |
| 메타 | `tool_search` `code_mode` `steer` `thinking` |

### 설계 철학 (`VISION.md` / `AGENTS.md`)

> **"작은 코어, 강력한 플러그인"**
>
> 코어는 호출마다 세금을 낸다. 코어 툴/프롬프트 한 줄은 모든 운영자의 모든 모델 요청에 매번 실려간다.
> 반면 플러그인 · 스킬 · 채널 · 앱은 그런 세금이 없으므로 계속 늘리고 싶어한다.

**새 기능 추가 시 우선순위 (AGENTS.md):**

1. 기존 소유자 확장 또는 기존 커맨드/스킬/플러그인/통합 사용
2. 기존 플러그인 계약 사용 (스킬·MCP·설정이면 **번들 플러그인**, 런타임 훅·공급자·채널·툴이 필요하면 **코드 플러그인**)
3. 계약이 없으면 **좁은 범용 코어/SDK 능력**을 정의하고 기존 구현·호출자를 함께 이전
4. 진짜 근본적이고 기존 확장점으로 표현 불가할 때만 **범용 코어 표면** 추가

**AGENTS.md 3대 설계 원칙:**

1. **One owner per responsibility** — 한 책임에 한 소유자. 중복 소유자 금지
2. **Small core, capable plugins** — 코어 추가는 지속적 컨텍스트 비용
3. **Stable conversation context** — 과거 컨텍스트 재구축은 프롬프트 프리픽스 재사용을 깬다

---

## 6. 설치 및 사용법

### 요건

- **Node.js 24.16+ 또는 26.1+** (26 권장) — `node --version` 확인
- 기존 Claude Code / Codex CLI 로그인 **또는** 공급자 API 키 (온보딩이 재사용 가능)

### 방법 A — 맛보기 (1줄)

```bash
npx openclaw@latest
```

기존 Claude Code / Codex CLI 로그인이나 API 키를 자동 탐지하고, 실제 completion 으로 검증한 뒤 설정 저장 + 웹 대시보드를 연다.
Ctrl+C 로 종료되지만 설정은 유지된다.

### 방법 B — 정식 설치 (권장)

```bash
# macOS / Linux / WSL2
curl -fsSL https://openclaw.ai/install.sh | bash
```

```powershell
# Windows PowerShell
iwr -useb https://openclaw.ai/install.ps1 | iex
```

```bash
# Node.js 를 직접 관리하는 경우
npm install -g openclaw@latest --allow-scripts=openclaw
# ⚠️ --allow-scripts 는 npm 12 또는 npm 11.16+ 에서만. 11.15 이하는 생략
```

### 방법 C — 소스 빌드

```bash
git clone https://github.com/bmshin94/openclaw.git
cd openclaw
pnpm install      # ⚠️ npm install 미지원 (pnpm 워크스페이스)
pnpm build
pnpm ui:build
```

> 소스 체크아웃에서 CLI 를 돌릴 때는 `pnpm openclaw ...` 또는 `pnpm dev` 를 쓴다.
> `node --import tsx src/index.ts` 같은 직접 실행은 지원되지 않는다 (빌드 신선도/프로세스 셋업을 래퍼가 소유).

### 방법 D — Docker / 클라우드

- `Dockerfile`, `docker-compose.yml`
- `fly.toml` (Fly.io), `render.yaml` (Render)
- 이미지: `ghcr.io/openclaw/openclaw:latest`

### 설치 후 필수 명령

```bash
openclaw onboard --install-daemon   # 온보딩 (모델 인증 + 워크스페이스 + 게이트웨이)
openclaw gateway status             # 상태 확인
openclaw dashboard                 # 웹 Control UI 열기
openclaw gateway install           # 백그라운드 상주 서비스 등록
openclaw                           # TUI
```

### 주요 CLI 서브커맨드

| 명령 | 용도 |
|---|---|
| `openclaw onboard` / `configure` | 최초 설정 / 설정 변경 |
| `openclaw doctor --fix` | **문제 진단 + 자동 수리 (설정 마이그레이션 포함)** |
| `openclaw gateway {start,stop,status,install}` | 게이트웨이 생명주기 |
| `openclaw channels add <채널>` | 메신저 연결 |
| `openclaw pairing approve <채널> <코드>` | 낯선 발신자 페어링 승인 |
| `openclaw models` | 모델 목록 / 선택 |
| `openclaw plugins` / `skills` | 플러그인 / 스킬 관리 |
| `openclaw mcp serve` | **OpenClaw 를 MCP 서버로 노출** |
| `openclaw mcp add/set/probe/status` | 외부 MCP 서버 등록/점검 |
| `openclaw cron` | 스케줄 작업 |
| `openclaw sessions` / `resume` / `transcripts` | 세션 관리 |
| `openclaw memory` | 기억 관리 |
| `openclaw secrets` / `vault` | 비밀값 관리 |
| `openclaw backup` | 백업 |
| `openclaw sandbox` | 샌드박스 |
| `openclaw fleet` | **멀티테넌트 격리 셀 (실험적)** |
| `openclaw workboard` | 작업 보드 / 워커 디스패치 |
| `openclaw nodes` / `devices` | 디바이스 노드 관리 |
| `openclaw acp` | 코딩 하네스 세션 호스팅 |
| `openclaw update` / `logs` / `health` | 업데이트 / 로그 / 헬스 |

### 설치 직후 반드시 읽을 보안 문서

- `docs/gateway/security` — 보안 가이드
- `docs/gateway/security/exposure-runbook` — 외부 노출 런북
- `docs/gateway/sandboxing` — 샌드박싱

> **경고:** 샌드박싱을 설정하지 않으면 메인 세션의 툴은 **호스트에서 그대로 실행된다.**
> 또한 인바운드 메시지는 신뢰할 수 없는 입력으로 취급해야 하며, DM 가능 채널은 낯선 발신자에 대해 기본적으로 페어링을 요구한다.

---

## 7. 플러그인? 스킬? MCP?

### 정답: 셋 다 아니다. **OpenClaw 는 "호스트 플랫폼"** 이다.

플러그인 · 스킬 · MCP 는 "꽂는 것"이고, OpenClaw 는 "꽂히는 곳(본체)"이다.

| 개념 | OpenClaw 에서의 위치 |
|---|---|
| **호스트 플랫폼** | ✅ OpenClaw 자신 (Gateway + 에이전트 런타임 소유) |
| 플러그인 | ✅ 167개 보유 (`extensions/`) + `packages/plugin-sdk` 로 직접 제작 가능 |
| 스킬 | ✅ 52개 보유 (`skills/`) + `SKILL.md` 로 직접 작성 가능 |
| **MCP 서버** | ✅ `openclaw mcp serve` — 채널 대화를 외부 MCP 클라이언트에 노출 |
| **MCP 클라이언트** | ✅ `openclaw mcp add/set/probe` — 외부 MCP 서버를 등록해 에이전트가 사용 |
| CLI 도구 | ✅ `openclaw` 전역 명령 |
| 데몬 / 서비스 | ✅ `openclaw gateway install` |
| 웹 앱 | ✅ Control UI (Lit) |
| 네이티브 앱 | ✅ mac / iOS / Android / Windows / Linux |

### Claude Code 와의 관계

```
Claude Code                     OpenClaw
  ↑                               ↑
코딩 전용 에이전트          상시 가동 개인 비서
(터미널 앞에 있을 때)       (밖에 있어도 메신저로)

        ←── MCP 로 상호 연결 ──→

추가로: OpenClaw 가 Claude Code / Codex 를
       에이전트 하네스로 내부 호스팅도 가능 (openclaw acp)
```

**경쟁이 아니라 보완 관계.**

---

## 8. API 토큰이 필요한가?

### OpenClaw 자체 토큰 = 없음

README 원문: **"has no paid tier, hosted service, or token."**

- OpenClaw 가입 ❌ / OpenClaw API 키 ❌ / 크립토 토큰 ❌
- 비영리 재단(501(c)(3)) 운영, MIT 라이선스

### 모델 인증 경로 5가지

| 경로 | 비용 | 특징 |
|---|---|---|
| 1. **기존 Claude Code / Codex CLI 로그인 재사용** | 구독료만 | 온보딩이 자동 감지. 새 키 발급 불필요 |
| 2. 구독 기반 (ChatGPT Plus, Claude Max, GitHub Copilot) | 월 구독 | 토큰당 과금 아님 |
| 3. API 키 (Anthropic, OpenAI, Google, Groq, DeepSeek 등 60+) | 사용량 과금 | |
| 4. **로컬 모델** (Ollama, LM Studio, llama.cpp, vLLM, SGLang) | **0원** | 인증·인터넷 불필요 |
| 5. 게이트웨이/라우터 (OpenRouter, LiteLLM, Cloudflare AI Gateway, Bedrock, Vertex) | 경유사 정책 | 한 키로 여러 모델 |

### 선택적으로 필요한 토큰

| 용도 | 필요 항목 | 필수? |
|---|---|---|
| 텔레그램 / 디스코드 / 슬랙 | 각 봇 토큰 | 해당 채널 사용 시만 |
| WhatsApp | QR 로그인 (Baileys) | 해당 채널 사용 시만 |
| iMessage | macOS 권한 | 해당 채널 사용 시만 |
| 웹 검색 | Brave / Tavily / Exa 키 | 해당 툴 사용 시만 |
| TTS / 음성 | ElevenLabs 등 | 옵션 |
| 게이트웨이 원격 접속 | `gateway.auth.token` (직접 정함) | 원격 사용 시 |

### 토큰 보관

- SQLite + `openclaw secrets` / `extensions/vault` / `extensions/onepassword`
- 모두 로컬 하드웨어에만 저장 (`docs/gateway/secrets.md`)
- `AGENTS.md` 규칙: 자격증명은 커밋·로그·전사·미디어에 절대 포함 금지

### 완전 무료 조합

```
OpenClaw (MIT, 무료) + Ollama (로컬 모델, 무료) + Telegram Bot (무료)
= 월 0원, 완전 오프라인 개인 AI 비서
```

---

## 9. 왜 깃허브에서 유명한가

> ⚠️ "39만 스타"는 레포 내 근거가 없어 검증되지 않음. 아래는 **구조적 인기 요인** 분석.

| # | 요인 | 근거 |
|---|---|---|
| 1 | **말만 하는 AI → 실제로 실행하는 AI** | `exec` `browser` `apply_patch` 등 실행 툴 + 승인 정책 |
| 2 | **새 앱 설치 불필요** — 진입 마찰 0 | 20+ 기존 메신저 채널 지원 → 바이럴 최적 |
| 3 | **프라이버시가 마케팅이 아니라 아키텍처** | 기본 텔레메트리 = 하루 1회 버전 체크. 통계는 옵트인. `update.checkOnStart:false` 로 완전 차단 |
| 4 | **벤더 락인 없음** | 플러그인 167개 / 모델 공급자 60+ / 두뇌 교체가 설정 한 줄 |
| 5 | **거버넌스가 깨끗함** | 독립 501(c)(3). "OpenAI is a donor, not an owner." 유료 티어·호스팅·토큰 전무 |
| 6 | **품질 신호 압도적** | 테스트 16,202개, 문서 1,307개, SECURITY.md 36KB, AGENTS.md 22KB |
| 7 | **브랜딩과 서사** | 우주 랍스터 Molty, "EXFOLIATE!", `soul.md`, 프로젝트 lore, 유명 개발자(steipete) |
| 8 | **기여 장벽 낮음** | "AI-assisted PRs are welcome" + AGENTS.md 로 AI 에게 규칙 선행 학습 |

### 교훈

> 기술이 좋아서 유명해진 것이 아니라,
> **기존 습관에 끼어들고(메신저) + 신뢰를 아키텍처로 증명하고(로컬) + 락인이 없고(플러그인) + 서사가 재밌어서(랍스터)** 유명해졌다.

---

## 10. 로컬 에이전트 구축에 도움이 되는가

### 결론: 매우 크게 도움이 된다. 사실상 "검증된 정답지".

### 활용 3단계

| 레벨 | 내용 | 난이도 |
|---|---|---|
| 1. **그대로 쓴다** | 설치 + 플러그인 조합만으로 로컬 에이전트 완성 | ⭐ |
| 2. **확장한다** | `packages/plugin-sdk` 계약으로 플러그인, `SKILL.md` 로 스킬 추가 | ⭐⭐ |
| 3. **설계를 배운다** | 어려운 문제들의 검증된 해답이 문서로 존재 | ⭐⭐⭐ (가치 최상) |

### 로컬 에이전트 만들 때 부딪히는 문제 ↔ OpenClaw 의 답

| 문제 | 답 | 문서 |
|---|---|---|
| 프로세스 구조 | 단일 게이트웨이 데몬 + WebSocket | `docs/concepts/architecture.md` |
| 컨텍스트 넘침 | 컴팩션 전략, 프롬프트 프리픽스 재사용 | `docs/concepts/compaction.md`, `context-engine.md` |
| 장기 기억 | 메모리 아키텍처 / 검색 / 출처추적 | `docs/concepts/memory-architecture.md`, `memory-search.md`, `memory-provenance.md` |
| 위험 명령 방어 | 승인 정책, 권한 모드, 샌드박스 | `docs/tools/exec-approvals.md`, `permission-modes.md`, `docs/gateway/sandboxing` |
| 모델 장애 | 폴백 체인 + 인증 프로파일 로테이션 | `docs/concepts/model-failover.md` |
| 상태 저장 | SQLite + Kysely, **동기 트랜잭션** (콜백 내 await 금지) | `docs/reference/database-schemas.md` |
| 멀티에이전트 조율 | 서브에이전트, 스웜, 위임, 병렬 레인 | `docs/concepts/multi-agent.md`, `delegate-architecture.md`, `docs/tools/swarm.md` |
| 툴 과다 → 프롬프트 폭발 | Tool Search 로 카탈로그 검색 | `docs/tools/tool-search.md` |
| 낯선 발신자 | 디바이스 페어링 + 챌린지 서명 | `docs/channels/pairing.md` |
| 설정 스키마 변경 | **Doctor 마이그레이션** 패턴 | `docs/gateway/doctor/config-migrations.md` |
| 무한 루프 | 루프 감지 | `docs/tools/loop-detection.md` |
| 스트리밍 / 타이핑 표시 | | `docs/concepts/streaming.md`, `typing-indicators.md` |

### 외부 연동 진입점

| 방법 | 사용 | 언어 제약 |
|---|---|---|
| A. WebSocket 직접 | `packages/gateway-protocol` 스펙 | 없음 |
| B. 참조 클라이언트 | `packages/gateway-client`(TS), `crates/openclaw-gateway-client`(Rust) | TS / Rust |
| C. MCP | `openclaw mcp serve` | MCP 지원 클라이언트 |
| D. 웹훅 | `extensions/webhooks` | 없음 (HTTP) |

### 주의할 점

- 학습 곡선 있음 (소스 604만 줄, 문서 1,307개) → 필요한 문서만 골라 읽을 것
- **terminal-first 설계** (`VISION.md`) — 편의 래퍼로 보안 결정을 숨기지 않는 것이 의도
- 상주 데몬 관리 필요
- 샌드박스 미설정 시 호스트에서 실제 명령 실행

---

## 11. React / PHP 로 만들 수 있는가

### 갈래 A — OpenClaw 자체를 React/PHP 로 재작성? ❌ 비추천

| 이유 |
|---|
| `src/` 604만 줄, TS 파일 34,794개 |
| PHP 는 장시간 WebSocket 다중 연결 상주 데몬에 부적합 |
| React 는 UI 라이브러리 — 백엔드 데몬을 만들 수 없음 |
| `VISION.md`: "TypeScript was chosen to keep OpenClaw hackable by default" |

### 갈래 B — OpenClaw 에 React/PHP 를 붙이기? ✅ 완전 가능

게이트웨이가 **WebSocket + JSON 프로토콜**이므로 언어 무관.

```
┌──────────────────┐        ┌──────────────────┐
│  React 앱        │        │  PHP / Laravel   │
│  (커스텀 UI)     │        │  (기존 백엔드)   │
└────────┬─────────┘        └────────┬─────────┘
         │  WebSocket / HTTP Webhook │
         └────────────┬──────────────┘
                      ▼
         ╔════════════════════════╗
         ║   OpenClaw Gateway     ║
         ║   ws://127.0.0.1:18789 ║
         ╚════════════════════════╝
```

### React 로 할 수 있는 것

| 할 것 | 방법 |
|---|---|
| 커스텀 대시보드 | `packages/gateway-client` (TS) 를 그대로 import |
| 타입 안전 | `packages/gateway-protocol` 타입 재사용 |
| 스트리밍 응답 UI | `event:agent` 이벤트 구독 |
| 세션 목록 / 재개 | `sessions` 메서드 |

**실전 체크리스트**

```
1) 첫 프레임은 반드시 connect
2) connect.challenge 논스에 서명 (payload v3: platform/deviceFamily 바인딩)
3) 새 디바이스는 페어링 승인 필요 → device token 발급받아 이후 재사용
4) send / agent 등 부수효과 메서드에는 멱등키 필수
5) 요청: {type:"req", id, method, params}
   응답: {type:"res", id, ok, payload|error}
   이벤트: {type:"event", event, payload, seq?, stateVersion?}
```

> ⚠️ 기존 Control UI 는 **Lit(Web Components)** 이다. 기존 UI 개조보다 **React 별도 앱 신규 작성**이 훨씬 수월하다.

### PHP 로 할 수 있는 것

| 할 것 | 방법 |
|---|---|
| **웹훅 수신** (가장 쉬움) | `extensions/webhooks` → PHP 엔드포인트로 HTTP POST 수신 |
| 웹훅 발신 | PHP → 게이트웨이 HTTP 엔드포인트 호출 |
| WebSocket 클라이언트 | Ratchet / ReactPHP / Swoole |
| Laravel 통합 | Queue + Reverb 조합 |
| 커스텀 채널 | PHP 브리지 서버 + 웹훅 플러그인 연결 |

**PHP 활용 예:** 기존 쇼핑몰 / CMS / 사내 ERP / 워드프레스에 AI 비서 연결

### 가장 정석적인 경로

내 서비스를 **OpenClaw 플러그인**으로 만드는 것 (`packages/plugin-sdk`, TypeScript):

```
장점: 설정 / 시크릿 / doctor / 생명주기를 공짜로 얻음
     에이전트가 자동으로 툴로 인식
     ClawHub 배포 → 유통 및 수익화 가능
     코어 미변경 → 업데이트에 깨지지 않음
단점: TypeScript 필요 (React 경험이 있으면 빠르게 적응)
```

### 최종 추천 우선순위

| 순위 | 할 것 | 언어 | 난이도 |
|---|---|---|---|
| 🥇 | React 커스텀 대시보드 | React + TS | ⭐⭐ |
| 🥈 | PHP 웹훅 연동 | PHP | ⭐ |
| 🥉 | OpenClaw 플러그인 | TypeScript | ⭐⭐⭐ |
| ❌ | OpenClaw 재작성 | — | ⭐⭐⭐⭐⭐ |

---

## 12. 수익화 아이디어 상세

### 12-0. 법적 기반

**MIT 라이선스로 가능한 것**

```
✅ 상업적 이용   ✅ 수정 후 판매   ✅ 재배포
✅ 사적 이용     ✅ 유료 서비스    ✅ 클로즈드 소스 제품에 포함
```

**의무:** 저작권 표시 + MIT 라이선스 사본 포함

**주의사항**

| 위험 | 대응 |
|---|---|
| 상표권 ("OpenClaw" 이름 / 로고 / 마스코트) | 독자 브랜드 사용. "OpenClaw 기반"이라는 **설명**은 가능 |
| 번들 의존성 라이선스 | `THIRD_PARTY_NOTICES.md` 확인 |
| 모델 공급자 ToS | 재판매 제한 조항 확인 (Anthropic / OpenAI 등) |
| 개인정보보호법 | 고객 데이터 취급 시 국내법 준수 |
| 메신저 플랫폼 정책 | **WhatsApp 비공식 연결(Baileys)은 상업 서비스에 리스크** |
| 커뮤니티 예의 | 수익 발생 시 업스트림 기여로 환원 |

**핵심 전략 인사이트**

```
README: "has no paid tier, hosted service, or token"
VISION: 안 머지할 것 = 코어 스킬 / 전체 문서 번역 / 상업 서비스 연동 / 클라우드 샌드박스

  → 재단이 "절대 안 하겠다"고 선언한 영역 = 서드파티가 독점 가능한 시장
```

---

### 12-1. 한국형 채널 플러그인

**기회:** 채널 167개 중 한국 서비스가 사실상 없음.

| 채널 | 지원 | 한국 점유율 |
|---|---|---|
| 카카오톡 | ❌ | 95%+ |
| 네이버웍스 | ❌ | 기업 다수 |
| 잔디(JANDI) | ❌ | 기업 |
| 플로우(Flow) | ❌ | 기업 |
| 두레이(Dooray) | ❌ | NHN |
| 라인 | ✅ `extensions/line` | 일본 중심 |

**수익 구조**

| 모델 | 가격 예시 |
|---|---|
| ClawHub 유료 플러그인 | $19 ~ 49 |
| 기업용 라이선스 | 월 10 ~ 50만원 |
| 커스터마이징 개발 | 프로젝트당 300 ~ 2,000만원 |
| 유지보수 계약 | 월 30 ~ 100만원 |

**구현 경로 (난이도 ⭐⭐⭐)**

```
1. extensions/line, extensions/telegram 구조 분석
2. docs/plugins/sdk-channel-plugins.md 채널 책임 계약 숙독
3. 대상 채널 공식 API 연동
4. packages/plugin-sdk 계약만 사용 (코어 내부 접근 금지 — AGENTS.md 규칙)
```

**리스크:** 카카오톡 개인 계정 연동은 정책 제약이 크다 → 비즈니스 채널 / 알림톡 경로 검토 필요.
**네이버웍스 / 잔디 / 두레이는 공식 API 가 있어 훨씬 안전 → 여기서 시작 권장.**

---

### 12-2. 기업 온프레미스 구축 · 운영 대행 ⭐ 최우선 추천

**무엇을:** "데이터를 외부로 내보낼 수 없는 조직"에 사내 AI 비서를 설치·운영해주는 SI 사업.

**OpenClaw 가 이 시장에 맞는 이유**

```
✅ 100% 온프레미스 (외부 전송 0)
✅ 로컬 모델(Ollama) 지원 → 완전 오프라인 가능
✅ MIT 라이선스 → 소프트웨어 비용 0원
✅ 감사 로그 내장 (docs/gateway/audit.md)
✅ 멀티테넌트 격리 (openclaw fleet)
✅ 백업 / 복구 (openclaw backup)
✅ Docker / Fly.io / Render 배포 자산 완비
```

**타겟 고객**

| 산업 | 필요 이유 |
|---|---|
| 병원 · 의료 | 환자정보 외부전송 불가 (의료법) |
| 금융 · 보험 | 전자금융감독규정 |
| 법무법인 | 의뢰인 비밀유지 |
| 공공기관 | 망분리 |
| 제조 대기업 | 설계도면 · 영업비밀 |
| 연구소 | 미공개 연구데이터 |

**수익 구조**

| 항목 | 가격대 |
|---|---|
| 초기 구축 | 500만 ~ 5,000만원 |
| 월 유지보수 | 50만 ~ 300만원 |
| 사내 교육 | 회당 100 ~ 300만원 |
| 커스텀 플러그인 | 건당 300 ~ 1,000만원 |
| 24/7 지원 SLA | 월 200만원+ |

**필요 역량 (난이도 ⭐⭐ — 기술 낮음 / 영업 높음)**

```
- OpenClaw 설치 · 설정 · doctor 숙련
- Docker / Fly.io / Render 배포
- 보안 문서 숙달 (docs/gateway/security/exposure-runbook)
- 샌드박싱 설정 (docs/gateway/sandboxing)
- Ollama 로컬 모델 튜닝
- openclaw fleet 멀티테넌트 구성
```

**강점:** 재고 0 · 초기자본 거의 0 · MRR 확보 가능 · 국내 경쟁자 희소 · 노하우 재사용으로 마진 상승

---

### 12-3. 관리형 호스팅 서비스

**기회:** 공식 호스팅이 영구히 없다고 선언됨 → 서드파티 독점 가능.

**기술 기반 (레포에 이미 존재)**

```
Dockerfile / docker-compose.yml / fly.toml / render.yaml / deploy/
openclaw fleet  ← 테넌트별 격리 셀 (각자 Gateway / 상태 / 자격증명 / 컨테이너 / 루프백 전용 포트)
ghcr.io/openclaw/openclaw:latest
```

`docs/cli/fleet.md`: "Use one cell for each tenant trust boundary; do not use one shared Gateway as a hostile multi-tenant boundary."

**수익 구조 (SaaS 3티어 예시)**

| 플랜 | 가격 | 내용 |
|---|---|---|
| Free | 0원 | 1채널, 월 100메시지, 본인 API 키 |
| Pro | 월 19,900원 | 5채널, 무제한, 모델 포함 |
| Team | 월 99,000원 | 팀 게이트웨이, 관리자 콘솔, 감사로그 |
| Enterprise | 협의 | 전용 인스턴스, SLA, SSO |

**리스크 (난이도 ⭐⭐⭐⭐)**

```
1. 컨테이너 탈출 위험 — exec 툴이 실제 명령을 실행함
   → openclaw fleet + Crabbox 격리 필수, 그래도 잔존 위험
2. 모델 API 비용 관리 (사용량 폭주)
3. 개인정보보호법 준수 (고객 대화 저장)
4. 메신저 플랫폼 ToS (특히 WhatsApp 비공식 연결)
5. 24/7 운영 인력
6. 포지셔닝 딜레마 — "데이터가 내 PC에 있음"이 셀링포인트인데 호스팅은 그것을 포기
```

**더 나은 변형 — 반(半)관리형**

```
"고객 서버에 설치 + 원격 관리는 우리가"
→ 데이터는 고객 소유, 운영은 대행 = 딜레마 해소
가격 예: 설치 300만원 + 월 관리 50만원
```

---

### 12-4. 한국어 교육 콘텐츠 & 커뮤니티

**기회:** 문서 1,307개 전부 영어. `VISION.md` 가 전체 문서 번역을 **보류(deferred)** 로 명시 → 한국어 자료 시장이 비어 있음.

| 경로 | 수익 | 난이도 |
|---|---|---|
| 유튜브 튜토리얼 시리즈 | 광고 + 협찬, 월 50 ~ 500만원 | ⭐ |
| 인프런 / 유데미 강의 | 강의당 5 ~ 15만원 × 수강생 | ⭐⭐ |
| 유료 뉴스레터 / 멤버십 | 월 9,900원 × 구독자 | ⭐ |
| 기업 출강 | 회당 100 ~ 300만원 | ⭐⭐ |
| 전자책 / 기술서 | 인세 | ⭐⭐ |
| 한국어 문서 사이트 | 광고 + 컨설팅 리드 | ⭐ |

**전략적 가치 (수익보다 중요)**

```
교육 콘텐츠 = 12-1 / 12-2 / 12-3 의 영업 파이프라인

시청자 → "우리 회사에 깔아주세요"  → 구축 대행 (12-2)
      → "카톡 연동 필요해요"      → 플러그인 (12-1)
      → "관리도 해주세요"         → 호스팅 (12-3)
```

---

### 12-5. 버티컬 특화 스킬 팩

**기회:** `VISION.md` — "New skills should be published through ClawHub first, not added to core by default."
→ 재단이 코어 스킬 추가를 거부 = ClawHub 유료 스킬 시장이 열려 있음.

| 스킬팩 | 내용 | 가격 예시 |
|---|---|---|
| 부동산 중개사 팩 | 매물 정리, 계약서 검토 체크리스트, 고객응대 템플릿 | $29 |
| 법무 팩 | 판례 검색, 계약서 리뷰 루브릭, 서면 초안 | $99 |
| 병원 원무 팩 | 예약 관리, 보험청구 체크, 환자 안내 | $79 |
| 온라인 셀러 팩 | 상품등록, 리뷰 분석, CS 자동응답, 재고 알림 | $49 |
| 개발팀 팩 | 코드리뷰 루브릭, PR 템플릿, 배포 런북 | $39 |
| 재무 / 회계 팩 | 세금계산서 정리, 월결산 체크리스트 | $69 |
| 콘텐츠 크리에이터 팩 | 기획서, 썸네일 아이디어, 댓글 분석 | $29 |

**구현 경로 (난이도 ⭐ — 가장 쉬움)**

```
1. skills/ 내 기존 52개 SKILL.md 구조 분석
2. docs/tools/creating-skills.md 숙독
3. 잘 아는 도메인의 워크플로를 SKILL.md 로 작성
4. ClawHub 등록
→ 코딩 거의 없음. 도메인 지식이 자산.
```

**강점:** 한 번 제작 후 반복 판매(디지털 자산) · 유지보수 부담 최소 · 포트폴리오화 용이

---

### 12-6. 니치 아이디어 모음

| # | 아이디어 | 수익 | 난이도 |
|---|---|---|---|
| 6-1 | **감사 / 컴플라이언스 애드온** — `docs/gateway/audit.md` 확장, 금융·의료 규제 리포트 자동생성 | 기업 라이선스 월 50만원+ | ⭐⭐⭐ |
| 6-2 | **한국 모델 공급자 플러그인** — 네이버 하이퍼클로바X, 카카오, LG 엑사원 연동 | 플러그인 판매 + 공급사 협업 | ⭐⭐ |
| 6-3 | **보안 감사 서비스** — `SECURITY.md`(36KB) 기반 설치본 보안 점검 리포트 | 건당 200 ~ 500만원 | ⭐⭐⭐ |
| 6-4 | **하드웨어 번들** — 미니PC(NUC / 라즈베리파이)에 사전설치 + Ollama = 플러그앤플레이 AI 비서 | 기기당 30 ~ 80만원 마진 | ⭐⭐ |
| 6-5 | **업무 자동화 템플릿 마켓** — `openclaw cron` 기반 레시피 ("매일 아침 브리핑" 등) | 구독제 | ⭐ |
| 6-6 | **React 커스텀 UI / 대시보드 판매** — `gateway-client` 기반 프리미엄 UI | 라이선스 판매 | ⭐⭐ |
| 6-7 | **PHP / 워드프레스 브리지 플러그인** — 국내 PHP 사이트 대량 시장 | WP 플러그인 유료화 | ⭐⭐ |
| 6-8 | **마이그레이션 서비스** — `extensions/migrate-claude`, `migrate-hermes` 참고, 타 봇 → OpenClaw 이전 | 건당 100 ~ 500만원 | ⭐⭐ |

---

### 12-7. 우선순위 결론

| 순위 | 아이디어 | 이유 |
|---|---|---|
| 🥇 | **12-2 기업 온프레미스 구축 · 운영 대행** | 초기자본 0 · 기술난이도 낮음 · 단가 높음 · MRR 확보 · 국내 경쟁자 희소 · 노하우 재사용 |
| 🥈 | **12-5 버티컬 스킬팩** | 난이도 최저 · 자산형 · 코딩 거의 불필요 |
| 🥉 | **12-4 한국어 교육 콘텐츠** | 모든 사업의 마케팅 엔진 · 즉시 시작 가능 |

---

## 13. 실행 로드맵

### 0 ~ 1개월 — 직접 써보고 감 잡기 (수익 0)

- [ ] `npx openclaw@latest` 실행
- [ ] 기존 Claude Code 로그인 재사용해 온보딩 완료
- [ ] 텔레그램 봇 1개 연결
- [ ] `AGENTS.md` + `VISION.md` 완독
- [ ] `docs/concepts/architecture.md` 숙독
- [ ] `docs/gateway/security` + `docs/gateway/sandboxing` 숙독
- [ ] `SKILL.md` 1개 직접 작성

### 1 ~ 3개월 — 신뢰 자산 만들기 (수익 소액)

- [ ] 유튜브 / 블로그 한국어 시리즈 시작 (12-4)
- [ ] React 커스텀 대시보드 제작 후 공개 (12-6 / 6-6)
- [ ] 스킬팩 1개 ClawHub 등록 (12-5)
- [ ] 업스트림에 PR 1건 기여

### 3 ~ 6개월 — 첫 현금 (목표 월 300 ~ 1,000만원)

- [ ] 구축 대행 첫 고객 확보 (12-2)
- [ ] 네이버웍스 / 잔디 채널 플러그인 (12-1)
- [ ] 스킬팩 3 ~ 5개로 확장 (12-5)
- [ ] 기업 출강 / 컨설팅 (12-4)

### 6 ~ 12개월 — 스케일

- [ ] 반관리형 호스팅 출시 (12-3 변형)
- [ ] 카카오 경로 정식 검토 (12-1)
- [ ] 감사 / 컴플라이언스 애드온 (6-1)

---

## 14. 참고 문서 링크 모음

### 레포 주소

- **분석 대상 (내 포크):** https://github.com/bmshin94/openclaw
- **원본 업스트림:** https://github.com/openclaw/openclaw
- npm 패키지: https://www.npmjs.com/package/openclaw

### 공식 사이트

- 웹사이트: https://openclaw.ai
- 공식 문서: https://docs.openclaw.ai
- 시작 가이드: https://docs.openclaw.ai/start/getting-started
- Why OpenClaw: https://docs.openclaw.ai/start/why-openclaw
- 플러그인 마켓 ClawHub: https://clawhub.ai
- 재단: https://openclaw.org
- DeepWiki: https://deepwiki.com/openclaw/openclaw
- Discord: https://discord.gg/clawd
- X: https://x.com/openclaw

### 레포 내 필독 문서

| 파일 | 왜 읽어야 하나 |
|---|---|
| `AGENTS.md` | AI 에이전트용 코드베이스 규칙서. 내 프로젝트 CLAUDE.md 작성 시 최고 참고자료 |
| `VISION.md` | 제품 범위 · 로드맵 · **머지 거절 목록(= 사업 기회 목록)** |
| `README.md` | 설치 · 구성 · 거버넌스 · 후원사 |
| `SECURITY.md` | 보안 정책 (36KB) |
| `CONTRIBUTING.md` | 기여 워크플로 |
| `THIRD_PARTY_NOTICES.md` | 번들 의존성 라이선스 (상업화 전 필수 확인) |
| `docs/concepts/architecture.md` | 게이트웨이 아키텍처 · 와이어 프로토콜 |
| `docs/tools/index.md` | 툴 / 스킬 / 플러그인 선택 가이드 |
| `docs/cli/mcp.md` | MCP 서버 / 클라이언트 양방향 사용법 |
| `docs/cli/fleet.md` | 멀티테넌트 격리 셀 (SaaS 설계 참고) |
| `docs/plugins/building-plugins` | 플러그인 제작 |
| `docs/plugins/sdk-channel-plugins.md` | 채널 플러그인 책임 계약 |
| `docs/tools/creating-skills.md` | 스킬 제작 |
| `docs/gateway/doctor/config-migrations.md` | 설정 마이그레이션 패턴 |
| `docs/reference/database-schemas.md` | SQLite 스키마 |

---

*이 문서는 `bmshin94/openclaw` 레포지토리를 전수조사한 결과를 정리한 분석 자료입니다.*
