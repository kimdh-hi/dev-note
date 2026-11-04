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

1. 브라우저가 키 쌍(개인키/공개키) 생성
2. 브라우저 -> api 서버 로그인 시 DPoP Proof(JWT, 정확히는 JWS) 전송
- DPoP Proof(JWS) 파싱시 클라이언트의 공개키를 알 수 있음
- 공개키로 DPoP Proof 서명 검증
- 검증 성공시, 서버는 jwt 발급시 이 공개키의 주인만 사용 가능 하다는 것을 기록 (`cnf.jkt` 클레임 추가)
- `cnf.jkt` 에 공개키의 해시값 (지문) 을 포함

서버가 응답하는 jwt 예시

```
{
  "access_token": "eyJhbGciOiJFUzI1NiIsImtpZCI6IkJlQUxrYiJ9.eyJzdWIiOiJzb21lb25lQGV4YW1wbGUuY29tIiwiaXNzIjoi...",
  "token_type": "DPoP",
  "expires_in": 3600
}
```

access token 파싱시 예시

```
{
  "sub": "someone@example.com",
  "iss": "<https://server.example.com>",
  "exp": 1562266216,
  "cnf": {
    "jkt": "0ZcOCORZNYy-DWpqq30jZyJGHTN0d2HglBV3uiguA4I"
  }
}
```

1. 이후 api 요청시 jwt와 개인키로 새롭게 만든 DPoP Proof(JWS)를 요청에 포함
- 매 요청마다 클라이언트는 본인의 개인키로 DPoP Proof 를 새롭게 만들어서 요청에 포함
- 클라이언트는 매 번 새롭게 생성. 만약 재사용한다면 bearer 와 다를게 없음
- DPoP Proof 클레임 중 `jti` 가 nonce와 유사한 역할을 해서 서버는 `jti` 를 기반으로 재전송이 있는지 검증 필요

```
// 요청 http header 예시
Authorization: DPoP <액세스 토큰>
DPoP: <새 Proof>
```

1. 서버측 검증
- 기본적인 jwt 유효성 검증
- 이 `DPoP: <새 Proof>` 가 공개키의 주인이 만든 것이 맞는지 검증
    - `verify(jwk, "Proof의 헤더.페이로드", Proof의 서명)`
- 공개키가 로그인 때 등록된 공개키와 같은지 검증
    - accessToken 의 `cnf.jkt` 와 새 Proof 의 공개키 지문 동일 여부 확인
    - Proof 의 공개키가 엑세스 토큰 발급시 사용된 공개키라는 것을 확인

```
DPoP 방식 적용시 AccessToken 탈취된 경우 클라이언트의 개인키만 안전하다면 공격자가 유효한 DPoP Proof 를 만들어 낼 수 없으므로 
탈취된 액세스 토큰 재사용 불가

단, AccessToken 을 만들어내는 서버측 서명키(jwt secret)이 털린 경우 공격자는 accessToken, DPoP Proof 모두 맘대로 만들어낼 수 있음
즉, DPoP Proof JWT 적용시에도 jwt 서명키 회전 또는 서명키 자체를 안전하게 관리하는(HSM, KMS...) 것 병행 필요
```

## Reference

- https://www.ibm.com/docs/ko/security-verify?topic=connect-demonstrating-proof-possession
- https://www.rfc-editor.org/rfc/rfc9449.html#section-4.2