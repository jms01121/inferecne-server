# inferecne-server

llama.cpp 를 **LLGuidance(구조화 출력 / constrained decoding) 포함**으로 빌드해
Windows x64 배포 zip 을 만드는 GitHub Actions 리포지토리.

## 왜 직접 빌드하는가

llama.cpp 공식 릴리스 zip 에는 LLGuidance 가 들어 있지 않다.
`LLAMA_LLGUIDANCE` 옵션이 기본 `OFF` 이고 업스트림 릴리스 워크플로도 이 옵션을 켜지 않는다.

그리고 LLGuidance 는 **백엔드 DLL 이 아니라 실행파일에 정적 링크**된다.

```
llguidance (Rust 정적 lib) -> llama-common -> llama-cli.exe / llama-server.exe
ggml-cuda.dll / ggml-vulkan.dll  <- LLGuidance 와 무관
```

따라서 공식 zip 에 DLL 만 갈아끼우는 방식으로는 안 되고, 실행파일을 새로 빌드해야 한다.

## 사용법

**Actions → "Build llama.cpp (LLGuidance)" → Run workflow**

| 입력 | 기본값 | 설명 |
|---|---|---|
| `llama_version` | `v0.1.2` | llama.cpp 릴리스 태그. nightly 태그(`b10485`)나 커밋 SHA 도 가능 |
| `targets` | `all` | `all` / `cuda-12.4` / `cuda-13.3` / `vulkan` |
| `create_release` | `false` | 켜면 이 리포지토리에 Release 를 만들고 zip 을 첨부 |

CLI:

```bash
gh workflow run build-llamacpp-llguidance.yml \
  -f llama_version=v0.1.2 -f targets=all -f create_release=false
gh run watch
```

## 산출물

`llama_version=v0.1.2` 기준:

| GPU 종류 | 파일 |
|---|---|
| nVidia | `llama-v0.1.2-bin-win-cuda-12.4-x64.zip` + `cudart-llama-bin-win-cuda-12.4-x64.zip` |
| nVidia (CUDA 13) | `llama-v0.1.2-bin-win-cuda-13.3-x64.zip` + `cudart-llama-bin-win-cuda-13.3-x64.zip` |
| Intel / AMD / nVidia | `llama-v0.1.2-bin-win-vulkan-x64.zip` |

**CUDA 패키지는 본체 zip 과 `cudart-*` zip 을 같은 폴더에 풀어야 동작한다.**
`ggml-cuda.dll` 이 `cudart64_*.dll` / `cublas64_*.dll` 을 동적 링크하기 때문이다.

받기:

```bash
gh run download <RUN_ID> -D ./release
```

## 버전 이름 규칙

llama.cpp 는 같은 커밋에 태그를 **두 개** 붙인다.

| 태그 | 성격 | 업스트림 용도 |
|---|---|---|
| `v0.1.2` | 시맨틱 버전 (현재 pre-release) | 릴리스 노트 / 체인지로그 |
| `b10485` | nightly 빌드 번호 | 실제 바이너리 zip 이 붙는 릴리스 |

이 워크플로는 요청한 ref 의 커밋을 가리키는 `v<x.y.z>` 태그를 우선 써서
`llama-v0.1.2-bin-win-...` 로 이름을 짓고, 그런 태그가 없으면 `b<커밋수>` 로 폴백한다.
업스트림 원본과 같은 이름을 원하면 `llama_version` 에 nightly 태그(`b10485`)를 직접 넣으면 된다.

## 빌드 옵션

```
-DLLAMA_LLGUIDANCE=ON        LLGuidance 포함 (핵심)
-DLLAMA_BUILD_IS_DEV=OFF     버전을 0.1.2-dev 가 아닌 0.1.2 로 고정
-DGGML_CUDA=ON / -DGGML_VULKAN=ON
-DGGML_CUDA_CUB_3DOT2=ON     CUDA 12.4 전용 (13.x 는 불필요)
-DGGML_NATIVE=OFF            배포용 - 빌드 머신 CPU/GPU 에 종속되지 않게
-DGGML_BACKEND_DL=ON         백엔드를 DLL 로 분리 (공식과 동일 구조)
-DGGML_CPU_ALL_VARIANTS=ON   AVX2/AVX512 등 CPU variant 를 모두 만들고 런타임 자동 선택
-DGGML_RPC=ON                공식 릴리스와 동일
-DLLAMA_BUILD_BORINGSSL=ON   HTTPS(모델 URL 다운로드) 지원
```

워크플로는 빌드 후 다음을 **자동 검증**하고, 실패하면 zip 을 만들지 않는다.

- `llguidance.lib` 존재 → LLGuidance 가 실제로 링크됨
- `llama-cli --version` 에 `-dev` 없음 → 릴리스 빌드
- `build 0, commit unknown` 아님 → git 메타데이터 정상
- 백엔드 DLL(`ggml-cuda.dll` / `ggml-vulkan.dll`) 존재

## 받은 zip 검증

```bat
llama-cli.exe --version
:: version: 0.1.2 (build 10485, commit 1511ce3bc)   <- -dev 가 없어야 함
```

LLGuidance 동작 확인 — `test.lark` 를 만들고:

```lark
%llguidance {}

start: "Hello, " NAME "!"
NAME: /[A-Z][a-z]{2,10}/
```

```bat
llama-cli.exe -m <model>.gguf --grammar-file test.lark -p "Greet someone." -n 32 -no-cnv
```

LLGuidance 가 빠진 빌드라면 `llguidance (cmake -DLLAMA_LLGUIDANCE=ON) is not enabled` 로 즉시 중단된다.
정상 출력이 나오면 포함된 것이다.

## 소요 시간

| 잡 | 대략 |
|---|---|
| CUDA (버전당) | 60~90분 |
| Vulkan | 30~50분 |

세 잡은 병렬로 돈다. **private 리포지토리에서는 Windows 러너가 분당 2배로 과금**되니 주의.

## 구성

```
.github/workflows/build-llamacpp-llguidance.yml   메인 워크플로
.github/actions/windows-setup-cuda/action.yml     CUDA 툴체인 설치 (llama.cpp 에서 가져옴, MIT)
```

`windows-setup-cuda` 는 ggml-org/llama.cpp 의 컴포짓 액션을 그대로 복사한 것이다.
로컬 액션(`uses: ./...`)은 체크아웃된 리포지토리 안에 있어야 하므로 vendoring 했다.
CUDA 버전을 추가하려면 이 파일에 블록을 추가하고 워크플로의 `targets` 선택지를 늘리면 된다.
