# AWS WAF

## aws waf

- WAF (Web Application Firewall)
- IP, 웹 요청(URL, 헤더, 본문) 등을 통해 HTTP 트래픽 검사
- web ACL(웹 접근 제어 목록)을 정의하고, 리소스에 web ACL 을 선택해서 연결 (리소스에 WAF 연결)
    - ALB, CloudFront, API Gateway 등의 매니지드 서비스에 연결
    - 해당 서비스로 요청이 도착하는 경우 해당 리소스가 요청을 WAF 로 전달
    - WAF 는 web ACL 규칙을 우선순위 대로 평가하여 결과에 따라 허용/차단/카운트/CAPTCHA 등을 처리

## web ACL

- 어떤 요청을 허용/차단할지를 정해둔 규칙 목록
- 구성 요소: 규칙(Rule), 규칙 그룹(Rule group), 우선순위(Priority), 기본 동작(Default action), 기타 설정
- 요청 판단시 사용 가능한 기준
    - 요청자 IP
    - 요청자 국가
    - 요청의 특정 부분(url 경로, http 메서드, 쿼리, 헤더, 쿠키, 본문 등..)에 대한 문자열, 정규식 판단
    - 요청의 특정 부분의 크기
    - 악성 sql 코드, 스크립트 탐지
- 규칙은 우선순위 숫자가 작은 것부터 차례대로 검사
    - 우선순위에 의해 결과가 바뀔 수 있음
    - 우선순위 0에 사내IP 인 경우 ALLOW 규칙 설정
    - 우선순위 1에 login 100회 초과 IP BLOCK 규칙 설정
    - login 100회 초과되어도 사내 IP인 경우 ALLOW 로 인해 다음 우선순위 규칙 검사하지 않아서 BLOCK 안됨
    - 우선순위가 바뀌는 경우 BLOCK
- 동작: ALLOW, BLOCK
    - COUNT, CAPCHA 와 동일하게 일반 규칙에도 ALLOW, BLOCK 동작 지정 가능 (단, 기본동작에는 ALLOW, BLOCK만 지정 가능)
    - ALLOW, BLOCK 모두 조건이 일치하면 다음 조건을 평가하지 않음.
    - 단, ALLOW 조건에 일치되면 요청을 통과시키고, BLOCK 조건에 일치되면 요청을 차단
- 동작: COUNT, CAPCHA
    - COUNT: 해당 규칙에 일치하는 경우 count 증가 (허용/차단을 평가하지 않고 다음 규칙을 계속 평가, cloudwatch, waf 로그 등에서 확인 가능)
    - CAPCHA: 조건에 일치하는 경우 CAPCHA 퍼즐 유도 (최근에 퍼즐을 풀어서 받은 토큰이 있는 경우 통과)
- rate-based
    - "IP별 N분간 /login POST 호출 100 건 초과" 등과 같은 조건 설정 가능
    - IP sets(IP 모음)을 미리 만들어두고(IP sets -> Create IP Set), 해당 IP sets 대상 특정 조건에서 차단/허용 여부를 지정
        - IP sets 은 일반 규칙에서도 사용 가능.
    - 이후 IP sets 에 추가/수정이 발생되면 web ACL, 규칙 등 수정할 필요없이 해당 IP sets 을 사용하는 모든 규칙에 적용

## 비용

- 연결한 리소스(ALB, CloudFront, API Gateway) 요금과 별도로 부과
- web ACL 수, web ACL 당 규칙 수, 수신 요청 수 기준 과금
- 기본 단가
    - web ACL: 월 $5
    - 규칙: 월 $1 (규칙 개수당 $1, 일반/rate-based 규칙 동일)
    - 요청 100만 건당 $0.60
    - web ACL 1개 + 규칙 20개 + 월 1,000만 요청: 월 $30
- CAPTCHA, Challenge 동작은 걸별 과금
- WCU (Web ACL Capacity Unit) 규칙을 실행하는데 소비되는 처리량을 수치로 나타낸 단위
    - 단순한 규칙은 적은 WCU 를, 복잡한 규칙은 많은 WCU 가 발생
    - 1,500 WCU 까지 기본 요금에 포함
    - WCU 1,500 초과시 500WCU 마다 100만 건당 $0.20 추가 과금
    - 크기 제한: 1WCU, 정규식: 2WCU, XSS: 40WCU 이런식으로 WCU 계산됨
    - https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statements-match.html
    - AWS 관리형 규칙 그룹을 사용하는 경우 WCU 금방 초과될 수 있음
    - https://docs.aws.amazon.com/waf/latest/developerguide/aws-managed-rule-groups-list.html
- 즉, 1,500 WCU 가 초과되는 web ACL 사용하지 않는 경우 기본 단가 내에서 비용 과금

## reference

- https://docs.aws.amazon.com/waf/latest/developerguide/how-aws-waf-works.html
- https://docs.aws.amazon.com/waf/latest/developerguide/web-acl.html
- https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-type-rate-based.html
- https://aws.amazon.com/ko/waf/pricing/