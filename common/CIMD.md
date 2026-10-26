# CIMD

## CIMD (Client Id Metadata Document)
- OAuth 클라이언트를 식별하고 등록하는 방식
- 최근 MCP 인가 스펙에서 권장 클라이언트 등록 방식
- 서버에 JSON 포맷의 CIMD 문서를 올려두면 인가서버는 해당 문서를 https get 통해 받아가서 redirect_uri(콜백주소)를 식별한다.
- OAuth 에서 결국 공격자가 노리는 것은 authorization code 인데 이 코드가 CIMD 문서상 정해진 목록으로만 가도록하는 것이 핵심
- oauth client 측은 CIMD json 문서를 획득 가능한 url(api or 파일주소) 을 client_id 로 사용한다.
  - client_id 를 클라이언트 측에서 정하는 것이 기존 OAuth 의 client_id 와 다름
  - 기존 OAuth 의 경우 인가 서버에서 client_id 를 불투명 값으로 생성
  - OAuth 에서 client_id 는 이 인가 요청을 보낸 앱이 어떤 앱인지를 인가 서버에 알리는 식별자
- 일반적으로 gcp console 에 앱을 등록할 때 redirect url 등을 설정(설정 결과 client_id 발급 받음)하게 되는데 이런 사전 등록을 사용할 수 없는 경우 CIMD 방식 채택 고려

## CIMD 스펙
- client_id URL 규칙
  - https 필수
  - https://example.com/oauth/mcp-client.json 과 같은 구조 권장
  - https://example.com/oauth/mcp-client.json 과 https://example.com:443/oauth/mcp-client.json 은 다른 클라언트로 취급됨
    - 두 url 같은 주소를 가리킴
    - CIMD 스펙은 client_id 를 글자 그대로 비교
- CIMD 문서 규칙
  - json 포맷
  - client_id 는 해당 문서이 주소와 일치해야 함
  - client_secret 등 민감정보 작성 금지
- loopback redirect 는 127.0.0.1 권장
  - localhost 는 비권장
  - 127.0.0.1 은 IP 주소 자체이므로 항상 loopback 인터페이스를 가리킴
  - localhost 는 hosts 파일, DNS 설정의 영향으로 바뀔 수 있음
- CIMD 문서 제공 규칙
  - 200 OK 응답 (그 외 상태 코드는 모두 오류로 처리됨)
  - 리다이렉트 금지 (인가 서버가 redirect_url 따라가지 않음)
  - 무인증 (외부 인가 서버가 서버 간 요청으로 가져감)
  - 도메인이 공인 IP 로 해석되어야 함

## reference
- https://www.ietf.org/archive/id/draft-ietf-oauth-client-id-metadata-document-02.html
- https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/client-registration
- https://datatracker.ietf.org/doc/html/rfc6749#section-2.2