# ADR-008: Message Queue - Kafka vs RabbitMQ

**Status:** Accepted  
**Date:** 2025

## Context

UITGo cần message broker để xử lý event streaming (driver location updates, trip events). Cần chọn message broker phù hợp cho high-throughput, multiple consumers, và durability.

## Decision

Chọn **Apache Kafka** thay vì RabbitMQ.

## Consequences

### Tích cực

- **High throughput**: Kafka được thiết kế cho high-throughput (hàng triệu messages/second)
- **Multiple consumers**: Hỗ trợ multiple consumer groups, mỗi group có thể consume độc lập
- **Durability**: Messages được lưu trữ trên disk, có thể replay
- **Partitioning**: Có thể partition theo key (driverId) để đảm bảo ordering
- **Scalability**: Có thể scale bằng cách thêm partitions và brokers

### Tiêu cực

- **Complexity**: Cần Zookeeper (hoặc KRaft mode), configuration phức tạp hơn RabbitMQ
- **Latency**: Có thể có latency cao hơn RabbitMQ cho low-volume scenarios
- **Learning curve**: Khó học hơn RabbitMQ

### Trade-off

- **Throughput vs Complexity**: Kafka tốt hơn cho high-throughput, nhưng phức tạp hơn
- **Durability vs Latency**: Kafka đảm bảo durability tốt hơn, nhưng có thể có latency cao hơn
- **Cost**: Kafka cần nhiều resources hơn (disk, memory), nhưng đổi lại có thể scale tốt hơn

## Implementation

- **Kafka**: driver-stream publish events vào topics `driver.location.hcm`, `driver.location.hn`
- **Producer**: Sử dụng Sarama (Go Kafka client) với sync producer
- **Topics**: Mỗi region có topic riêng để partition theo địa lý
- **Future**: Có thể thêm consumers để xử lý analytics, notifications

