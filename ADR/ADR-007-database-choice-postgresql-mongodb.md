# ADR-007: Database Choice - PostgreSQL vs MongoDB

**Status:** Accepted  
**Date:** 2025

## Context

UITGo cần lưu trữ 2 loại data khác nhau: trip data (structured, relational) và user profile data (document-based, flexible schema). Cần chọn database phù hợp cho từng use case.

## Decision

Sử dụng **PostgreSQL** cho trip data và **MongoDB** cho user profile data.

## Consequences

### Tích cực

- **PostgreSQL cho Trip**:
  - ACID compliance: Đảm bảo consistency cho financial transactions (fare calculation)
  - Relational queries: Hỗ trợ complex queries (JOIN, aggregation) cho trip history, analytics
  - Prisma ORM: Type-safe, migration support, tốt cho NestJS
  - Read replicas: Có thể tách read replica cho trip-query-service
- **MongoDB cho User**:
  - Flexible schema: Dễ thêm fields mới (avatar, preferences) mà không cần migration
  - Document-based: Phù hợp cho user profile (nested data)
  - Mongoose ODM: Tốt cho NestJS, validation support

### Tiêu cực

- **Complexity**: Cần maintain 2 databases, 2 connection pools
- **Cost**: Cần infrastructure cho cả 2 databases
- **Data consistency**: Khó đảm bảo consistency giữa 2 databases (cần eventual consistency)

### Trade-off

- **Consistency vs Flexibility**: PostgreSQL đảm bảo consistency tốt hơn, MongoDB linh hoạt hơn
- **Cost vs Performance**: 2 databases tăng cost, nhưng đổi lại tối ưu cho từng use case
- **Complexity vs Optimization**: Tăng complexity, nhưng đổi lại có thể optimize tốt hơn

## Implementation

- **PostgreSQL**: trip-command-service và trip-query-service sử dụng Prisma ORM
- **MongoDB**: user-service sử dụng Mongoose ODM
- **Connection**: Mỗi service chỉ connect đến database của mình (Database-per-Service pattern)

