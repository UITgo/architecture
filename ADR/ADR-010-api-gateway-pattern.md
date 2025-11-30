# ADR-010: API Gateway Pattern

**Status:** Accepted  
**Date:** 2025

## Context

UITGo có nhiều microservices (auth, user, trip-command, trip-query, driver-stream). Client cần gọi nhiều services khác nhau. Cần chọn approach: direct client-to-service hoặc API Gateway.

## Decision

Sử dụng **API Gateway Pattern** với gateway-service làm single entry point.

## Consequences

### Tích cực

- **Single entry point**: Client chỉ cần biết 1 endpoint (gateway), không cần biết internal services
- **Centralized authentication**: Gateway xử lý JWT verification một lần, các service không cần verify lại
- **Request routing**: Gateway route requests đến đúng service dựa trên path
- **Cross-cutting concerns**: Gateway có thể xử lý logging, rate limiting, CORS, error handling tập trung
- **Service abstraction**: Có thể thay đổi internal services mà không ảnh hưởng client

### Tiêu cực

- **Single point of failure**: Gateway trở thành single point of failure (cần high availability)
- **Latency**: Thêm 1 hop (client → gateway → service), tăng latency
- **Complexity**: Gateway cần maintain routing logic, có thể trở thành bottleneck

### Trade-off

- **Simplicity vs Performance**: Gateway đơn giản hóa client, nhưng tăng latency
- **Centralization vs Distribution**: Gateway tập trung logic, nhưng tăng dependency
- **Cost**: Cần đảm bảo gateway có high availability, nhưng đổi lại giảm complexity ở client

## Implementation

- **Gateway**: gateway-service (NestJS) trên port 3004
- **Routing**: Route `/api/v1/trips` POST/PUT/DELETE → trip-command-service, GET → trip-query-service
- **Authentication**: JWT verification ở gateway, gắn `X-User-Id`, `X-User-Role` headers
- **Logging**: Gateway log tất cả requests với correlation ID (`X-Request-Id`)

