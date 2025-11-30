# ADR-016: Security - JWT vs OAuth2

**Status:** Accepted  
**Date:** 2025

## Context

UITGo cần authenticate users. Cần chọn authentication mechanism: JWT (simple) hoặc OAuth2 (standard, more complex).

## Decision

Sử dụng **JWT** với AWS Cognito cho authentication, để **OAuth2** cho future third-party integrations.

## Consequences

### Tích cực

- **JWT**:
  - Simplicity: Đơn giản hơn OAuth2, dễ implement
  - Stateless: Không cần session storage, scale tốt
  - Self-contained: Token chứa user info, không cần query database mỗi request
  - AWS Cognito: Managed service, handle user registration, email verification, password reset
- **OAuth2 (future)**:
  - Standard: Industry standard, dễ integrate với third-party (Google, Facebook)
  - Security: More secure với refresh tokens, authorization codes

### Tiêu cực

- **JWT**:
  - No revocation: Không thể revoke token trước khi expire (cần blacklist)
  - Token size: Token có thể lớn nếu chứa nhiều claims
- **OAuth2**:
  - Complexity: Phức tạp hơn JWT, cần authorization server
  - Overhead: More network calls (authorization code flow)

### Trade-off

- **Simplicity vs Security**: JWT đơn giản hơn, nhưng OAuth2 secure hơn
- **Stateless vs Stateful**: JWT stateless, OAuth2 cần authorization server
- **Cost**: AWS Cognito có cost, nhưng đổi lại managed service

## Implementation

- **JWT**: auth-service sử dụng AWS Cognito để generate JWT tokens
- **Verification**: Gateway verify JWT với `JWT_SECRET`, extract `userId` và `role`
- **Headers**: Gateway gắn `X-User-Id` và `X-User-Role` vào headers cho downstream services
- **Future**: Có thể thêm OAuth2 cho third-party login (Google, Facebook)

