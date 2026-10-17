# 클라우드 저장

구글 로그인으로 진행 상황을 계정별로 저장합니다. `gridtd.unbrik.ro`에서만 켜지고(`index.html`의 `CLOUD.host`), claude.ai 아티팩트에서는 보이지 않습니다.

## Firebase

- 프로젝트: `nuker-2f286`
- Authentication: Google 로그인 사용, 승인된 도메인에 `gridtd.unbrik.ro`
- Firestore: `(default)` 데이터베이스, 위치 `asia-northeast3`
- 로그인은 구글 로그인 버튼(Google Identity Services, `accounts.google.com/gsi/client`)으로 받은 ID 토큰을 Firebase에 넘깁니다(`signInWithCredential`). 그래서 로그인 창에 firebaseapp.com이 아니라 `gridtd.unbrik.ro`가 보입니다.
  - 웹 클라이언트 ID: `1036223456136-2cdcjsnu2gmmo7ig66o4no33d3d4bcfo.apps.googleusercontent.com` (`index.html`의 `CLOUD.clientId`)
  - Google Cloud 콘솔 → API 및 서비스 → 사용자 인증 정보 → Web client (auto created by Google Service)의 **승인된 자바스크립트 원본**에 `https://gridtd.unbrik.ro`
  - 버튼 스크립트를 못 불러오면 예전 Firebase 팝업 로그인 버튼으로 돌아갑니다.
- 게임은 Firebase Auth SDK(`www.gstatic.com/firebasejs/10.12.2/`)만 불러오고, Firestore는 REST API에 ID 토큰을 붙여 씁니다.
- `firebaseConfig`의 apiKey는 공개 식별자입니다. 접근은 아래 보안 규칙과 승인된 도메인이 막습니다.

## 저장 구조

| 경로 | 내용 |
|---|---|
| `saves/{uid}` | `data`: 로컬 저장(`grid-defense-v1`)과 같은 JSON 문자열 · `updatedAt`: 그 저장의 시각 · `version`: 1 |
| `saves/{uid}/backups/{0–6}` | 클라우드 저장을 덮어쓰기 직전 상태를 하루에 한 번 보관(요일 칸 7개를 돌려 씀) · `day`: 로컬 기준 날짜 번호 |

이 기기 쪽 백업은 `localStorage`의 `grid-defense-v1-prev`에 하나 남습니다(클라우드 받기·저장 불러오기·백업 복구 직전 상태). 설정의 **백업 복구**에서 두 백업을 모두 고를 수 있습니다.

## 동기화

- 저장 시각은 내용이 바뀔 때만 움직입니다.
- 바뀐 뒤 15초 안에 올리고, 앱을 나갈 때도 올립니다. 돌아오면 다른 기기의 저장을 확인합니다.
- 이 기기만 바뀌었으면 올리고, 클라우드만 바뀌었으면 받습니다(전투 중에는 받지 않음).
- 둘 다 바뀌었으면 플레이어가 고릅니다. 진행이 없는 새 기기는 클라우드를 그대로 받습니다.

## 보안 규칙

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /saves/{uid} {
      allow read: if request.auth != null && request.auth.uid == uid;
      allow write: if request.auth != null && request.auth.uid == uid
        && request.resource.data.keys().hasOnly(['data', 'updatedAt', 'version'])
        && request.resource.data.data is string
        && request.resource.data.data.size() < 1000000
        && request.resource.data.updatedAt is int;
      match /backups/{slot} {
        allow read: if request.auth != null && request.auth.uid == uid;
        allow write: if request.auth != null && request.auth.uid == uid
          && slot.matches('[0-6]')
          && request.resource.data.keys().hasOnly(['data', 'updatedAt', 'version', 'day'])
          && request.resource.data.data is string
          && request.resource.data.data.size() < 1000000;
      }
    }
  }
}
```

저장과 백업은 게임에서 지울 수 없습니다. 지워야 하면 Firebase 콘솔에서 직접 지웁니다.
