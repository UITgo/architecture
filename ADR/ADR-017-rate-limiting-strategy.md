# ADR-017: Rate Limiting Strategy

**Status:** Accepted  
**Date:** 2025

## Context

UITGo cần protect services khỏi abuse và DDoS attacks. Cần chọn rate limiting strategy: per-user, per-IP, hoặc per-endpoint.

## Decision

Để **rate limiting** cho future implementation, có thể implement ở gateway hoặc sử dụng AWS API Gateway rate limiting.

## Consequences

### Tích cực

- **Protection**: Bảo vệ services khỏi abuse và DDoS
- **Fair usage**: Đảm bảo fair usage giữa users
- **Cost control**: Giảm cost nếu có abuse
- **Future implementation**: Có thể implement với Redis (sliding window) hoặc AWS API Gateway

### Tiêu cực

- **Current limitation**: Chưa có rate limiting
- **Complexity**: Cần implement rate limiting logic, handle edge cases
- **False positives**: Có thể block legitimate users nếu rate limit quá strict

### Trade-off

- **Security vs Usability**: Rate limiting bảo vệ services, nhưng có thể ảnh hưởng legitimate users
- **Complexity vs Protection**: Rate limiting tăng complexity, nhưng đổi lại bảo vệ services
- **Cost**: Rate limiting cần infrastructure (Redis hoặc AWS API Gateway), nhưng đổi lại giảm cost nếu có abuse

## Implementation

- **Current**: Chưa có rate limiting
- **Future options**:
  - **Redis-based**: Sử dụng Redis với sliding window algorithm
  - **AWS API Gateway**: Sử dụng AWS API Gateway rate limiting (nếu deploy lên AWS)
  - **Per-user**: Rate limit dựa trên `userId` (sau khi authenticate)
  - **Per-IP**: Rate limit dựa trên IP address (cho unauthenticated requests)
  - **Per-endpoint**: Rate limit khác nhau cho từng endpoint (ví dụ: location update có thể có rate limit cao hơn)

