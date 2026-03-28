# VS Server — 로드맵

최종 업데이트: 2026-03-28

---

## 전체 비전

VS Server는 P100 기반 로컬 AI 범용 서버로, Jingyeskan(징계 절차 관리)의 백엔드이자 영수증 정리·문서 OCR 등 부가 기능을 제공한다.

```
VS Server (FastAPI + Ollama, 단일 포트 8000)
├── AI 엔진 계층
│   ├── Whisper large-v3 (음성 전사)
│   ├── Ollama
│   │   ├── Qwen3.5 9B — 녹취록 다듬기 (thinking OFF)
│   │   ├── Gemma 3 12B — 녹취록 다듬기 (보조 옵션)
│   │   ├── Qwen3-VL 8B — 이미지 OCR
│   │   └── DeepSeek OCR — 인쇄물 OCR
│   └── PaddleOCR PP-OCRv5 (경량 전통 OCR)
│
├── API 계층 (FastAPI)
│   ├── /transcribe — Whisper 전사
│   ├── /transcript/polish — 녹취록 다듬기 (모델 선택 가능)
│   ├── /ocr/paddle — PaddleOCR
│   ├── /ocr/ollama — Ollama VLM OCR (모델 선택 가능)
│   └── /health
│
└── UI 계층 (StaticFiles, ~/vs-server/ui/)
    ├── Tailscale IP (100.114.30.82) → 전체 기능
    │   ├── 녹취록 정리 (전사 + 화자매칭 + 다듬기)
    │   ├── 문서 OCR
    │   ├── 영수증 정리
    │   └── Jingyeskan (징계 절차 관리) [장기]
    │
    └── 내부망 IP (192.168.0.66) → 영수증 정리만
```

### 접근 제어

- Tailscale IP 접속 → 전체 기능 (관리자 = 본인만)
- 내부망 IP 접속 → 영수증 정리만 (사무실 동료)
- Tailscale은 같은 LAN에서 P2P 직접 연결 → 내부망과 속도 동일
- 서버 하나, 포트 하나, 추가 인증 없음

---

## Phase 1: 서버 정리 + 클라이언트 통합 (v0.6)

**목표**: 산재한 클라이언트를 통합하고, Ollama 엔드포인트를 추가하여 현재 동작하는 모든 기능을 하나의 서버에서 제공

### 서버 (vs_server.py)

- [ ] VARCO 관련 코드 전부 제거
- [ ] Qwen2.5-VL-3B (Transformers 경로) 제거
- [ ] Ollama 프록시 엔드포인트 추가
  - `/transcript/polish` — 녹취록 다듬기 (model 파라미터로 gemma3:12b / qwen3.5:9b 선택)
  - `/ocr/ollama` — VLM OCR (model 파라미터로 qwen3-vl:8b / deepseek-ocr 선택)
- [ ] StaticFiles로 `ui/` 서빙
- [ ] 접근 제어 미들웨어: 요청 Host IP로 Tailscale/내부망 구분
- [ ] health 엔드포인트 업데이트 (Ollama 모델 상태 포함)
- [ ] Whisper, PaddleOCR 엔드포인트 유지 (변경 없음)

### 클라이언트 (ui/)

- [ ] HTML 파일들을 `~/vs-server/ui/`로 이동
- [ ] vs-client.html (녹취록 정리) — 다듬기 기능 추가, 모델 선택 드롭다운
- [ ] vs-document.html (문서 OCR) — Ollama 모델 선택 옵션, hwpx-templates.js 포함
- [ ] vs-receipt.html (영수증 정리) — VARCO → Ollama OCR로 전환
- [ ] 내부망 접속 시 영수증 정리 전용 UI (또는 라우팅)

### 기타

- [ ] CLAUDE.md 업데이트 (v0.6 반영)
- [ ] whisper_server.py 삭제 (레거시)
- [ ] start.sh 업데이트

---

## Phase 1.5: Qwen3.5 35B 테스트 (Phase 1 이후 검토)

**배경**: Qwen3.5는 Gated DeltaNet + MoE 아키텍처로 CPU 오프로딩에 매우 유리. 48개 레이어 중 어텐션 레이어가 12개뿐이고 KV캐시도 12개 레이어에만 필요. CPU↔GPU 전환이 빈번하지만 성능 저하가 적음. RTX 3090에서 122B를 VRAM 5.5GB만 쓰고 초당 7토큰 달성 사례 있음.

**P100에서의 가능성**:
- Qwen3.5 35B A3B (Q4): ~22GB. P100 VRAM 16GB + 나머지 CPU 오프로딩
- 서버 RAM 32GB 업그레이드 필요 (현재 미확인 → 확인 후 결정)
- P100 대역폭 732GB/s(HBM2)는 양호하나, i3-7100(2코어) CPU가 병목 가능성
- 녹취록 다듬기는 실시간 불필요 → 속도 느려도 품질 향상이 크면 가치 있음

**할 일**:
- [ ] 서버 RAM 용량 확인 및 32GB 업그레이드 검토
- [ ] `ollama pull qwen3.5:35b` 후 녹취록 다듬기 품질 테스트
- [ ] 9B 대비 품질/속도 비교 → 실사용 가능 여부 판단

---

## Phase 2: 영수증 정리 완성도 (미끼 전략)

**목표**: 사무실 동료들이 "이거 좋다"고 느낄 수준으로 영수증 정리 프로그램 완성 → AI 투자(GPU 예산) 근거 확보

### 기능 개선

- [ ] OCR 엔진 교체: VARCO → Qwen3-VL 8B (Ollama)
- [ ] OCR 정확도 검증 및 프롬프트 최적화 (영수증 특화)
- [ ] 자동 분류 로직 개선 (현재 정규식 기반 → LLM 보조 검토)
- [ ] 에러 처리 강화 (서버 연결 실패, OCR 실패 시 안내)
- [ ] 사용성 개선 (로딩 상태 표시, 결과 편집 UX)

### 문서 이미지 전처리 서비스

사진 찍은 문서를 깔끔한 스캔 이미지로 자동 변환. OCR 전 단계이기도 하지만 독립 서비스로도 활용 가능.

- [ ] `/preprocess` 엔드포인트 추가 (OpenCV 기반, GPU 미사용)
  - 가장자리 자동 감지 + 트리밍 (Canny + findContours)
  - 원근 보정 (getPerspectiveTransform — 비스듬히 찍은 문서 → 정면)
  - 밝기/대비 자동 보정 (adaptiveThreshold)
- [ ] 배치 처리 지원 (여러 이미지 일괄 전처리)
- [ ] OCR 파이프라인 연동 (전처리 → OCR 자동 연결 옵션)
- [ ] 영수증 사진에도 적용 → OCR 정확도 향상 (미끼 전략 시너지)

### 배포/운영

- [ ] 내부망 사용자 가이드 (간단한 사용법 문서)
- [ ] 서버 자동 시작: systemd 서비스 등록
- [ ] 사용 통계 수집 (몇 건 처리했는지, GPU 예산 정당화 근거)

---

## Phase 3: Jingyeskan 징계 절차 관리 (장기)

**목표**: VS Server 위의 웹 앱으로, 징계 절차 전체를 단계별로 관리

### 핵심 기능 (기존 구상 기반)

- [ ] 사건 접수 및 관리 (징계 사유, 관련자, 상태 추적)
- [ ] 단계별 공고문 확인 및 보관
- [ ] 업체별 조합원 데이터 관리
- [ ] 통지서 자동 생성 (3일 전 통지 규정 준수 확인)
- [ ] 투표 관리
- [ ] 녹취록 파이프라인 통합
  - 음성 → Whisper 전사 → Clova Note 화자매칭 → LLM 다듬기 → Claude 요약
- [ ] 징계위원회 의결 기록 관리

### 기술 스택 (예정)

- 프론트엔드: HTML/JS (기존 VS 브랜딩 다크 테마)
- 백엔드: VS Server FastAPI
- 데이터: 파일 기반 또는 SQLite (규모에 따라 결정)
- 외부 API: Claude Sonnet (요약/논점 추출)

---

## 모델 현황 (2026-03-28)

### 사용 중

| 모델 | 서빙 | 용도 |
|------|------|------|
| Qwen3.5 9B | Ollama | 녹취록 다듬기 (주력). thinking OFF 필수 |
| Gemma 3 12B | Ollama | 녹취록 다듬기 (보조 옵션). OCR 불가 |
| Qwen3-VL 8B | Ollama | 이미지 OCR (인쇄물+손글씨) |
| DeepSeek OCR | Ollama | 인쇄물 전용 OCR |
| PP-OCRv5 | FastAPI | 경량 전통 OCR |
| Whisper large-v3 | FastAPI | 음성 전사 |
| Claude Sonnet | 외부 API | 논점 추출/요약 |
| Gemini Flash Lite | 외부 API | 녹취록 다듬기 (외부 대안) |

### 테스트 완료 — 제외

| 모델 | 제외 사유 |
|------|----------|
| Gemma 3 12B (OCR) | 한국어 OCR 불가. temp 1.0에서 "텍스트" 반복 hallucination, temp 0.7에서도 부분 인식 + 반복. Qwen3-VL 대비 현격히 열위 |
| PaddleOCR-VL-1.5 | P100 모든 경로 불가 |
| VARCO-VISION-2.0-1.7B-OCR | 단어 조각 이슈 |
| GLM-OCR | 한국어 부적합 |
| Nanonets-OCR2-3B | 한국어 인식 불가 |
| Ocean-OCR 3B | GGUF/Ollama 미지원 |

### GPU 업그레이드 시 기대 효과

현재 P100 16GB (CC 6.0) → 차세대 GPU 확보 시:
- bfloat16 지원 → 모델 호환성 대폭 향상
- VRAM 증가 → 더 큰 모델 또는 동시 로딩 가능
- Flash Attention 2 → 추론 속도 향상
- Jingyeskan 실시간 처리 가능

### GPU 선정 시 고려사항 (Qwen3.5 아키텍처 특성)

Qwen3.5는 Gated DeltaNet + MoE 구조로 CPU 오프로딩이 매우 효율적. VRAM보다 **메모리 대역폭**이 더 중요한 요소:

| GPU | VRAM | 대역폭 | 비고 |
|-----|------|--------|------|
| P100 (현재) | 16GB | 732 GB/s (HBM2) | 대역폭은 양호, VRAM 부족 |
| RTX 3090 (중고) | 24GB | 936 GB/s (GDDR6X) | 가성비 최고. 122B 오프로딩 실증됨 |
| RTX 4080 | 16GB | 717 GB/s | 대역폭 3090보다 낮음 |
| RTX 5060 | — | 320 GB/s (GDDR7) | 대역폭 부족, 오프로딩에 불리 |

- 3090은 384bit 버스로 전송률 우수 → CPU 오프로딩 빈번한 Qwen3.5에 최적
- VRAM 크기보다 대역폭을 우선 고려할 것

---

## 예정 작업 (우선순위 순)

1. vs_server.py v0.6 작성 (Phase 1 서버)
2. ui/ 폴더 통합 + StaticFiles 서빙 (Phase 1 클라이언트)
3. Qwen3.5 35B 테스트 검토: RAM 확인 → 업그레이드 → 녹취록 품질 비교 (Phase 1.5)
4. 영수증 정리 OCR 엔진 교체 및 테스트 (Phase 2)
5. 문서 이미지 전처리 서비스 구현 (Phase 2)
6. systemd 서비스 등록
7. 영수증 사용자 가이드 작성
8. Jingyeskan 설계 및 프로토타입 (Phase 3)
