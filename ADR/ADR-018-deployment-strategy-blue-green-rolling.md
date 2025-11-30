# ADR-018: Deployment Strategy - Blue-Green vs Rolling

**Status:** Accepted  
**Date:** 2025

## Context

UITGo cần deploy updates mà không downtime. Cần chọn deployment strategy: blue-green (2 environments, switch) hoặc rolling (gradual update).

## Decision

Để **deployment strategy** cho future implementation, có thể sử dụng blue-green hoặc rolling tùy vào use case.

## Consequences

### Tích cực

- **Blue-Green**:
  - Zero downtime: Switch từ blue sang green ngay lập tức
  - Easy rollback: Có thể rollback bằng cách switch lại
  - Testing: Có thể test green environment trước khi switch
- **Rolling**:
  - Gradual update: Update từng instance một, không cần 2 environments
  - Resource efficient: Chỉ cần 1 environment, scale up/down gradually

### Tiêu cực

- **Blue-Green**:
  - Resource cost: Cần 2 environments (2x resources)
  - Database migration: Cần handle database migration carefully
- **Rolling**:
  - Compatibility: Cần đảm bảo backward compatibility giữa old và new versions
  - Gradual risk: Nếu có bug, có thể affect một phần users

### Trade-off

- **Zero downtime vs Cost**: Blue-green đảm bảo zero downtime, nhưng tốn resources
- **Simplicity vs Safety**: Rolling đơn giản hơn, nhưng blue-green an toàn hơn
- **Cost**: Blue-green tốn resources hơn, nhưng đổi lại an toàn hơn

## Implementation

- **Current**: Docker Compose, manual deployment (stop old, start new)
- **Future options**:
  - **Kubernetes**: Sử dụng Kubernetes rolling updates hoặc blue-green deployment
  - **ECS**: Sử dụng AWS ECS blue-green deployment
  - **Database migration**: Sử dụng Prisma migrations với zero-downtime strategy
  - **Health checks**: Đảm bảo new version healthy trước khi switch traffic

