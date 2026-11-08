# DPoP Proof JWT

## DPoP Proof JWT (Demonstrating Proof of Possession, 소유 증명)

- RFC 9449 (OAuth 2.0 Demonstrating Proof of Possession)
- 클라이언트가 특정 공개키에 대응하는 개인키를 보유하고 있음을 매 http 요청마다 증명하는 용도로 사용
- 일반적인 Bearer 토큰은 토큰을 가진 누구나 사용 가능
    - 서버는 Bearer 토큰 이 유효한지만 확인하고 누가 보냈는지 출처에 대한 확인은 없음
    - 때문에 토큰이 탈취 당했을 때 공격자는 그대로 사용 가능
- DPoP Proof JWT 은 accessToken 을 발행받은 클라이언트가 아닌 다른 클라이언트가 accessToken 을 탈취하여 사용하는 것을 방지하기 위함

## 브라우저 -> api 서버 bearer jwt 인증 flow

1. 브라우저 -> api 서버 로그인 시 jwt 발급
2. 이후 api 요청시 발급받은 jwt 를 포함
3. 서버는 요청의 jwt 의 유효성 검증

```
서버는 jwt 의 유효성만을 검증
jwt 가 탈취되어 다른 어떤 주체가 사용해도 알 수 없음
```

## 브라우저 -> api 서버 DPoP jwt 인증 flow

### 클라이언트 측에서 개인키/공개키 쌍 생성

- 키쌍은 재사용

### 로그인 api 호출

- 개인키/공개키 쌍으로 DPoP Proof(JWT) 생성
- 생성한 DPoP Proof 는 `DPoP` 헤더에 포함

#### DPoP Proof JWT 예시

- 페이로드 영역은 매 요청마다 바뀜
- jti (JwtId): 중복되지 않도록 클라이언트에서 책임
- htm/htu (http method/http uri): 대상 api 에 대한 http 요청 정보
- iat (Issued At): Proof 생성시간
- ath (Access Token Hash): proof 가 어떤 AT와 함께 쓰이는지 묶어주는 클레임
    - ath = base64url(sha-256(AT))

```
{
  "typ": "dpop+jwt",
  "alg": "ES256",
  "jwk": {
    "kty": "EC",
    "crv": "P-256",
    "x": "gE_Wj5aOKoNwQ07V_731_528M2wtt4nqBrdrrCDP82Q",
    "y": "ikZJHUqwuHfeXaRQpgsnc0QDRnoLMNOqjXGmEV76NVU"
  }
}

{
  "jti": "ceae5b71-a448-4144-963b-4bb365767c1a",
  "htm": "GET",
  "htu": "https://api.example.com/orders",
  "iat": 1762650030,
  "ath": "hHkaQeixNWiybC6ICtf7mSydDKYMg4m3eOaOdwswrc4"
}
```

### 서버측 로그인 처리 (AT 발급)

- dpop proof 검증
    - proof 헤더의 공개키로 서명 검증
    - 필수 클레임 (jti, htm/htu, iat) 존재 여부 검증
    - jti 중복여부 검증 (iat 기반 특정 기간 동안 중복 있는지 검증)
    - 호출된 api가 htm/htu 와 일치하는지 검증
- dpop proof 검증 통과시 username/password 검증 등 로그인 검증 진행
- AT 발급
- AT 의 `cnf.jkt` 클레임에는 proof의 공개키의 지문(해시) 가 포함됨

```
// AccessToken 예시

{ "alg": "ES256", "typ": "at+jwt", "kid": "as-key-1" }

{
  "iss": "https://auth.example.com",
  "sub": "user123",
  "aud": "https://api.example.com",
  "iat": 1762650000,
  "exp": 1762650600,
  "jti": "f92a8fcf-47f4-4b82-a3c5-33e2b50292dc",
  "cnf": {
    "jkt": "zT4XWt7p4uocBGre0_yG4UFk2UcLlw-_pEGSU9tAUgw"
  }
}
```

### 인증 필요 API 호출

- 클라이언트 측에서 dpop proof 새로 생성
    - dpop proof 는 매 번 새로 생성되어야 함
- Authorization 헤더에는 AT를 DPOP 헤더에는 proof 를 포함하여 요청

```
GET /orders HTTP/1.1
Host: api.example.com
Authorization: DPoP eyJhbGciOiJFUzI1NiIsInR5cCI6ImF0K2p3dCIsImtpZCI6ImFzLWtleS0xIn0.eyJpc3MiOiJodHRwczovL2F1dGguZXhhbXBsZS5jb20iLCJzdWIiOiJ1c2VyMTIzIiwiYXVkIjoiaHR0cHM6Ly9hcGkuZXhhbXBsZS5jb20iLCJpYXQiOjE3NjI2NTAwMDAsImV4cCI6MTc2MjY1MDYwMCwianRpIjoiZjkyYThmY2YtNDdmNC00YjgyLWEzYzUtMzNlMmI1MDI5MmRjIiwiY25mIjp7ImprdCI6InpUNFhXdDdwNHVvY0JHcmUwX3lHNFVGazJVY0xsdy1fcEVHU1U5dEFVZ3cifX0.PS-vQJcPOwn74CBRCykV7MVw0y4-D2jVlA0m4zfWUoL4R5yVE5hzFAnZBQ_0-RBcXsJrD2JUmD1ElfZk6xHM5Q
DPoP: eyJ0eXAiOiJkcG9wK2p3dCIsImFsZyI6IkVTMjU2IiwiandrIjp7Imt0eSI6IkVDIiwiY3J2IjoiUC0yNTYiLCJ4IjoiZ0VfV2o1YU9Lb053UTA3Vl83MzFfNTI4TTJ3dHQ0bnFCcmRyckNEUDgyUSIsInkiOiJpa1pKSFVxd3VIZmVYYVJRcGdzbmMwUURSbm9MTU5PcWpYR21FVjc2TlZVIn19.eyJqdGkiOiJjZWFlNWI3MS1hNDQ4LTQxNDQtOTYzYi00YmIzNjU3NjdjMWEiLCJodG0iOiJHRVQiLCJodHUiOiJodHRwczovL2FwaS5leGFtcGxlLmNvbS9vcmRlcnMiLCJpYXQiOjE3NjI2NTAwMzAsImF0aCI6ImhIa2FRZWl4TldpeWJDNklDdGY3bVN5ZERLWU1nNG0zZU9hT2R3c3dyYzQifQ.zpvHC0tJV0YrHqvC3kQbUWIBzYLUyczQ8vRs55GC8-GsV2C0uAxJEdscFO3Q43hZVYRAI8elWaEJ3gXPuEXGYA
```

### 인증 필요 api 서버측 검증

- Authorization 헤더로부터 AT 파싱
- proof 검증
    - htm/htu 기반 검증
    - 서명 검증 (proof 파싱 결과 jwk 로 proof 서명 검증)
- jti (iat) 검증
    - iat 로부터 특정 시간동안 동일 jti 가 요청된 경우 해당 요청은 탈취된 proof 로 취급
    - jti 는 클라이언트가 중복되지 않도록 보장해야하는 값이고 이를 서버측에서 보장하려면 별도 nonce 발급 필요
- AT 검증
    - exp, iss, aud 등 기본적인 검증
- ath 검증
    - ath(AccessTokenHash), proof 검증
    - 이 proof 가 현재 요청에 포함된 AT 를 위해 생성된 것이 맞는지 검증
    - proof 의 ath 클레임과 Authorization 헤더에 전달된 DPoP JWT 비교
        - DPoP 뒤 부분 base64url(sha-256(AT)) 해서 ath 와 비교
- jkt 검증
    - 이 proof를 서명한 키가 AT의 주인의 키인가 맞는지 검증
    - proof 의 공개키 지문과 AT 의 cnf.jkt 비교
    - AT를 탈취한 공격자가 자신의 키로 proof 를 만드는 것 방지

```
DPoP 방식 적용시 AccessToken 탈취된 경우 클라이언트의 개인키만 안전하다면 공격자가 유효한 DPoP Proof 를 만들어 낼 수 없으므로 
탈취된 액세스 토큰 재사용 불가

단, AccessToken 을 만들어내는 서버측 서명키(jwt secret)이 털린 경우 공격자는 accessToken, DPoP Proof 모두 맘대로 만들어낼 수 있음
즉, DPoP Proof JWT 적용시에도 jwt 서명키 회전 또는 서명키 자체를 안전하게 관리하는(HSM, KMS...) 것 병행 필요
```

## Claim

- htm, htu (http method/http uri)
    - proof로 보낼 목적지의 url 및 http 메서드 명시
    - htm, htu 가 없을 때, proof 로 확인 가능한 것은 이 proof 의 개인키를 가진 클라이언트가 서명했다는 것은 확신 가능하지만 클라이언트가 목적으로 한 api 호출인지는 확신 불가
    - proof 구성시 클라이언트는 목적 api 를 htm, htu 에 명시
    - 서버는 클아이언트가 목적지로 삼은 htm, htu 와 현재 호출된 api 가 동일한지 검증
    - 즉, 클라이언트가 proof 구성시 목적지로 지정한 api가 호출되었는지 서버에서 한 번 더 검증 가능
    - `유출된 proof의 다른 API 재사용 차단 목적`
- ath (AccessTokenHash)
    - proof 가 어떤 AT와 함께 쓰이는지 묶어주는 클레임
    - AT를 발행받은 클라이언트가 proof 생성시 proof 의 ath 클레임에 AT 를 포함시켜서 proof 와 AT를 묶는 용도

## Reference

- https://www.ibm.com/docs/ko/security-verify?topic=connect-demonstrating-proof-possession
- https://www.rfc-editor.org/rfc/rfc9449.html#section-4.2