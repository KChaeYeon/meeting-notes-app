# 회의지옥 🔥

암호화된 연구 회의록(요약/스크립트)을 열람하는 정적 PWA. 무빌드, 바닐라 JS + Web Crypto.

## 설치 (홈 화면에 추가)

배포된 URL로 접속 후 암호(초기값 `0122`)를 입력하면 목록이 열립니다.
브라우저 메뉴 → "홈 화면에 추가" / "앱 설치"로 오프라인에서도 실행 가능한 PWA로 설치할 수 있습니다.

## 암호 변경법

1. `~/.local/share/meeting-pipeline/secrets.toml`의 `app_passcode` 값을 새 암호로 변경.
2. `~/.local/share/meeting-pipeline/.venv/bin/python generate_app_data.py` 재실행 → `data/` 전체가 새 암호로 재암호화됨(salt는 기존 `data/crypto.json`이 있으면 유지).
3. 이 앱 repo를 커밋·push(또는 로컬 정적 서빙 갱신).
4. 기존 기기의 "암호 기억하기" 저장값은 다음 자동 잠금해제 시도에서 실패 → 자동으로 게이트로 돌아가며, 새 암호를 다시 입력해야 합니다.

## 구조

```
index.html        # 전체 UI (게이트/목록/상세), 인라인 CSS+JS
crypto.js         # Web Crypto: deriveKey / decryptBlob (PBKDF2-SHA256 + AES-256-GCM)
manifest.json     # PWA manifest
sw.js             # 서비스워커 (network-first, 앱 셸 캐싱)
vendor/marked.min.js  # 로컬 번들 마크다운 렌더러
fonts/Gaegu-Regular.ttf  # 로컬 번들 손글씨 폰트 (OFL)
icons/icon-192.png, icon-512.png
data/
  crypto.json     # {v, kdf, iter, salt} — salt는 전역 공용
  index.enc       # 암호화된 목록 [{id,date,title,keywords}]
  <id>/summary.enc  # 암호화된 요약 (markdown)
  <id>/script.enc   # 암호화된 스크립트 전문 (plain text)
```

## 암호화 스킴

- KDF: PBKDF2-HMAC-SHA256, iterations=200000, salt 16B (전역, `data/crypto.json`)
- 암호화: AES-256-GCM, IV 12B (파일별 랜덤)
- 블롭 포맷: `{"iv": base64, "ct": base64}`
- 파이썬 측 구현: `~/.local/share/meeting-pipeline/app_crypto.py`, `generate_app_data.py`
