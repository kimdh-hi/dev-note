# Passkeys

## passkey?

### FIDO/FIDO2

- 비밀번호 없는 인증/2단계 인증 등 표준을 만드는 업계 단체
- FIDO2 는 FIDO Alliance가 만든 개방형 인증 표준

### WebAuthn (Web Authentication)

- FIDO2를 구성하는 표준 중 하나
- 표준
    - RP 서버로부터 받은 challenge, rpid 등을 FE(RP 웹페에지) 에서 브라우저로 키쌍을 등록하는 api 표준
        - `create()`: 등록 - 인증기에 키 쌍을 만들고, 공개키를 응답 받음
    - 브라우저 → 인증기로 사용자 인증 통해 인증 완료 후 서명 결과 응답 표준
        - `get()`: 인증 - 기존 키로 challenge에 서명하고, 서명을 응답 받음
    - RP서버에서 공개키로 서명을 검증하는 검증 절차에 대한 표준
- RP 가 공개키 자격증명을 등록하고 검증하는 방식을 정함

### CTAP (Client to Authenticator Protocols)

- FIDO2를 구성하는 표준 중 하나 (클라이언트 ↔ 인증기)
    - 외부 장치 인증기(USB, NFC, BLE 등) 과 데이터를 주고받는 방법 표준
    - 기기 내장 인증기인 경우 CTAP 아닐 수 있음.
- 브라우저/OS 와 인증기(휴대폰/보안키) 사이 통신 규약

### Passkey

- 자격증명
- 위 표준으로 만들어지는 공개키/개인키 쌍
- 단, 위 표준으로 만들지는 공개키/개인키 쌍 모두를 passkey 라고 지칭하진 않음.
    - discoverable 자격증명
    - 인증기에 개인키와 함께 계정 정보(user handle) 도 저장되어야 함.
    - 인증기가 RP 서버의 도움(credential ID) 없이 스스로 PR ID 기반으로 자격 증명을 찾아낼 수 있는지
    - 일반적으로 위 여부와 관계없이 passkey 라고 부르기도 함.


## passkey 등록/인증 Flow

- flow
    
    ```mermaid
    sequenceDiagram
        autonumber
        actor U as 사용자
        participant FE as 클라이언트<br/>(FE + 브라우저)
        participant A as 인증기<br/>(iCloud 키체인 등)
        participant BE as RP 서버(BE)
        participant R as Redis
        participant DB as DB
    
        rect rgb(235, 245, 255)
        Note over U,DB: [등록] 로그인 상태에서 패스키 2차 인증 등록
    
        FE->>BE: 1. POST /webauthn/register/options (AT)
        BE->>DB: user handle 조회 (없으면 랜덤 생성 후 저장)
        BE->>DB: 기존 credentialId 목록 조회 (excludeCredentials)
        BE->>BE: challenge 생성
        BE->>R: 옵션 저장 (key: webauthn:reg:{user_id}, TTL 5분)
        BE-->>FE: 옵션 응답 (rp, user, challenge, excludeCredentials ...)
    
        FE->>A: 2. navigator.credentials.create()
        Note over FE: 브라우저: clientDataJSON 생성<br/>{type: webauthn.create, challenge, origin}
        A->>U: 사용자 확인 (Touch ID 등)
        U-->>A: 확인
        A->>A: 키쌍 생성 + 저장<br/>(개인키, credentialId, RP ID, user handle)
        A-->>FE: 공개키, credentialId, authData, transports
    
        FE->>BE: 3. POST /webauthn/register
        BE->>R: 4. 옵션 조회 + 삭제 (GETDEL)
        BE->>BE: 검증: type, challenge, origin,<br/>rpIdHash, UP, 알고리즘, credentialId 중복
        BE->>DB: credential 저장 (credentialId, 공개키,<br/>signCount, BE/BS, transports ...)
        BE-->>FE: 등록 완료
        end
    
        rect rgb(240, 255, 240)
        Note over U,DB: [2차 인증] 로그인 시
    
        FE->>BE: 0. 비밀번호 로그인
        BE-->>FE: 2차 인증 대기 임시 토큰
    
        FE->>BE: 5. POST /webauthn/authenticate/options (임시 토큰)
        BE->>DB: 사용자의 credentialId + transports 조회
        BE->>BE: challenge 생성
        BE->>R: 옵션 저장 (TTL 5분)
        BE-->>FE: 옵션 응답 (challenge, rpId,<br/>allowCredentials, userVerification: preferred)
    
        FE->>A: 6. navigator.credentials.get()
        Note over FE: 브라우저: clientDataJSON 생성<br/>{type: webauthn.get, challenge, origin}
        A->>U: 사용자 확인 (터치, 필요 시 Touch ID)
        U-->>A: 확인
        A->>A: authenticatorData 생성<br/>개인키로 서명 (authData + SHA-256(clientDataJSON))
        A-->>FE: signature, authenticatorData, userHandle
    
        FE->>BE: POST /login/webauthn (id, clientDataJSON,<br/>authenticatorData, signature, userHandle)
        BE->>R: 7. 옵션 조회 + 삭제 (GETDEL)
        BE->>DB: credentialId로 공개키 조회
        BE->>BE: 검증: challenge, credentialId ∈ allowCredentials,<br/>type, origin, rpIdHash, UP
        BE->>BE: 서명 검증: verify(공개키, authData + 해시, signature)
        BE->>DB: signCount, BS, last_used_at 갱신
        BE-->>FE: AT/RT 발급
        end
    
    ```
    

**1. 클라이언트(FE) → RP 서버(BE): 등록 옵션 요청 (등록 준비)**

- 현재 사용자 계정 대상 passkey 를 만들 수 있도록, 인증기에 넘길 데이터들을 서버가 만들어주는 과정
- `POST /webauthn/register/options`
- 기존 계정에 2차인증 추가인 경우 로그인 상태를 검증
- challenge 생성
    - 서버가 매번 새로 만드는 1회용 난수 (16바이트 이상)
    - 재전송 공격 방지: 이번 요청에 대한 응답인지 확인하는 용도
    - 등록(1단계), 인증(5단계)에서 각각 새로 생성
    - 브라우저가 clientDataJSON에 넣어 전달 → BE는 저장값과 비교
- user handle(`user.id`) 조회 또는 발급
    - user handle 가 이미 DB에 있는 경우 재사용
    - user handle(user.id) 는 난수값(랜덤바이트)으로 설정
    - user handle 새로 발급인 경우 db 에 저장 필요 (user 테이블에 칼럼으로 추가)
- 기존 credentialId 목록 조회 (`excludeCredentials`용)
    - 같은 인증기에 중복으로 passkey 등록되지 않도록 하기 위함
    - 중복 확인을 위한 조회만 하고 credentialId 저장은 4단계에서 진행

```json
//응답예시
{
  "rp": { "name": "SimpleWebAuthn Example", "id": "example.localhost" },
  "user": {
    "id": "oWJtkJ6vJ_m5b84LB4_K7QKTCTEwLIjCh4tFMCGHO4w",  
    "name": "alice@example.com",
    "displayName": "alice@example.com"
  },
  "challenge": "q7lCdd3SVQxdC-v8pnRAGEn1B2M-t7ZECWPwCAmhWvc",
  "pubKeyCredParams": [
    { "type": "public-key", "alg": -8 },
    { "type": "public-key", "alg": -7 },
    { "type": "public-key", "alg": -257 }
  ],
  "timeout": 300000,
  "excludeCredentials": [],
  "authenticatorSelection": {
    "residentKey": "required",
    "userVerification": "preferred"
  },
  "attestation": "none",
  "extensions": { "credProps": true }
}
```

**2. 클라이언트 → 인증기: 키쌍 및 자격증명 생성 (passkey 생성)**

- `navigator.credentials.create()`
- 브라우저가 매체 선택 창을 띄우고, 인증기가 사용자를 확인한 뒤 키쌍 생성 (개인키는 인증기 밖으로 나가지 않음)

```jsx
// 요청 예시 (값은 전부 1단계 응답에서 받음)
const credential = await navigator.credentials.create({
  publicKey: {
    challenge,            // 필수
    rp,                   // 필수
    user,                 // 필수
    pubKeyCredParams,     // 필수 (허용할 서명 알고리즘 목록, 예: ES256 -7, RS256 -257)
    // 이하 선택
    timeout, excludeCredentials, authenticatorSelection,
    hints, attestation, attestationFormats, extensions
  }
});
// authenticatorSelection 권장: residentKey "required", userVerification "preferred"(2차 인증 용도), authenticatorAttachment 생략
```

**3. 클라이언트 → RP 서버: 등록 결과 전달**

- `POST /webauthn/register`
- JSON 구성
    - `id`: credential ID
    - `clientDataJSON`: type, challenge, origin
    - `attestationObject`: authData(rpIdHash, 플래그 UP/UV/BE/BS, signCount, 공개키)
    - `transports` : 브라우저가 패스키의 인증기에 어떤 방식으로 연결할 수 있는지
        - internal: 클라이언트 기기에 묶여 있어 분리할 수 없는 인증기 (iCloud키체인..)
        - hybrid: 근접 확인 수단 함께 사용 (모바일, 데스크탑에서 사용) → 휴대폰 QR..
- 어떤 인증기를 사용했는지(platform, cross-platform) → 이 값(`authenticatorAttachment`)은 브라우저가 채우지 않을 수 있어 검증에 사용하지 않음

**4. RP 서버: 검증 후 저장**

- 검증 항목
    - type이 webauthn.create인지
    - challenge가 저장된 값과 일치하는지
    - origin이 허용된 값인지
    - rpIdHash가 RP ID와 일치하는지
    - UP 플래그 (UV는 preferred이므로 필수 아님)
    - 알고리즘이 허용 목록에 있는지
    - credential ID가 이미 등록되어 있지 않은지
- 검증 후 challenge 즉시 삭제 (1회용)
- DB 스키마 예시

```json
CREATE TABLE webauthn_user (
  user_id      
  user_handle  
  created_at
);

CREATE TABLE webauthn_credential (
  credential_id
  user_handle  
  public_key   
  sign_count   
  uv_initialized 
  backup_eligible
  backup_state   
  transports     
  label          
  aaguid         
  created_at     
  last_used_at 
);
```

**5. 클라이언트 → RP 서버: 인증 옵션 요청**

- `POST /webauthn/authenticate/options`
- BE는 새 challenge를 만들어 저장하고 응답 (rpId, userVerification, allowCredentials 포함)
- allowCredentials
    - 이번 인증에 사용 가능한 패스키 목록 (사전에 username 등 사용자 식별 정보를 서버가 알아야 함)
    - 빈 배열 응답시 RP ID 기준 기기에 저장된 모든 패스키 목록중 사용자가 선택
- userVerification
    - 인증기가 생체/PIN 통해 본인 확인을 해야하는지 여부
    - 해당 기기를 가지고 있다 정도만 인증해도 되는지 여부에 따라 결정
    - required, preferred, discouraged

**6. 클라이언트 → 인증기: 서명**

- FE: navigator.credentials.get() 호출 (5단계 옵션 전달)
- 브라우저: rpId 확인 → clientDataJSON 생성 {type, challenge, origin} → allowCredentials의 패스키를 가진 인증기 찾기
- 인증기: 사용자 확인(터치, 필요 시 생체·PIN) → authenticatorData 생성 {rpIdHash, UP/UV, signCount} → 개인키로 서명 (authenticatorData + clientDataJSON 해시)
- FE: POST /login/webauthn
    - id: 사용한 패스키
    - clientDataJSON: 브라우저가 만든 값 (challenge 포함)
    - authenticatorData: 인증기가 만든 값
    - signature: 위 두 개에 대한 서명
    - userHandle: 계정 식별값

**7. RP 서버: 검증 후 인증 완료**

- 저장소에서 옵션 조회 후 즉시 삭제 (GETDEL) → clientDataJSON의 challenge와 비교
- credential ID가 allowCredentials 목록에 있는지 = 1차 인증한 사용자의 패스키인지 확인
- type webauthn.get, origin, rpIdHash, UP 검증
- 공개키로 서명 검증: verify(공개키, authenticatorData + SHA-256(clientDataJSON), 서명)
    - challenge를 바꾸면 해시가 달라져 실패, 재사용하면 저장값이 없어 실패
- signCount 확인 (동기화 패스키는 0으로 오므로 0은 허용)
- 갱신: sign_count, backup_state, uv_initialized(false였으면), last_used_at
- AT/RT 발급

```json
# 서명 검증
verify(공개키, authenticatorData + clientDataJSON 해시, 서명)
```

## Reference
- https://www.passkeycentral.org/ko/introduction-to-passkeys/how-passkeys-work
- https://docs.spring.io/spring-security/reference/servlet/authentication/passkeys.html
- https://www.w3.org/TR/webauthn-3/
- https://auth-wiki.logto.io/ko/webauthn
- claude