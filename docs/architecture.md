# VS Server — 아키텍처 & 인프라

최종 업데이트: 2026-03-27

---

## 하드웨어

| 항목 | 상태 | 비고 |
|------|------|------|
| GPU | Tesla P100-PCIE-16GB (CC 6.0) | bfloat16 불가, Flash Attention 불가, float16 사용 |
| CPU | i3-7100 (AVX2 지원) | PaddlePaddle 네이티브 가능. LGA 1151, B150 보드 |
| CUDA | cu118 필수 | cu126은 CC 6.0 커널 미포함 (CUDA error 209) |
| 네트워크 | SSH + Tailscale | 외부: 100.114.30.82:8000 (FastAPI), :11434 (Ollama) |
| 쿨링 | P100 끝단 초고속 소형팬 2개 + 몰렉스 팬 컨트롤러 | 쿨링 양호. 컨트롤러로 인해 소음 변동 이슈 있음 |

NUC(sussertod)도 같은 네트워크에서 운용 중.

---

## 서빙 구조: 이중 백엔드

```
클라이언트 (브라우저)
    │
    ├── http://100.114.30.82:8000   → FastAPI (vs_server.py)
    │       ├── /transcribe          → Whisper large-v3
    │       ├── /ocr/paddle          → PP-OCRv5 (PaddlePaddle 네이티브)
    │       ├── /ocr/qwen (레거시)   → Qwen2.5-VL-3B (교체 예정)
    │       └── /ocr/varco (제거 예정)
    │
    └── http://100.114.30.82:11434  → Ollama
            ├── qwen3.5:9b           → 녹취록 정리
            ├── qwen3-vl:8b          → 이미지 OCR
            └── deepseek-ocr:latest  → 인쇄물 OCR
```

---

## Ollama 서빙 (주력)

Ollama가 자체 llama.cpp를 번들하며, P100 CC 6.0에서 VLM을 안정적으로 GPU 가속 실행. Flash Attention, bfloat16, Transformers 버전 충돌 등 기존 Python 경로의 모든 호환성 문제를 우회.

### 모델 목록

| 모델 | Ollama 모델명 | 크기 | 용도 |
|------|-------------|------|------|
| Qwen3.5 9B | `qwen3.5:9b` | ~6.6GB (Q4) | 녹취록 정리 (구어체→문어체). **Thinking OFF 필수** |
| Qwen3-VL 8B | `qwen3-vl:8b` | ~5GB (Q4) | 이미지 OCR (인쇄물+손글씨+문서 이해) |
| DeepSeek OCR | `deepseek-ocr:latest` | ~6.7GB | 인쇄물 전용 OCR (텍스트 전용 모델, mmproj 없음) |

### Qwen3.5 9B 상세

- Early-fusion 멀티모달 학습. 텍스트 처리에서 Qwen3-VL 대비 상위
- 한국어 현대 문어체 변환 정확 (Qwen3-VL은 고전 문어체로 오해하는 문제 있음)
- **Thinking ON 시 주의사항**: pathological self-doubt 패턴 — 자기 의심 반복으로 품질 저하, 속도 3배 느림. 정체성 hallucination 발생 가능 (구글이라고 자칭 등)
- 텍스트 처리 태스크에서는 **thinking OFF 필수**

### Qwen3-VL 8B 상세

- 범용 VLM: OCR + 문서 이해, 요약, 번역, 질의응답 가능
- 한국어 인쇄물 OCR 우수 (캘리번과 마녀 서적 테스트 통과 — 오탈자, 띄어쓰기, 그림 인식 완벽)
- 한국어 손글씨 OCR 가능 (일반 필체. 극악 필체는 문맥 추론으로 보완, hallucination 주의)
- **텍스트 전용 태스크에서는 열위**: 한국어 문체 이해 부정확, `think: false`가 무시됨

### DeepSeek OCR 상세

- 텍스트 전용 모델 (비전 인코더 mmproj 없음, LLM 단독)
- 인쇄물 OCR에서 완벽한 정확도
- 손글씨는 반복 hallucination 발생

### Thinking 제어 (Ollama)

| 방법 | API | 동작 |
|------|-----|------|
| `think: false` (top-level) | Chat API(`/api/chat`)만 | ✅ 정상 |
| `/no_think` (프롬프트 끝) | Generate/Chat 둘 다 | ✅ 동작 (소프트 스위치) |
| `think: true` | Chat API | ✅ 정상 |
| Thinking budget | — | ❌ 미지원 |

> **주의**: Generate API(`/api/generate`)에서는 `think: false`가 무시됨 (알려진 버그).

---

## FastAPI 서버 (vs_server.py v0.5)

실행: `~/vs-server/start.sh` (conda ocr 활성화 + uvicorn)  
환경변수: `PYTORCH_CUDA_ALLOC_CONF=expandable_segments:True`

### 엔드포인트

| 엔드포인트 | 모델 | VRAM | 상태 |
|-----------|------|------|------|
| `GET /health` | — | — | ✅ 현재 로드 모델 + GPU 상태 반환 |
| `POST /transcribe` | Whisper large-v3 | ~9.7GB | ✅ word_timestamps + 중복 제거 포함 |
| `POST /ocr/paddle` | PP-OCRv5 (korean) | ~1.2GB | ✅ PaddlePaddle cu118 네이티브 |
| `POST /ocr/qwen` | Qwen2.5-VL-3B | ~7.0GB | 레거시, Qwen3-VL로 교체 예정 |
| `POST /ocr/varco` | VARCO-VISION-2.0-1.7B-OCR | ~4-8GB | 제거 예정 (단어 조각 이슈) |

### ModelManager

모든 모델 on-demand 스왑: 요청 시 현재 모델 언로드 → 요청 모델 로드 → 추론.  
시작 시 아무 모델도 로드하지 않음. `gc.collect()` + `torch.cuda.empty_cache()`로 VRAM 정리.

---

## Whisper 전사 서비스

- 모델: OpenAI Whisper large-v3 (openai/whisper 패키지, PyTorch, fp16=True)
- VRAM: ~9.7GB
- 성능: 1시간 녹취 → ~15-23분 처리 (realtime factor 0.4)
- 출력: segments + words (중복 제거 적용) + meta
- 중복 제거: `deduplicate_phrases()` (반복 구문) + `deduplicate_stutters()` (시간 근접 동일 단어)
- word_timestamps: True — Clova Note 화자분리 매칭용

---

## PaddleOCR PP-OCRv5

- 엔진: PaddlePaddle cu118 네이티브, PP-OCRv5 korean
- VRAM: ~1.2GB
- 출력: lines (텍스트 + score + bbox) + plain_text
- 특징: 전통 OCR 파이프라인 (detection + recognition), 빠르고 가벼움
- PP-DocLayoutV3: safetensors float16에서 P100 동작 확인 (레이아웃 분석용)
