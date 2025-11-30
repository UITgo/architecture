# ADR-001: Kiến trúc Microservices

**Status:** Accepted  
**Date:** 2025

## Context

UITGo cần một kiến trúc có thể scale theo từng phần, cho phép các team phát triển độc lập, và hỗ trợ công nghệ đa dạng cho các use case khác nhau. Hệ thống cần xử lý các workload khác nhau: read-heavy (trip queries), write-heavy (location updates), và latency-sensitive (find driver).

## Decision

Chọn kiến trúc microservices với các service độc lập:
- gateway-service: API Gateway
- auth-service: Authentication
- user-service: User profile (MongoDB)
- trip-command-service: Trip write operations (PostgreSQL)
- trip-query-service: Trip read operations (PostgreSQL + Redis cache)
- driver-stream: Driver location & status (Go + Redis Geo + Kafka)

Mỗi service có database riêng (Database-per-Service pattern).

## Consequences

### Tích cực

- **Scalability**: Mỗi service có thể scale độc lập (ví dụ: driver-stream cần scale cao hơn cho location updates)
- **Flexibility**: Có thể chọn công nghệ phù hợp cho từng service (NestJS cho business logic, Go cho driver-stream)
- **Team autonomy**: Các team có thể phát triển và deploy độc lập
- **Fault isolation**: Lỗi ở một service không ảnh hưởng toàn bộ hệ thống

### Tiêu cực

- **Complexity**: Tăng complexity (network calls, distributed transactions, service discovery)
- **Debugging**: Khó debug hơn (cần distributed tracing)
- **Deployment**: Cần orchestration tốt (Docker Compose, Kubernetes)
- **Data consistency**: Khó đảm bảo consistency giữa các services (cần eventual consistency)

### Trade-off

- **Chi phí**: Cần nhiều infrastructure hơn (multiple databases, message brokers), nhưng đổi lại có thể scale tốt hơn
- **Development**: Tăng thời gian phát triển ban đầu, nhưng đổi lại có thể phát triển nhanh hơn sau này
- **Operational**: Tăng operational complexity, nhưng đổi lại có thể maintain tốt hơn với team nhỏ

