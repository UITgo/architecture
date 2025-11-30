# ADR-003: Driver Location Storage - Redis Geo vs DynamoDB

**Status:** Accepted  
**Date:** 2025

## Context

UITGo cần lưu trữ và query driver location real-time để tìm tài xế gần nhất. Yêu cầu: hiệu năng cao (latency <50ms), throughput cao (hàng nghìn updates mỗi giây), và spatial queries (tìm tài xế trong bán kính).

## Decision

Sử dụng **Redis Geo** để lưu driver location real-time, publish events qua **Kafka**, và để **DynamoDB** cho tương lai (location history).

## Consequences

### Tích cực

- **Redis Geo**: 
  - Hiệu năng cao: Spatial queries (GEORADIUS) <10ms
  - Real-time: Location được update liên tục, không cần query database
  - In-memory: Nhanh hơn database quan hệ
- **Kafka**: 
  - Decoupling: Các service khác có thể subscribe để xử lý events
  - Scalability: Hỗ trợ multiple consumers
  - Durability: Events được lưu trữ, có thể replay
- **DynamoDB (tương lai)**: 
  - Time-series data: Phù hợp cho location history
  - Auto-scaling: Tự động scale theo traffic

### Tiêu cực

- **Redis**: 
  - Không persistent: Cần backup (AOF/RDB)
  - Memory limit: Cần đủ RAM
  - Single point of failure: Cần high availability
- **Kafka**: 
  - Infrastructure: Cần Zookeeper, brokers
  - Configuration: Phức tạp
- **DynamoDB**: 
  - Cost: Đắt hơn Redis
  - Latency: Chậm hơn Redis (network call)

### Trade-off

- **Performance vs Cost**: Redis nhanh hơn DynamoDB, nhưng cần infrastructure riêng
- **Real-time vs History**: Redis cho real-time queries, DynamoDB cho location history
- **Complexity**: Cần maintain Redis và Kafka, nhưng đổi lại có thể scale tốt hơn

## Implementation

- **Redis Geo**: driver-stream sử dụng `GEOADD` để lưu location, `GEORADIUS` để tìm tài xế gần nhất
- **Kafka**: driver-stream publish events vào topic `driver.location.{region}`
- **DynamoDB**: Có thể sử dụng trong tương lai để lưu location history với TTL

