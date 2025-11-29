# ADR-009: Caching Strategy - Redis vs Memcached

**Status:** Accepted  
**Date:** 2024

## Context

UITGo cần cache cho 2 use cases: Trip data (trip-query-service) và driver location (driver-stream với Redis Geo). Cần chọn caching solution phù hợp.

## Decision

Sử dụng **Redis** cho cả 2 use cases (Trip cache và Redis Geo).

## Consequences

### Tích cực

- **Redis Geo**: Redis hỗ trợ Geo commands (GEOADD, GEORADIUS) mà Memcached không có
- **Data structures**: Redis hỗ trợ nhiều data structures (strings, hashes, sets, sorted sets) phù hợp cho nhiều use cases
- **Persistence**: Redis có thể persist data (AOF/RDB) nếu cần
- **Pub/Sub**: Redis hỗ trợ pub/sub nếu cần real-time notifications
- **Single solution**: Chỉ cần maintain 1 caching solution thay vì 2

### Tiêu cực

- **Memory**: Redis sử dụng nhiều memory hơn Memcached (do data structures phức tạp hơn)
- **Complexity**: Redis phức tạp hơn Memcached (nhiều commands, configurations)
- **Cost**: Redis có thể đắt hơn Memcached (do cần nhiều memory hơn)

### Trade-off

- **Features vs Simplicity**: Redis có nhiều features hơn (Geo, pub/sub), nhưng phức tạp hơn
- **Memory vs Functionality**: Redis sử dụng nhiều memory hơn, nhưng đổi lại có nhiều functionality hơn
- **Cost**: Redis có thể đắt hơn, nhưng đổi lại chỉ cần 1 solution cho nhiều use cases

## Implementation

- **Trip cache**: trip-query-service sử dụng Redis strings với key `trip:{tripId}`, TTL 60s
- **Redis Geo**: driver-stream sử dụng Redis Geo set `geo:drivers` và hash `presence:driver:{id}`
- **Connection**: Mỗi service có Redis client riêng, có thể dùng connection pooling

