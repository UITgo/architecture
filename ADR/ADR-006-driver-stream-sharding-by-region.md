# ADR-006: Driver Stream Sharding by Region

**Status:** Accepted  
**Date:** 2025

## Context

UITGo cần scale driver-stream để xử lý nhiều regions (HCM, HN, v.v.). Có 2 options: single instance với routing logic hoặc shard theo region. Cần chọn approach phù hợp để balance giữa scalability, performance, và complexity.

## Decision

Shard driver-stream theo **region/city** (HCM, HN), mỗi region có container riêng, Redis DB index riêng, và Kafka topic riêng.

## Consequences

### Tích cực

- **Scalability**: Mỗi region có thể scale độc lập (ví dụ: HCM có nhiều tài xế hơn, cần scale cao hơn)
- **Performance**: Giảm latency (tài xế HCM không cần query tài xế HN)
- **Isolation**: Lỗi ở một region không ảnh hưởng region khác
- **Data locality**: Data gần với users hơn (có thể deploy gần region)

### Tiêu cực

- **Complexity**: 
  - Tăng complexity (cần routing logic, maintain multiple containers)
  - Configuration: Cần maintain config cho mỗi region
- **Cost**: 
  - Cần nhiều containers hơn
  - Cần nhiều Redis DB indexes hoặc instances
  - Cần nhiều Kafka topics

### Trade-off

- **Scalability vs Complexity**: Sharding cho phép scale tốt hơn, nhưng tăng complexity
- **Performance vs Cost**: Sharding giảm latency, nhưng tăng cost (nhiều containers)
- **Isolation vs Consistency**: Sharding tăng isolation, nhưng khó đảm bảo consistency giữa regions

## Implementation

- **REGION_SHARD_CONFIG**: File config mapping cityCode → driverStreamBaseUrl
- **Docker Compose**: 2 containers (driver-stream-hcm, driver-stream-hn) với:
  - Redis DB index khác nhau (0 cho HCM, 1 cho HN)
  - Kafka topics khác nhau (`driver.location.hcm`, `driver.location.hn`)
  - Ports khác nhau (8081 cho HCM, 8082 cho HN)
- **Routing**: trip-command-service sử dụng `getDriverStreamUrl(cityCode)` để route request đến shard tương ứng
- **Trip.cityCode**: Lưu cityCode trong Trip để routing

