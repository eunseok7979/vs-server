# VS Server — 프로젝트 대화 히스토리

시간순 정렬. 각 대화의 주요 결정/발견사항 기록.

---

| 날짜/순서 | 링크 | 주요 내용 |
|-----------|------|----------|
| 1 | [중고 GPU 채굴 손상 여부 소프트웨어 검사](https://claude.ai/chat/ea3d43ad-2f5d-4119-8899-0adb11c19e46) | P100 구매 전 진단 가이드, A320 보드 + iGPU 구성 계획 |
| 2 | [파일 인코딩 변환 오류 해결](https://claude.ai/chat/cd7108cc-84c8-40da-bd3e-0b3dbbb2546c) | PDF-to-EPUB OCR 파이프라인 (PaddleX 레이아웃 + Tesseract/EasyOCR), PaddlePaddle 3.x 호환성 이슈 |
| 3 | [PaddleOCR-VL을 Transformers + PyTorch로 실행하기](https://claude.ai/chat/41d61b8d-5443-46a4-b787-959f68091e1f) | PaddleOCR-VL-1.5 품질 확인 (캘리번과 마녀), Transformers 경로 검증 |
| 4 | [P100 16GB OCR 서버 구축 사전 검토](https://claude.ai/chat/1798bffc-eeaa-4351-bc0b-68118dabd2df) | 모델 비교 문서, 소프트웨어 스택 설계, PaddleOCR-VL 1순위 확정 |
| 5 | [드라이버 설치 시작](https://claude.ai/chat/c1acf12e-f3fa-4d6a-aed2-89715d0d2ba1) | Ubuntu 24.04 셋업, conda ocr 환경, PyTorch cu121, PaddleOCR-VL-1.5 첫 GPU 테스트, Qwen2.5-VL-3B 테스트, FastAPI 통합 서버 구성 |
| 6 | [우분투 네트워크 설정 및 SSH 연결](https://claude.ai/chat/b7507a0c-13e5-4c65-9f02-5b3cbd81cd3a) | faster-whisper 계획, Clova Note 형식 확인, 화자매칭 설계 |
| 7 | [로컬 OCR 모델 파라미터 개선](https://claude.ai/chat/871fc737-1629-47d8-9009-e06ab7d9757b) | VARCO-VISION 테스트 (CPU→GPU, 단어 조각 이슈로 부적합 판정) |
| 8 | [음성전사 결과 대기 중](https://claude.ai/chat/96a7f6b7-41a1-42a3-a3f0-b14bba0a2b5f) | Whisper 서버 구축, vs-receipt.html 녹취록 정리 UI, 화자분리 매칭 클라이언트 |
| 9 | [Partial PDF text summarization](https://claude.ai/chat/dd01b602-6a6d-4110-bb29-15acb2eb0b6a) | vs-document.html (PDF OCR 클라이언트), 프롬프트 프리셋, pdf.js 썸네일 이슈, OCR 모델 추가 후보 조사 |
| 10 | Ollama/llama.cpp 경로 확립 및 모델 비교 테스트 | Ollama 서빙 발견, DeepSeek OCR 인쇄물 완벽, PaddleOCR-VL-1.5 모든 경로 불가 확인, Qwen3-VL 8B 주력 확정 |
| 11 | Qwen3.5 벤치마크 및 모델 역할 확정 | Qwen3.5 9B P100 동작 확인, thinking on/off 비교, 역할 분담 확정 (녹취록=Qwen3.5 OFF, 논점=Claude Sonnet, 이미지OCR=Qwen3-VL) |
