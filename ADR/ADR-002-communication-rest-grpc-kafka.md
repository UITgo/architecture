# ADR-002: Communication Patterns - REST + gRPC + Kafka

**Status:** Accepted  
**Date:** 2025

## Context

UITGo cần giao tiếp giữa client và services, giữa các services với nhau, và xử lý events real-time. Cần chọn protocol phù hợp cho từng use case: external API (client → services), internal communication (service → service), và event streaming (async events).

## Decision

Sử dụng 3 protocols khác nhau:
- **REST (HTTP)**: Cho external API (client → gateway → services)
- **gRPC**: Cho internal communication (service → service, ví dụ: trip-command-service → user-service GetProfile)
- **Kafka**: Cho event streaming (driver location updates, trip events)

## Consequences

### Tích cực

- **REST**: Standard, dễ debug, phù hợp cho external API, hỗ trợ tốt cho web/mobile clients
- **gRPC**: Binary protocol, hiệu năng cao, type-safe với proto files, phù hợp cho internal communication
- **Kafka**: Decoupling, hỗ trợ multiple consumers, durable storage, phù hợp cho event streaming

### Tiêu cực

- **Complexity**: Cần maintain nhiều protocols (REST, gRPC, Kafka)
- **gRPC**: Cần maintain proto files, generate code, handle versioning
- **Kafka**: Cần infrastructure (Zookeeper, brokers), configuration phức tạp

### Trade-off

- **Performance**: gRPC nhanh hơn REST cho internal communication (binary protocol), nhưng REST dễ debug hơn
- **Flexibility**: Kafka cho phép decoupling và multiple consumers, nhưng tăng complexity
- **Cost**: Cần infrastructure cho Kafka, nhưng đổi lại có thể scale tốt hơn

## Implementation

- **REST**: Gateway sử dụng HTTP client để gọi các services
- **gRPC**: trip-command-service sử dụng gRPC client để gọi user-service GetProfile
- **Kafka**: driver-stream publish events vào Kafka, các service khác có thể subscribe

