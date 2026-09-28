# PaddleOCR 전수조사 분석 및 활용·수익화 정리

> 본 문서는 PaddleOCR 저장소를 폴더 단위로 전수조사한 결과와, 설치/사용법,
> 에이전트 연동 형태(Skill / MCP / SDK), 수익화 아이디어를 한국어로 정리한 자료입니다.

- **작성일**: 2026-09-28
- **분석 대상 저장소**: <https://github.com/bmshin94/PaddleOCR>
- **업스트림 원본**: <https://github.com/PaddlePaddle/PaddleOCR>
- **공식 문서**: <https://www.paddleocr.ai> / **체험 사이트**: <https://www.paddleocr.com>
- **라이선스**: Apache License 2.0

---

## 목차

1. [PaddleOCR란 무엇인가](#1-paddleocr란-무엇인가)
2. [핵심 모델 3대장](#2-핵심-모델-3대장)
3. [폴더별 전수조사 결과](#3-폴더별-전수조사-결과)
4. [동작 원리 (3단계 파이프라인)](#4-동작-원리-3단계-파이프라인)
5. [언제 쓰는가 — 활용 시나리오](#5-언제-쓰는가--활용-시나리오)
6. [경쟁 솔루션 비교](#6-경쟁-솔루션-비교)
7. [설치 및 사용법](#7-설치-및-사용법)
8. [플러그인 / Skill / MCP 정체 정리](#8-플러그인--skill--mcp-정체-정리)
9. [API 토큰이 필요한가](#9-api-토큰이-필요한가)
10. [GitHub에서 유명한 이유](#10-github에서-유명한-이유)
11. [로컬 AI 에이전트 구축 활용법](#11-로컬-ai-에이전트-구축-활용법)
12. [React / PHP 로 만들 수 있는가](#12-react--php-로-만들-수-있는가)
13. [수익화 아이디어 7선](#13-수익화-아이디어-7선)
14. [추천 실행 로드맵](#14-추천-실행-로드맵)
15. [알려진 단점 및 주의사항](#15-알려진-단점-및-주의사항)
16. [참고 링크 모음](#16-참고-링크-모음)

---

## 1. PaddleOCR란 무엇인가

바이두(Baidu)가 주도하는 **오픈소스 OCR + 문서 AI(Document AI) 엔진**입니다.

> 한 줄 요약: **PDF/이미지를 넣으면 LLM이 바로 소비할 수 있는 JSON/Markdown으로 변환해 주는 엔진**

### 주요 지표 (README 기준)

| 항목 | 값 |
| --- | --- |
| GitHub Stars | 70,000+ |
| 의존 저장소 수 | 6,000+ |
| 지원 언어 | 100+ (PaddleOCR-VL-1.5 기준 111개) |
| 지원 OS | Linux / Windows / macOS |
| 지원 하드웨어 | CPU / GPU / XPU(쿤룬) / NPU |
| Python | 3.8 ~ 3.13 |
| 라이선스 | Apache 2.0 (상업 이용 가능) |

### 이 저장소 규모 (실측)

| 항목 | 값 |
| --- | --- |
| 총 파일 수 | 2,456 |
| Python 파일 수 | 593 |
| 저장소 용량 | 약 235MB |

### PaddleOCR를 채택한 대표 프로젝트

Dify, RAGFlow, MinerU, Umi-OCR, Cherry Studio, Haystack, OmniParser,
QAnything, pathway 등. (전체 목록: `awesome_projects.md`)

---

## 2. 핵심 모델 3대장

| 모델 | 역할 | 특징 |
| --- | --- | --- |
| **PP-OCRv6** | 일반 텍스트 인식 (Scene OCR) | 단일 모델로 50개 언어(중/영/일 + 라틴계 46종). tiny(1.5M) / small(7.7M) / medium(34.5M) 3단계. PP-OCRv5 대비 검출 +4.6%, 인식 +5.1%. CPU 5.2배 가속(OpenVINO), Apple M4에서 6.1배, A100에서 0.13초 |
| **PP-StructureV3** | 레이아웃 분석 + 구조화 | 표 셀 좌표·텍스트 좌표까지 세밀하게 제공. PDF/이미지 → Markdown/JSON |
| **PaddleOCR-VL-1.6** (0.9B) | 비전-언어 모델 문서 파싱 | OmniDocBench v1.6 **96.3%** SOTA. 표·고문서·희귀문자·도장·차트까지. VL-1.5와 아키텍처 동일해 교체 비용 0 |

### 추가 파이프라인

- **HPD-Parsing** (2026.07.22): 계층적 병렬 디코딩 + P-MTP. 최대 **4,752 tokens/s** 처리량. OpenAI 호환 서빙 + 커스텀 vLLM 런타임 지원
- **PP-DocLayoutV3**: 기울어짐/왜곡/스캔/조명/화면촬영 등 5가지 난조건 대응
- **PP-DocTranslation**: 레이아웃 유지 문서 번역
- **PP-ChatOCRv4**: 핵심정보추출(KIE) + LLM 결합
- **PaddleOCR.js**: 브라우저에서 PP-OCRv5 직접 실행하는 공식 SDK

---

## 3. 폴더별 전수조사 결과

### A. 코어 엔진 (Python 본체)

| 폴더 | 파일 수 | 내용 |
| --- | --- | --- |
| `paddleocr/` | 72 | **메인 패키지**(`pip install paddleocr`). `_pipelines/`(ocr, pp_structurev3, paddleocr_vl, pp_chatocrv4_doc, pp_doctranslation, seal_recognition, table_recognition_v2, doc_preprocessor, formula_recognition, doc_understanding), `_models/`(text_detection, text_recognition, layout_detection, formula_recognition, chart_parsing, doc_vlm, seal_text_detection, table_*, textline_orientation_classification 등), `_api_client/`(클라우드 API 클라이언트), `_cli.py`, `_doc2md/`(Word/Excel/PPT → Markdown) |
| `ppocr/` | 340 | **학습용 딥러닝 코드**. `modeling/`(백본·넥·헤드), `losses/`, `optimizer/`, `postprocess/`, `metrics/`, `data/`(데이터로더·증강), `ext_op/` |
| `ppstructure/` | 46 | 구조 분석 전용. `kie/`(핵심정보추출), `layout/`, `table/`, `recovery/`, `pdf2word/`, `predict_system.py` |
| `tools/` | 30 | `train.py`, `eval.py`, `export_model.py`, `infer_det.py`, `infer_rec.py`, `infer_kie*.py`, `infer_table.py`, `program.py` 등 학습/추론 진입점 |
| `configs/` | 157 | YAML 설정: `det/`(검출), `rec/`(인식), `cls/`(분류), `kie/`, `table/`, `sr/`(초해상도), `e2e/` |
| `benchmark/` | 84 | 성능 측정 스크립트, `PaddleOCR_DBNet/` |

### B. AI 에이전트 연동층 ⭐

| 폴더 | 내용 |
| --- | --- |
| `skills/` | **공식 Agent Skills 2개**. `paddleocr-text-recognition`(이미지/PDF 텍스트 인식), `paddleocr-doc-parsing`(문서 구조 파싱). 각 `SKILL.md`에 name/description/trigger terms YAML frontmatter + `metadata.openclaw`(requires.env, bins, install) 포함 |
| `mcp_server/` | **공식 MCP 서버** (FastMCP v2). MCP 툴 3개: `ocr`, `pp_structurev3`, `paddleocr_vl`. `providers.py`, `selection.py`, `tasks/`(ocr, doc_parsing, factory, base, mcp_image), `inference/`(ocr, pp_structurev3, paddleocr_vl, shared, errors, factory, types). PyPI: `paddleocr-mcp` |
| `langchain-paddleocr/` | LangChain 통합. `PaddleOCRVLLoader` 도큐먼트 로더. PyPI: `langchain-paddleocr` |

### C. 다국어 SDK

| 폴더 | 내용 |
| --- | --- |
| `api_sdk/typescript/` | TypeScript SDK. `client.ts`, `models.ts`, `results.ts`, `errors.ts`, `internal/`, `examples/`(ocr-url, doc-parsing-file), `tests/` |
| `api_sdk/go/` | Go SDK. `client.go`, `ocr.go`, `poller.go`(비동기 폴링), `transport.go`, `options.go`, `resource.go`, `errors.go`, `examples/` |
| `paddleocr-js/` | **브라우저 전용 SDK** (`@paddleocr/paddleocr-js`). ONNX Runtime Web + OpenCV.js 기반. `packages/core/`(SDK) + `apps/demo/`(Vite 데모). 문서: architecture.md, development.md, monorepo.md |

### D. 배포층 (`deploy/`, 456 파일)

`android_demo/`, `ios_demo/`, `ppocr-android/`, `cpp_infer/`(C++),
`lite/`(Paddle Lite 모바일), `paddle2onnx/`(ONNX 변환), `docker/`,
`paddleocr_vl_docker/`, `hubserving/`(서빙), `paddlecloud/`(K8s),
`slim/`(경량화·양자화), `avh/`(Arm Virtual Hardware — MCU급 임베디드)

### E. 문서 & 테스트

| 폴더 | 내용 |
| --- | --- |
| `docs/` | 621 파일. mkdocs 사이트 소스, `version3.x/`, `version2.x/`, `integrations/`(mcp_server, skills 문서 영/중), FAQ, 데이터셋 가이드, 논문 목록 |
| `readme/` | 9개 언어 README (중/번체/일/한/불/러/스/아랍) |
| `tests/` | 52 파일. `api_client/`, `models/`, `pipelines/`, `ppocr/`, `security/`, `unit/`, `tools/` |
| `test_tipc/` | 학습·추론 전과정 자동 검증 (CI) |

### F. 이 포크에 추가된 파일

| 파일 | 내용 |
| --- | --- |
| `CLAUDE.md` | 프로젝트 페르소나 가이드 (커밋 `641b68c`, PR #1로 머지) |
| `TEST_REPORT.md` | API SDK 통합 테스트 리포트. 11개 테스트 전부 PASS. `fetch_jsonl`이 BOS 프리사인 URL에 불필요한 `Authorization` 헤더를 붙여 400을 받던 블로킹 버그 발견·수정 기록 |
| `PADDLEOCR_ANALYSIS_KR.md` | 본 문서 |

### 계층 구조 요약

```
┌─────────────────────────────────────────────────┐
│ 4층. 연동층 — 외부 프로그램 어댑터                 │
│   skills/  mcp_server/  langchain-paddleocr/     │
│   api_sdk/{typescript,go}  paddleocr-js/         │
├─────────────────────────────────────────────────┤
│ 3층. 사용층 — pip install 로 오는 부분             │
│   paddleocr/                                     │
├─────────────────────────────────────────────────┤
│ 2층. 배포층 — 어디서 돌릴지                        │
│   deploy/ (Android/iOS/C++/Docker/K8s/ONNX/MCU)  │
├─────────────────────────────────────────────────┤
│ 1층. 연구층 — 모델 학습·평가                       │
│   ppocr/  tools/  configs/  benchmark/           │
└─────────────────────────────────────────────────┘
```

일반 사용자는 **3층과 4층**만 다루면 충분하며, 1~2층은 완성된 엔진룸입니다.

---

## 4. 동작 원리 (3단계 파이프라인)

```
[입력]              [처리]                      [출력]
이미지 / PDF  →  ① Detection (글자 위치)   →  Markdown
스캔본        →  ② Recognition (글자 내용) →  JSON
스크린샷      →  ③ Structure (구조/순서)   →  DOCX / Excel
```

1. **Detection(검출)** — DBNet 계열 모델이 글자 영역의 4점 좌표를 산출
   예: `[[120,45],[380,45],[380,80],[120,80]]`
2. **Recognition(인식)** — CRNN / SVTR 계열 모델이 잘라낸 영역의 문자열과
   신뢰도를 산출. 예: `"안녕하세요"`, `0.987`
3. **Structure(구조화)** — PP-DocLayoutV3가 제목/본문/표/그림/수식/도장/
   머리글을 분류하고 읽는 순서(다단 편집 대응)를 정렬

> **PaddleOCR-VL** 계열은 위 3단계를 하나의 0.9B 비전-언어 모델이
> 한 번에 처리하여 Markdown을 직접 생성하는 방식입니다.

### 일반 OCR 대비 결정적 차이

일반 OCR 출력:
```
이름 김철수 나이 30 이름 이영희 나이 25
```

PaddleOCR(PP-StructureV3) 출력:
```markdown
| 이름 | 나이 |
|------|------|
| 김철수 | 30 |
| 이영희 | 25 |
```

표 구조가 유지되므로 LLM이 정확하게 해석할 수 있습니다. RAG 품질에서
결정적인 차이를 만드는 부분입니다.

---

## 5. 언제 쓰는가 — 활용 시나리오

1. **RAG 시스템 구축** — 사내 PDF 대량을 Markdown으로 변환 → 청킹 → 임베딩 → 벡터DB
2. **문서 자동화** — 영수증·세금계산서·계약서에서 표/금액을 JSON으로 추출
3. **AI 에이전트의 시각 기능** — LLM이 스크린샷·손글씨 메모를 읽도록 지원
4. **다국어 문서 번역** — PP-DocTranslation으로 레이아웃 유지 번역
5. **모바일/엣지 앱** — Android/iOS 온디바이스 OCR (명함 스캐너, 번역 카메라)
6. **커스텀 모델 학습** — 특수 폰트·번호판·산업 각인 등 도메인 파인튜닝

### 공식 문서에 수록된 실제 데모

| 데모 | 내용 |
| --- | --- |
| Demo 1 | Claude Desktop에서 손글씨 이미지 → 텍스트 추출 → Notion 저장 (PaddleOCR MCP + Notion MCP) |
| Demo 2 | VSCode에서 손글씨 의사코드 → 실행 가능 Python → GitHub 업로드 (PaddleOCR MCP + filesystem MCP) |
| Demo 3 | 복잡한 표/워터마크 PDF → 편집 가능한 DOCX / 수식·표 이미지 → CSV·Excel |

---

## 6. 경쟁 솔루션 비교

| 항목 | PaddleOCR | Google Vision | AWS Textract | Tesseract |
| --- | --- | --- | --- | --- |
| 비용 | 무료(자체 호스팅) | 페이지 과금 | 페이지 과금 | 무료 |
| 표 인식 | 매우 우수 | 보통 | 우수 | 미지원 |
| 수식(LaTeX) | 매우 우수 | 미지원 | 미지원 | 미지원 |
| 한국어 | 우수 | 우수 | 보통 | 보통 |
| 오프라인 | 가능 | 불가 | 불가 | 가능 |
| 상업 이용 | Apache 2.0 | 유료 | 유료 | 가능 |
| 브라우저 실행 | 가능(paddleocr-js) | 불가 | 불가 | 부분(tesseract.js) |

**핵심 차별점**: "무료 + 오프라인 + 표·수식 정확도"의 조합.

---

## 7. 설치 및 사용법

### 7.1 클라우드 API 모드 (가장 가벼움, 모델 다운로드 없음)

```bash
pip install "paddleocr>=3.7.0"

# 토큰 발급: https://aistudio.baidu.com/account/accessToken
export PADDLEOCR_ACCESS_TOKEN="<발급받은_토큰>"

# 텍스트 인식
paddleocr api --model_type ocr --file_path ./image.jpg

# 문서 파싱 (PDF -> Markdown)
paddleocr api --model_type doc_parsing --file_path ./document.pdf \
  --output result.json --prettify_markdown True
```

주요 CLI 옵션:

```bash
# 모델 지정
paddleocr api --model_type doc_parsing --model PP-StructureV3 --file_path ./report.pdf

# 전처리 비활성화 (평평하고 방향이 올바른 입력에서 더 빠름)
paddleocr api --model_type doc_parsing --file_path ./doc.pdf \
  --use_doc_unwarping False --use_doc_orientation_classify False

# 페이지 범위 지정
paddleocr api --model_type doc_parsing --file_path ./large.pdf --page_ranges "1-5,10,15-20"

# 결과 + 리소스 저장
paddleocr api --model_type doc_parsing --file_url "https://..." \
  --output result.json --save_resources ./resources
```

전처리를 **유지해야 하는 경우**: 휘어진/접힌 문서 사진, 원근 왜곡이 큰 경우,
방향(90/180/270도 회전)이 불확실한 경우.

### 7.2 완전 로컬 모드 (토큰 불필요, 오프라인 동작)

```bash
# CPU
pip install paddlepaddle
pip install "paddleocr[doc-parser]"

# GPU (CUDA 12.6 예시)
# pip install paddlepaddle-gpu -i https://www.paddlepaddle.org.cn/packages/stable/cu126/

# 실행 (첫 실행 시 모델 자동 다운로드)
paddleocr ocr -i ./image.jpg
```

Python 코드:

```python
from paddleocr import PaddleOCR, PPStructureV3

# 텍스트 인식
ocr = PaddleOCR(lang="korean")
for res in ocr.predict("./image.jpg"):
    res.print()
    res.save_to_json("out/")

# 문서 구조 파싱 (PDF -> Markdown)
pipeline = PPStructureV3()
for res in pipeline.predict("./document.pdf"):
    res.save_to_markdown("out/")
    res.save_to_json("out/")
```

의존성 충돌을 피하기 위해 **가상환경 격리가 강력히 권장**됩니다(공식 문서 명시).

### 7.3 선택적 의존성 (pyproject.toml)

| extras | 포함 내용 |
| --- | --- |
| `doc-parser` | `paddlex[ocr,genai-client]` — 문서 파싱 |
| `ie` | `paddlex[ie]` — 정보 추출 |
| `trans` | `paddlex[trans]` — 번역 |
| `doc2md` | python-docx, python-pptx, openpyxl, pylatexenc |
| `all` | 위 전체 |

MCP 서버 전용 extras: `paddleocr-mcp[local]`(문서파싱 의존성),
`paddleocr-mcp[local-cpu]`(+ CPU PaddlePaddle 엔진)

### 7.4 Docker

```bash
# deploy/docker/ , deploy/paddleocr_vl_docker/ 에 Dockerfile 및 README 존재
cd deploy/paddleocr_vl_docker && cat README*.md
```

### 7.5 브라우저 (paddleocr-js)

```bash
cd paddleocr-js
npm install
npm run dev:demo      # Vite 데모 실행 — 브라우저에서 PP-OCRv5 동작
```

기타 명령: `npm run build`, `npm run test`, `npm run typecheck`, `npm run check`

### 7.6 SDK 검증 명령 (저장소 루트 기준)

```bash
python -m pytest tests/api_client/         # Python
cd api_sdk/typescript && npm run lint && npm test   # TypeScript
cd api_sdk/go && go test ./...             # Go
```

### 7.7 추천 진행 순서

1. `paddleocr api` 로 클라우드 모드에서 결과 확인 (5분)
2. 로컬 모드 설치 후 오프라인 동작 확인
3. 서비스화 단계에서 Docker + 서빙 구성

---

## 8. 플러그인 / Skill / MCP 정체 정리

**본질은 "Python 라이브러리 + AI 모델"** 이며, 그 위에 여러 연동 형태가 제공됩니다.

```
        PaddleOCR (라이브러리 + 모델)
                   │
   ┌───────────┬───┴────────┬──────────────┐
   ▼           ▼            ▼              ▼
Agent Skills  MCP 서버    SDK 4종       브라우저 SDK
(skills/)   (mcp_server/) (Py/TS/Go)  (paddleocr-js)
```

| 형태 | 위치 | 정체 | 소비 주체 |
| --- | --- | --- | --- |
| 라이브러리 (본체) | `paddleocr/` | PyPI 패키지 | 개발자 코드 |
| Agent Skill | `skills/` | `SKILL.md` 지시서 | Claude Code, claude.ai, OpenClaw |
| MCP 서버 | `mcp_server/` | FastMCP v2 프로세스 | Claude Desktop, VSCode, Cursor |
| LangChain 통합 | `langchain-paddleocr/` | 도큐먼트 로더 | LangChain 앱 |
| 다국어 SDK | `api_sdk/` | TS/Go 클라이언트 | 웹/백엔드 |

### Skill vs MCP 비교

| 항목 | Skill | MCP |
| --- | --- | --- |
| 정체 | Markdown 문서 | 실행되는 프로세스 |
| 동작 방식 | AI에게 CLI 사용법을 지시 | AI에게 함수(tool)를 제공 |
| 설치 | 파일 복사 | 프로세스 등록 + 설정 JSON |
| 무게 | 매우 가벼움 | 상대적으로 무거움 |
| 제공 단위 | `paddleocr-text-recognition`, `paddleocr-doc-parsing` | `ocr`, `pp_structurev3`, `paddleocr_vl` |

Claude Code 의미의 "플러그인"은 아니지만, Skill이 실질적으로 동일한 역할을 하며
`metadata.openclaw` 필드를 통해 OpenClaw/clawhub 생태계에도 대응합니다.

### Skill 설치

```bash
# 방법 1: skills CLI (Node.js 필요)
npx skills add PaddlePaddle/PaddleOCR -g --skill paddleocr-text-recognition -y
npx skills add PaddlePaddle/PaddleOCR -g --skill paddleocr-doc-parsing -y

# 저장소가 크므로 타임아웃 시 로컬 클론 후 설치
git clone https://github.com/PaddlePaddle/PaddleOCR.git
npx skills add ./PaddleOCR/skills/paddleocr-text-recognition
npx skills add ./PaddleOCR/skills/paddleocr-doc-parsing

# 방법 2: clawhub (OpenClaw)
clawhub install paddleocr-text-recognition
clawhub install paddleocr-doc-parsing

# 방법 3: skills/ 디렉터리를 AI 앱이 요구하는 위치에 수동 복사
```

Claude Code 환경변수 설정 (`.claude/settings.local.json`):

```json
{
  "env": {
    "PADDLEOCR_ACCESS_TOKEN": "<ACCESS_TOKEN>"
  }
}
```

### MCP 서버 설치

```bash
pip install -U paddleocr-mcp      # PyPI
# 또는
git clone https://github.com/PaddlePaddle/PaddleOCR.git
pip install -e mcp_server

paddleocr_mcp --help              # 설치 확인
```

`claude_desktop_config.json` 위치:

| OS | 경로 |
| --- | --- |
| macOS | `~/Library/Application Support/Claude/claude_desktop_config.json` |
| Windows | `%APPDATA%\Claude\claude_desktop_config.json` |
| Linux | `~/.config/Claude/claude_desktop_config.json` |

로컬 추론 설정 예시 (토큰 불필요):

```json
{
  "mcpServers": {
    "paddleocr": {
      "command": "paddleocr_mcp",
      "args": [],
      "env": {
        "PADDLEOCR_MCP_MODEL": "PP-OCRv5",
        "PADDLEOCR_MCP_PPOCR_SOURCE": "local"
      }
    }
  }
}
```

`paddleocr_mcp` 가 PATH에 없으면 `command`에 절대경로를 지정합니다.
`uvx` 를 통한 무설치 실행도 지원합니다.

---

## 9. API 토큰이 필요한가

**필수가 아니며, 4가지 추론 모드 중 선택**합니다.

| 모드 | 토큰 | 모델 다운로드 | GPU | 데이터 외부 전송 | 적합 상황 |
| --- | --- | --- | --- | --- | --- |
| `local` | 불필요 | 필요 | 권장 | 없음 | 개인정보 문서, 오프라인, 대량 처리 |
| `aistudio` (공식 API) | 필요 | 불필요 | 불필요 | 있음 | 빠른 검증, 노코드 |
| `qianfan` (바이두 클라우드) | API Key 필요 | 불필요 | 불필요 | 있음 | 대규모 상용 |
| `self_hosted` | 불필요(직접 관리) | 필요 | 권장 | 없음 | 사내 서버 운영 |

`qianfan` 모드는 `PP-StructureV3` 와 `PaddleOCR-VL` 만 지원합니다.
`self_hosted` 모드는 현재 기본 서빙 솔루션만 지원합니다.

### 환경변수 정리

```bash
# Skill / CLI
PADDLEOCR_ACCESS_TOKEN=<token>        # API 모드 필수
PADDLEOCR_BASE_URL=<url>              # 선택

# MCP 서버
PADDLEOCR_MCP_MODEL=<model>
PADDLEOCR_MCP_PPOCR_SOURCE=local|aistudio|qianfan|self_hosted
PADDLEOCR_MCP_AISTUDIO_ACCESS_TOKEN=<token>
PADDLEOCR_MCP_AISTUDIO_BASE_URL=<url>
PADDLEOCR_MCP_QIANFAN_API_KEY=<key>
PADDLEOCR_MCP_QIANFAN_BASE_URL=<url>
PADDLEOCR_MCP_SELF_HOSTED_BASE_URL=<url>
PADDLEOCR_MCP_PIPELINE_CONFIG=<yaml 절대경로>   # 선택
```

### 보안 주의사항

- 토큰은 저장소에 커밋하지 않습니다. `.env` 또는 `.claude/settings.local.json`
  에 보관하고 `.gitignore` 적용 여부를 확인합니다.
- 프런트엔드 코드에 토큰을 넣지 않습니다. 반드시 서버 라우트를 경유합니다.
- 토큰 발급에는 바이두 AI Studio 계정이 필요합니다. 국내 환경에서는
  `local` 모드가 현실적인 선택입니다.

---

## 10. GitHub에서 유명한 이유

1. **유료 서비스를 무료로 대체** — Google Vision / AWS Textract는 페이지당 과금.
   PaddleOCR는 무료 + Apache 2.0으로 대량 처리 조직에 직접적인 비용 절감 효과
2. **RAG 붐의 병목을 해결** — "PDF를 정확히 읽는 문제"를 선점.
   Dify·RAGFlow·MinerU·Haystack·QAnything 채택 → 네트워크 효과
3. **실제로 SOTA 성능** — OmniDocBench v1.6 96.3%, 파라미터 0.9B로
   Qwen3-VL-235B·GPT-5.5 등 대형 모델을 상회
4. **100개+ 언어 지원** — 중/일/한/아랍/티베트/벵골 등 비라틴 문자권 커버
5. **엣지부터 클라우드까지 전 범위** — 1.5M 초경량 ~ 0.9B VLM,
   `deploy/`에 Android/iOS/C++/ONNX/TensorRT/K8s/MCU 대응
6. **기업 주도 지속 유지보수** — 2020년부터 3.3 → 3.4 → 3.5 → 3.6 → 3.7 +
   HPD-Parsing까지 꾸준한 릴리스
7. **생태계 대응 속도** — MCP 공식 서버, Agent Skills 공식 제공,
   LangChain 통합, 브라우저 SDK를 빠르게 출시

---

## 11. 로컬 AI 에이전트 구축 활용법

로컬 에이전트의 3요소 중 **"시각(문서 이해)" 부품**에 해당합니다.

```
두뇌 = LLM (Ollama / vLLM / llama.cpp)
시각 = PaddleOCR          <-- 해당 영역
손   = MCP 툴 (파일시스템, 브라우저, DB)
```

### 적합한 이유

- `local` 모드로 **완전 오프라인 동작** → 민감 문서 처리 가능
- **공식 MCP 서버 제공** → 별도 래핑 코드 불필요
- 경량 모델(1.5M ~ 34.5M) 존재 → GPU 없이도 동작
- 표준 출력 포맷(JSON/Markdown) → LLM 컨텍스트에 직접 투입
- 파이프라인 기능별 on/off로 속도·메모리 조절 가능

### 추천 아키텍처

```
┌──────────────────────────────────────────────┐
│  MCP 호스트 (Claude Desktop / Code / Cursor)  │
└──────────────┬───────────────────────────────┘
               │ MCP 프로토콜
     ┌─────────┼─────────┬──────────────┐
     ▼         ▼         ▼              ▼
 PaddleOCR  filesystem  sqlite       web-search
 MCP(local)    MCP       MCP            MCP
     │
     ▼
 로컬 모델 캐시
```

### 성능 최적화 (공식 문서 기재 설정)

```python
from paddleocr import PPStructureV3

pipeline = PPStructureV3(
    use_doc_orientation_classify=False,  # 문서 방향 분류 비활성화
    use_doc_unwarping=False,             # 왜곡 보정 비활성화
    use_textline_orientation=False,      # 텍스트라인 방향 분류 비활성화
    use_formula_recognition=False,       # 수식 인식 비활성화
    use_seal_recognition=False,          # 도장 인식 비활성화
    use_table_recognition=False,         # 표 인식 비활성화
    use_chart_recognition=False,         # 차트 파싱 비활성화
    text_detection_model_name="PP-OCRv5_mobile_det",
    text_recognition_model_name="PP-OCRv5_mobile_rec",
    layout_detection_model_name="PP-DocLayout-S",
)
pipeline.export_paddlex_config_to_yaml("PP-StructureV3.yaml")
```

생성된 YAML의 절대경로를 `PADDLEOCR_MCP_PIPELINE_CONFIG` 에 지정합니다.

### 주의

- **PaddleOCR-VL 계열은 CPU 추론 비권장**(공식 명시). CPU 환경에서는
  PP-OCRv6 / PP-StructureV3 조합을 사용합니다.
- `paddlepaddle` 설치 용량이 크므로 가상환경 격리가 필요합니다.

---

## 12. React / PHP 로 만들 수 있는가

가능하며, 저장소가 이미 이를 위한 SDK를 제공합니다.

### 12.1 React — 3가지 방법

**방법 1: 브라우저에서 직접 실행 (서버 비용 없음)**

```bash
npm install @paddleocr/paddleocr-js
```

```jsx
import { useState } from 'react';
import { createOCR } from '@paddleocr/paddleocr-js';

function OcrUploader() {
  const [text, setText] = useState('');

  const handleFile = async (e) => {
    const ocr = await createOCR();              // ONNX Runtime Web + OpenCV.js
    const result = await ocr.detect(e.target.files[0]);
    setText(result.map((r) => r.text).join('\n'));
  };

  return (
    <>
      <input type="file" accept="image/*" onChange={handleFile} />
      <pre>{text}</pre>
    </>
  );
}
```

- 장점: 서버 비용 0, 데이터가 브라우저를 벗어나지 않음(프라이버시)
- 단점: 초기 모델 다운로드 용량, 대용량 PDF 처리 한계
- 참고: `paddleocr-js/apps/demo/` 가 동일 구조의 예제

**방법 2: TypeScript SDK를 서버사이드에서 사용 (권장)**

```ts
// Next.js API Route 등 서버 환경에서만 실행
import { PaddleOCRClient } from '@paddleocr/sdk';

const client = new PaddleOCRClient({
  accessToken: process.env.PADDLEOCR_ACCESS_TOKEN,   // 서버 환경변수
});
const result = await client.docParsing({ filePath: './doc.pdf' });
```

토큰이 브라우저에 노출되지 않도록 반드시 서버 라우트를 경유합니다.

**방법 3: 자체 Python 서버 + React (프로덕션 권장)**

```
React (Vite/Next)  --fetch-->  FastAPI (paddleocr 로컬)  -->  결과 JSON
```

### 12.2 PHP — 공식 SDK는 없으나 문제없음

**방법 1: REST API 직접 호출**

```php
<?php
$ch = curl_init('https://paddleocr.aistudio-app.com/api/v2/ocr/jobs');
curl_setopt_array($ch, [
    CURLOPT_POST           => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_HTTPHEADER     => [
        'Authorization: bearer ' . getenv('PADDLEOCR_ACCESS_TOKEN'),
        'Content-Type: application/json',
    ],
    CURLOPT_POSTFIELDS     => json_encode([
        'model'   => 'PP-OCRv5',
        'fileUrl' => 'https://example.com/doc.pdf',
    ]),
]);
$res = json_decode(curl_exec($ch), true);
// 비동기 잡이므로 jobId 로 폴링 필요 (api_sdk/go/poller.go 참고)
```

**방법 2: 자체 PaddleOCR 서빙을 PHP가 호출 (권장)**

```
PHP (Laravel / WordPress)  --HTTP-->  PaddleOCR 서빙 컨테이너
```

`deploy/` 의 Docker 구성을 그대로 활용할 수 있습니다.

**방법 3: PHP에서 Python CLI 직접 실행 (소규모 한정)**

```php
<?php
$file = escapeshellarg('/path/doc.pdf');   // 이스케이프 필수
exec("paddleocr api --model_type doc_parsing --file_path $file 2>&1", $out, $code);
```

사용자 입력을 그대로 전달하면 커맨드 인젝션 위험이 있습니다.
`escapeshellarg()` 적용은 필수이며, 가능하면 방법 2를 사용합니다.

### 12.3 목적별 권장 스택

| 만들 것 | 권장 스택 |
| --- | --- |
| 개인 토이/데모 | React + paddleocr-js (서버 불필요) |
| SaaS 서비스 | Next.js + FastAPI(PaddleOCR) + Docker |
| WordPress 플러그인 | PHP + 자체 PaddleOCR 서빙 API |
| 사내 시스템 | React + PaddleOCR `self_hosted` (완전 온프레미스) |

---

## 13. 수익화 아이디어 7선

### 법적 전제

Apache 2.0 라이선스이므로 상업 이용·수정·재배포·클로즈드소스 제품 포함이
모두 허용됩니다. 조건은 ① LICENSE 사본 포함 ② 저작권 고지 유지
③ 변경사항 명시입니다. 상표 조항에 따라 **PaddleOCR가 공식 후원·인증한다는
오해를 유발하는 표현은 사용할 수 없습니다.** 또한 클라우드 API 모드를
재판매할 경우 AI Studio / Qianfan 의 이용약관을 별도로 확인해야 합니다.

### 핵심 원리

PaddleOCR는 "기술"이며 "제품"이 아닙니다. **기술과 제품 사이의 간극
— UI, 워크플로, 도메인 지식, 운영 책임 — 이 과금 가능한 영역**입니다.

---

### 아이디어 1. 업종 특화 문서 자동화 SaaS (최우선 추천)

범용 OCR 시장은 대기업이 점유했으나, 좁은 업종 시장은 비어 있습니다.
도메인 지식이 진입장벽이 되고 이탈률이 낮습니다.

| 타겟 | 상품 | 핵심 기능 |
| --- | --- | --- |
| 세무사/회계사무소 | 증빙 자동 분류기 | 영수증·세금계산서 → 계정과목 분류 → 세무 프로그램용 CSV |
| 병원/의원 | 진료의뢰서 디지털화 | 팩스 스캔 → 구조화 JSON → EMR 연동 |
| 법무법인 | 판결문/계약서 분석 | PDF → 조항 분해 → 위험조항 하이라이트 |
| 물류/수출입 | 선적서류 자동입력 | B/L·Invoice·P/L → ERP 자동입력 |
| 건설 | 도면 텍스트 추출 | 도면 PDF → 부재 리스트 |
| 학원/출판 | 시험지 디지털화 | 수식 포함 문제 → LaTeX → 문제은행 |

위 업종 문서는 표와 수식이 핵심이므로 일반 OCR로는 처리가 어렵고,
PaddleOCR의 표 셀 좌표 + LaTeX 수식 인식이 사실상 유일한 무료 해법입니다.

**수익 모델 예시**

```
Free       : 월 20페이지
Starter    : 월 3만원  / 500페이지
Pro        : 월 9만원  / 3,000페이지 + API
Business   : 월 30만원 / 무제한 + 다중사용자 + 전용지원
Enterprise : 별도 견적 (온프레미스)
```

자체 호스팅이므로 변동비가 거의 없습니다. GPU 서버 월 30~50만원 수준으로
수천 페이지 처리가 가능하여, 고객 20명 내외에서 손익분기에 도달합니다.

- 난이도: 중 · MVP 4~8주 · React/PHP 스택으로 구현 가능
- 리스크: 도메인 지식 필요 → 해당 업종 베타 고객 1명 확보 후 시작
- 개인정보 이슈는 `local` 모드 온프레미스 옵션으로 차별화 요소로 전환 가능

---

### 아이디어 2. 저가 OCR API 재판매

```
Google Vision : 1,000페이지 약 $1.5
AWS Textract  : 1,000페이지 $1.5 ~ $65 (표 포함 시 급증)
자체 서비스   : 1,000페이지 약 $0.3
```

**판매 채널**: RapidAPI, AWS Marketplace 등록으로 자연 유입 확보.
OpenAI 호환 엔드포인트를 제공하면 `base_url` 변경만으로 이전 가능해
스위칭 코스트가 사라집니다. 한국 리전 운영 시 "데이터 국외 미전송"이
공공·금융 대상 핵심 판매 포인트가 됩니다.

- 난이도: 하 · 2~4주 · `deploy/` 구성 재사용
- 주의: 순수 API 재판매는 마진이 얇은 볼륨 게임입니다.
  아이디어 1과 결합해 "API도 제공하는 SaaS" 형태가 현실적입니다.

---

### 아이디어 3. 플랫폼 플러그인/확장

이미 형성된 사용자 풀에 진입하는 전략입니다.

| 플랫폼 | 상품 | 수익 모델 |
| --- | --- | --- |
| WordPress | 이미지 업로드 시 자동 OCR → 검색 가능화 + alt 텍스트 자동생성(SEO) | Freemium, Pro 연 $49 |
| Notion | PDF 첨부 → 자동 Markdown 페이지 변환 | 월 $5 |
| Obsidian | 스캔 노트 → 검색 가능 텍스트 | 일시불 $15 |
| Chrome 확장 | 우클릭 → 이미지 텍스트 복사 (paddleocr-js 로컬 처리) | Free + Pro |
| Excel / Google Sheets 애드온 | 영수증 사진 → 행 자동 입력 | 월 $9 |
| Figma 플러그인 | 시안 이미지 → 텍스트 레이어 추출 | 일시불 $29 |
| Slack / Discord 봇 | 이미지 업로드 시 텍스트 자동 응답 | 워크스페이스당 과금 |

WordPress는 유료 플러그인 판매 생태계가 확립되어 결제·배포 인프라를
직접 구축할 필요가 없어 PHP 개발자에게 유리합니다.

- 난이도: 하~중 · 3~6주

---

### 아이디어 4. 기업 온프레미스 구축 대행 (수익성 최상)

병원, 로펌, 은행, 공공기관, 방산, 제조 대기업은 문서를 외부 클라우드로
전송할 수 없습니다. PaddleOCR는 완전 오프라인 구동이 가능하므로 이 시장에
진입할 수 있습니다.

```
[1] 컨설팅 / PoC            : 500만 ~ 1,500만원
[2] 구축 (설치+연동+튜닝)    : 2,000만 ~ 8,000만원
[3] 커스텀 모델 파인튜닝     : 1,000만 ~ 3,000만원
[4] 연간 유지보수            : 구축비의 15~20% / 년
[5] 교육 / 기술이전          : 300만 ~ 800만원
```

기업은 오픈소스 자체가 아니라 **책임과 보증**에 비용을 지불합니다.
이 저장소에는 `tools/train.py`, `configs/` 157개, `test_tipc/` 가 포함되어
있어 **커스텀 파인튜닝까지 제공 가능**하며, 이는 클라우드 API 사업자가
제공할 수 없는 차별 서비스입니다.

- 난이도: 상 · 영업력과 레퍼런스가 핵심
- 진입 전략: 나라장터 등 공공 입찰에서 "OCR", "문서 디지털화",
  "비정형 데이터" 키워드 모니터링. 중소 SI와 컨소시엄 구성 시
  레퍼런스 없이도 진입 가능

---

### 아이디어 5. 모바일 스캐너 앱

`deploy/android_demo/`, `deploy/ios_demo/`, `deploy/ppocr-android/`,
`deploy/lite/` 가 이미 제공되며, 온디바이스 처리로 서버 비용이 없습니다.

| 앱 | 차별점 |
| --- | --- |
| 명함 스캐너 → 연락처 자동저장 | 국내 명함 레이아웃 특화 |
| 영수증 가계부 | 카드사 연동 없이 사진만으로 |
| 여행 번역 카메라 | 111개 언어 + 오프라인(로밍 불필요) |
| 수식 스캐너 → LaTeX | Photomath 대안, 학생 타겟 |
| 시험지 → 오답노트 자동생성 | 학부모 타겟, 구독 전환율 높음 |

- 수익: 인앱결제(월 4,900원 / 평생 29,000원) + 광고
- 난이도: 중 · React Native + 네이티브 브릿지 가능
- 리스크: 앱스토어 경쟁이 치열하므로 틈새 특화 필수

---

### 아이디어 6. 교육 콘텐츠 / 인포프로덕트 (가성비 최상)

개발 부담이 없고 재고가 없으며 마진이 높습니다.

| 상품 | 가격대 | 채널 |
| --- | --- | --- |
| "PaddleOCR로 RAG 구축" 강의 | 5~15만원 | 인프런, 유데미, 클래스101 |
| 유튜브 튜토리얼 시리즈 | 광고 + 협찬 | YouTube |
| 전자책 "문서AI 실전 가이드" | 2~5만원 | 크몽, 리디, Gumroad |
| **보일러플레이트 판매** | 10~30만원 | Gumroad, LemonSqueezy |
| 1:1 기술 컨설팅 | 시간당 10~20만원 | 크몽, 숨고 |
| 유료 뉴스레터 | 월 1만원 | 스티비, Substack |

특히 **"PaddleOCR + FastAPI + Next.js + 결제 연동"이 포함된 문서파싱 SaaS
스타터킷**을 판매하는 방식이 효과적입니다. 구매자는 아이디어 1을 구현하려는
개발자이므로, 직접 경쟁하지 않고 도구를 공급하는 위치를 차지합니다.

- 난이도: 최하 · 즉시 시작 가능

---

### 아이디어 7. AI 에이전트 번들 (미래성 최상)

MCP / Skills 생태계는 아직 초기 단계이며, 킬러 제품이 부재합니다.

| 상품 | 설명 | 수익 |
| --- | --- | --- |
| "문서비서" 에이전트 팩 | PaddleOCR MCP + 파일시스템 + DB + 프롬프트 = 원클릭 설치 패키지 | 월 구독 $9 |
| 업종별 커스텀 Skill 세트 | 세무/의료/법률용 프롬프트 + Skill 묶음 | 일시불 $49 |
| 에이전트 구축 컨설팅 | 사내 로컬 에이전트 구축 | 건당 500~3,000만원 |
| 사내 에이전트 호스팅 | 완전 온프레미스 운영 대행 | 월 100~500만원 |

PaddleOCR가 공식 MCP 서버와 공식 Skill을 모두 제공하므로 조립 난이도가 낮습니다.

- 난이도: 중

---

### 아이디어 비교표

| 아이디어 | 난이도 | 초기비용 | 수익규모 | 수익화 속도 |
| --- | --- | --- | --- | --- |
| 1. 업종 SaaS | 중 | 중 | 매우 큼 | 중 |
| 2. API 재판매 | 하 | 중 | 보통 | 빠름 |
| 3. 플러그인 | 하 | 낮음 | 큼 | 빠름 |
| 4. 온프레미스 구축 | 상 | 낮음 | 최대 | 느림 |
| 5. 모바일 앱 | 중 | 낮음 | 보통 | 중 |
| 6. 교육 콘텐츠 | 최하 | 없음 | 보통 | 매우 빠름 |
| 7. 에이전트 번들 | 중 | 낮음 | 큼 | 중 |

---

## 14. 추천 실행 로드맵

```
1개월차   - 아이디어 6 (콘텐츠)
            PaddleOCR 로컬 설치 및 실험 → 블로그/영상으로 기록
            → 개인 브랜딩 시작, 리드 확보 (비용 0)

2~3개월차 - 아이디어 3 (플러그인)
            WordPress 플러그인 또는 Chrome 확장 1개 출시
            → 첫 매출 경험 + 실사용 피드백 확보

4~6개월차 - 아이디어 1 (SaaS) — 본 사업
            업종 1개 선정하여 MVP 출시
            → 베타 고객 5명 확보 → 유료 전환

6~12개월차 - 아이디어 4 (온프레미스 구축)
            SaaS 고객 중 대형 기업의 온프레미스 요청이
            고액 계약의 출발점이 됨
```

### 핵심 원칙 3가지

1. **기술이 아니라 문제를 판다** — "OCR 서비스"가 아니라
   "세무사 업무 월 40시간 절감"으로 제안한다
2. **좁게 시작한다** — 범용 시장은 대기업이 유리하다
3. **온프레미스가 차별 무기다** — 클라우드 API 사업자가 제공할 수 없는 영역

---

## 15. 알려진 단점 및 주의사항

1. **설치 용량이 큼** — `paddlepaddle` 프레임워크가 수백 MB ~ 수 GB
2. **PaddleOCR-VL은 CPU 추론 비권장** (공식 문서 명시) — GPU 필요
3. **문서가 중국어 중심** — 영문 문서도 있으나 최신 내용은 중국어가 앞설 수 있음
4. **`paddlex` 의존** — `paddlex[ocr-core]>=3.7.0,<3.8.0` 에 고정되어
   버전 충돌 주의. 가상환경 격리 권장
5. **한국어 최적화 수준** — 중국어/영어보다 상대적으로 낮으나 실용 수준은 충족
6. **토큰 발급에 바이두 AI Studio 계정 필요** — 국내에서는 `local` 모드 권장
7. **대형 저장소** — `npx skills add` 가 느린 네트워크에서 타임아웃될 수 있음.
   로컬 클론 후 설치로 우회

---

## 16. 참고 링크 모음

### 저장소

- 이 포크: <https://github.com/bmshin94/PaddleOCR>
- 업스트림: <https://github.com/PaddlePaddle/PaddleOCR>
- 이슈: <https://github.com/PaddlePaddle/PaddleOCR/issues>

### 공식 사이트 / 문서

- 공식 문서: <https://www.paddleocr.ai>
- 체험 센터 / API: <https://www.paddleocr.com>
- DeepWiki: <https://deepwiki.com/PaddlePaddle/PaddleOCR>
- PP-OCR 문서: <https://www.paddleocr.ai/latest/en/version3.x/pipeline_usage/OCR.html>
- PaddleOCR-VL 문서: <https://www.paddleocr.ai/latest/en/version3.x/pipeline_usage/PaddleOCR-VL.html>
- PP-StructureV3 문서: <https://www.paddleocr.ai/latest/en/version3.x/pipeline_usage/PP-StructureV3.html>
- HPD-Parsing 문서: <https://www.paddleocr.ai/latest/en/version3.x/pipeline_usage/HPD-Parsing.html>
- 공식 API CLI 문서: <https://www.paddleocr.ai/latest/en/version3.x/inference_deployment/serving/paddleocr_official_api/cli.html>

### 모델 배포처

- HuggingFace PP-OCRv6: <https://huggingface.co/collections/PaddlePaddle/pp-ocrv6>
- HuggingFace PaddleOCR-VL-1.6: <https://huggingface.co/PaddlePaddle/PaddleOCR-VL-1.6>
- ModelScope PP-OCRv6: <https://www.modelscope.cn/collections/PaddlePaddle/PP-OCRv6>

### 토큰 / 클라우드

- AI Studio Access Token: <https://aistudio.baidu.com/account/accessToken>
- Qianfan API 문서: <https://cloud.baidu.com/doc/qianfan-api/s/ym9chdsy5>

### 생태계

- MCP 소개: <https://modelcontextprotocol.io/introduction>
- FastMCP: <https://gofastmcp.com>
- Claude Code Skills 문서: <https://code.claude.com/docs/en/skills>
- claude.ai Skills 사용법: <https://support.claude.com/en/articles/12512180-use-skills-in-claude>
- OpenClaw Skills 문서: <https://docs.openclaw.ai/tools/skills>
- LangChain 통합 패키지: <https://pypi.org/project/langchain-paddleocr/>

### 저장소 내부 문서

- Agent Skills 가이드: `docs/version3.x/integrations/skills.en.md`
- MCP 서버 가이드: `docs/version3.x/integrations/mcp_server.en.md`
- API SDK 개요: `api_sdk/README.md`
- 브라우저 SDK: `paddleocr-js/README.md`
- 활용 프로젝트 목록: `awesome_projects.md`
- API SDK 테스트 리포트: `TEST_REPORT.md`
