# ADR-005: Trip Read Path - Redis Cache + CQRS

**Status:** Accepted  
**Date:** 2025

## Context

UITGo có read-heavy traffic cho trip queries (GET /trips/:id). Cần tối ưu read path để giảm latency và giảm tải database. Có 2 options: thêm cache vào service hiện tại hoặc tách thành CQRS (command/query separation).

## Decision

Tách trip service thành **CQRS** (trip-command-service cho write, trip-query-service cho read), và sử dụng **Redis cache** cho read-path.

## Consequences

### Tích cực

- **CQRS**:
  - Scalability: Scale read và write độc lập
  - Optimization: Tối ưu từng path (read có thể dùng read replica, write có thể dùng primary)
  - Flexibility: Có thể chọn database phù hợp cho từng path
- **Redis Cache**:
  - Performance: Giảm latency từ ~50-100ms xuống <10ms (cache hit)
  - Database load: Giảm tải database (đọc từ cache thay vì database)
  - Cost: Giảm cost database queries

### Tiêu cực

- **Complexity**: 
  - Tăng complexity (cần maintain 2 services)
  - Cache invalidation: Cần strategy để invalidate cache khi data thay đổi
  - Data consistency: Cache có thể stale (TTL 60s)
- **Cost**: 
  - Cần Redis infrastructure
  - Cần maintain 2 services

### Trade-off

- **Performance vs Complexity**: Cache giảm latency, nhưng tăng complexity
- **Consistency vs Performance**: Cache có thể stale, nhưng acceptable cho read operations
- **Cost**: Cần Redis infrastructure, nhưng đổi lại giảm database load

## Implementation

- **trip-command-service**: Xử lý write operations, sử dụng `PRIMARY_DB_URL`
- **trip-query-service**: Xử lý read operations, sử dụng `READ_DB_URL` và Redis cache
- **Cache strategy**: Key `trip:{tripId}`, TTL 60s, fallback to database nếu cache miss
- **Cache invalidation**: TTL-based (auto expire sau 60s), có thể thêm event-based invalidation trong tương lai

