# 서버 간(Server-to-Server) 인증 방식 정리

## 방식

### API Key

- 구현상 가장 단순
- 같은 비밀값이 매 요청에 포함되므로 로그, 프록시, APM 등 노출될 지점이 많아 보안상 취약
- 노출되면 만료도 범위 제한도 없이 재사용 가능

### HMAC 서명

- secret은 보내지 않고 서명 전송
- 서명 대상

| 항목 | 예시 |
| --- | --- |
| HTTP 메서드 | `POST` |
| 경로 + 쿼리 (쿼리는 키 이름순으로 정렬) | `/api/conferences` |
| 요청 시각 (Unix 초) | `1763452800` |
| nonce (요청마다 새로 만든 UUID) | `550e8400-e29b-41d4-a716-446655440000` |
| body의 SHA-256 해시 | `e22b792a...74ef33f9` |

```
POST /api/conferences
X-Timestamp: 1763452800
X-Nonce: 550e8400-e29b-41d4-a716-446655440000
X-Signature: NfiAzUzkkLSr1truBqpeA2aTxQbaLAGpJxQX2C/4zcY=

{"key1":10,"data1":2}
```

- 요청 변조를 감지 및 타임스탬프 + nonce로 재전송 공격 방지
- 서버는 secret과 요청정보, timestamp, nonce 등을 통해 동일한 HMAC 을 생성해서 검증
    - timestamp 로부터 특정 시간동안 동일 nonce 에 대한 예외처리(재전송 방지 처리 필요)

### OAuth Client Credentials (client_secret)

- 인가 서버에서 client_id/secret으로 access token을 발급받고, API를 `Authorization: Bearer`로 호출
    - 인가 서버가 별도로 없는 경우 인가서버/리소스서버 역할을 한 개 서버에서 수행
- 클라이언트는 client_id/client_secret 등록 후 인가서버로 client_id/client_secret 포함 요청하여 AT 획득
- 만료, scope 기반 인가, 클라이언트 중앙 관리
- refresh token은 발급하지 않는 것이 권장 (RFC 6749)
    - client 는 이미 AT 발급 가능한 client_id/client_secret 을 알고 있으므로 refresh 대신 재인증을 통해 AT 획득
- 토큰은 Bearer라서 유출되면 만료 전까지 사용 가능하고, 요청 단위의 무결성은 보장 불가
    - 단, access token 만료시간은 짧게 가져간다.
- 각 클라이언트에 대해 client_id, client_secret 을 등록하는 백오피스 ui 필요

### OAuth Client Credentials (private_key_jwt)

- client_secret(대칭키)를 발급하지 않고 클라이언트 측에서 공개키/개인키를 생성 및 관리하며 공개키를 인가 서버에 등록하여, 인가서버로부터 신원을 증명
    - 이 방식에 client_secret(대칭키) 는 사용되지 않음
- 위 방식에서 client_secret 대신, 개인키로 서명한 JWT(client assertion)로 클라이언트를 인증
- 인가 서버에는 공개키만 등록하므로, 인증 서버가 유출돼도 클라이언트를 사칭 불가
- 보호 범위는 토큰 발급 단계뿐이고, 발급된 AT는 공개

```
1. 클라이언트 측에서 키 쌍 생성
2. 개인키는 클라이언트 측에서 안전하게 보관 (KMS...)
3. 공개키를 인가 서버에 등록
- jwks_uri: 클라이언트가 공개키 JWK Set 을 URL로 공개하고, 클라이언트 등록시 jwks_uri 를 등록
  인가서버는 jwtks_uri 통해 클라이언트 공개키 획득
- jwks: 공개키 JWT Set을 클라이언트 등록 정보에 포함
4. 인가서버로 AT 요청 (POST /oauth2/token)
- client_id 포함 (client_secret X)
- client_assertion 포함 => 개인키로 서명한 JWT(JWS) 생성하여 
- client_assertion 은 매 요청마다 새롭게 생성하며, kid, aud, jti, exp 필수 클레임을 포함
5. 서버는 client_id 로 해당 클라이언트의 공개키를 찾아 client_assertion 서명 검증 후 AT 발행 

```

```
# jwks_uri 등록시 예시
{
  "client_name": "clientA",
  "grant_types": ["client_credentials"],
  "token_endpoint_auth_method": "private_key_jwt",
  "token_endpoint_auth_signing_alg": "RS256",
  "scope": "clientA.write",
  "jwks_uri": "https://client-a.example.com/.well-known/jwks.json"
}

# jwks 등록 예시
{
  "client_name": "clientA",
  "grant_types": ["client_credentials"],
  "token_endpoint_auth_method": "private_key_jwt",
  "token_endpoint_auth_signing_alg": "RS256",
  "scope": "clientA.write",
  "jwks": {
    "keys": [
      {
        "kty": "RSA",
        "kid": "client-a-2026-11",
        "use": "sig",
        "alg": "RS256",
        "e": "AQAB",
        "n": "nj3YJwsLUFl9BmpAbkOswCNVx17Eh9wMO-_AReZwBqfa..."
      }
    ]
  }
}
```

## 참고 문서

- RFC 2104 – HMAC
- RFC 9421 – HTTP Message Signatures
- RFC 6749 §4.4 – Client Credentials Grant
- RFC 6750 §1.2 – Bearer Token
- RFC 7523 §2.2 – JWT를 이용한 클라이언트 인증
- RFC 9700 §2.5 – OAuth 2.0 Security BCP: Client Authentication
- OpenID Connect Core §9 – Client Authentication
- RFC 7517 §4.5 – JWK `kid`
- Spring Security – OAuth 2.0 Resource Server JWT
- PHP – hash_hmac
- OWASP – Secrets Management Cheat Sheet