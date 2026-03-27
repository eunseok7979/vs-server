# VS Server — OCR 모델 테스트 현황

최종 업데이트: 2026-03-27

---

## 주력 모델 (사용 중)

| 모델 | 크기 | 서빙 | 인쇄물 | 손글씨 | 텍스트 처리 | 비고 |
|------|------|------|:---:|:---:|:---:|------|
| Qwen3.5 9B | 9B | Ollama | — | — | ✅ 최고 | 녹취록 정리 전용. thinking OFF 필수. 현대 문어체 정확 |
| Qwen3-VL 8B | 8B | Ollama | ✅ 완벽 | ✅ 가능 | ⚠️ 열위 | 이미지 OCR 전용. 텍스트 처리 시 고전체 혼동, thinking 제어 불가 |
| DeepSeek OCR | ~6.7B | Ollama | ✅ 완벽 | ❌ hallucination | — | 인쇄물 강점. 텍스트 전용 모델 (mmproj 없음) |

## 보조/레거시 모델

| 모델 | 크기 | 서빙 | 상태 | 비고 |
|------|------|------|------|------|
| PP-OCRv5 | 경량 | FastAPI | ✅ 동작 | PaddlePaddle cu118 네이티브, 전통 OCR |
| PP-DocLayoutV3 | 경량 | FastAPI | ✅ 동작 | safetensors float16, 레이아웃 분석용 |
| Qwen2.5-VL-3B | 3B | FastAPI (레거시) | 교체 예정 | Qwen3-VL 8B로 대체 |

---

## 테스트 완료 — 제외된 모델

| 모델 | 제외 사유 |
|------|----------|
| **PaddleOCR-VL-1.5** | P100에서 모든 경로 불가. PaddlePaddle 네이티브 불가, Transformers 경로 전후처리 미지원(000000 반환), llama.cpp paddleocr 아키텍처 CC 6.0 CUDA 미지원, Ollama도 아키텍처 미인식. bf16→f16 변환 시도했으나 근본적으로 paddleocr 아키텍처의 CUDA 커널이 CC 6.0 미지원 |
| **VARCO-VISION-2.0-1.7B-OCR** | GPU 테스트에서 단어가 조각조각 남 → 실사용 부적합 |
| **GLM-OCR** | 한국어 부적합 |
| **Nanonets-OCR2-3B** | Ollama에서 P100 GPU 동작 확인 (qwen2vl 아키텍처). 그러나 한국어 인식 불가 — 인쇄물에서도 일본어/영어 혼합 출력. 학습 데이터에 Korean 포함 표기되었으나 실제 한국어 OCR 능력 없음 |
| **Ocean-OCR 3B** | GGUF/Ollama 미지원 (HF 원본 weights만 공개, 커스텀 아키텍처) |

---

## llama.cpp 직접 빌드 관련 (참고)

- 빌드: build 8547, `CMAKE_CUDA_ARCHITECTURES=60`, `libggml-cuda.so`에 sm_60 커널 포함 확인
- 문제: qwen2vl 아키텍처 모델이 llama.cpp 직접 빌드에서 MUL_MAT CUDA error 발생 (bf16/f16 무관, Flash Attention off 무관)
- 동일 모델이 Ollama에서는 GPU 정상 동작 → Ollama 번들 llama.cpp 버전이 다른 것으로 추정
- **결론: VLM 서빙은 llama.cpp 직접 빌드 대신 Ollama 사용**

---

## 신규 모델 테스트 체크리스트 (P100 + Ollama)

새로운 OCR/VLM 모델을 P100에서 Ollama로 테스트할 때 아래 순서로 확인:

### 1단계: Ollama 공식 라이브러리
- https://ollama.com/library
- 여기 있으면 `ollama run 모델명`으로 바로 실행 가능. 가장 안전한 경로.

### 2단계: Ollama 커뮤니티 업로드
- https://ollama.com/search 에서 검색
- 누구나 GGUF를 Modelfile로 업로드 가능. 품질 보장 없음 (변환 오류, 템플릿 문제 가능)
- 이상하면 HuggingFace에서 공식 GGUF를 받아 직접 Modelfile로 등록하는 것이 나음

### 3단계: HuggingFace GGUF → Ollama 직접 실행
- `ollama run hf.co/{사용자}/{레포}:{양자화}` 로 HF GGUF 바로 실행
- 또는 GGUF 다운로드 후 Modelfile 작성 → `ollama create`
- 비전 모델은 mmproj를 Modelfile에서 ADAPTER로 지정 필요

### 4단계: 아키텍처 확인 (가장 중요)

Ollama가 아키텍처를 인식해야 모델이 동작함. GGUF 내부 `general.architecture` 메타데이터로 확인:

```bash
python3 -c "
from gguf import GGUFReader
r = GGUFReader('모델.gguf')
[print(f'{f.name}: {bytes(f.parts[-1])}') for f in r.fields.values() if 'architecture' in f.name]
"
```

| 아키텍처 | P100 동작 여부 |
|---------|-------------|
| qwen2vl | ✅ 동작 확인 |
| llama | ✅ 동작 확인 |
| deepseek | ✅ 동작 확인 |
| paddleocr | ❌ Ollama 미인식 |

qwen2vl, llama, gemma 같은 메이저 아키텍처 기반이면 안심하고 시도.

### GGUF bf16 주의

P100은 bf16 하드웨어 미지원. bf16 GGUF는 변환 필요:
```bash
./build/bin/llama-quantize input.gguf output.gguf F16
# LLM과 mmproj 둘 다 변환 필요
```
