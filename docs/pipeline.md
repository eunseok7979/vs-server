# VS Server — 파이프라인 & 클라이언트 UI

최종 업데이트: 2026-03-27

---

## 화자분리 매칭 파이프라인

완성도 가장 높음. 현재 잘 동작 중.

### 흐름

```
음성 파일
    │
    ├── VS Server /transcribe → Whisper large-v3
    │       └── words: [{start, end, word}, ...]
    │
    └── Clova Note → 화자별 타임스탬프 텍스트
            └── [{speaker, start_sec, text}, ...]

매칭 로직:
- Clova Note 화자 전환 시점 기준으로 각 Whisper word에 화자 태깅
- 종료 시간 = 다음 화자의 시작 시간
- 연속 화자 단어 묶기 → 타임스탬프 제거 → 최종 텍스트
```

### Clova Note 출력 형식

```
화자이름 MM:SS
발화 텍스트 (여러 줄 가능)

화자이름 MM:SS
다음 발화...
```

### 녹취록 정리 후처리 (Qwen3.5 9B)

Whisper + Clova Note 매칭 완료 후, Qwen3.5 9B (thinking OFF)로 구어체 → 문어체 변환.  
논점 추출/요약은 Claude Sonnet 외부 API 사용.

---

## 클라이언트 UI

현재 Windows 로컬에 HTML 파일 산재 → `~/vs-server/ui/` 통합 예정.

### vs-receipt.html (녹취록 정리)

- 음성 파일 업로드 → VS서버 `/transcribe` 전사
- Clova Note 텍스트 붙여넣기 → 화자분리 매칭
- 매칭 결과 표시 + 텍스트/JSON 다운로드 + 복사
- 디자인: 다크 테마 (VS 브랜딩, Noto Sans KR + JetBrains Mono)
- **실행**: `file://`로 직접 열기 가능 (pdf.js 미사용)

### vs-document.html (문서 OCR)

- PDF 업로드 → pdf.js 썸네일 → 페이지 선택(체크박스 + 범위입력) → OCR → .txt 내보내기
- 렌더링 배율 선택 (1.5x~3.0x)
- 문서 유형 + 프롬프트 프리셋 선택
- 원본 이미지 ↔ OCR 텍스트 나란히 비교, 텍스트 편집 가능
- **실행**: `file://` 불가 (pdf.js 보안 제한)
  - `start-vs-document.bat` 더블클릭 → `python -m http.server 8090`
  - `http://localhost:8090/vs-document.html` 접근

### 예정: 클라이언트 통합

```
~/vs-server/ui/
├── vs-receipt.html
├── vs-document.html
└── (공통 에셋)
```

FastAPI `StaticFiles`로 서빙 → `http://100.114.30.82:8000/ui/` 단일 접근점.  
bat 파일 불필요, pdf.js 문제 해소, 외부에서도 Tailscale로 접근 가능.

---

## 파생 프로젝트

### PDF-to-EPUB

- GitHub: [eunseok7979/PDF-to-Epub](https://github.com/eunseok7979/PDF-to-Epub)
- conda 환경: `pdfepub`
- 현재: PaddleX PP-DocLayout-M 레이아웃 분석 + 3-way OCR (PyMuPDF/Tesseract/EasyOCR)
- 예정: Ollama (Qwen3-VL / DeepSeek OCR) 엔진으로 교체

### 영수증 정리 프로그램

- 기반: vs-receipt.html
- 현재 동작하지만 정교화 필요
