# TURN

## TURN (Traversal Using Relays around NAT)

- NAT 또는 방화벽 때문에 두 기기가 직접 연결 못하는 경우 `공인망에 있는 서버가 두 기기 사이 트래픽을 대신 중계`해주는 프로토콜
- TURN 프로토콜을 구현한 서버를 TURN 서버라 한다.
- Symmetric NAT 또는 방화벽 이슈로 STUN 통한 p2p가 어려운 경우 TURN 서버 고려
    - 어떤 이유로든 NAT 뒤에 호스트와 직접 연결이 불가할 때

```yaml
1. A -> TURN서버
- 내 중계 주소 생성(할당) 요청
- TURN 서버는 A의 전용주소 할당 (5.5.5.5:49195)

2. A -> 시그널링 서버(websocket..)
- B에게 TURN 중계 주소 전달
- STUN 통해 알아낸 내 공인ip/포트 알리는 것과 목적 및 방식은 동일

3. B -> TURN
- 데이터 전송
- B -> 5.5.5.5:49195
```

## 관련 개념

- NAT
    - 각 기기는 사설IP를 갖고, 라우터가 고유한 포트를 붙여서 라우터의 공인 IP로 변환하여 인터넷으로 트래픽을 내보내고 응답을 중계함
- STUN
    - STUN 서버로 패킷을 보내면 그 순간 NAT 테이블에는 사설ip/포트, 공인ip/포트가 생기고 STUN은 이렇게 생긴 매핑을 보고 응답으로 알려주는 기능을 제공
    - STUN 서버로 패킷을 보내면 STUN 서버는 출발지의 공인ip/포트(nat 테이블 매핑정보)를 응답
    - 응답받은 클라이언트는 상대방에게 시그널링 서버(websocket..) 통해 공인/ip포트 전달
- Symmetric NAT
    - 목적지마다 공인 포트를 새로 할당하는 NAT
    - 아웃바운드로 나갈 때 nat 테이블에 쓰여지는 공인 ip/port 매핑정보가 목적지에 따라 달라짐
    - 같은 목적지 ip/port 로 가는 요청은 같은 공인ip/port 로 매핑
    - 즉, STUN으로 알아낸 공인 포트는 STUN 서버용 매핑이라, 상대에게는 다른 포트로 보임
        - A 가 STUN 서버에 패킷을 보내고 STUN 서버로부터 받은 공인ip/포트를 B에게 시그널링 서버 통해 보내 공유해도, A가 B에게 실제 패킷을 보낼 때 나가는 것은 Symmetric NAT 에 의해 다른 포트가 매핑됨.
- ICE (Interactive Connectivity Establishment)
    - 두 기기가 연결에 쓸 수 있는 주소 후보를 모두 모은 뒤, 우선순위대로 하나씩 직접 시도해서 실제 통하는 경로를 고르는 절차
        - 서로의 네트워크 구조를 모르는 상태에서 데이터 경로를 만들 수 있음
    - STUN, TURN, 시그널링 등을 하나로 묶어 자동 처리
    - UDP 기반 통신을 위한 NAT traversal

## Reference

- https://webrtc.org/getting-started/turn-server?hl=ko
- RFC 8656 – Traversal Using Relays around NAT (TURN)
- RFC 8489 – Session Traversal Utilities for NAT (STUN)
- RFC 8445 – Interactive Connectivity Establishment (ICE)