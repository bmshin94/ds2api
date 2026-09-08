# DS2API 스터디 노트 (한국어)

> 이 문서는 DS2API 프로젝트를 처음 접한 사람이 **"이게 뭐하는 건지 / 어떻게 쓰는지 / 어디에 도움이 되는지"**를
> 빠르게 파악할 수 있도록 정리한 한국어 학습 노트입니다.
> 공식 문서가 아니라 **개인 학습용 요약본**이며, 정확한 내용은 항상 원본 문서를 우선하세요.

## 저장소 주소

| 구분 | 주소 |
| --- | --- |
| 원본 저장소 (Upstream) | https://github.com/CJackHwang/ds2api |
| 이 포크 저장소 (Fork) | https://github.com/bmshin94/ds2api |
| Releases (실행파일 다운로드) | https://github.com/CJackHwang/ds2api/releases |
| 컨테이너 이미지 (GHCR) | `ghcr.io/cjackhwang/ds2api:latest` |

- 분석 기준 버전: **v4.6.1** (`VERSION` 파일 기준)
- 라이선스: **AGPL-3.0** (`LICENSE`)

## 목차

- [1. 한 줄 요약](#1-한-줄-요약)
- [2. 동작 원리](#2-동작-원리)
- [3. 폴더 구조 분석](#3-폴더-구조-분석)
- [4. 핵심 기능 정리](#4-핵심-기능-정리)
- [5. 설치 및 사용법](#5-설치-및-사용법)
- [6. 클라이언트 연결 방법](#6-클라이언트-연결-방법)
- [7. 모델 목록](#7-모델-목록)
- [8. 트러블슈팅](#8-트러블슈팅)
- [9. React / PHP 로도 만들 수 있을까?](#9-react--php-로도-만들-수-있을까)
- [10. 수익화 관련 정리](#10-수익화-관련-정리)
- [11. 주의사항 (필독)](#11-주의사항-필독)

---

## 1. 한 줄 요약

> **DS2API = DeepSeek 웹 채팅 기능을 OpenAI / Claude / Gemini API 형식으로 바꿔주는 변환 계층(어댑터)**

비유하자면 **해외여행용 콘센트 어댑터**입니다.

- 대부분의 앱/툴은 "OpenAI API 모양 콘센트"에 꽂도록 만들어져 있음
- DeepSeek 웹 채팅은 콘센트 모양이 다름
- DS2API가 그 사이에 끼어서 **모양을 맞춰 줌**

즉, 내 컴퓨터에 **"가짜 OpenAI 서버"를 하나 띄우는 것**과 같습니다.

---

## 2. 동작 원리

```
[내 앱 / Cursor / Claude Code]
        |  "OpenAI API 형식으로 요청"
        v
[ DS2API (로컬에서 실행) ]
        |  요청을 '웹 채팅용 순수 텍스트'로 번역
        v
[ DeepSeek 웹 서버 ]
        |  응답 (SSE 스트림)
        v
[ DS2API 가 다시 OpenAI/Claude/Gemini 형식으로 재조립 ]
        v
[ 내 앱은 정상적인 API 응답으로 인식 ]
```

핵심 처리 단계:

1. **요청 정규화** — OpenAI / Claude / Gemini 각각 다른 메시지 구조를 하나의 내부 표준 모델로 통일
2. **웹 컨텍스트 변환** (`promptcompat`) — role/content 배열 → 웹 채팅에 넣을 수 있는 긴 순수 텍스트
3. **세션 준비 + PoW** — DeepSeek이 요구하는 작업증명(Proof of Work) 계산
4. **업스트림 호출** — 실제 DeepSeek 웹 completion API 호출
5. **출력 정규화** (`assistantturn`) — thinking / 본문 / 인용 / 도구호출 분리
6. **프로토콜별 렌더링** — 요청이 들어온 규격(OpenAI/Claude/Gemini)에 맞춰 응답 재조립

> 설계 원칙: `AGENTS.md`에 **"프로토콜별 포맷이 공통 비즈니스 로직을 소유하면 안 된다"**고 명시되어 있음.
> → 요청은 먼저 표준화하고, 공통 로직은 한 곳에서 실행하고, 마지막 경계에서만 프로토콜별로 렌더링.

---

## 3. 폴더 구조 분석

| 경로 | 역할 |
| --- | --- |
| `cmd/ds2api/main.go` | 프로그램 진입점 (시동 버튼) |
| `internal/httpapi/` | HTTP API 창구. OpenAI / Claude / Gemini / Ollama / Admin 라우트 |
| `internal/promptcompat/` | **핵심 1.** API 메시지 → 웹 채팅용 순수 텍스트 변환 |
| `internal/completionruntime/` | **핵심 2.** 세션 생성 / PoW / 실제 completion 실행 |
| `internal/assistantturn/` | 출력 의미 정규화 (thinking / 본문 / 인용) |
| `internal/toolcall/`, `internal/toolstream/` | Tool Calling 감지 및 구조화 (누수 방지 처리 포함) |
| `internal/account/` | 계정 풀, 동시 실행 제한, 대기열 관리 |
| `internal/deepseek/` | DeepSeek 통신 클라이언트 (utls로 TLS 지문 위장) |
| `internal/auth/` | API key / bearer / x-goog-api-key 인증 처리 |
| `internal/sse/`, `internal/stream/` | SSE 스트리밍 파싱 및 재조립 |
| `internal/config/` | 설정 로딩 (파일 / 환경변수 / Base64) |
| `webui/` | React 관리자 페이지 소스 (빌드 결과는 `static/admin/`) |
| `api/index.go`, `api/chat-stream.js` | Vercel 배포용 진입점 + Node 스트리밍 브릿지 |
| `docs/` | 아키텍처 / 배포 / 테스트 / 기여 가이드 |
| `tests/` | Go + Node 테스트, 실제 SSE 원본 샘플 포함 |
| `config.example.json` | 설정 템플릿 |
| `Dockerfile`, `docker-compose.yml`, `vercel.json`, `zeabur.yaml` | 배포 설정 |

---

## 4. 핵심 기능 정리

| 기능 | 설명 |
| --- | --- |
| OpenAI 호환 | `/v1/chat/completions`, `/v1/responses`, `/v1/models`, `/v1/embeddings`, `/v1/files` |
| Claude 호환 | `/anthropic/v1/messages`, `/v1/messages`, `count_tokens` |
| Gemini 호환 | `/v1beta/models/{model}:generateContent`, `:streamGenerateContent` |
| Ollama 호환 | `/api/version`, `/api/tags`, `/api/show` |
| 모델 alias | `gpt-4o`, `claude-sonnet-4-6`, `gemini-2.5-pro` 등으로 불러도 DeepSeek 모델로 자동 매핑 |
| 멀티 계정 | 여러 DeepSeek 계정 자동 로테이션 + 토큰 자동 갱신 |
| 동시성 제어 | 계정별 in-flight 제한 + 대기열. 한도 초과 시에만 `429` |
| PoW | DeepSeekHashV1을 순수 Go로 구현 (밀리초 단위) |
| Tool Calling | DSML 태그 기반 도구 호출 파싱, 텍스트 누수 방지 |
| Admin API + WebUI | `/admin` 에서 설정 핫 리로드, 계정 테스트, 대화기록 조회 |
| 운영 프로브 | `/healthz` (liveness), `/readyz` (readiness) |

---

## 5. 설치 및 사용법

### 5.0 준비물

- DeepSeek 계정 (https://chat.deepseek.com) — **부계정 사용 권장**
- (소스 실행 시) Go 1.26+ / Node.js 20.19+ 또는 22.12+

### 5.1 방법 A — Release 실행파일 (가장 쉬움, 권장)

1. https://github.com/CJackHwang/ds2api/releases 에서 OS에 맞는 파일 다운로드
   - Windows: `..._windows_amd64.zip`
   - macOS (Apple Silicon): `..._darwin_arm64.tar.gz`
   - macOS (Intel): `..._darwin_amd64.tar.gz`
   - Linux: `..._linux_amd64.tar.gz`
2. 압축 해제
3. `config.example.json` → `config.json` 으로 복사 후 편집
4. 실행
   - Windows: `ds2api.exe` 더블클릭
   - macOS / Linux: `chmod +x ds2api && ./ds2api`
5. 확인: 브라우저에서 `http://127.0.0.1:5001/healthz` → `{"status":"ok"}`

### 5.2 최소 `config.json` 예시

```json
{
  "keys": [
    "sk-mykey1234"
  ],
  "accounts": [
    {
      "name": "내계정",
      "email": "내딥시크이메일@example.com",
      "password": "내딥시크비밀번호"
    }
  ],
  "runtime": {
    "account_max_inflight": 2
  }
}
```

| 항목 | 의미 |
| --- | --- |
| `keys` | **내가 직접 정하는 클라이언트 접근 키.** Cursor 등에 넣을 "가짜 API 키". `sk-` 로 시작하면 호환성 좋음 |
| `accounts` | **실제 DeepSeek 로그인 정보.** 휴대폰 가입이면 `email` 대신 `mobile` 사용 |
| `accounts` 여러 개 | 배열에 계정을 추가하면 자동 로테이션되어 처리량 증가 |

### 5.3 방법 B — Docker

```bash
cp .env.example .env
cp config.example.json config.json
# config.json 편집 (위 5.2 참고)
# .env 편집: DS2API_ADMIN_KEY=강한비밀번호, DS2API_HOST_PORT=5001

docker-compose up -d      # 실행
docker-compose logs -f    # 로그 확인
docker-compose down       # 종료
```

> 주의: `DS2API_HOST_PORT`를 지정하지 않으면 호스트 포트 기본값이 **6011** 입니다 (컨테이너 내부는 항상 5001).
> `config.json` 권한 문제가 나면 `chmod 644 config.json`.

### 5.4 방법 C — 소스 실행 (개발/학습용)

```bash
git clone https://github.com/CJackHwang/ds2api.git
cd ds2api

cp config.example.json config.json
# config.json 편집

DS2API_ADMIN_KEY=내관리자비번 go run ./cmd/ds2api
```

- 기본 주소: `http://127.0.0.1:5001` (실제 바인딩은 `0.0.0.0:5001`, `PORT`로 변경 가능)
- WebUI 정적 파일이 없으면 첫 실행 시 자동으로 npm 빌드를 시도함
- 수동 빌드: `./scripts/build-webui.sh`
- 자동 빌드 끄기: `DS2API_AUTO_BUILD_WEBUI=false`
- 바이너리 빌드: `go build -o ds2api ./cmd/ds2api`

### 5.5 관리자 화면

`http://127.0.0.1:5001/admin` 접속 → `DS2API_ADMIN_KEY`로 로그인

| 탭 | 기능 |
| --- | --- |
| 계정 관리 | 계정 추가/삭제, 연결 테스트, 대기열 상태 |
| API 키 | 클라이언트 키 발급/관리 |
| 설정 | 모델 매핑, 동시성, 자동 삭제 정책 등 **재시작 없이 반영** |
| API 테스터 | 브라우저에서 바로 대화 테스트 |
| 대화 기록 | 서버에 저장된 대화 조회 |
| 프록시 | 프록시 설정 |

> 처음 세팅했다면 **"계정 테스트"** 를 먼저 눌러 로그인 성공 여부를 확인하세요.

### 5.6 배포 후 점검 체크리스트

```bash
curl -s http://127.0.0.1:5001/healthz   # {"status":"ok"}
curl -s http://127.0.0.1:5001/readyz    # {"status":"ready"}
curl -s http://127.0.0.1:5001/v1/models # 모델 목록
curl -s -o /dev/null -w "%{http_code}" http://127.0.0.1:5001/admin  # 200
```

---

## 6. 클라이언트 연결 방법

### 6.1 curl 로 먼저 확인

```bash
curl http://127.0.0.1:5001/v1/chat/completions \
  -H "Authorization: Bearer sk-mykey1234" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"안녕!"}]}'
```

### 6.2 Cursor / Continue / Cline

| 항목 | 값 |
| --- | --- |
| Base URL | `http://127.0.0.1:5001/v1` |
| API Key | `config.json`의 `keys` 값 |
| Model | `deepseek-v4-flash` |

### 6.3 Claude Code

```bash
export ANTHROPIC_BASE_URL=http://127.0.0.1:5001
export ANTHROPIC_API_KEY=sk-mykey1234
export NO_PROXY=127.0.0.1,localhost
```

- `ANTHROPIC_BASE_URL`에는 **`/v1`을 붙이지 않습니다** (루트 주소 사용)
- 시스템 프록시가 있으면 `NO_PROXY` 설정이 필수

### 6.4 Python (OpenAI SDK)

```python
from openai import OpenAI

client = OpenAI(base_url="http://127.0.0.1:5001/v1", api_key="sk-mykey1234")

res = client.chat.completions.create(
    model="deepseek-v4-flash",
    messages=[{"role": "user", "content": "안녕!"}],
)
print(res.choices[0].message.content)
```

### 6.5 JavaScript / TypeScript

```js
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "http://127.0.0.1:5001/v1",
  apiKey: "sk-mykey1234",
});

const res = await client.chat.completions.create({
  model: "deepseek-v4-flash",
  messages: [{ role: "user", content: "안녕!" }],
});
console.log(res.choices[0].message.content);
```

---

## 7. 모델 목록

| 모델 ID | 특징 |
| --- | --- |
| `deepseek-v4-flash` | 기본. 빠름 |
| `deepseek-v4-pro` | 고성능. 복잡한 추론/코딩 |
| `deepseek-v4-flash-search` / `deepseek-v4-pro-search` | 웹 검색 사용 |
| `deepseek-v4-vision` | 이미지 인식 |
| `*-nothinking` | 사고 과정(thinking) 강제 비활성화 → 더 빠름 |

- `gpt-4o`, `gpt-5`, `o3`, `claude-sonnet-4-6`, `gemini-2.5-pro` 등 **다른 벤더 모델명으로 호출해도 자동 매핑**
- 매핑은 `config.json`의 `model_aliases` 로 직접 수정 가능
- 기본 Claude 매핑: `claude-sonnet-4-6` → `deepseek-v4-flash`, `claude-opus-4-6` → `deepseek-v4-pro`

---

## 8. 트러블슈팅

| 증상 | 원인 / 해결 |
| --- | --- |
| 접속 불가 | 서버 미실행 또는 포트 오인 (Docker 기본 호스트 포트는 `6011`) |
| `401 Unauthorized` | 요청 키가 `config.json`의 `keys`와 불일치 |
| 로그인/계정 오류 | DeepSeek 계정 정보 오류. 브라우저에서 직접 로그인 확인 |
| `429 Too Many Requests` | 동시 요청 한도 초과. 계정 추가 또는 `account_max_inflight` 상향 |
| `429 upstream_empty_output` | 업스트림 빈 응답. 자동 재시도되며, 잦으면 계정 추가 권장 |
| `/admin` 404 | WebUI 미빌드. `./scripts/build-webui.sh` 실행 |
| Tool call이 텍스트로 출력됨 | 모델이 DSML 형식을 안 지킨 경우. `deepseek-v4-pro` 등 상위 모델 시도 |
| Claude Code 연결 실패 | `NO_PROXY` 설정 확인 + BASE_URL에 `/v1` 제거 |

로그 확인:

```bash
docker-compose logs -f          # Docker
LOG_LEVEL=DEBUG ./ds2api        # 직접 실행 시 상세 로그
```

---

## 9. React / PHP 로도 만들 수 있을까?

결론부터: **React 단독은 불가능, Node/Next.js는 가능, PHP는 가능하지만 매우 비효율적.**

### 9.1 React

- React는 **브라우저에서 도는 UI 라이브러리**라 서버 역할을 할 수 없음
- 브라우저에서 직접 DeepSeek을 호출하면 **CORS 차단 + 계정 비밀번호가 사용자에게 그대로 노출**됨
- 단, **관리자 화면 용도로는 이미 React가 쓰이고 있음** (`webui/`)
- 정리: **React = 프론트엔드 담당. 백엔드는 별도 필요.**

### 9.2 Node.js / Next.js — 가능 (현실적인 대안)

이 저장소도 **Vercel 배포 시 스트리밍 구간을 Node로 처리**하고 있습니다 (`api/chat-stream.js`).
즉 Node로 구현 가능하다는 것이 이미 저장소 안에서 증명되어 있습니다.

| 항목 | 평가 |
| --- | --- |
| SSE 스트리밍 | 매우 좋음 (Web Streams API) |
| 비동기 동시성 | 좋음 (이벤트 루프) |
| 프론트+백엔드 통합 | 최고 (Next.js 하나로 해결) |
| PoW 같은 CPU 연산 | 보통 (worker_threads 필요, Go보다 느림) |
| TLS 지문 위장 (utls) | 어려움 |
| 배포 | 좋음 (Vercel 등) |

### 9.3 PHP — 가능하지만 권장하지 않음

| 항목 | 평가 |
| --- | --- |
| SSE 스트리밍 | 가능하지만 출력 버퍼링(`ob_flush`) 이슈가 잦음 |
| 동시성 | 기본 PHP-FPM은 **요청당 프로세스** → 장시간 스트리밍 연결에 매우 취약 |
| 해결책 | Swoole / RoadRunner / ReactPHP / Workerman 같은 상주형 런타임 필요 |
| 백그라운드 토큰 갱신 | 별도 cron/데몬 필요 (PHP는 상주 프로세스가 기본이 아님) |
| PoW 계산 | 느림. C 확장이나 외부 프로세스 위임 필요 |
| TLS 지문 위장 | 사실상 불가능 |
| 배포 | 공유 호스팅에서는 사실상 불가 |

**PHP를 굳이 쓴다면** — 이미 Laravel 스택을 쓰고 있고, **PoW/TLS 위장이 필요 없는 정식 API 게이트웨이**를 만들 때 정도는 괜찮습니다 (Laravel + Octane + Swoole).

### 9.4 왜 원작자는 Go를 골랐을까

| 이유 | 설명 |
| --- | --- |
| 고루틴 | 수백 개의 동시 스트리밍 연결을 가볍게 처리 |
| 단일 바이너리 | 런타임 설치 없이 실행파일 하나로 배포 |
| `utls` | TLS 지문을 브라우저처럼 위장 (Go 생태계가 독보적) |
| CPU 성능 | PoW를 밀리초 단위로 계산 |
| 정적 파일 임베딩 | React 빌드 결과를 바이너리에 포함 |

### 9.5 추천 스택

| 만들려는 것 | 추천 |
| --- | --- |
| DS2API 같은 웹 리버싱 계층 | **Go** (다른 선택지가 사실상 없음) |
| 정식 API 기반 게이트웨이 | **Next.js(TypeScript)** 또는 Go |
| 관리자 UI | **React** (지금 구조 그대로) |
| 기존 PHP 팀의 사내 게이트웨이 | Laravel + Octane (스트리밍 검증 필수) |

---

## 10. 수익화 관련 정리

### 10.1 DS2API 자체로는 수익화 불가

| 이유 | 내용 |
| --- | --- |
| **라이선스** | AGPL-3.0. 네트워크 서비스로 제공만 해도 **전체 소스 공개 의무** 발생 |
| **명시적 금지** | README에 "상업적 라이선스를 제공하지 않으며, 상업적 사용 전 저자의 서면 허가 확인 필요" |
| **약관 위반** | 무료 웹 계정 자동화 재판매는 DeepSeek ToS 위반 → 계정 정지 및 법적 리스크 |

### 10.2 대신 가능한 방향

| # | 아이디어 | 핵심 |
| --- | --- | --- |
| 1 | **정식 API 기반 AI 게이트웨이** | 웹 리버싱 부분만 제거하고 공식 API로 교체. 프로토콜 변환/로드밸런싱/폴백/사용량 추적 로직은 그대로 가치 있음. 경쟁: LiteLLM, OpenRouter, Portkey |
| 2 | **기업 AI 인프라 구축 대행** | 사내 게이트웨이 구축, 키·예산 관리, 온프레미스 LLM 세팅. 진입 난이도가 가장 낮고 현금화가 빠름 |
| 3 | **버티컬 AI 서비스** | 계약서 검토, 상품설명 생성, 문제 출제, 상담일지 요약 등. 게이트웨이는 원가 절감 수단, 수익은 앱에서 |
| 4 | **기술 콘텐츠** | Go SSE 스트리밍, 프로토콜 어댑터 패턴, Tool Calling 파싱 — 국내 자료가 희소한 주제 |
| 5 | **커리어** | "LLM 게이트웨이 설계·구현 경험"은 현재 시장에서 희소 스킬. 기댓값이 가장 높음 |

> **중요**: 아이디어 1을 진행하더라도 **DS2API 코드를 복사하면 AGPL 전염 조건이 적용**됩니다.
> 아키텍처(설계 아이디어)만 참고하고 **처음부터 새로 구현**해야 합니다.

### 10.3 추천 로드맵

1. **1~2주** — DS2API 코드 이해 (`promptcompat`, `completionruntime`, `toolcall` 중심)
2. **2~4주** — 정식 API 기반 게이트웨이를 직접 새로 구현 (코드 복붙 금지, 설계만 참고)
3. **1개월** — 그 위에 버티컬 앱 하나 실제 출시
4. **병행** — 기술 블로그 연재 → 강의 / 컨설팅 / 이직 기회로 연결

---

## 11. 주의사항 (필독)

1. **`config.json`을 절대 Git에 커밋하지 마세요.** DeepSeek 비밀번호가 평문으로 들어갑니다.
   - 다행히 `.gitignore`에 `config.json`, `.env`가 이미 등록되어 있습니다.
2. **부계정을 사용하세요.** 이용약관상 계정 정지 가능성이 있습니다.
3. **`DS2API_ADMIN_KEY`를 반드시 변경하세요.** Docker Compose 기본값은 `ds2api` 입니다.
4. **외부에 공개 노출하지 마세요.** 로컬(`127.0.0.1`) 사용을 권장합니다.
5. **상업적 이용 금지.** 학습 / 연구 / 개인 실험 용도로만 사용하세요.
6. **업스트림 변경에 취약합니다.** DeepSeek 웹 구조가 바뀌면 동작이 깨질 수 있어 업데이트가 잦습니다.

---

## 참고 문서

| 문서 | 경로 |
| --- | --- |
| 전체 개요 | [README.MD](../README.MD) / [README.en.md](../README.en.md) |
| 아키텍처 | [docs/ARCHITECTURE.md](./ARCHITECTURE.md) |
| 배포 가이드 | [docs/DEPLOY.md](./DEPLOY.md) |
| API 명세 | [API.md](../API.md) |
| 프롬프트 호환 계층 | [docs/prompt-compatibility.md](./prompt-compatibility.md) |
| Tool Calling 의미 | [docs/toolcall-semantics.md](./toolcall-semantics.md) |
| 프로젝트 가치 | [docs/project-value.md](./project-value.md) |
| 에이전트 규칙 | [AGENTS.md](../AGENTS.md) |
