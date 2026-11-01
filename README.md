# OHiggs-Unified — 통합 저장소 (OH + Comfy 연동)

**OHiggs-Unified**는 두 저장소를 **git 서브모듈**로 묶은 메타(유니파이드) 저장소입니다.
한 번에 클론·갱신하고, **OH 화면 → comfy-webui → ComfyUI**로 이어지는 로컬 생성 경로를
함께 돌리는 것이 목적입니다.

**이 저장소의 실제 구성**(`.gitmodules` 기준):

| 서브모듈 path | 실제 리포 |
|---|---|
| `openhiggs/`    | `baramofme/open-higgsfield` (OH, Next.js 생성 UI) |
| `comfy-webui/`  | `baramofme/comfy-webui` (Gradio+ComfyUI 생성 백엔드/WaaS, :7861) |

> **주의**: 생성 백엔드 **comfy-webui**(:7861)는 이제 이 메타 저장소의 **서브모듈**입니다
> (path `comfy-webui/`). 반면 실제 **ComfyUI** 백엔드(:8189)는 메타 밖의 런타임 서비스로
> 계속 별도로 돌아갑니다. 아래 4-4 런칭 절이 이 구분을 전제로 합니다.
>
> 이 저장소는 2026-11-01 서브모듈 재구성으로 이전 `comfy` 래퍼(시드 커밋)를 제거하고
> comfy-webui를 서브모듈로 교체했습니다. 과거 클론은 `git submodule update --init --recursive`
> 후 갱신 시 경로를 `comfy/` → `comfy-webui/`로 바꿔 주세요.
>
> 아래 예시 커맨드에서 서브모듈 경로는 실 path(`openhiggs/`, `comfy-webui/`)를 씁니다.
> 애매하면 `git submodule status`와 `cat .gitmodules`로 실제 경로를 먼저 확인하세요.

---

## 1. 저장소 구조

| 서브모듈 | 역할 | 실행 포트 | GitHub |
|---|---|---|---|
| `openhiggs/` (OH) | Next.js 생성 UI 프론트엔드 | **3100** (실 운영. `npm run dev` 기본값은 3000) | `baramofme/open-higgsfield` |
| `comfy-webui/` | Gradio+ComfyUI 생성 백엔드(WaaS 래퍼) | **7861** | `baramofme/comfy-webui` |

- 실제 생성은 메타 서브모듈인 **comfy-webui**가 **7861**에서 Gradio+ComfyUI 래퍼(WaaS)로,
  그 뒤의 **ComfyUI 백엔드**(메타 밖 서비스)가 **8189**에서 돌아갑니다. 라이브 리스너는 `ss -ltn`으로 찍어 확인합니다
  (`config.yaml`의 기본 포수와 다를 수 있으니 반드시 실 포트를 확인).
- 모델·입출력 실체는 별도 공유 디렉토리(`ComfyUI-Shared/`)에 있고, comfy-webui의 `allowed_paths`가
  그 `output/`을 Gradio로 노출합니다. (연동절 3번 참조)

---

## 2. 클론 — 서브모듈까지 한 번에

메타 저장소만 클론하면 서브모듈 디렉토리는 **빈 껍데기**로 남습니다. 반드시 재귀 플래그를 붙입니다.

```bash
git clone --recurse-submodules https://github.com/baramofme/OHiggs-Unified.git
cd OHiggs-Unified
```

이미 재귀 없이 클론해버렸다면, 한 번에 초기화합니다:

```bash
git submodule update --init --recursive
```

서브모듈이 제대로 붙었는지 확인:

```bash
git submodule status
# 각 서브모듈이 커밋 SHA + (detached HEAD) 로 찍히면 정상
```

> `comfy-webui`(백엔드)도 서브모듈이므로 위 재귀 클론/`--init --recursive`로 함께 붙습니다 — 별도 클론 불필요.

---

## 3. 서브모듈 갱신

### 3-1. 부모(메타)가 가리키는 고정 커밋으로 되돌리기

```bash
git submodule update --init --recursive
```

메타 저장소가 "이 커밋을 써라"고 못 박아둔 SHA로 checkout 합니다. 재현성·안정성 기본값.

### 3-2. 각 서브모듈의 최신 upstream으로 따라올리기

```bash
git submodule update --remote --merge
```

`--remote`는 서브모듈이 추적하는 원격 브랜치의 **최신**을 가져옵니다. 갱신 후 메타 저장소는
"새 SHA를 가리키겠다"는 변경이 생기므로, 그 가리킴(gitlink SHA)을 커밋해 확정합니다:

```bash
git add openhiggs comfy-webui
git commit -m "bump: 서브모듈 최신 커밋으로 갱신"
git push
```

### 3-3. 두 서브모듈을 한 번에 pull (동시 갱신)

각 서브모듈 안에서 자기 브랜치를 당기는 가장 실용적인 방법:

```bash
git submodule foreach 'git fetch --all && git checkout main && git pull --ff-only'
```

- 서브모듈은 기본이 **detached HEAD**라 `pull`이 애매할 수 있습니다. 그래서 `foreach`로 먼저 자기 브랜치(`main`)에 붙이고 당깁니다. 서브모듈 추적 브랜치가 `main`이 아니면 `checkout`의 브랜치명을 바꿔 주세요.
- 또는 전체를 메타 기준으로 한 번에:

```bash
git pull --recurse-submodules
```

### 3-4. 서브모듈 브랜치 확인/전환

```bash
# 특정 서브모듈 상태를 보는 shortcut
git -C openhiggs branch --show-current
git -C comfy-webui branch --show-current

# 메타가 지정한 SHA가 아니라 자기 원격 브랜치를 추적하게 하려면 .gitmodules의 branch 확인
cat .gitmodules
```

---

## 4. 두 저장소가 연동되는 방식 (X-bridge)

OH의 플랫폼 transport가 **comfy-webui를 직접 호출**하도록 리와이어된 상태입니다(별도 shim 서버 없음).
즉 UI 위젯은 OH에 이미 있고, 이식 단위는 **transport + 모델별 카탈로그 매퍼**입니다.
(comfy-webui는 이 메타의 **서브모듈**입니다 — path `comfy-webui/`, 클론 절 참조)

### 4-1. LAN 모드 스위치 (핵심)

OH의 `openhiggs/.env.local` 안에 이 한 줄이 켜져 있으면 **로컬(무인증) 모드**로 comfy-webui를 호출합니다:

```bash
CFWUI_BASE_URL=http://127.0.0.1:7861
```

- **있으면** → 클라이언트 팩토리가 cfwui 클라이언트를 고르고, readCredentials가 더미(`local:local`)를 반환 → 로컬 ComfyUI로 생성.
- **없으면** → 원래 Higgsfield 클라우드 경로가 그대로 유지됩니다.
- 이 줄은 커밋하지 말고 `.env.local`(gitignore)에만 두세요. `.env.example`에는 클라우드용(`HF_API_BASE_URL`, `OPEN_HIGGSFIELD_READ_WRITE_TOKEN`)만 들어 있습니다.

### 4-2. transport 계약 (Gradio queue REST)

경로: 브라우저 → OH Next 서버 액션 → comfy-webui의 Gradio queue REST → ComfyUI 백엔드.

```bash
# 제출: POST {base}/gradio_api/call/{api_name}  body {"data":[params]}  -> {"event_id": ...}
# 조회: GET  {base}/gradio_api/call/{api_name}/{event_id}               -> SSE (event: complete = JSON 배열)
# 이름 목록: GET {base}/gradio_api/info                                  (dict keyed by /Name)
```

- **기계를 읽을 때 MCP JSON-RPC 채널을 쓰지 마세요.** 그 `result.content[0].text`는 Python dict repr(따옴표 한쪽)이고 `structuredContent`가 없습니다. queue REST가 클린 JSON 채널입니다.
- 완결 시 `result.images`는 **URL 문자열 리스트**(이미지), 영상은 mp4 URL 하나. 실패 시 top-level `error:{code,message}`.

### 4-3. 모델 경로 & 출력 노출 (연동의 가장 조용히 깨지는 부분)

- 모델 트리: `ComfyUI-Shared/models/` 아래 카테고리별(`diffusion_models`, `clip`, `vae`, `loras`, `text_encoders` …). OH 카탈로그가 cfwui의 injector `max_count`(role cap)를 미러로 씁니다.
- **출력 URL 재작성**: `GRADIO_SERVER_NAME=0.0.0.0`로 띄우면 host가 `0.0.0.0`이 됩니다. OH가 **ingest 시점에** origin을 재작성(경로 유지)하므로 저장된 run URL은 이미 정규화됩니다. 갈색/빈 갤러리는 호스트/재작성 문제가 아니라 **파일 실재·history·렌더링** 쪽을 추적하세요.
- **403 함정**: Gradio는 `allowed_paths` 아래 파일만 `/gradio_api/file=` 으로 서빙합니다. 이 환경은 `allowed_paths=[COMFYUI_OUTPUT_PATH]`이고 **TMPDIR 아래 파일은 403** → 갤러리는 살아있는데 asset-picker 카드가 죽어 보인다면 산출물 경로를 `output/` 실제 파일이 있는 디렉토리로 박습니다(백엔드가 컨테이너면 **bind-mount의 호스트 경로**로 `COMFYUI_OUTPUT_PATH` 지정). 파일 URL에 `curl` → `200 image/png` 확인. 403 = 경로/allowed set 오설정.

### 4-4. 띄우기 (3개 레이어를 순서대로)

```bash
# 1) ComfyUI 백엔드 (:8189) — 보통 이미 떠 있음. 실 포트 확인:
ss -ltn | grep ':8189' && curl -s http://127.0.0.1:8189/object_info >/dev/null && echo "ComfyUI OK"

# 2) comfy-webui (:7861) — 이 메타의 서브모듈 comfy-webui/ (COMFYUI_BACKEND_DEFAULT를 라이브 ComfyUI 포트에 맞춘다)
cd comfy-webui   # ← 이 메타의 서브모듈
GRADIO_SERVER_NAME=0.0.0.0 GRADIO_SERVER_PORT=7861 \
  COMFYUI_BACKEND_DEFAULT=http://127.0.0.1:8189 \
  .venv/bin/python app.py

# 3) OH (:3100) — .env.local에 CFWUI_BASE_URL이 켜져 있는지 확인 후 dev 실행
cd openhiggs     # ← 이 메타의 서브모듈
# .env.local: CFWUI_BASE_URL=http://127.0.0.1:7861
npx next dev -p 3100
```

- `COMFYUI_BACKEND_DEFAULT`가 `config.yaml`의 기본 포수를 **_override_**합니다. 컨테이너가 `8189→8188`로 매핑되어 HOST에 `:8188` 리스너가 없다면 라이브 ComfyUI는 컨테이너의 `:8189`입니다. 반드시 실 포트로.
- gradio가 부팅된 것만으로 생성이 되는 게 아닙니다. 앱 로그에 `Loaded ComfyUI backends from environment variables.` 와 `NodeInfoManager … nodes from 1 backend(s)` 두 줄이 떠야 백엔드 override가 실제로 먹은 것입니다.
- 이미지 업로드는 Vercel Blob 대신 **base64 data URI**(클라이언트 생성)를 씁니다. cfwui가 `data:image/...`, http(s) URL, 로컬 경로를 모두 받므로 업로드 서비스·공개 URL이 필요 없습니다.

---

## 5. 자주 쓰는 커맨드 모음

```bash
# 메타 저장소 + 서브모듈 상태를 한눈에
git submodule status
git submodule summary

# 두 서브모듈 동시 pull (자기 브랜치 main 기준)
git submodule foreach 'git fetch --all && git checkout main && git pull --ff-only'

# 서브모듈 갱신 후 메타의 gitlink SHA를 커밋으로 확정
git add openhiggs comfy-webui && git commit -m "bump submodules" && git push

# 서브모듈을 latest로 따라올리고 커밋
git submodule update --remote --merge && git commit -am "latest submodules"

# 한쪽 서브모듈만 갱신
git submodule update --remote comfy-webui

# 서브모듈 안에서 브랜치 작업 후 메타에 반영: (서브모듈 push) -> 메타의 gitlink SHA 갱신 커밋
git -C openhiggs push origin main
```

> 서브모듈에서 코드를 고쳐 push 한 뒤에는 반드시 **메타 저장소에서 그 gitlink(SHA)를 `git add`로 커밋**해야
> 다른 클론이 같은 commit을 받습니다. 서브모듈 push만 하고 메타를 안 고치면 메타는 여전히 옛 SHA를 가리킵니다.

---

## 6. 트러블슈팅

**A. 서브모듈 디렉토리가 비어 있다 (clone 후 내용 없음)**
- 재귀로 안 클론했을 확률이 높습니다. → `git submodule update --init --recursive`

**B. `git submodule status`에 `-` 표시가 붙는다 (미초기화)**
- 해당 경로가 아직 init 안 됨. → `git submodule update --init <path>`

**C. `git pull` 후에도 서브모듈 코드가 안 바뀐다**
- 일반 `git pull`은 서브모듈을 건드리지 않습니다. → `git pull --recurse-submodules` 또는 `git submodule update --init --recursive`

**D. 서브모듈이 detached HEAD라 브랜치가 "없어" 보인다**
- 정상 동작입니다(메타가 SHA로 고박). 작업하려면 자기 브랜치로: → `git -C openhiggs checkout main`
- 추적 브랜치를 메타에 박아두고 싶으면 → `git config -f .gitmodules submodule.openhiggs.branch main` 후 `git submodule update --remote`

**E. `.gitmodules`의 path가 예시와 다르다**
- 실제 경로를 기준으로 명령을 재작성: → `cat .gitmodules` 로 `[submodule "..."] path = ...` 확인

**F. 갤러리는 뜨는데 asset-picker/편집 캔버스가 빈 화면**
- 산출물 파일이 지워졌거나 **allowed_paths** 밖(TMPDIR 403)일 수 있음. → 스킬 `scripts/task_truth.sh <gradio_base> <task_id>` 로 TASKS_DB의 실 산출 경로를 읽고 그 URL을 `curl` 해서 `200` + ~MB 본문을 확인(가정 디렉토리는 2차 증거일 뿐).
- 또는 **PNA(CORS) 함정**: `localhost` 페이지에서 loopback(`127.0.0.1:7861`)을 CORS 모드(`<img crossOrigin="anonymous">`)로 부르면 preflight가 죽어 `ERR_FAILED`. 평범한(non-CORS) `<img>`은 살아있는데 편집 캔버스만 죽는 클래스. → OH의 same-origin 라우트(`/api/ohf-image?file=<abs path>`)로 프록시하여 동일- 오리진으로 바이트를 서비스(예: OH 리포에 이미 구현). 갤러리/viewer-main는 raw URL 유지.

**G. 생성이 "backend에 queue 실패"로 죽는다 (EXECUTION_ERROR)**
- 400은 ComfyUI가 **살아있는데 그래프 내용을 거부**했다는 뜻(연두절 아님). message의 포트를 믿고 `ss -ltn` + `curl :<port>/object_info` 로 확인.
- 주범은 대개 **transformer/unet 파일명 mismatch** — 카탈로그가 특정 quant 변형(예 `…-nvfp4.safetensors`)을 하드코딩했는데 실제 설치 파일이 다른 변형(`…-int8-convrot.safetensors`)인 경우. → 편집 대신 **비침습 symlink**: `ln -s <있는파일> <원하는이름>.safetensors` 를 `ComfyUI-Shared/models/diffusion_models/` 에(메타 밖 ComfyUI 설치·공유 트리, 서브모듈 path 아님). 재스캔 후 `/object_info`의 UNETLoader enum에 이름이 뜨면(무재기동) 같은 task_type 재제출.

**H. cfwui(:7861)가 죽고 ComfyUI는 살아있어 갤러리 전채가 빈다**
- 시스템드 유닛이 없어 자율 재기동합니다. `ss -ltn | grep :7861` + `curl :7861/`(HTTP=000/리스너 없음 = cfwui 다운)로 먼저 진단.
- 재기동: `cd comfy-webui && GRADIO_SERVER_NAME=0.0.0.0 GRADIO_SERVER_PORT=7861 COMFYUI_BACKEND_DEFAULT=http://127.0.0.1:<라이브_Comfy포트> .venv/bin/python app.py` (백그라운드·세션종료 유지). 라이브 Comfy 포트는 `ss -ltn` / `curl :<port>/object_info`(200) 로 실측 — config 기본 포수는 아님.

**I. img2img/inpaint가 파일권한으로 죽는다 (Errno 13, 400 아님)**
- 파이프라인이 `{comfyui_path}/input/temp_image_*.png` 를 씁니다. 컨테이너 bind-mount에서 `input/`이 root 소유인데 cfwui가 호스트 유저로 돌면 권한 거부. → `sudo chown root:<user> …/input && sudo chmod 775` 로 컨테이너-root와 호스트-user 모두 쓰게.

**J. dev-server 재시작 시 포트가 조용히 꺼진다**
- `pkill -f "next dev"` 같은 패턴킬은 패턴이 살해 커맨드 자신의 cmdline에도 있어 자기 자신을 SIGTERM하고 wrapper와 함께 포트가 크래시 라인 없이 꺼집니다. → dev 서버를 `exec`로 띄워 트래킹되는 프로세스가 `next`가리게 하고, 패턴킬은 절대 커맨드 안에 그 문자열이 있게 하지 마세요. 재시작 확인은 `ss -tln` + `curl`. 포트 소유 pid 만 자르기(`ss -ltnp | grep :<port>`).

---

## 부록: 관련 스킬·문서

- 연동 계약 전체: 스킬 **`oh-cfwui-x-bridge`**(OH↔comfy-webui X-bridge, Gradio queue REST, 출력 URL/allowed_paths, 모델 파일명·symlink 절).
- 정본 계획/인계: `/home/baramofme/oh-cfwui-ui-transplant-plan.md`, `/home/baramofme/oh-cfwui-KICKOFF.md`.
- 태스크 실체 검증(서버와 UI가 어긋날 때): `scripts/task_truth.sh`, 영상 재현: `scripts/cfwui_video_test.py`.

*이 README는 클론/갱신/연동/트러블슈팅을 다루며 모든 커맨드는 복사-붙여넣기 가능합니다.*
