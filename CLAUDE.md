# VS Server (Visual-Sonic) — CLAUDE.md

최종 업데이트: 2026-03-27 v3  
서버: sussertod | P100 16GB (CC 6.0) | i3-7100 (AVX2) | Ubuntu 24.04  
Tailscale: 100.114.30.82 | 로컬: 192.168.0.66 | 프로젝트: ~/vs-server/  
conda 환경: `ocr`

---

## 현재 상태

### 완성된 기능
- ✅ 녹취록 정리: Whisper + Clova Note 화자매칭 (완성도 최고)
- ✅ ModelManager v0.5: 4모델 on-demand 스왑
- ✅ Ollama 서빙 확립: Qwen3.5 9B + Qwen3-VL 8B + DeepSeek OCR (P100 GPU 동작)

### 지금 당장 할 일 (최우선)
- [ ] vs_server.py에 Ollama 프록시 엔드포인트 추가 (`/ocr/deepseek`, `/ocr/qwen3vl`)
- [ ] vs-document.html에 Ollama 모델 선택 옵션 추가
- [ ] VARCO-VISION 엔드포인트 제거, Qwen2.5-VL-3B → Qwen3-VL 교체

### 다음 작업
- [ ] 클라이언트 통합: HTML 파일을 `~/vs-server/ui/`로 모으고 FastAPI StaticFiles 서빙
      → `http://100.114.30.82:8000/ui/` 단일 접근, bat 파일 불필요, pdf.js 문제 해소
- [ ] faster-whisper 전환 검토 (CTranslate2, 속도 ~4배, VRAM 절감)
- [ ] systemd 서비스 등록 (현재 start.sh 수동 실행)
- [ ] whisper_server.py 정리 (vs_server.py에 통합되어 불필요)
- [ ] 팬 컨트롤러 소음 변동 해결 (몰렉스 → 고정 전압 검토)

---

## 모델 역할 분담 (확정)

| 태스크 | 모델 | 서빙 |
|--------|------|------|
| 녹취록 정리 (구어체→문어체) | Qwen3.5 9B, **thinking OFF 필수** | Ollama |
| 논점 추출/요약 | Claude Sonnet (외부 API) | — |
| OCR — 이미지/손글씨/인쇄물 | Qwen3-VL 8B | Ollama |
| OCR — 인쇄물 전용 (빠름) | DeepSeek OCR | Ollama |
| OCR — 전통 경량 | PP-OCRv5 | FastAPI `/ocr/paddle` |
| 음성 전사 | Whisper large-v3 | FastAPI `/transcribe` |

---

## 핵심 제약 (건드리기 전 반드시 확인)

1. **CC 6.0**: bfloat16 불가, Flash Attention 2 불가 → float16 + `attn_implementation="sdpa"` or `eager`
2. **CUDA cu118 필수**: cu126은 CC 6.0 커널 미포함 → CUDA error 209
3. **VRAM 16GB**: 모델 동시 로딩 불가, 순차 스왑 필수 (Ollama도 모델 간 스왑)
4. **VLM 서빙은 Ollama만**: llama.cpp 직접 빌드에서 qwen2vl 아키텍처 MUL_MAT CUDA error 발생
5. **GGUF bf16**: P100 bf16 하드웨어 미지원 → `llama-quantize input.gguf output.gguf F16`으로 변환
6. **Ollama Thinking 제어**: `think: false`는 Chat API(`/api/chat`)에서만 동작. Generate API에서는 무시됨 → 프롬프트에 `/no_think` 삽입으로 우회
7. **Transformers 경로 주의**: model weights만 로드되는 경우 있음 — 전처리/후처리 포함 여부 확인 필수

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

**Whisper 전사 (PowerShell)**
```powershell
curl.exe -X POST http://100.114.30.82:8000/transcribe `
  -F "file=@C:\경로\파일.m4a" -F "language=ko"
```

---

## 파일 구조

```
~/vs-server/
├── vs_server.py        # 통합 서버 v0.5 (FastAPI)
├── start.sh            # conda ocr 활성화 + uvicorn
├── whisper_server.py   # 레거시 (불필요, 정리 예정)
└── __pycache__/

클라이언트 (현재 Windows 로컬에 산재 → ui/ 통합 예정)
├── vs-receipt.html     # 녹취록 정리 UI (file:// 직접 실행 가능)
└── vs-document.html    # 문서 OCR UI (python -m http.server 8090 필요)

파생 프로젝트
├── PDF-to-EPUB         # GitHub: eunseok7979/PDF-to-Epub / conda: pdfepub
└── 영수증 정리          # vs-receipt.html 기반, 정교화 예정
```

환경변수: `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`

---

## 참고 문서

- 하드웨어/서빙 아키텍처 상세: `docs/architecture.md`
- OCR 모델 테스트 현황 + 신규 테스트 체크리스트: `docs/models.md`
- 화자분리 파이프라인 + 클라이언트 UI: `docs/pipeline.md`
- 프로젝트 대화 히스토리: `docs/history.md`
