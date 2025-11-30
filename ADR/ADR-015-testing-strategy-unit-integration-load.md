# ADR-015: Testing Strategy - Unit, Integration, Load Testing

**Status:** Accepted  
**Date:** 2025

## Context

UITGo cần đảm bảo quality và performance. Cần chọn testing strategy: unit tests, integration tests, và load tests.

## Decision

Sử dụng **k6** cho load testing, để **unit tests** và **integration tests** cho future development.

## Consequences

### Tích cực

- **k6 Load Testing**:
  - Script-based: Dễ viết và maintain test scripts
  - Realistic scenarios: Test với realistic data và traffic patterns
  - Metrics: Cung cấp detailed metrics (latency, throughput, error rate)
  - Thresholds: Có thể set SLA thresholds (p95 < 200ms)
- **Future unit/integration tests**:
  - Unit tests: Test individual functions/services
  - Integration tests: Test service interactions
  - E2E tests: Test end-to-end flows

### Tiêu cực

- **Current limitation**: Chưa có unit tests và integration tests
- **Manual testing**: Cần manually run k6 scripts
- **No CI/CD**: Chưa integrate tests vào CI/CD pipeline

### Trade-off

- **Coverage vs Effort**: Load testing đã có, unit/integration tests cần thêm effort
- **Manual vs Automated**: Current approach manual, automated tests cần CI/CD setup
- **Cost**: Tests cần time và resources, nhưng đổi lại đảm bảo quality

## Implementation

- **k6 scripts**: Trong `infra/k6/`:
  - `trips_create.js`: Test create trip latency
  - `drivers_update_location.js`: Test location update throughput
  - `trips_read_cached.js`: Test read-heavy scenario với cache
- **Test data**: Sử dụng realistic data (HCM coordinates, multiple drivers)
- **Thresholds**: Set SLA thresholds (p95 < 200ms cho create, p95 < 50ms cho location update)
- **Future**: Có thể thêm:
  - **Jest** (Node.js) cho unit tests
  - **Supertest** cho integration tests
  - **GitHub Actions** cho CI/CD pipeline

