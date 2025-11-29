# ADR-014: Data Consistency - Eventual Consistency

**Status:** Accepted  
**Date:** 2024

## Context

UITGo là distributed system với nhiều services và databases. Cần chọn consistency model: strong consistency (ACID transactions) hoặc eventual consistency (BASE).

## Decision

Sử dụng **strong consistency** cho critical operations (trip creation, payment) và **eventual consistency** cho non-critical operations (location updates, events).

## Consequences

### Tích cực

- **Strong consistency cho Trip**:
  - ACID transactions: Đảm bảo trip được tạo đúng, không duplicate
  - Financial accuracy: Fare calculation, payment cần chính xác
  - User experience: User thấy data consistent ngay lập tức
- **Eventual consistency cho Location**:
  - Performance: Location updates không cần wait cho consistency
  - Scalability: Có thể scale location updates độc lập
  - Real-time: Location updates được process nhanh, không block

### Tiêu cực

- **Complexity**: Cần handle 2 consistency models
- **Stale data**: Eventual consistency có thể có stale data (location có thể không sync ngay)
- **Conflict resolution**: Cần handle conflicts nếu có (ví dụ: 2 drivers claim cùng trip)

### Trade-off

- **Consistency vs Performance**: Strong consistency đảm bảo accuracy, nhưng chậm hơn
- **Consistency vs Scalability**: Eventual consistency scale tốt hơn, nhưng có thể có stale data
- **Complexity vs Correctness**: 2 models tăng complexity, nhưng đổi lại đúng cho từng use case

## Implementation

- **Strong consistency**: Trip operations (create, accept, cancel) sử dụng PostgreSQL transactions
- **Eventual consistency**: Location updates sử dụng Redis Geo (in-memory, không persistent), publish events qua Kafka
- **Conflict resolution**: Trip assignment sử dụng Lua script (atomic) để tránh race condition
- **Future**: Có thể thêm idempotency keys để handle duplicate requests

