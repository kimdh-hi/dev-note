# AWS VPC

## VPC (Virtual Private Cloud)
- aws 의 같은 믈리 인프라를 여러 고객이 함께 쓰지만, vpc 라는 논리적인 분리를 통해 각 vpc는 중복된 사설 IP를 사용 가능
  - A VPC 10.0.0.5 와 B VPC 의 10.0.0.5 는 논리적으로 분리됨
- aws 는 패킷을 목적지 ip 만으로 구분하지 않고 어느 vpc 의 ip 인지를 보고 전달
- 즉, 서로 다른 vpc 에서 동일한 사설 ip를 사용해도 충돌 발생하지 않음
- 기본적으로 같은 vpc 내에서만 사설 ip로 통신이 가능
  - 다른 vpc 간 사설 ip 통신 필요하다면 vpc 피어링 등 다른 vpc 와 연결 작업 추가로 필요
- vpc 는 계정 + 리전 단위로 생성
- vpc 생성 후 vpc 안에 aws 리소스를 만드려면 서브넷 지정 필요
  - vpc 자체는 서브넷 없이 생성 가능 단, 서브넷 없이 vpc 내 리소스 생성 불가
  - vpc 내 리소스는 반드시 어떤 서브넷에 속해야 함.
  - 서브넷에는 AZ(가용영역)가 지정되므로 결과적으로 리소스를 어떤 AZ에 위치시킬지를 지정하는 것.
  - 참고, aws 는 리소스를 여러 AZ에 분산해서 배치하는 것을 권장

## VPC 구조

```
VPC (리전)
 ├─ 라우팅 테이블      => 서브넷에 연결
 ├─ 인터넷 게이트웨이   => VPC에 연결(attach), 라우팅 테이블 경로로 사용
 ├─ VPC 엔드포인트     => 게이트웨이형은 라우팅 테이블에 경로로 추가
 ├─ 네트워크 ACL       => 서브넷에 연결
 ├─ 보안 그룹          => 리소스(EC2 등)에 연결
 └─ 서브넷 (AZ)
      ├─ NAT 게이트웨이 => 특정 서브넷(AZ) 안에 생성
      └─ EC2, RDS 등 리소스
```

## Reference
- https://docs.aws.amazon.com/vpc/latest/userguide/how-it-works.html
- https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html
- https://docs.aws.amazon.com/vpc/latest/userguide/create-vpc.html