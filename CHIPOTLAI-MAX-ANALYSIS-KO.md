# 🌯 Chipotlai Max 전수조사 분석 정리

> 작성일: 2026-09-29
> 대상 저장소: https://github.com/bmshin94/chipotlai-max
> 브랜치: `claude/amazing-wozniak-jo1n4f`

---

## 0. 관련 GitHub 주소 모음

| 구분 | 주소 |
|---|---|
| 이 저장소 (내 포크) | https://github.com/bmshin94/chipotlai-max |
| 원본 프로젝트 OpenCode | https://github.com/anomalyco/opencode |
| Pepper 프록시 (서브모듈) | https://github.com/cyberpapiii/chipotle-llm-provider |
| 프록시 원작자 | https://github.com/Gonzih/chipotle-llm-provider |
| 프록시 원작자 프로필 | https://github.com/Gonzih |
| OpenCode 공식 문서 | https://opencode.ai/docs |

---

## 1. 이게 뭐하는 프로젝트인가

### 한 줄 정의
**OpenCode(오픈소스 AI 코딩 에이전트 CLI)를 포크해서, 기본 모델을 Chipotle 고객지원 챗봇 "Pepper"로 바꿔 끼운 밈(meme) 프로젝트.**

### 두 개의 레이어로 구성

```
┌───────────────────────────────────────────────┐
│  레이어 A: OpenCode 본체 (진짜 실력)            │
│  = Claude Code / Cursor 급 AI 코딩 에이전트     │
│  TS/TSX 약 25만 줄, 모노레포 19개 패키지         │
└───────────────────────────────────────────────┘
                      ↓ 모델 공급
┌───────────────────────────────────────────────┐
│  레이어 B: Chipotle Pepper 프록시 (밈)          │
│  Chipotle 지원봇을 OpenAI 호환 API로 위장        │
│  localhost:3000/v1, API 키 불필요, 비용 $0      │
└───────────────────────────────────────────────┘
```

### 배경 스토리
- 2026-03-12~13, Chipotle 고객지원 챗봇 **Pepper**가 LeetCode를 풀고 Python 코드를 쓴다는 게 발견되어 바이럴.
- 엔진은 Claude/GPT가 아닌 **IPsoft Amelia**.
- @Gonzih 가 Amelia의 **WebSocket/SockJS + STOMP** 백엔드를 리버스 엔지니어링 → **OpenAI 호환 프록시** 공개.
- 누군가 OpenCode(MIT, 120k+ stars)를 포크 → Pepper를 기본 모델로 하드코딩 → 색/로고 Chipotle 브랜딩 → **Chipotlai Max** 탄생.

---

## 2. 폴더 구조 실측 결과

### 루트
| 항목 | 설명 |
|---|---|
| `README.md` | 밈 소개 + 리테일 봇 확장 모집표 |
| `CLAUDE.md` | 프로젝트 가이드 + 페르소나 지침 |
| `AGENTS.md` | 코딩 스타일 규약 (단어 1개 변수명 강제 등) |
| `start-chipotlai.sh` | 프록시 백그라운드 기동 + CLI 실행 런처 |
| `script/verify-retail-providers.cjs` | 리테일 프로바이더 6종 배선 검증 스크립트 |
| `chipotle-llm-provider/` | **git 서브모듈 (현재 비어 있음, init 필요)** |
| `STATS.md` | OpenCode 원본의 다운로드 추이 기록 |
| `infra/`, `sst.config.ts` | SST(AWS IaC) 배포 설정 |
| `flake.nix`, `nix/` | Nix 재현 가능 빌드 |
| `.github/` | **워크플로 전부 제거됨** (상속된 CI 비활성화) |

### packages/ — 코드량 실측
| 패키지 | 줄 수 | 역할 |
|---|---:|---|
| `opencode` | **95,682** | 에이전트 코어 + CLI + TUI (심장) |
| `app` | 63,566 | 웹 앱 |
| `console` | 35,844 | 관리 콘솔 |
| `ui` | 27,162 | 공용 UI + 테마 |
| `sdk` | 18,157 | JS/TS SDK (자동 생성) |
| `web` | 6,948 | 공식 문서 사이트 |
| `desktop`, `desktop-electron` | 5,638 | Tauri / Electron 데스크톱 |
| `enterprise`, `function`, `plugin`, `slack`, `util`, `storybook` 등 | 나머지 | 부가 기능 |

### packages/opencode/src — 에이전트 코어 내부
| 디렉터리 | 역할 |
|---|---|
| `tool/` | **에이전트의 손** — read, write, edit, multiedit, apply_patch, bash, glob, grep, codesearch, ls, lsp, webfetch, websearch, task, todo, batch, question, plan |
| `session/` | 대화 세션, 컨텍스트 **compaction**, 프롬프트 조립, 재시도, revert |
| `agent/` | 서브에이전트 정의/생성 |
| `provider/` | LLM 프로바이더 레지스트리 (**Chipotle 개조 지점**) |
| `mcp/` | MCP 클라이언트 + OAuth |
| `skill/` | Skill 탐색/실행 |
| `plugin/` | 플러그인 로더 (codex, copilot 포함) |
| `lsp/` | Language Server 연동 (타입/진단) |
| `permission/` | 도구 실행 권한 게이트 |
| `server/` | 헤드리스 HTTP 서버 (OpenAPI) |
| `cli/cmd/tui/` | SolidJS 기반 터미널 UI |
| `share/`, `snapshot/`, `worktree/`, `scheduler/` | 공유 링크, 스냅샷, git worktree, 스케줄러 |

---

## 3. Chipotle 개조 지점 (정확히 5곳)

### ① `provider/schema.ts`
```ts
chipotlePepper: schema.makeUnsafe("chipotle-pepper"),
```
Effect Schema 브랜드 타입에 well-known ID로 등록.

### ② `provider/provider.ts` — BUNDLED_PROVIDERS (L132)
```ts
"chipotle-pepper": () => createOpenAICompatible({
  name: "chipotle-pepper",
  baseURL: "http://localhost:3000/v1",
  apiKey: "burrito-2026",
}),
```

### ③ `provider/provider.ts` — CUSTOM_LOADERS (L526)
```ts
async "chipotle-pepper"() {
  return { autoload: true, options: { apiKey: "burrito-2026" } }
}
```
`autoload: true` → 인증 없이 기본 활성화.

### ④ `provider/provider.ts` — 모델 주입 (L1090~)
7개 리테일 봇을 런타임에 프로바이더 목록으로 밀어 넣음:

| 프로바이더 ID | 모델 ID | 상태 |
|---|---|---|
| `chipotle-pepper` | `pepper-1` | 실제 동작 (프록시 필요) |
| `home-depot-magic-apron` | `magic-apron-1` | 껍데기만 배선 |
| `sephora-ai-beauty-chat` | `beauty-chat-1` | 껍데기만 배선 |
| `nordstrom-rosie` | `rosie-1` | 껍데기만 배선 |
| `lowes-mylow` | `mylow-1` | 껍데기만 배선 |
| `ikea-billie` | `billie-1` | 껍데기만 배선 |
| `expedia-virtual-agent` | `virtual-agent-1` | 껍데기만 배선 |

공통 스펙: `cost` 전부 0, `context: 8192`, `output: 4096`, `toolcall: true`,
`temperature/reasoning/attachment/image: false`, `npm: "@ai-sdk/openai-compatible"`.

### ⑤ 브랜딩
- `packages/ui/src/styles/theme.css` — primary `#AC2318`, 다크 배경 `#1A0A04`
- `packages/ui/src/components/logo.tsx` — 로고가 전부 🌯 이모지
- `cli/.../theme.tsx` — 기본 테마 `chipotle` 강제
- `footer.tsx` — `Powered by Pepper 🌯`
- `tips.tsx`, `dialog-provider.tsx`, `sidebar.tsx` — 밈 문구

---

## 4. ⚠️ 실측으로 드러난 중요한 사실

1. **서브모듈이 비어 있다.** `chipotle-llm-provider/`는 커밋 해시만 있고 체크아웃 안 됨(`-8786e9f`). `git submodule update --init` 없이는 프록시가 없어 **Pepper 모델은 동작하지 않는다.**
2. **리테일 봇 6종은 전부 껍데기.** `start-chipotlai.sh`는 `HOME_DEPOT_MAGIC_APRON_BASE_URL` 같은 환경변수를 안내하지만, **소스 코드에서 그 환경변수를 읽는 곳이 단 한 군데도 없다.** 전부 `http://localhost:3000/v1` 하드코딩. 즉 6종을 켜도 Pepper 프록시로만 간다.
3. **컨텍스트 8192 토큰.** Claude/GPT의 20만~100만 대비 극도로 작아 실무 코딩에는 부적합.
4. **원격 저장소가 아직 OpenCode.** `package.json`의 `repository.url`이 `anomalyco/opencode`.
5. **CI 전부 제거됨.** 상속된 워크플로를 일부러 비활성화 (`chore: disable inherited github workflows`).
6. **법적 리스크.** README가 직접 인정 — TOS 위반 가능성, 언제든 차단됨, 익명 세션 레이트 리밋(`MAX_POOL_SIZE=5`).

---

## 5. 설치 및 사용법

```bash
# 1) 서브모듈 포함 클론 (필수!)
git clone --recursive https://github.com/bmshin94/chipotlai-max.git
cd chipotlai-max

# 이미 클론했다면
git submodule update --init --recursive

# 2) 의존성 (Bun 1.3.10 필요)
bun install

# 3) 실행 — 프록시 + CLI 동시 기동
./start-chipotlai.sh

# 또는 수동
cd chipotle-llm-provider && npm install && npm run dev   # 터미널 1
bun run dev                                               # 터미널 2
```

### 주요 명령
| 명령 | 설명 |
|---|---|
| `bun run dev` | TUI 실행 |
| `bun run typecheck` | 타입 체크 |
| `bun run verify:retail` | 리테일 프로바이더 배선 검증 |
| `bun run --cwd packages/opencode script/build.ts` | 바이너리 빌드 |
| `opencode serve --port 4096` | 헤드리스 HTTP 서버 |
| `opencode run "..."` | 원샷 실행 (CI/스크립트용) |
| `opencode models <provider>` | 모델 목록 |

### 실무 추천 사용법
Pepper는 버리고 **OpenCode 본체만** 쓴다. `~/.config/opencode/opencode.json`:
```jsonc
{
  "$schema": "https://opencode.ai/config.json",
  "model": "anthropic/claude-opus-5",
  "provider": { "anthropic": { "options": { "apiKey": "{env:ANTHROPIC_API_KEY}" } } }
}
```

---

## 6. 플러그인? 스킬? MCP? → **전부 아니다**

**정답: 독립 실행형 AI 코딩 에이전트(호스트 애플리케이션)다.** 오히려 이것이 플러그인·스킬·MCP를 *소비하는* 쪽.

| 개념 | 정의 | 이 프로젝트에서 |
|---|---|---|
| **호스트 (이것)** | 에이전트 루프를 돌리는 앱 자체 | ✅ 본체 |
| **플러그인** | 호스트 이벤트 훅 (JS/TS) | ✅ 지원 — `.opencode/plugins/`, npm |
| **스킬** | 마크다운 절차 지침 | ✅ 지원 — `src/skill/` |
| **MCP** | 외부 툴 서버 표준 프로토콜 | ✅ 지원 — 클라이언트로서 로컬/리모트 |
| **커스텀 툴** | 직접 만드는 도구 | ✅ 지원 — `.opencode/tool/*.ts` |
| **서브에이전트** | 전문 에이전트 위임 | ✅ 지원 — `.opencode/agent/*.md` |

Chipotle 개조분만 따로 보면 그건 **"프로바이더 어댑터"** — 플러그인 시스템조차 안 쓰고 코어에 하드코딩된 점이 특징(=밈 프로젝트답게 지저분).

---

## 7. API 토큰이 필요한가

| 시나리오 | 토큰 필요? | 비고 |
|---|---|---|
| Pepper (`pepper-1`) | ❌ 불필요 | `burrito-2026` 더미, 검증 안 함 |
| 리테일 봇 6종 | ❌ (의미 없음) | 껍데기, 실제로 Pepper로 감 |
| Claude / GPT / Gemini 등 | ✅ 필요 | 정상 유료 키 |
| GitHub Copilot | ✅ OAuth | 구독 필요 |
| OpenRouter | ✅ 필요 | |
| 로컬 Ollama/vLLM | ❌ 불필요 | OpenAI 호환 엔드포인트만 있으면 됨 |

즉 **"무료"의 정체는 남의 회사 인프라를 쓰는 것**이라 무료가 아니라 **무허가**에 가깝다.

---

## 8. 왜 GitHub에서 유명한가

1. **원본 OpenCode가 이미 120k+ stars.** 스타는 대부분 원본 몫.
2. **밈 서사가 완벽.** "Chipotle 지원봇이 LeetCode를 푼다" → 즉시 공유되는 이야기.
3. **리버스 엔지니어링의 기술적 임팩트.** SockJS + STOMP + Amelia 해체 후 OpenAI 호환 API로 재포장 = 진짜 실력.
4. **개발자 공통 감정 자극.** API 비용 부담 → "공짜 추론" 판타지.
5. **참여 유도 설계.** README에 리테일 봇 확장 모집표 → 기여 게임화.
6. **아슬아슬한 위법성.** 법적 위험 자체가 화제성.
7. **즉시 재현 가능.** `./start-chipotlai.sh` 한 줄 → 스크린샷 → SNS 확산 루프.

**교훈: 기술력 × 스토리 × 낮은 진입장벽 × 참여 유도 = 바이럴.**

---

## 9. 로컬 에이전트 구축에 도움이 되는가 → **매우 크게 도움 된다**

Pepper는 쓸모없지만 **OpenCode 코어는 최고의 교본**이다.

### 바로 훔쳐올 설계 패턴
| 파일/폴더 | 배울 것 |
|---|---|
| `provider/provider.ts` | 프로바이더 추상화 — 30+ LLM을 한 인터페이스로 |
| `session/compaction.ts` | **컨텍스트 압축** — 긴 대화를 요약해 윈도우 유지 (에이전트 최대 난제) |
| `session/prompt.ts`, `system.ts` | 시스템 프롬프트 동적 조립 |
| `session/retry.ts` | 실패/레이트리밋 재시도 전략 |
| `tool/registry.ts`, `tool/schema.ts` | 도구 등록·스키마 검증 구조 |
| `tool/truncation.ts` | 거대 출력 잘라내기 (컨텍스트 폭발 방지) |
| `permission/` | **위험 작업 권한 게이트** — 프로덕션 에이전트 필수 |
| `tool/task.ts`, `agent/` | 서브에이전트 오케스트레이션 |
| `lsp/` | LSP로 코드 이해도 끌어올리기 |
| `mcp/` | MCP 클라이언트 구현 참고 |
| `session/revert.ts`, `snapshot/` | 에이전트 변경 되돌리기 |
| `server/` + `sdk/` | 헤드리스 API + 타입 세이프 클라이언트 |

### 가장 실용적인 활용: 로컬 LLM 붙이기
```jsonc
{
  "provider": {
    "local": {
      "npm": "@ai-sdk/openai-compatible",
      "options": { "baseURL": "http://localhost:11434/v1", "apiKey": "x" },
      "models": { "qwen3-coder": { "name": "Qwen3 Coder" } }
    }
  }
}
```
Chipotle 개조 방식 그대로 → Ollama/vLLM/LM Studio 연결. **완전 무료 + 완전 합법 + 오프라인.**

---

## 10. 수익화 아이디어 (합법 노선만)

> 전제: Pepper 자체 수익화는 **절대 금지**. TOS 위반 + 상표권 + 언제든 차단.
> 아래는 전부 **OpenCode(MIT) 기반**의 합법 노선.

### A. 사내 전용 에이전트 구축 SaaS ⭐ 추천
- **문제**: 금융/의료/공공은 데이터 외부 유출 금지 → Cursor/Copilot 도입 불가.
- **해법**: OpenCode + 사내 GPU(vLLM) 온프레미스 배포 + 감사로그/권한(`permission/` 활용).
- **모델**: 구축비 + 월 유지보수, 또는 시트당 과금.
- **왜 되나**: `enterprise/` 패키지와 SST 인프라 코드가 이미 있어 시작점이 좋다.

### B. 도메인 특화 에이전트 배포판
- Laravel/WordPress 전용, React Native 전용, 레거시 마이그레이션 전용 등.
- OpenCode에 도메인 스킬 + MCP + 서브에이전트 프리셋을 얹어 유료 배포판.
- **모델**: 월 구독 or 라이선스.

### C. 플러그인 / MCP 서버 마켓플레이스
- 이 프로젝트가 플러그인·MCP·스킬을 *소비*하는 호스트라는 점을 이용.
- 유료 MCP 서버(사내 위키, Jira, 결제 시스템 커넥터) 판매.
- **모델**: 개별 판매 + 마켓 수수료.

### D. 컨텍스트 압축 / 비용 최적화 미들웨어
- `session/compaction.ts`를 발전시켜 **토큰 비용 30~50% 절감** 프록시로 상품화.
- 캐싱 + 요약 + 모델 라우팅(쉬운 건 Haiku, 어려운 건 Opus).
- **모델**: 절감액의 일정 비율(성과 과금) — 설득력 최고.

### E. 코드리뷰 / CI 자동화 봇
- `opencode run` + `github/` 액션 코드를 활용한 PR 자동 리뷰 봇.
- **모델**: 리포지토리당 월 과금.

### F. 교육 콘텐츠 ⭐ 가장 빠른 현금화
- "AI 코딩 에이전트 직접 만들기" 강의/전자책 — 이 코드베이스를 교보재로.
- 밈 스토리("Chipotle 봇 해킹 사건")를 후킹으로 사용.
- **모델**: 강의 판매, 유튜브, 유료 뉴스레터, 기업 워크샵.

### G. 리버스 엔지니어링 기술의 합법 전환
- Amelia/SockJS 해체 기술 → **합법 레거시 API 래핑 컨설팅**.
- 구형 SOAP/WebSocket 사내 시스템을 REST/OpenAI 호환으로 현대화.
- **모델**: 프로젝트 단위 고액 계약.

### 우선순위 추천
1. **F (교육)** — 자본 0, 즉시 시작, 스토리 있음
2. **A (사내 에이전트 구축)** — 단가 높고 수요 확실
3. **D (비용 최적화)** — 성과 과금으로 진입장벽 낮음

---

## 11. React나 PHP로 만들 수 있는가

### React → ✅ 이미 그렇게 만들어져 있다
| 레이어 | 실제 기술 |
|---|---|
| 웹 UI | **SolidJS** (`packages/app`, `ui`, `web`) — React와 JSX 문법 거의 동일 |
| 터미널 UI | **SolidJS 렌더러** (`.tsx`로 TUI 작성) |
| 데스크톱 | Tauri / Electron |
| 서버 | Hono (`server/`) |
| 런타임 | Bun + TypeScript |

**React로 하려면**: `opencode serve`로 헤드리스 서버를 띄우고, `@opencode-ai/sdk`로 React 프론트만 새로 만들면 된다. **코어를 다시 쓸 필요가 전혀 없다.** 가장 가성비 높은 루트.

```tsx
import { createOpencode } from "@opencode-ai/sdk"
const { client } = await createOpencode()
// React 컴포넌트에서 세션 생성 / 메시지 전송 / 스트리밍 수신
```

### PHP → ⚠️ 가능하지만 권장 안 함
**가능한 것**
- PHP가 `opencode serve`(HTTP/OpenAPI)를 **호출하는 클라이언트**: ✅ 매우 쉬움. Laravel 대시보드 + Guzzle로 충분.
- 간단한 에이전트 루프를 PHP로 직접 구현: ✅ 가능 (OpenAI 호환 API 호출 + 함수 호출 파싱 + 도구 실행).

**PHP가 불리한 이유**
| 요구사항 | PHP 현실 |
|---|---|
| 스트리밍 응답 | SSE 처리가 번거롭고 FPM 워커를 오래 점유 |
| 장시간 실행 | `max_execution_time`, 요청-응답 모델과 충돌 |
| 병렬 도구 실행 | 동시성 약함 (ReactPHP/Swoole 별도 도입 필요) |
| 상태 유지 세션 | 요청마다 초기화 → 외부 저장소 의존 |
| LSP 연동 | 장수 프로세스 관리가 어려움 |
| 생태계 | AI SDK / MCP 라이브러리가 거의 없음 |

**결론적 추천 아키텍처**
```
[React 프론트엔드]  ←→  [OpenCode 헤드리스 서버 (Bun/TS)]
        ↑
[Laravel/PHP]  ←→  인증·과금·관리자 백오피스 담당
```
에이전트 코어는 TS에 맡기고, PHP는 잘하는 영역(웹 백오피스, 결제, 인증)에 쓰는 하이브리드가 정답.

---

## 12. 최종 요약

| 관점 | 평가 |
|---|---|
| **실사용 가치 (Pepper)** | ❌ 거의 0 — 서브모듈 미초기화, 컨텍스트 8192, 법적 리스크 |
| **학습 가치 (OpenCode 코어)** | ⭐⭐⭐⭐⭐ 에이전트 아키텍처 최고급 교본 |
| **실무 전환 가치** | ⭐⭐⭐⭐⭐ 모델만 바꾸면 무료·합법·오프라인 로컬 에이전트 완성 |
| **수익화 가치** | ⭐⭐⭐⭐ 밈이 아니라 OpenCode 기반으로 접근할 때 |

**핵심 한 줄**: 🌯는 껍데기, 알맹이는 **OpenCode**다. Pepper는 떼어내고 코어만 가져가면 최고의 자산이 된다.
