# 회의지옥 🔥

암호화된 연구 회의록(요약/스크립트)을 열람하는 정적 PWA. 무빌드, 바닐라 JS + Web Crypto.

**배포 URL**: https://kchaeyeon.github.io/meeting-notes-app/

## 사용법

1. 위 URL로 접속 → 암호(초기값 `0122`) 입력 → 목록 화면이 열립니다.
2. **"이 기기에서 암호 기억하기"**(기본 체크)를 켜두면 다음 방문부터 암호 재입력 없이 자동으로 열립니다. 헤더의 **🔒 잠금** 버튼으로 언제든 기억된 암호를 지우고 게이트로 돌아갈 수 있습니다.
3. **홈 화면에 추가**: 브라우저 메뉴 → "홈 화면에 추가"/"앱 설치"로 오프라인에서도 실행되는 PWA로 설치할 수 있습니다.

### 메인 화면 — 두 가지 보기 모드

헤더 아래 알약형 토글로 전환하며, 선택은 이 기기에 기억됩니다.

- **📚 주제별** (기본값): 회의를 주제(topic)별로 묶은 카드로 보여줍니다(예: "전기장 시뮬레이션 4건"). 카드를 클릭하면 그 주제의 회의만 최신순 표로 필터링됩니다.
- **📅 표 형식**: 전체 회의를 최신순 표로 나열합니다. 날짜/시간은 두 줄로 나눠 컬럼을 좁히고, 제목 컬럼을 넓게, 키워드는 항목별로 줄바꿈해 가독성을 높였습니다.

주제 색상은 주제 이름을 해시해 파스텔 팔레트에서 자동 배정됩니다 — 새 주제가 생겨도 코드 수정 없이 계속 동작합니다.

### 상세 화면

회의를 클릭하면 **요약본**(마크다운 렌더링) / **스크립트 전문**(타임스탬프 포함 원문) 두 탭으로 봅니다.

## 데이터가 만들어지는 곳

이 repo의 `data/`는 사람이 직접 편집하지 않습니다. 회의 녹음 파이프라인(`~/.local/share/meeting-pipeline`, 로컬 전용)이 매일 05:00 새 회의를 처리한 뒤 `generate_app_data.py`로 암호화해 이 repo에 커밋·push합니다. 원본 스크립트·요약본(평문)은 이 repo가 아니라 별도 **private** 저장소에 보관되며, 여기엔 **암호화된 사본만** 올라옵니다.

## 암호 변경법

1. `~/.local/share/meeting-pipeline/secrets.toml`의 `app_passcode` 값을 새 암호로 변경.
2. `~/.local/share/meeting-pipeline/.venv/bin/python generate_app_data.py` 재실행 → `data/` 전체가 새 암호로 재암호화됨(salt는 기존 `data/crypto.json`이 있으면 유지).
3. 이 앱 repo를 커밋·push.
4. 기존 기기의 "암호 기억하기" 저장값은 다음 자동 잠금해제 시도에서 실패 → 자동으로 게이트로 돌아가며, 새 암호를 다시 입력해야 합니다.

## 회의 주제 재분류(백엔드에서)

특정 회의의 주제를 다른 이름으로 합치거나 바꾸려면:
1. `~/.local/share/meeting-pipeline/logs/meta_cache.json`에서 해당 회의(stem id)의 `topic` 값을 원하는 문자열로 수정.
2. `generate_app_data.py` 재실행(캐시에 keywords/summary/topic이 모두 있으면 `claude` 재호출 없이 즉시 재생성됨) → 커밋·push.
3. 주제는 데이터에서 동적으로 도출되므로, 어떤 회의도 쓰지 않는 주제는 목록에서 자동으로 사라집니다(별도 삭제 절차 불필요).

## 구조

```
index.html        # 전체 UI (게이트/목록 토글/주제 카드/표/상세), 인라인 CSS+JS
crypto.js         # Web Crypto: deriveKey / decryptBlob (PBKDF2-SHA256 + AES-256-GCM)
manifest.json     # PWA manifest
sw.js             # 서비스워커 (network-first, 앱 셸 캐싱)
vendor/marked.min.js       # 로컬 번들 마크다운 렌더러
fonts/NotoSansKR-Regular.woff2, NotoSansKR-Bold.woff2  # 로컬 번들 본문 폰트(가독성)
fonts/Gaegu-Regular.ttf    # 로컬 번들 손글씨 폰트(장식용, 현재 본문에는 미사용)
icons/icon-192.png, icon-512.png
data/
  crypto.json     # {v, kdf, iter, salt} — salt는 전역 공용
  index.enc       # 암호화된 목록 [{id,date,title,keywords,topic}]
  <id>/summary.enc  # 암호화된 요약 (markdown)
  <id>/script.enc   # 암호화된 스크립트 전문 (plain text)
```

## 암호화 스킴

- KDF: PBKDF2-HMAC-SHA256, iterations=200000, salt 16B (전역, `data/crypto.json`)
- 암호화: AES-256-GCM, IV 12B (파일별 랜덤)
- 블롭 포맷: `{"iv": base64, "ct": base64}`
- 파이썬 측 구현: `~/.local/share/meeting-pipeline/app_crypto.py`, `generate_app_data.py`
- 이 repo는 **public**이지만 `data/` 내용은 암호 없이는 읽을 수 없습니다. 유일한 방어선은 앱 암호이므로, 4자리 숫자보다 긴 암호를 권장합니다.
