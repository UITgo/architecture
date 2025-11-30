# ADR-013: Monitoring and Observability Strategy

**Status:** Accepted  
**Date:** 2025

## Context

UITGo là distributed system với nhiều services. Cần monitor health, performance, errors để debug và optimize. Cần chọn strategy cho logging, metrics, và tracing.

## Decision

Sử dụng **structured logging** với correlation IDs, **health check endpoints** cho mỗi service, và để **metrics/tracing** (Prometheus, Jaeger) cho production.

## Consequences

### Tích cực

- **Structured logging**: Log với JSON format, dễ parse và search
- **Correlation ID**: Gateway generate `X-Request-Id`, propagate qua các services, dễ trace request flow
- **Health checks**: Mỗi service có `/healthz` endpoint, Docker Compose sử dụng để check readiness
- **Future metrics/tracing**: Có thể thêm Prometheus metrics và Jaeger tracing trong production

### Tiêu cực

- **Current limitation**: Chưa có centralized logging (mỗi service log riêng)
- **No metrics**: Chưa có metrics collection (latency, throughput, error rate)
- **No tracing**: Chưa có distributed tracing (khó debug cross-service calls)

### Trade-off

- **Simplicity vs Observability**: Current approach đơn giản, nhưng ít observability
- **Cost vs Features**: Centralized logging/metrics/tracing cần infrastructure, nhưng đổi lại có observability tốt hơn
- **Development vs Production**: Current approach đủ cho development, production cần thêm metrics/tracing

## Implementation

- **Logging**: Mỗi service sử dụng NestJS Logger hoặc Go log, format structured
- **Correlation ID**: Gateway generate UUID, gắn vào `X-Request-Id` header, log trong mỗi service
- **Health checks**: Mỗi service có `/healthz` endpoint, check database/Redis connections
- **Future**: Có thể thêm:
  - **Prometheus**: Metrics (request count, latency, error rate)
  - **Jaeger**: Distributed tracing
  - **ELK Stack**: Centralized logging (Elasticsearch, Logstash, Kibana)
  - **Grafana**: Visualization cho metrics và logs

