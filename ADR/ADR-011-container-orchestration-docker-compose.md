# ADR-011: Container Orchestration - Docker Compose & ECS

**Status:** Accepted  
**Date:** 2025

## Context

UITGo gồm nhiều services và infrastructure components (PostgreSQL, MongoDB, Redis, Kafka, OSRM). Cần chọn cách orchestrate containers cho development và production:
- Development: cần setup nhanh, dễ test local.
- Production: cần auto-scaling, high availability, service discovery, monitoring, integrate với AWS stack (ECS, ALB, IAM, VPC, SG).

## Decision

Sử dụng **Docker Compose** cho local development và demo. Sử dụng AWS ECS (Fargate/EC2) kết hợp ALB để deploy microservices, với auto-scaling, health checks và service discovery

## Consequences

### Tích cực

**Docker Compose**:
- Simplicity: Dễ setup, không cần cluster
- Development: Phù hợp cho local development, testing
- Quick start: Có thể start toàn bộ stack với 1 command
**ECS / AWS Production**:
- Production-ready: Auto-scaling, rolling updates, self-healing
- High availability: Pod/task replication, ALB health checks
- Service discovery: ECS service registry + ALB routing
- Security: IAM roles, Security Groups, VPC isolation

### Tiêu cực

**Docker Compose**:
- Single node: Chỉ chạy trên 1 machine, không có high availability
- No auto-scaling: Không tự động scale services
- Manual management: Cần manually start/stop services
- 
**ECS**:
- Complexity: cần hiểu AWS concepts (ECS, ALB, IAM, VPC, SG)
- Learning curve: setup infra phức tạp hơn Docker Compose
- Cost: sử dụng cloud resources

### Trade-off

- **Simplicity vs Features**: Docker Compose đơn giản nhưng ít features; ECS mạnh hơn nhưng phức tạp.
- **Development vs Production**: Docker Compose tốt cho local dev/test; ECS phù hợp production.
- **Cost**: Docker Compose free; ECS cần cloud resources.

## Implementation

**Docker Compose**:
- File: infra/docker-compose.yml định nghĩa tất cả services
- Networks: Docker network uitgo-net để các service giao tiếp
- Health checks: đảm bảo service ready trước khi start dependent services
- Dependencies: depends_on đảm bảo thứ tự start services

**ECS**:
- ECS Cluster cho tất cả microservices
- ECS Service mỗi service riêng biệt, kết hợp ALB route gRPC/HTTP requests
- IAM roles cho mỗi service để truy cập DB/Kafka/Redis
- Security Groups & VPC để bảo vệ network
- Health checks và auto-scaling policy
- CI/CD deploy qua Terraform / CloudFormation / GitHub Actions
