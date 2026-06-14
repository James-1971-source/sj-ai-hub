# S&J희망나눔 AI 도구 허브 — Firebase 회원 DB 연동 가이드

이 가이드를 따라하면 약 15분 안에 **실제 회원 데이터베이스**가 작동합니다.
Firebase 무료 플랜(Spark)으로 충분하며, 신용카드 등록이 필요 없습니다.

> 무료 플랜 한도: 인증 무제한, Firestore 일일 읽기 5만/쓰기 2만 회, 저장 1GB
> — 법인 규모의 회원 수에서는 한도에 도달할 가능성이 사실상 없습니다.

---

## 1단계. Firebase 프로젝트 생성 (3분)

1. https://console.firebase.google.com 접속 → Google 계정 로그인
2. **"프로젝트 추가"** → 이름 입력 (예: `sj-ai-hub`) → 계속
3. Google 애널리틱스는 **사용 안 함** 선택 (단순화) → 프로젝트 만들기

## 2단계. 웹 앱 등록 및 설정값 복사 (2분)

1. 프로젝트 메인 화면에서 **웹 아이콘( `</>` )** 클릭
2. 앱 닉네임 입력 (예: `ai-hub-web`) → 앱 등록
   - "Firebase 호스팅 설정"은 체크하지 않아도 됩니다
3. 화면에 표시되는 `firebaseConfig` 값을 복사:

```javascript
const firebaseConfig = {
  apiKey: "AIzaSy...",
  authDomain: "sj-ai-hub.firebaseapp.com",
  projectId: "sj-ai-hub",
  storageBucket: "sj-ai-hub.firebasestorage.app",
  messagingSenderId: "1234567890",
  appId: "1:1234567890:web:abc123"
};
```

4. `index.html` 상단 `CONFIG.FIREBASE`에 **같은 항목끼리 그대로 붙여넣기**:

```javascript
const CONFIG = {
  FIREBASE: {
    apiKey:            "AIzaSy...",
    authDomain:        "sj-ai-hub.firebaseapp.com",
    projectId:         "sj-ai-hub",
    storageBucket:     "sj-ai-hub.firebasestorage.app",
    messagingSenderId: "1234567890",
    appId:             "1:1234567890:web:abc123"
  },
  ...
```

> 참고: Firebase의 apiKey는 비밀번호가 아니라 공개용 식별자이므로
> HTML에 노출되어도 안전합니다. 실제 보안은 4단계의 보안 규칙이 담당합니다.

## 3단계. 로그인 방법 활성화 (3분)

Firebase Console 왼쪽 메뉴 → **빌드 > Authentication** → "시작하기"

### 이메일/비밀번호
1. "로그인 방법" 탭 → **이메일/비밀번호** 선택 → 사용 설정 → 저장

### Google
1. 같은 탭에서 **Google** 선택 → 사용 설정
2. 프로젝트 지원 이메일 선택 → 저장
3. 끝! (별도 Google Cloud Console 작업이나 GOOGLE_CLIENT_ID가 필요 없습니다)

### 승인된 도메인 등록
1. Authentication → **설정** 탭 → "승인된 도메인"
2. 배포 도메인 추가 (예: `계정명.github.io` 또는 `ai.sj-hs.or.kr`)
   - `localhost`는 기본 등록되어 있어 로컬 테스트가 바로 가능합니다

## 4단계. Firestore 데이터베이스 생성 (3분)

1. 왼쪽 메뉴 → **빌드 > Firestore Database** → "데이터베이스 만들기"
2. 위치: `asia-northeast3 (서울)` 선택 → **프로덕션 모드**로 시작
3. 생성 후 **"규칙" 탭**에서 아래 내용으로 교체 → 게시:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // 본인의 회원 문서만 읽기/쓰기 가능
    match /users/{uid} {
      allow read, write: if request.auth != null && request.auth.uid == uid;
    }
  }
}
```

이 규칙으로 각 회원은 자신의 정보만 접근할 수 있고,
비로그인 상태에서는 어떤 데이터도 읽거나 쓸 수 없습니다.

## 5단계. 배포 및 확인 (2분)

1. 수정한 `index.html`을 호스팅에 다시 업로드 (배포가이드.md 참고)
2. 사이트 접속 → 회원가입 테스트
3. Firebase Console → Authentication → "사용자" 탭에 계정이 보이면 성공
4. Firestore Database → `users` 컬렉션에 프로필 문서 확인:

```
users/
└── {자동생성 UID}/
    ├── name: "홍길동"
    ├── email: "hong@example.com"
    ├── provider: "email"          ← email 또는 google
    └── lastLogin: 2026-06-13 ...   ← 로그인할 때마다 자동 갱신
```

---

## 연동 후 달라지는 점

| 항목 | 연동 전 (데모) | 연동 후 (Firebase) |
|---|---|---|
| 회원 정보 저장 위치 | 사용자 브라우저에만 | Firebase 클라우드 DB |
| 다른 기기에서 로그인 | ❌ 불가 | ✅ 가능 |
| 비밀번호 보안 | 평문 저장 | Firebase 암호화 처리 |
| Google 로그인 | GIS 직접 연동 필요 | ✅ 자동 전환 (설정 1분) |
| 회원 목록 관리 | 불가 | ✅ 콘솔에서 조회·삭제·비활성화 |
| 로그인 상태 유지 | 새로고침 시 유지 | ✅ 재방문 시에도 자동 복원 |

## 자주 묻는 질문

**Q. Kakao/Naver 로그인도 Firebase에 저장되나요?**
Firebase는 Kakao/Naver를 공식 지원하지 않아, 두 로그인은 기존 방식
(브라우저 세션)으로 동작합니다. Firebase에 통합하려면 별도 서버(Cloud
Functions 유료 플랜 등)가 필요하므로, 무료 운영을 위해서는 **이메일·Google
가입을 주 채널**로 안내하시는 것을 권장합니다.

**Q. `auth/unauthorized-domain` 오류가 나요.**
3단계의 "승인된 도메인"에 배포 도메인이 등록되지 않은 경우입니다.
도메인을 추가한 뒤 새로고침하세요.

**Q. 회원 탈퇴 기능은요?**
콘솔의 Authentication > 사용자 탭에서 관리자가 직접 삭제할 수 있습니다.
사이트 내 자가 탈퇴 버튼이 필요하시면 다음 버전에서 추가해 드립니다.

---
사단법인 S&J희망나눔 · 2026
