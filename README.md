# VS Server (Visual-Sonic)

P100 16GB GPU 서버 기반의 OCR + 음성 전사 통합 서비스.

서버: sussertod | Ubuntu 24.04 | Tailscale: 100.114.30.82

---

## 주요 기능

- **녹취록 정리**: 음성 전사 (Whisper) + Clova Note 화자분리 매칭 + Qwen3.5 문체 변환
- **문서 OCR**: PDF → 텍스트 (Qwen3-VL, DeepSeek OCR, PP-OCRv5 선택)
- **음성 전사**: Whisper large-v3, word-level 타임스탬프

---

## 빠른 시작

**서버 실행**
```bash
cd ~/vs-server
./start.sh
```

**클라이언트**
- 녹취록 정리: `vs-receipt.html` (file://로 직접 열기)
- 문서 OCR: `start-vs-document.bat` 실행 후 `http://localhost:8090/vs-document.html`

**API**
- FastAPI: `http://100.114.30.82:8000`
- Ollama: `http://100.114.30.82:11434`
- 헬스체크: `GET http://100.114.30.82:8000/health`

---

## 문서

- **[CLAUDE.md](./CLAUDE.md)** — Claude Code 작업용 현재 상태 + TODO
- **[docs/architecture.md](./docs/architecture.md)** — 하드웨어, 서빙 구조, 엔드포인트 상세
- **[docs/models.md](./docs/models.md)** — OCR 모델 테스트 현황 + 신규 모델 테스트 가이드
- **[docs/pipeline.md](./docs/pipeline.md)** — 화자분리 파이프라인, 클라이언트 UI
- **[docs/history.md](./docs/history.md)** — 프로젝트 대화 히스토리
