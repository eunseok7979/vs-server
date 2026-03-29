# VS Server (Visual-Sonic) — CLAUDE.md

최종 업데이트: 2026-03-29 v5 (v0.6)
서버: sussertod | P100 16GB (CC 6.0) | i3-7100 (AVX2) | Ubuntu 24.04  
Tailscale: 100.114.30.82 | 로컬: 192.168.0.66 | 프로젝트: ~/vs-server/  
conda 환경: `ocr`

---

## 프로젝트 비전

VS Server는 P100 기반 로컬 AI 범용 서버.
- **Jingyeskan(징계 자료 자동화)** — Virtual World(텍스트 분석)의 GPU 백엔드
- 영수증 정리, 문서 OCR 등 부가 기능 제공
- 상세 로드맵: `roadmap.md`

### Virtual World / Material World 구조

징계 자료는 두 갈래로 처리:
- **Virtual World (VS Server 웹)**: 원본 → OCR → 텍스트 → 다운로드 → Windows에서 쟁점 추출/요약 (Claude API)
- **Material World (Windows)**: 원본 → 이미지 그대로 한글(HWPX) 문서로 변환 (Jingyeskan-1.0)

---

## 현재 상태 (v0.6)

### 완성된 기능
- ✅ vs_server.py v0.6: Whisper + PaddleOCR + Ollama 프록시
- ✅ Ollama 프록시 엔드포인트: `/transcript/polish`, `/ocr/ollama`
- ✅ VRAM 관리: 로컬 모델 ↔ Ollama 모델 자동 스왑
- ✅ 녹취록 정리: Whisper + Clova Note 화자매칭
- ✅ Ollama 서빙: Qwen3.5 9B + Qwen3-VL 8B + DeepSeek OCR + Gemma 3 12B

### 다음 할 일
- [ ] Jingyeskan 웹 UI 프로토타입 (사건별 자료 업로드 → OCR/전사 → 텍스트 다운로드)
- [ ] 기존 HTML 클라이언트 통합: StaticFiles로 `ui/` 서빙
- [ ] 접근 제어 미들웨어

---

## API 엔드포인트 (v0.6)

| 엔드포인트 | 메서드 | 입력 | 엔진 |
|-----------|--------|------|------|
| `/transcribe` | POST | UploadFile + language | Whisper large-v3 |
| `/transcript/polish` | POST | JSON {text, model, system, options} | Ollama (Qwen3.5/Gemma3) |
| `/ocr/paddle` | POST | UploadFile | PaddleOCR PP-OCRv5 |
| `/ocr/ollama` | POST | UploadFile + model + prompt + system | Ollama (Qwen3-VL/DeepSeek) |
| `/health` | GET | — | 상태 확인 |

### 프록시 설계 원칙
- 시스템 프롬프트는 클라이언트(HTML)가 보냄 — 서버는 순수 프록시
- Ollama 요청 시 FastAPI가 로컬 모델(Whisper/PaddleOCR) 언로드
- 로컬 모델 요청 시 FastAPI가 Ollama 모델 언로드 (keep_alive=0)

---

## 모델 역할 분담 (확정)

| 태스크 | 모델 | 서빙 |
|--------|------|------|
| 녹취록 정리 (구어체→문어체) | Qwen3.5 9B, **thinking OFF 필수** | Ollama |
| 녹취록 정리 (보조 옵션) | Gemma 3 12B | Ollama |
| 논점 추출/요약 | Claude Sonnet (외부 API) | Windows에서 호출 |
| 녹취록 다듬기 (외부 대안) | Gemini Flash Lite (외부 API) | — |
| OCR — 이미지/손글씨/인쇄물 | Qwen3-VL 8B | Ollama |
| OCR — 인쇄물 전용 (빠름) | DeepSeek OCR | Ollama |
| OCR — 전통 경량 | PP-OCRv5 | FastAPI `/ocr/paddle` |
| 음성 전사 | Whisper large-v3 | FastAPI `/transcribe` |

---

## 핵심 제약 (건드리기 전 반드시 확인)

1. **CC 6.0**: bfloat16 불가, Flash Attention 2 불가 → float16 + `attn_implementation="sdpa"` or `eager`
2. **CUDA cu118 필수**: cu126은 CC 6.0 커널 미포함 → CUDA error 209. 단, CC가 영향하는 건 GPU 커널 코드뿐. CUDA 호스트 라이브러리(libcusparse 등)는 CC와 무관하고 API/ABI 버전 호환만 필요.
3. **VRAM 16GB**: 모델 동시 로딩 불가, 순차 스왑 필수 (Ollama도 모델 간 스왑)
4. **VLM 서빙은 Ollama만**: llama.cpp 직접 빌드에서 qwen2vl 아키텍처 MUL_MAT CUDA error 발생
5. **GGUF bf16**: P100 bf16 하드웨어 미지원 → `llama-quantize input.gguf output.gguf F16`으로 변환
6. **Ollama Thinking 제어**: `think: false`는 Chat API(`/api/chat`)에서만 동작. Generate API에서는 무시됨
7. **GPU 열 스로틀링**: P100이 80°C 근접 시 스로틀링 발생 → 12B 모델 추론 시 속도 저하
8. **PaddleOCR cuDNN 8 의존성**: conda 환경에 cuDNN 8 수동 설치 필요. `libcusparse`/`libnvJitLink` 충돌 시 conda 환경 것을 제거(`~/vs-server/lib/backup/`)하고 시스템 CUDA 사용. `start.sh`에 `LD_LIBRARY_PATH` 설정 필수.

---

## Ollama API 빠른 참조

엔드포인트: `http://localhost:11434` (로컬) / `http://100.114.30.82:11434` (Tailscale)

**Qwen3.5 9B — 녹취록 정리 (thinking OFF)**
```bash
curl http://localhost:11434/api/chat -d '{
  "model": "qwen3.5:9b",
  "messages": [
    {"role": "system", "content": "당신은 한국어 녹취록 전문 편집자입니다..."},
    {"role": "user", "content": "녹취록 텍스트..."}
  ],
  "stream": false,
  "think": false,
  "options": {"num_ctx": 32768, "temperature": 0.7, "top_p": 0.8, "top_k": 20}
}'
```

**Qwen 권장 파라미터**
- Thinking OFF: `temperature=0.7, top_p=0.8, top_k=20`
- Thinking ON: `temperature=0.6, top_p=0.95, top_k=20` (Greedy/temperature=0 금지 — 반복 출력)

**Gemma 3 권장 파라미터**
- 녹취록 정리용: `temperature=0.7, top_p=0.8, top_k=20`
- Thinking mode 없음 — 프롬프트로 "결과만 출력" 지시 필요

---

## 접근 제어 구조

| 접속 경로 | IP | 보이는 기능 |
|----------|-----|-----------|
| Tailscale | 100.114.30.82:8000 | 전체 (녹취록, OCR, 영수증, Jingyeskan) |
| 내부망 | 192.168.0.66:8000 | 영수증 정리만 |

---

## 파일 구조

```
~/vs-server/
├── vs_server.py        # 통합 서버 v0.6 (FastAPI + Ollama 프록시)
├── start.sh            # conda ocr 활성화 + LD_LIBRARY_PATH + uvicorn
├── lib/
│   ├── cudnn8/         # cuDNN 8 심볼릭 링크
│   └── backup/         # 충돌 라이브러리 백업 (libcusparse, libnvJitLink)
├── ui/                 # 클라이언트 HTML (StaticFiles 서빙) [통합 예정]
│   ├── vs-client.html  # 녹취록 정리 (전사 + 화자매칭 + 다듬기)
│   ├── vs-document.html# 문서 OCR
│   └── vs-receipt.html # 영수증 정리
├── docs/
│   ├── architecture.md
│   ├── models.md
│   ├── pipeline.md
│   ├── roadmap.md
│   └── history.md
├── roadmap.md          # 프로젝트 로드맵 (최신)
└── CLAUDE.md           # 이 파일

별도 프로젝트 (GitHub: eunseok7979)
├── claudemain          # 징계 자료 자동화 CLI (Windows) — 쟁점 추출/요약
├── Jingyeskan-1.0      # PDF→한글 변환 (Material World)
├── PDF-to-Epub
├── transcript_tool.py / transcript_tool_gui.py  # 녹취록 다듬기 독립 도구
```

환경변수: `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`
