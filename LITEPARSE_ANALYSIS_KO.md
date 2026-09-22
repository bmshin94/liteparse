# LiteParse 전수조사 분석 & 활용 가이드 (한국어)

> 이 문서는 Claude Code(카리나 페르소나)와 함께 이 저장소를 전수조사하며 나눈 대화를
> 정리한 결과물입니다. 코드 5만 5천여 줄, 폴더 구조, 문서, CI 워크플로우, 벤치마크를
> 직접 확인한 내용을 기반으로 작성했습니다.

- 작성일: 2026-09-22
- 대상 버전: **v2.14.6**
- 분석 브랜치: `claude/hopeful-dijkstra-wq9ytg`

---

## 📎 관련 GitHub 주소

### 이 저장소 (포크)
- **내 저장소**: https://github.com/bmshin94/liteparse
- 분석 브랜치: https://github.com/bmshin94/liteparse/tree/claude/hopeful-dijkstra-wq9ytg

### 원본 (업스트림)
- **원본 저장소**: https://github.com/run-llama/liteparse
- 공식 문서: https://developers.llamaindex.ai/liteparse/
- LiteParse V1 (구버전 코드): https://github.com/run-llama/liteparse/tree/logan/liteparse-v1

### 패키지 레지스트리
| 플랫폼 | 주소 |
|---|---|
| npm (Node.js) | https://www.npmjs.com/package/@llamaindex/liteparse |
| npm (WASM) | https://www.npmjs.com/package/@llamaindex/liteparse-wasm |
| PyPI (Python) | https://pypi.org/project/liteparse/ |
| crates.io (Rust) | https://crates.io/crates/liteparse |

### 생태계 / 확장
| 항목 | 주소 |
|---|---|
| HTTP 서버 | https://github.com/run-llama/liteparse-server |
| 에이전트 스킬 | https://github.com/run-llama/llamaparse-agent-skills |
| 스킬 정의(SKILL.md) | https://github.com/run-llama/llamaparse-agent-skills/blob/main/skills/liteparse/SKILL.md |
| 에이전트 플러그인 | https://github.com/run-llama/llamaparse-agent-plugins |
| skills CLI | https://github.com/vercel-labs/skills |
| 유료 클라우드(LlamaParse) | https://cloud.llamaindex.ai |

### 벤치마크 / 의존성
| 항목 | 주소 |
|---|---|
| ParseBench | https://github.com/run-llama/parse-bench |
| opendataloader-bench | https://github.com/opendataloader-project/opendataloader-bench |
| olmOCR-bench | https://github.com/allenai/olmocr/tree/main/olmocr/bench |
| PDFium | https://pdfium.googlesource.com/pdfium/ |
| Tesseract | https://github.com/tesseract-ocr/tesseract |
| EasyOCR | https://github.com/JaidedAI/EasyOCR |
| PaddleOCR | https://github.com/PaddlePaddle/PaddleOCR |
| napi-rs | https://napi.rs/ |
| PyO3 | https://pyo3.rs/ |
| wasm-bindgen | https://github.com/wasm-bindgen/wasm-bindgen |

---

## 1. 프로젝트 정체 (전수조사 결과)

### 한 줄 요약
> **LiteParse** = PDF / 워드 / 엑셀 / PPT / 이미지를 **AI가 먹기 좋은 텍스트·마크다운**으로
> 바꿔주는, **Rust로 만든 초고속 로컬 문서 파서**.

### 이 저장소의 정체
```
origin    https://github.com/bmshin94/liteparse    ← 내 포크
upstream  https://github.com/run-llama/liteparse   ← LlamaIndex 공식 원본
```

커밋 로그에 `Merge pull request #462 from run-llama/...` 같은 원본 커밋이 그대로 남아 있음.
즉 **LlamaIndex(run-llama)의 오픈소스를 포크한 상태**이며, 내가 직접 추가한 변경은 다음 하나뿐이다.

| 커밋 | 내용 |
|---|---|
| `68c7399` | `docs: appended CLAUDE.md persona guide` — CLAUDE.md에 카리나 페르소나 27줄 추가 |
| `da00293` | PR #1 머지 |

나머지 코드는 100% 원본. `run-llama`는 RAG 프레임워크 **LlamaIndex**의 개발사이며,
유료 클라우드 파서 **LlamaParse**도 같은 회사 제품이다.

### 핵심 문제 정의: "PDF는 왜 짜증나는가"

PDF 내부에는 **문장이 없다.** "몇 좌표에 이 글자를 찍어라"는 인쇄 명령서만 있다.

```
"안" → (72, 300)에 그려라
"녕" → (85, 300)에 그려라
```

PDF는 "화면과 프린터 출력이 같아야 한다"는 목적으로 만든 포맷이라 **보이는 것**만 저장하고
**의미**는 저장하지 않는다. 그래서 이런 문제가 생긴다.

| 사람이 보는 것 | PDF가 아는 것 |
|---|---|
| 깔끔한 3x4 표 | 좌표에 흩어진 글자 12덩이 + 별개의 선 그림 |
| 2단 논문 (좌 → 우) | 좌우로 뒤섞인 글자 순서 |
| "제목" | 그냥 좀 큰 글자 |
| 세로 회전 표 | 90도 돌아간 좌표들 |
| 스캔한 계약서 | 사진 1장, 글자 0개 |

기존 도구 결과:
```
매출액영업이익2024년1,2003002023년1,000250
```
→ 이걸 LLM에 넣으면 할루시네이션 공장.

LiteParse 결과 (`--format markdown`):
```markdown
## 재무 요약

| 구분 | 2024년 | 2023년 |
|------|-------:|-------:|
| 매출액 | 1,200 | 1,000 |
| 영업이익 | 300 | 250 |
```

---

## 2. 아키텍처 (데이터 흐름)

```
[입력] PDF / DOCX / XLSX / PPTX / 이미지(jpg,png,svg...)
   |
[1] 변환        conversion.rs (1,459줄)
                - Office → LibreOffice 호출 → PDF
                - 이미지 → Rust image/resvg/usvg 크레이트로 직접 PDF화
                → 입구를 PDF 하나로 통일 (뒤 로직 단일화)
   |
[2] 텍스트 추출  extract.rs (4,518줄) + pdfium-sys (C FFI)
                - Google PDFium(크롬 내장 PDF 엔진)으로
                  글자 + 좌표(bbox) + 폰트 + 이미지 + 벡터 추출
   |
[3] 선택적 OCR   ocr/tesseract.rs, ocr/http_simple.rs, ocr/oar.rs
                - 글자 없는 페이지/이미지 영역만 렌더링 후 OCR
   |
[4] OCR 병합     ocr_merge.rs (2,178줄)
                - 원본 텍스트 + OCR 결과 중복 제거 후 병합
                - confidence, source 플래그 보존
   |
[5] 레이아웃 복원 projection.rs (5,783줄) ← 프로젝트의 심장
                - 격자 투영(grid projection): 페이지를 모눈종이로 보고 글자 배치
                - 컬럼 감지: 세로 빈 공간으로 다단 레이아웃 판별 → 읽기 순서 교정
                - 앵커 정렬: 좌/우/중앙/부동 정렬 추적, forward anchor로 줄 간 전달
                - 회전 처리: 90도 / 180도 / 270도 텍스트 정상화
   |
[6] 구조 분류    markdown_layout/
                - classify.rs (1,591줄) 블록 분류
                - headings.rs (1,029줄) 제목 레벨 추론
                - tables.rs (6,535줄) 표 복원 (선 있는/없는 표, 병합 셀)
                - repetition.rs (654줄) 반복 머리말/꼬리말 제거
                - blocks.rs, cross_region.rs, inline.rs
   |
[출력] Text / JSON(bbox 포함) / Markdown / PNG 스크린샷
   |
[바인딩] Rust · Node.js(napi-rs) · Python(PyO3) · 브라우저(WASM) · CLI(lit)
```

### 폴더별 내용

| 폴더 | 내용 |
|---|---|
| `crates/liteparse/` | 핵심 엔진. `projection.rs`(228KB), `extract.rs`(168KB), `ocr_merge.rs`(88KB) |
| `crates/liteparse/src/markdown_layout/` | 마크다운 복원 로직 (`tables.rs` 6,535줄) |
| `crates/pdfium-sys/` | PDFium C FFI 바인딩 (`bindings.rs` 3,316줄, 자동 생성) |
| `crates/pdfium/` | 안전한 Rust 래퍼 (`page.rs` 2,258줄) |
| `crates/liteparse-napi/` | Node.js 네이티브 모듈 |
| `crates/liteparse-python/` | Python 네이티브 모듈 (PyO3) |
| `crates/liteparse-wasm/` | 브라우저용 WASM (1,657줄) |
| `packages/node/` | npm `@llamaindex/liteparse`. TS 래퍼 + 워커풀(`pool.ts`) + CLI |
| `packages/python/` | PyPI `liteparse`. 동일 구성 + `_pool.py` |
| `packages/wasm/` | npm `@llamaindex/liteparse-wasm` |
| `ocr/` | OCR 서버 예제 3종: EasyOCR(8828), PaddleOCR(8829), SuryaOCR(8830) |
| `docs/` | 공식 문서 소스 MDX 17개 (시각적 인용, 복잡도, 브라우저, 서버 사용법 등) |
| `demo/docs/` | 테스트 PDF (Apple 10-K 연차보고서, 초소형 텍스트 PDF) |
| `integration_tests_data/` | 지옥 난이도 테스트 PDF (대각선 텍스트, 180도 회전, 음수 폰트 크기, AcroForm, 영수증 이미지) |
| `.github/workflows/` | CI 12종 + 릴리스 5종 (crates.io / npm / PyPI / WASM / Docker 자동 배포) |
| `wasm-demo-site/` | 브라우저 데모 단일 HTML |
| `dataset_eval_utils/` | 벤치마크 평가 도구 (Python) |

전체 Rust 코드: **55,074줄**

---

## 3. 핵심 기능

### 3.1 공간 텍스트 추출 (Spatial Extraction) — 최대 차별점
글자마다 정확한 좌표를 제공한다. PDF 포인트 단위(1pt = 1/72inch), 좌측 상단 원점.

```json
{ "text": "매출 15% 성장", "x": 72, "y": 200, "width": 150, "height": 12 }
```

→ "이 답변은 3페이지 이 위치에서 나왔다"는 **근거 하이라이트(Visual Citation)** 구현 가능.

### 3.2 복잡도 사전 검사 (`lit is-complex`)
파싱 **전에** 싸게 훑어 OCR 필요 여부를 판정. 판정 이유도 제공:
`scanned`, `no-text`, `sparse-text`, `embedded-images`, `garbled`, `vector-text`, `annotation-text`

- stdout: 페이지별 JSON
- stderr: `COMPLEX` / `SIMPLE` 판정
- **OCR이 필요하면 종료 코드 non-zero** → 쉘 조건문으로 파이프라인 분기 가능

```bash
lit is-complex doc.pdf --quiet && lit parse doc.pdf --no-ocr
lit is-complex doc.pdf --compact | jq '[.[] | select(.needs_ocr) | .page_number]'
```

대량 처리에서 이 기능은 **파싱 기능이 아니라 비용 관리 기능**이다.

### 3.3 선택적 OCR
전체 페이지가 아니라 텍스트 추출이 실패한 영역만 OCR한다.

| 엔진 | 설정 | 특징 |
|---|---|---|
| Tesseract (내장) | 불필요 | 즉시 사용, 라틴 문자 양호, CJK 보통 |
| PaddleOCR | `uv` | **CJK 최강**, EasyOCR 대비 2~3배 빠름 |
| EasyOCR | `uv` | 범용 균형 |
| Surya | `uv` | 다국어 정확도 우수, GPU 권장 |

OCR 서버 규격은 매우 단순하다: `POST /ocr` → `{ results: [{ text, bbox, confidence }] }`
(상세: `OCR_API_SPEC.md`)

### 3.4 워커 풀 (Python / Node 전용)
PDFium이 동시 파싱을 직렬화하는 문제 때문에 **프로세스 분리 워커풀**을 제공한다.

```ts
new LiteParse({ poolSize: 4, parseTimeoutMs: 30_000 })
```

타임아웃 시 워커 프로세스를 죽여서, 문제 문서 하나가 전체 파이프라인을 멈추는 것을 막는다.
`ParseTimeoutError.source`로 범인 문서를 식별할 수 있다.

### 3.5 추출 옵션 (전부 opt-in)

| 플래그 | 얻는 것 |
|---|---|
| `--extract-images` | 내장 이미지 바이트 + 메타데이터 |
| `--extract-blocks` | 제목/문단/리스트/표/코드/도형 블록 + 셀 단위 bbox |
| `--extract-form-fields` | AcroForm 입력값 (고장난 위젯 메모리상 복구 포함) |
| `--extract-annotations` | PDF 주석 |
| `--extract-structure-tree` | 태그드 PDF 논리 구조 |
| `--extract-vector-graphics` | 벡터 패스, 병합된 수평/수직 선 |
| `--extract-xfa-packets` | XFA 폼 원본 XML |
| `--extract-text-metadata` | 글자별 폰트/타이포 정보 |
| `--extract-content-bounds` | 페이지 실제 콘텐츠 영역 bbox |
| `--complexity` / `include_complexity` | 페이지별 복잡도 통계 |

기본 ON: `extract_links` (`--no-links`로 해제)

### 3.6 스크린샷
```bash
lit screenshot doc.pdf --dpi 300 --target-pages "1,3,5" -o ./shots
```
AcroForm 필드 외관(입력값, 체크박스)까지 렌더링되어 Vision 모델 입력으로 적합.
`is_solid_fill`(빈 페이지 여부), `detect_screenshot_rects`(솔리드 사각형/선 검출) 신호 제공.

---

## 4. 벤치마크 (README 기록, 2026-09-09 기준)

| 벤치마크 | LiteParse | +Tesseract | +PaddleOCR | 2등 (모델 없는 도구) |
|---|---:|---:|---:|---|
| ParseBench (2,049 문서) | 0.364 | 0.380 | **0.389** | pdf-inspector 0.283 |
| opendataloader-bench (200) | 0.886 | 0.896 | **0.901** | opendataloader 0.842 |
| olmOCR-bench (1,403 페이지) | 39.6 | 41.1 | **42.2** | pdf-inspector 33.7 |

- **ML 모델 0개, GPU 0개, LLM 0개**로 달성한 점수
- 속도: **페이지당 2~5ms**
- opendataloader-bench에서 상업용 nutrient(0.885)를 0.901로 상회
- 강점: `multi_column` 69.1 (2등 49.7), `long_tiny_text` 46.4, `baseline` 99.9
- **약점(정직하게)**: 수식(LaTeX) 카테고리 `arxiv_math` / `old_scans_math` **0.0점**,
  차트 데이터 복원 `Charts` 0.013 (모든 도구가 0점대)

재현:
```bash
./run_benchmarks.sh --liteparse-only
./run_benchmarks.sh --liteparse-only --ocr=tesseract
( cd ocr/rapidocr && uv run server.py ) & ./run_benchmarks.sh --liteparse-only --ocr=paddle
```

---

## 5. 설치 및 사용법

### 5.1 CLI 설치 (4가지, 모두 동일한 `lit` 명령 제공)

```bash
npm i -g @llamaindex/liteparse     # 추천: 가장 간단
pip install liteparse
cargo install liteparse
docker build -t liteparse . && docker run -v $(pwd):/data liteparse lit parse /data/doc.pdf
```

지원: Linux, macOS(Intel/ARM), Windows / Node 18+ / Python 3.10~3.14

### 5.2 CLI 주요 명령

```bash
lit parse doc.pdf                                  # 텍스트 (기본)
lit parse doc.pdf --format markdown -o out.md      # 마크다운
lit parse doc.pdf --format json -o out.json        # bbox 포함 JSON
lit parse doc.pdf --target-pages "1-5,10,15-20"    # 특정 페이지
lit parse doc.pdf --no-ocr                         # OCR 끄기
lit parse doc.pdf --ocr-language kor               # 한국어 OCR
lit parse doc.pdf --ocr-server-url http://localhost:8829   # PaddleOCR 서버 사용
lit is-complex doc.pdf                             # 복잡도 검사
lit screenshot doc.pdf --dpi 300 -o ./shots        # 스크린샷
lit batch-parse ./in ./out --recursive             # 폴더 일괄 처리
curl -sL https://example.com/a.pdf | lit parse -   # 파이프 입력
```

### 5.3 라이브러리 사용

**Python**
```python
from liteparse import LiteParse

parser = LiteParse(output_format="markdown", ocr_enabled=True, ocr_language="kor")
result = parser.parse("계약서.pdf")
for page in result.pages:
    print(page.page_num, page.text)
    for item in page.text_items:
        print(item.text, item.x, item.y, item.width, item.height)

# async
result = await parser.aparse("doc.pdf")

# 프로덕션: 워커풀 + 타임아웃
with LiteParse(pool_size=4, parse_timeout_ms=30000) as p:
    result = p.parse("거대문서.pdf")
```

**Node.js / TypeScript**
```typescript
import { LiteParse, searchItems } from "@llamaindex/liteparse";

const parser = new LiteParse({
  outputFormat: "json",
  extractImages: true,
  extractBlocks: true,
  poolSize: 4,
  parseTimeoutMs: 30_000,
});

const result = await parser.parse("report.pdf");
for (const page of result.pages) {
  console.log(page.pageNum, searchItems(page.textItems, "매출"));
}
parser.close();
```

**Rust**: `cargo add liteparse`
**브라우저(WASM)**: `npm i @llamaindex/liteparse-wasm`

### 5.4 부가 설치

| 항목 | 필요한 경우 | 설치 |
|---|---|---|
| LibreOffice | DOCX/XLSX/PPTX 파싱 | `apt install libreoffice` / `brew install --cask libreoffice` |
| 한국어 tessdata | 한국어 OCR | `apt install tesseract-ocr-kor` 또는 `TESSDATA_PREFIX` 지정 |
| PaddleOCR 서버 | 한국어 OCR 정확도 향상(권장) | `cd ocr/paddleocr && uv run server.py` (포트 8829) |

환경변수: `TESSDATA_PREFIX` — 오프라인/에어갭 환경용 traineddata 디렉터리

---

## 6. 플러그인? 스킬? MCP?

**본질은 "라이브러리 + CLI"이며, 스킬/플러그인으로도 공식 제공된다.**

| 형태 | 지원 | 설명 |
|---|:---:|---|
| 라이브러리 + CLI | O (본질) | npm/pip/cargo 패키지 + `lit` 명령 |
| Agent Skill | O (공식) | `npx skills add run-llama/llamaparse-agent-skills --skill liteparse` |
| 플러그인 (Claude Code / Codex) | O (공식) | `run-llama/llamaparse-agent-plugins` 마켓플레이스 |
| MCP 서버 | **X (없음)** | 이 저장소에 MCP 구현 없음. MCP는 유료 LlamaParse 쪽 |

### 에이전트 스킬이 주입하는 사용 패턴 (문서 명시)
1. **한 번만 파싱해 파일로 저장하고, 그 파일을 검색한다** (검색마다 재파싱 방지)
2. **왕복 최소화** — 매치와 주변 문맥을 한 명령으로, 독립 조회는 배치로
3. **출력에 상한** — 한 번의 조회가 컨텍스트 창을 넘치지 않게
4. **키워드 검색이 막히면 랭크 검색으로 승급**

스킬의 가치는 설치가 아니라 **컨텍스트 절약 패턴**이며, 실제 토큰 비용을 크게 줄인다.

---

## 7. API 토큰 / 비용

**API 토큰 불필요. 100% 무료. 완전 로컬.**

코드 전체를 grep한 결과:

| 확인 항목 | 결과 |
|---|---|
| API 키 요구 코드 | 없음 |
| 클라우드 인증 | 없음 |
| 텔레메트리 / 사용량 전송 | 없음 |
| 계정 가입 | 불필요 |
| 라이선스 | Apache 2.0 (상업 이용/수정/재배포 허용) |
| 페이지당 과금 | 없음 (무제한) |

공식 문서: *"No API key is required since everything runs on your machine."*

`Authorization: Bearer <token>`이 등장하는 곳은 **사용자가 직접 세운 OCR 서버**에 인증을
붙이는 옵션(`--ocr-server-header`)이며, LiteParse가 요구하는 토큰이 아니다.

네트워크가 나가는 경우는 3가지뿐:
1. `--ocr-server-url`을 직접 지정했을 때
2. URL에서 PDF를 받아올 때
3. Tesseract 언어 데이터 최초 다운로드 (`TESSDATA_PREFIX`로 오프라인 가능 → 에어갭 동작)

> 주의: 같은 회사의 **LlamaParse는 유료 클라우드 + API 키 필수**다. README가 "어려운 문서는
> LlamaParse"로 유도하는 것은 OSS 유입 → 클라우드 전환 비즈니스 모델이다.
> **LiteParse 자체는 완전 무료.**

---

## 8. GitHub에서 유명한 이유 (코드에서 찾은 근거)

> 실시간 스타 수는 이 분석 환경에서 확인하지 않았다. 아래는 코드/문서에서 확인된 구조적 요인.

1. **개발사가 LlamaIndex** — RAG 프레임워크 최상위 인지도. 공개 첫날부터 유통 채널이 확보됨.
2. **벤치마크를 숫자와 재현 스크립트로 제시** — `run_benchmarks.sh`(11KB), 같은 채점기,
   실행 날짜/머신 명시, 리더보드 수치 미혼합. 자기 약점(`headers_footers` OCR 역효과, Charts 0점)까지 공개.
3. **"AI 없이 AI를 이긴" 서사** — 순수 규칙 기반으로 페이지당 2~5ms, 상업용 도구 상회.
4. **시장 타이밍** — RAG 붐 + 기존 선택지의 결함(클라우드=비싸고 외부전송, 로컬 파이썬=느리고 표 못 읽음, LLM 파싱=비싸고 느림)을 동시에 해결.
5. **엔지니어링 완성도** — 바인딩 4개, 플랫폼 7개 타깃(musl 포함), 릴리스 파이프라인 5개,
   지옥 난이도 테스트 데이터, 프로덕션 배려(워커풀/타임아웃).
6. **기여 문턱이 낮다** — CHANGELOG에 외부 컨트리뷰터 다수, 이슈 템플릿 4종,
   `AGENTS.md`/`CLAUDE.md` 제공으로 AI 에이전트 기여 지원.

---

## 9. 로컬 에이전트 구축 활용법

LiteParse는 로컬 에이전트의 **문서 입력 장치**로서 현재 최선의 선택이다.

| 활용 | 설명 |
|---|---|
| 철학 일치 | API 키 없음 / 오프라인 / 프라이버시 → 로컬 에이전트 목적과 부합 |
| 즉시 도입 | 공식 에이전트 스킬 존재 → 바퀴 재발명 불필요 |
| 컨텍스트 관리 | `--target-pages`, `--format markdown`, `--extract-blocks`, 반복 머리말 자동 제거 |
| 라우터 | `lit is-complex`로 OCR/Vision 필요 페이지만 선별 → 비싼 호출 절약 |
| 시각 인식 | `lit screenshot`으로 차트/도형/손글씨 페이지만 Vision 모델에 전달 |
| 신뢰도 | bbox 좌표로 근거 제시 → 사람이 검증 가능 |
| 안정성 | 워커풀 + 하드 타임아웃으로 대량 배치 중단 방지 |

### 권장 아키텍처
```
사용자: "작년 계약서에서 위험 조항 찾아줘"
  |
로컬 에이전트 (Ollama / Claude Code)
  1. lit is-complex          → 스캔 문서 분류
  2. lit parse --format markdown → 파일로 저장
  3. 저장 파일을 grep / 랭크 검색  (재파싱 금지)
  4. 히트 주변 문맥만 LLM에 전달
  5. 차트 페이지는 lit screenshot → Vision
  6. 답변 + bbox로 근거 하이라이트
  |
답변 + "출처: contract.pdf 12p" + 형광펜 표시
```

---

## 10. React / PHP에서 쓸 수 있는가

### 10-A. LiteParse를 React/PHP로 재구현? → 하지 말 것

| 이유 | 설명 |
|---|---|
| 코드량 | Rust 55,074줄. `projection.rs` 5,783줄 + `tables.rs` 6,535줄 → 재구현 최소 1~2년 |
| PDFium 의존 | 핵심이 C 라이브러리. PHP는 FFI, JS는 WASM 필요 → 결국 LiteParse가 한 일 반복 |
| 성능 | 페이지당 2~5ms는 Rust여서 가능. PHP는 수십~수백 배 느려짐 |
| 엣지 케이스 | 회전 텍스트, 음수 폰트 크기, 고장난 AcroForm 등 수천 개 함정 재경험 |
| 결정타 | **Apache 2.0이라 그냥 쓰면 된다** |

### 10-B. React/PHP 앱에서 LiteParse 사용 → 둘 다 완전히 가능

**React 방법 1: WASM (서버 비용 0원, 파일 업로드 없음)**
```jsx
import init, { LiteParse } from "@llamaindex/liteparse-wasm";

async function handleFile(file) {
  await init();
  const parser = new LiteParse({ outputFormat: "markdown" });
  const bytes = new Uint8Array(await file.arrayBuffer());
  const result = await parser.parse(bytes);
  return result.pages.map(p => p.text).join("\n\n");
}
```
장점: 서버비 0원, 프라이버시 최강, 무한 확장 / 주의: WASM 초기 로딩, 브라우저 메모리 한계

**React 방법 2: Next.js API Route**
```typescript
// app/api/parse/route.ts
import { LiteParse } from "@llamaindex/liteparse";
const parser = new LiteParse({ outputFormat: "markdown", poolSize: 4 });

export async function POST(req: Request) {
  const form = await req.formData();
  const file = form.get("file") as File;
  const result = await parser.parse(Buffer.from(await file.arrayBuffer()));
  return Response.json({ markdown: result.pages.map(p => p.text).join("\n\n") });
}
```
주의: 네이티브 모듈이므로 Edge 런타임 불가. Node 런타임 / Docker 필요.

**React 방법 3: 별도 HTTP 서버**
```bash
docker run -p 5000:5000 ghcr.io/run-llama/liteparse-server:main
# POST /parse , POST /screenshots
```

**PHP 방법 1: CLI 호출 (가장 쉬움)**
```php
function parseDocument(string $path, string $format = 'markdown'): string {
    $cmd = sprintf('lit parse %s --format %s --quiet 2>&1',
        escapeshellarg($path), escapeshellarg($format));   // escapeshellarg 필수
    exec($cmd, $output, $code);
    if ($code !== 0) throw new RuntimeException("파싱 실패: " . implode("\n", $output));
    return implode("\n", $output);
}

function needsOcr(string $path): bool {
    exec(sprintf('lit is-complex %s --compact --quiet', escapeshellarg($path)), $o, $code);
    return $code !== 0;   // OCR 필요 시 non-zero
}
```
Laravel:
```php
use Illuminate\Support\Facades\Process;
$markdown = Process::run(['lit', 'parse', $path, '--format', 'markdown', '--quiet'])->output();
```

**PHP 방법 2: HTTP 서버 + curl (프로덕션 권장)**
```php
$ch = curl_init('http://localhost:5000/parse');
curl_setopt_array($ch, [
    CURLOPT_POST => true,
    CURLOPT_RETURNTRANSFER => true,
    CURLOPT_POSTFIELDS => ['file' => new CURLFile($path)],
]);
$json = json_decode(curl_exec($ch), true);
```

**PHP 방법 3: FFI로 .so 직접 호출** → 헤더 수동 정의 + 메모리 관리 부담. 비권장.

### 최종 추천

| 상황 | 추천 방식 |
|---|---|
| React 단독 (서버 없음) | WASM |
| Next.js 풀스택 | API Route + `@llamaindex/liteparse` |
| PHP 소규모 / 사내 | CLI (`exec`) |
| PHP 프로덕션 | HTTP 서버 + curl |
| 대량 배치 | Python/Node + 워커풀 |

---

## 11. 수익화 아이디어

### 11.0 전략 프레임

**피해야 할 함정**
1. "파싱 API를 판다" → LiteParse가 무료라 고객도 같은 걸 무료로 씀. 가격 경쟁 패배.
2. "LlamaParse보다 싸게" → 클라우드+LLM 경쟁자에게 정확도로 패배.
3. "개발자에게 판다" → 개발자는 직접 만들 수 있어 지불의사가 낮음.

**대신 3축으로**
- 축1 **로컬/온프레미스** — 클라우드가 진입 못 하는 시장
- 축2 **버티컬 워크플로우** — "파싱"이 아니라 "업무 완결"을 판매
- 축3 **한국 특화** — 글로벌이 하지 않는 영역

**가치 사다리**
```
lit parse (원본)                     → 무료. 판매 불가
+ 웹 UI, 배치, 큐                     → 편의성. 소액
+ 업종 스키마, 검증, 룰                → 도메인 지식. 유료
+ 근거 하이라이트, 감사 로그            → 신뢰/컴플라이언스. 고가
+ 온프레미스 설치 / 유지보수            → 안심. 최고가
```

---

### 아이디어 1: 온프레미스 문서 AI 솔루션 (수익 최대)

**무엇**: 금융/의료/법무/공공 내부망에 설치하는 완전 로컬 문서 처리 + RAG 시스템.

**왜 돈이 되나**: 은행은 계약서를 외부 API에 못 보내고, 병원은 진료기록을 클라우드에 못 올리고,
법무법인은 소송자료를 외부로 못 보낸다. **클라우드 경쟁자가 진입 불가한 시장**이며,
이 고객군은 예산이 크고, 계약이 길고, 이탈하지 않으며, 가격보다 컴플라이언스를 우선한다.

**구성**
```
고객 내부망 (인터넷 차단)
  LiteParse (문서 → 마크다운 + bbox)
    → 로컬 LLM (Ollama / vLLM)
    → 벡터DB (Qdrant / pgvector)
    → 웹 UI (React): 검색 / 질의 / 근거 하이라이트
    → 감사 로그 (열람 이력 전체 기록)
```

**가격**
| 항목 | 가격대 |
|---|---|
| 초기 구축 | 2,000만 ~ 1억원 |
| 연간 유지보수 | 구축비의 15~20% |
| 사용자 시트 | 1인 월 3~10만원 |
| 커스텀 스키마 | 건당 500만원~ |

**실행**: PoC 무료 제공(고객 문서 100개로 증명, `is-complex`로 난이도 정직 고지) →
Docker Compose 원클릭 패키지 → 에어갭 지원(`TESSDATA_PREFIX` + 오프라인 모델) → 감사 로그/권한

**리스크**: 영업 사이클 6개월~1년, 초기 레퍼런스 확보가 관문. 첫 고객은 저가/무료 + 레퍼런스화.

---

### 아이디어 2: 한국어 문서 특화 파서 (1순위 추천)

**무엇**: LiteParse를 한국 서식 전용으로 튜닝한 파서 + API + SaaS.

**공략 대상 서식** (글로벌 파서의 사각지대)
세금계산서, 등기부등본, 사업자등록증, 4대보험 납부확인서, 급여명세서,
건강보험자격득실확인서, 가족관계증명서, 표준근로계약서

LiteParse는 "표"로 읽어주지만 "이 칸이 공급가액"이라는 것은 모른다. **그 매핑이 판매 가치.**

**왜 1순위인가**
- 한국 기업은 데이터 외부 전송에 극도로 보수적 → 로컬이 강점
- 한국어 OCR은 PaddleOCR로 해결 가능 (`ocr/paddleocr` 이미 존재)
- 경쟁자가 거의 없음

**구성**
```
LiteParse (--extract-blocks, bbox 포함)
  → 한국 서식 분류기 ("이건 세금계산서")
  → 서식별 필드 추출 룰 (좌표 + 라벨 기반 매핑)
  → 검증 (사업자번호 체크섬, 합계 검산, 날짜 형식)
  → 구조화 JSON { 공급자, 공급가액, 세액, 품목[] }
```

**핵심 기술**: `--extract-blocks`가 표 셀마다 bbox를 주므로
"공급가액 라벨 오른쪽 셀 = 금액" 같은 좌표 기반 룰로 AI 없이 95%+ 정확도 가능.

**가격**
| 상품 | 가격 |
|---|---|
| API 종량제 | 1,000건당 5,000~2만원 |
| SaaS 스타터 | 월 3~5만원 (월 500건) |
| SaaS 비즈니스 | 월 20~50만원 (월 1만건) |
| 온프레미스 | 연 500만~3,000만원 |
| 신규 서식 추가 | 건당 100~300만원 |

**MVP 4주 플랜**
| 주 | 할 일 |
|---|---|
| 1주 | 서식 1종 선정(세금계산서 권장) + 샘플 100장 수집 |
| 2주 | `lit parse --format json --extract-blocks` → 필드 매핑 룰 작성 |
| 3주 | 검증 로직 + 정확도 측정 (목표 95%) |
| 4주 | 웹 업로드 UI + API 배포 |

> 원칙: 1종을 95%로 하는 것이 10종을 70%로 하는 것보다 훨씬 가치 있다.

**리스크**: 서식 변경 시 유지보수. 기존 OCR 업체와 경쟁 → 차별점은 로컬 + 저렴 + 빠름.

---

### 아이디어 3: 계약서 AI 검토 SaaS (bbox 하이라이트가 무기)

**무엇**: 계약서 업로드 → 위험 조항 자동 탐지 + 원본 위치 하이라이트 + 수정 제안.

**왜 돈이 되나**: 변호사 검토는 30만~200만원 + 3~7일. 변호사를 부를 만큼 크지 않은 계약
(프리랜서 계약, 임대차, NDA, 업무위탁, 공급계약)이 훨씬 많다. → **1차 스크리닝 시장**.

**킬러 기능**: bbox 기반 근거 하이라이트
```
AI: "제12조 손해배상에 상한이 없어 위험합니다"  [원본 보기]
  → PDF 4페이지 렌더링 + 해당 문장 하이라이트
```
법률 AI에서 가장 큰 불신 요소("AI가 지어낸 것 아닌가")를 3초 만에 해소한다.
**bbox 없는 파서로는 구현 불가한 방어선.**

**구성**
```
업로드 (PDF/DOCX)
  → LiteParse (--format json, bbox) + lit screenshot
  → 조항 단위 분할 (제N조 패턴 + 블록 정보)
  → 위험 판정: 룰 기반(상한 없는 배상, 자동갱신, 일방 해지, 불리한 관할)
               + LLM 기반(뉘앙스)
  → 리포트 + 원본 하이라이트 + 수정 문구 제안
```

**가격**
| 플랜 | 가격 |
|---|---|
| 무료 | 월 1건 |
| 개인/프리랜서 | 월 9,900원 (5건) |
| 스타트업 | 월 49,000원 (30건 + 팀 공유) |
| 기업 | 월 30만원~ (무제한 + 온프레미스) |
| 건당 | 9,900원 |

**실행**: 계약 종류 1개(NDA 또는 프리랜서 계약)부터 → 위험 조항 체크리스트 30개 작성(핵심 IP)
→ 룰 기반 우선, LLM 나중 → 하이라이트 UI를 최우선으로 완성

**리스크**: 법률 자문 오해 방지 문구 필수, 오탐 관리, 변호사법 저촉 여부 확인.
초기에는 "검토 보조 도구"로 명확히 포지셔닝.

---

### 아이디어 4: 영수증 / 증빙 → 회계 자동화

**무엇**: 영수증·세금계산서·카드전표 사진/PDF → 회계 전표 데이터 자동 생성.

**왜**: 소상공인/프리랜서/세무사무소의 최대 반복 노동. 수요가 계절적으로 폭발(부가세 1·4·7·10월, 종소세 5월).

**LiteParse 활용**: 이미지 → PDF 변환 내장(imagemagick 불필요),
`integration_tests_data/receipt.png` 존재, `--extract-blocks`로 품목/금액 매핑, PaddleOCR로 한국어 정확도.

**구성**
```
사진 다중 업로드 (또는 모바일 촬영)
  → LiteParse 배치 (워커풀 병렬)
  → 유형 분류 (카드전표 / 현금영수증 / 세금계산서 / 간이영수증)
  → 필드 추출 (사업자번호, 날짜, 공급가액, 부가세, 품목)
  → 검증 (사업자번호 체크섬, 합계 = 공급가액 + 세액)
  → 출력: 엑셀 / 회계 프로그램 포맷 / API
```

**가격**: 프리랜서 월 9,900원(100장) / 소상공인 월 29,000원(500장) / 세무사무소 월 15만원~ / 장당 50~100원

**리스크**: 기존 경쟁자 존재 → 차별점은 로컬 처리 / 저가 / 세무사무소 벌크.
정확도 90% 미달이면 수동이 더 빠르다는 평가를 받는다. 정확도가 생명.

---

### 아이디어 5: LiteParse MCP 서버 (인지도 자산)

**무엇**: LiteParse를 MCP 서버로 감싸 Claude Desktop / Cursor / Claude Code에 연결.
(현재 LiteParse에는 MCP 구현이 없음 → 빈 자리)

**왜**: 직접 수익은 작지만 최고의 마케팅 자산. 구현 난이도가 매우 낮다(약 200줄).

**툴 설계**
```
parse_document(path, format, targetPages)  → 마크다운 / JSON
check_complexity(path)                     → OCR 필요 페이지
screenshot_pages(path, pages, dpi)         → PNG 경로 (Vision용)
search_document(path, query)               → 매치 + bbox + 주변 문맥
batch_parse(dir, pattern)                  → 요약 인덱스
```

**차별 포인트**: 공식 스킬의 컨텍스트 절약 패턴을 서버가 강제
(1회 파싱 후 캐시, 검색은 캐시에서, 출력 항상 상한) → "컨텍스트 안 터지는 문서 MCP" 포지셔닝.

**수익**: 본체 오픈소스 무료 + Pro(랭크 검색, 캐시 DB, 다중 문서 교차 질의, 팀 공유) 월 1~2만원.

**실행**: 주말 2일 MVP. `@modelcontextprotocol/sdk` + `child_process`로 `lit` 호출 →
해시 기반 캐시 → GitHub 공개 + GIF 데모 → 커뮤니티 공유.

---

### 아이디어 6: 브라우저 전용 문서 도구 (서버비 0원 SaaS)

**무엇**: WASM으로 100% 브라우저에서 동작하는 문서 도구. 파일 업로드 자체가 없음.

**경제성**: 사용자가 10만 명이어도 인프라 비용이 늘지 않는다. 개인정보를 받지 않으므로
GDPR/개인정보법 대응 부담도 없다.

**제품 후보**
| 도구 | 설명 |
|---|---|
| PDF → 마크다운 변환기 | "AI에 넣기 좋게" 포지셔닝 |
| PDF 표 → 엑셀 추출기 | `--extract-blocks` 활용 |
| 다중 파일 텍스트 검색 | 배치 검색 |
| **문서 익명화 도구** | 정규식으로 주민번호/전화/계좌 탐지 + bbox로 위치 찾아 마스킹 |
| 계약서 버전 비교 | 두 버전 diff + 하이라이트 |

**수익**: 무료(페이지 제한) + Pro 월 4,900원(무제한/배치/OCR/히스토리) + 팀 월 29,000원

**실행**: `wasm-demo-site/index.html` 참고 → 도구 1개만 완성도 높게 →
Vercel/Cloudflare Pages 무료 정적 호스팅 → SEO 키워드 공략

---

### 아이디어 7: RAG 전처리 파이프라인 API

**무엇**: 문서 → RAG 준비 완료 청크 API (파싱 + 청킹 + 메타데이터 + 임베딩).

**차별점**: 경쟁자는 "문서 → 텍스트 청크", 우리는 "청크 + bbox + 페이지 + 블록타입 + 스크린샷 링크".
고객이 근거 표시 기능을 부가 비용 없이 얻는다.

**가격**: 무료 100페이지/월 / Dev 월 29,000원(5,000페이지) / Pro 월 99,000원(5만 페이지) / 온프레미스 연 계약

**리스크**: 개발자 대상 지불의사 낮음, LlamaIndex/Unstructured와 경쟁.
살 길은 "bbox 붙은 청크" + "로컬 배포 가능"으로 좁게 파기.

---

### 아이디어 8: 논문 / 리서치 도구

**무엇**: 논문 PDF → 마크다운, 참고문헌 파싱, 표 추출, 다중 논문 비교.

**궁합**: `multi_column` 69.1점(2등 49.7), `long_tiny_text` 46.4점 → 2단 논문/각주에 강함.
**약점**: 수식(LaTeX) 0점 → 수식은 별도 도구(Nougat 등) 필요.

**수익**: 연구자 월 4,900원 / 연구실 팀 월 29,000원 / 대학 기관 라이선스.
난이도 낮고 수요 확실하지만 수익 규모는 작음. 첫 프로젝트 연습용으로 적합.

---

### 11.9 종합 비교

| # | 아이디어 | 개발 난이도 | 영업 난이도 | 수익 규모 | 경쟁 | 추천도 |
|---|---|:---:|:---:|:---:|:---:|:---:|
| 1 | 온프레미스 문서 AI | 높음 | 매우 높음 | 최상 | 낮음 | 4/5 |
| 2 | **한국어 특화 파서** | 중 | 중 | 상 | 매우 낮음 | **5/5** |
| 3 | 계약서 검토 SaaS | 중 | 중 | 상 | 중 | 4/5 |
| 4 | 영수증 회계 자동화 | 중 | 중 | 상 | 높음 | 3/5 |
| 5 | **MCP 서버** | 낮음 | 낮음 | 하 | 없음 | **5/5** |
| 6 | 브라우저 도구 | 낮음 | 낮음 | 중 | 중 | 4/5 |
| 7 | RAG 전처리 API | 중 | 높음 | 중 | 높음 | 2/5 |
| 8 | 논문 도구 | 낮음 | 낮음 | 하 | 중 | 2/5 |

### 11.10 3단계 로드맵

```
[1단계 — 1개월]  인지도 + 실력
  아이디어 5 (MCP 서버) 오픈소스 공개
  아이디어 6 (브라우저 도구 1개) 배포
  → 난이도 낮음, LiteParse 체득, 포트폴리오 + 인지도 확보

[2단계 — 3~6개월]  첫 매출
  아이디어 2 (한국어 특화, 세금계산서 1종부터)
  → 경쟁 거의 없음, B2B 단가 양호, 1종만으로도 판매 가능
  → 1단계 인지도로 초기 고객 확보

[3단계 — 6~12개월]  스케일
  아이디어 3 (계약서 SaaS) 또는 아이디어 1 (온프레미스)
  → 2단계 레퍼런스로 영업
  → bbox 하이라이트 = 경쟁사가 따라오기 어려운 기술 방어선
```

### 11.11 핵심 원칙 5개

1. **파서를 팔지 말고 결과를 팔아라.** "PDF 파싱"이 아니라 "월말 정산 3시간 → 3분".
2. **로컬이 최고의 세일즈 포인트다.** "데이터가 서버 밖으로 나가지 않습니다" 한 줄로 클라우드 경쟁자가 탈락한다.
3. **범위를 좁혀라.** "모든 문서"가 아니라 "세금계산서 1종 95% 정확도".
4. **bbox 하이라이트는 기술 방어선이다.** 대부분의 파서가 제공하지 못하며, AI 불신 시대에 가격을 올려주는 기능이다.
5. **`is-complex`는 마진 관리 도구다.** 싸게 미리 분류해 비싼 처리를 아끼는 것이 곧 이익률이다.

---

## 12. 결론

LiteParse는 AI/RAG 파이프라인에서 **가장 어렵고 품질을 가장 크게 좌우하는 첫 단계**를
세계 최고 수준으로, 무료로, 오프라인에서 해결해 주는 도구다.

- 만들어야 할 것: **파서 위에 얹을 도메인 가치** (서식 매핑, 검증 룰, 워크플로우, 근거 UI)
- 만들지 말아야 할 것: **파서 자체** (Apache 2.0으로 이미 공짜)

---

## 부록: 이 문서가 다룬 대화 순서

| # | 질문 | 문서 위치 |
|---|---|---|
| 1 | 전수조사 — 뭐하는 건지, 언제 쓰는지, 무슨 도움이 되는지 | 1~4장 |
| 2 | 더 쉽게 상세히 설명 | 1장(문제 정의), 2장(아키텍처) |
| 3 | 설치/사용법, 플러그인·스킬·MCP, API 토큰, 유명한 이유, 로컬 에이전트, 수익화, React·PHP | 5~11장 |
| 4 | 수익화 아이디어 상세 | 11장 |
| 5 | 대화 정리 + GitHub 주소 포함 저장 후 머지 | 이 문서 |
