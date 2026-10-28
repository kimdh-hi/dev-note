# Grafana Drilldown

## Grafana Drilldown?
- 쿼리 작성없이 직관적으로 텔레메트리 데이터를 탐색하는 도구
  - 이전에는 Explore 앱으로 불렸음
- Drilldown 종류
  - Metrics Drilldown: prometheus 호환 데이터소스
  - Logs Drilldown: Loki
  - Traces Drilldown: Tempo
  - Profiles Drilldown: Pyroscope

## useful

### Traces Drilldown
- Drilldown > Traces > duration > Slow traces
- pinpoint 와 유사하게 call tree 확인 가능
- 단, springboot/jpa 의 경우 datasource-micrometer, ObservationFilter 등 커스텀하여 sql 단위 span 및 호출 메서드 파악 가능
