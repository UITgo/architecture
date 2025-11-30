# ADR-011: Container Orchestration - Docker Compose

**Status:** Accepted  
**Date:** 2025

## Context

UITGo có nhiều services và infrastructure components (PostgreSQL, MongoDB, Redis, Kafka, OSRM). Cần chọn cách orchestrate containers: Docker Compose (local development) hoặc Kubernetes (production).

## Decision

Sử dụng **Docker Compose** cho local development và demo, để **Kubernetes** cho production deployment.

## Consequences

### Tích cực

- **Docker Compose**:
  - Simplicity: Dễ setup, không cần cluster
  - Development: Phù hợp cho local development, testing
  - Quick start: Có thể start toàn bộ stack với 1 command
- **Kubernetes (tương lai)**:
  - Production-ready: Auto-scaling, self-healing, rolling updates
  - High availability: Pod replication, health checks
  - Service discovery: Built-in service discovery

### Tiêu cực

- **Docker Compose**:
  - Single node: Chỉ chạy trên 1 machine, không có high availability
  - No auto-scaling: Không tự động scale services
  - Manual management: Cần manually start/stop services
- **Kubernetes**:
  - Complexity: Phức tạp hơn Docker Compose, cần cluster
  - Learning curve: Khó học hơn Docker Compose

### Trade-off

- **Simplicity vs Features**: Docker Compose đơn giản hơn, nhưng ít features hơn Kubernetes
- **Development vs Production**: Docker Compose tốt cho development, Kubernetes tốt cho production
- **Cost**: Docker Compose free (local), Kubernetes cần cloud resources

## Implementation

- **Docker Compose**: File `infra/docker-compose.yml` định nghĩa tất cả services
- **Networks**: Tất cả services trong cùng Docker network `uitgo-net`
- **Health checks**: Mỗi service có health check để đảm bảo ready trước khi start
- **Dependencies**: Sử dụng `depends_on` để đảm bảo services start theo thứ tự
- **Future**: Có thể migrate lên Kubernetes với Helm charts hoặc Terraform

