# ADR-004: Authentication - Gateway-level JWT vs Per-Service JWT

**Status:** Accepted  
**Date:** 2025

## Context

UITGo cần xác thực user requests. Có 2 options: verify JWT ở gateway (centralized) hoặc ở mỗi service (distributed). Cần chọn approach phù hợp để balance giữa security, performance, và complexity.

## Decision

Đặt JWT verification ở **gateway**, gắn `X-User-Id` và `X-User-Role` vào headers cho downstream services.

## Consequences

### Tích cực

- **Centralized authentication**: Gateway xử lý một lần, các service không cần verify lại
- **Performance**: Giảm latency (không cần verify JWT ở mỗi service)
- **Security**: Gateway có thể implement rate limiting, IP whitelisting, DDoS protection
- **Simplicity**: Các service không cần maintain JWT verification logic

### Tiêu cực

- **Single point of failure**: Gateway trở thành single point of failure (cần high availability)
- **Trust**: Các service phải trust headers từ gateway (cần network security)
- **Flexibility**: Khó customize authentication logic cho từng service

### Trade-off

- **Performance vs Security**: Gateway verification nhanh hơn, nhưng cần đảm bảo gateway secure
- **Centralization vs Distribution**: Centralized dễ maintain, nhưng tăng dependency vào gateway
- **Cost**: Cần đảm bảo gateway có high availability, nhưng đổi lại giảm complexity ở các service

## Implementation

- **Gateway**: Sử dụng `JwtAuthGuard` và `JwtStrategy` để verify JWT, extract `userId` và `role`, gắn vào headers `X-User-Id`, `X-User-Role`
- **Services**: Đọc `X-User-Id` từ headers để xác định user thực hiện request
- **Network**: Các service chỉ có thể truy cập từ gateway (Docker network), không expose ra ngoài

