# RabbitMQ QoS

## RabbitMQ QoS?
- rabbitmq 에서 QoS 는 한 번에 메세지를 몇 개까지 보낼지 정하는 설정을 의미
  - mqtt 에서 메세지 전달 보장 수준을 의미하는 QoS와 다름
- rabbitmq가 컨슈머로 메세지를 보낸 뒤 컨슈머로부터 처리 완료됨을 의미하는 응답을 받아야 메세지를 큐에서 제거함. 이 응답을 ack 라 하고, ack 에는 두 가지 방식이 있음
  - 자동 ack: 보내는 순간 처리된 것으로 간주
  - 수동 ack: 컨슈머가 `basic.ack` 를 보내야 처리된 것으로 간주
- 컨슈머로 메세지를 보냈지만 ack 를 받지 못한 메세지를 unacked 메세지라 부르고 QoS는 unacked 개수를 다루는 설정
- rabbitmq 는 기본적으로 컨슈머에게 메세지를 push 하는 방식
  - 기본적으로 보낼 수 있는 만큼 계속 보냄
  - 컨슈머는 큐 구독 전에 rabbitmq 로 qos 설정을 보냄 (`basic.qos`)
  - `channel.basicQos(10);`
  - ack를 보내지 않은 메세지가 10개가 되면 나(컨슈머) 에게 더이상 메세지를 보내지 말라는 뜻
  - basic.qos 설정은 prefetch_count 를 설정하는 것과 동일
  - rabbitmq 는 각 컨슈머 별로 unacked 메세지 개수를 관리하며, 최대 `prefetch_count(basic.qos)-unacked 메세지 수` 만큼 한 번에 컨슈머로 메세지를 push 한다. 
- 자동 ack 는 unacked 메세지가 생기지 않으므로 QoS 동작하지 않는다.

## 용어
- 자동 ack: rabbitmq 는 컨슈머로 메세지 push 한 순간 ack 된 것으로 처리(클라이언트 측에서 정상적으로 받아 처리했는지 여부와 관계없음)
- 수동 ack: rabbitmq 는 컨슈머로 메세지 push 이후 컨슈머가 처리 후 `basic.ack` 를 받은 뒤 ack 처리 (메세지 큐로부터 제거)
- prefetch_count: ack 하지 않은 채 동시에 들고 있을 수 있는 최대 메세지 개수
  - 수동 ack 인 경우 최대 prefetch_count-unacked수 만큼 메세지 push됨
  - 자동 ack 에서는 prefetch_count 동작하지 않는다

## 참고
- https://www.rabbitmq.com/docs/consumer-prefetch
- https://www.rabbitmq.com/docs/confirms
- https://www.rabbitmq.com/docs/consumers