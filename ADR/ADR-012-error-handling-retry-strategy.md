# ADR-012: Error Handling and Retry Strategy

**Status:** Accepted  
**Date:** 2024

## Context

UITGo có nhiều network calls giữa services (gateway → services, trip-command → user-service gRPC, trip-command → driver-stream HTTP). Network calls có thể fail do timeout, network issues, service unavailable. Cần strategy để handle errors và retry.

## Decision

Sử dụng **exponential backoff retry** cho network calls, **circuit breaker pattern** cho critical paths, và **graceful degradation** cho non-critical operations.

## Consequences

### Tích cực

- **Exponential backoff**: Retry với delay tăng dần (1s, 2s, 4s), tránh thundering herd
- **Circuit breaker**: Ngừng retry nếu service fail liên tục, tránh waste resources
- **Graceful degradation**: Nếu service không available, return partial data hoặc default values
- **Error logging**: Log errors để debug và monitor

### Tiêu cực

- **Complexity**: Cần implement retry logic, circuit breaker logic
- **Latency**: Retry tăng latency nếu service fail
- **Consistency**: Graceful degradation có thể return stale data

### Trade-off

- **Reliability vs Latency**: Retry tăng reliability, nhưng tăng latency
- **Availability vs Consistency**: Graceful degradation tăng availability, nhưng có thể giảm consistency
- **Complexity vs Resilience**: Retry và circuit breaker tăng complexity, nhưng đổi lại tăng resilience

## Implementation

- **HTTP client**: Gateway sử dụng Axios với timeout 5s, có thể thêm retry interceptor
- **gRPC client**: trip-command-service có thể implement retry với exponential backoff
- **Error handling**: Các service catch errors, log, và return appropriate HTTP status codes
- **Fallback**: Nếu driver-stream không available, trip-command-service log error và continue (trip vẫn được tạo, nhưng không tìm được tài xế)
- **Future**: Có thể implement circuit breaker với libraries như `opossum` (Node.js) hoặc `gobreaker` (Go)

