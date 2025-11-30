# UITGo - Hệ thống Gọi Xe

## 1. Giới thiệu UITGo

UITGo là một hệ thống gọi xe được xây dựng theo kiến trúc **microservices**, hỗ trợ các chức năng đặt chuyến, tìm tài xế gần nhất, và quản lý trạng thái tài xế theo thời gian thực. Hệ thống được thiết kế để scale theo region (HCM, HN) và tối ưu cho cả read-heavy và write-heavy workloads.

**Repo `architecture` này là "cửa chính" (entry point) của toàn bộ hệ thống UITGo.** Repo này chứa:
- Tài liệu kiến trúc tổng thể (`ARCHITECTURE.md`)
- Báo cáo chuyên sâu Module A (`REPORT.md`)
- Architectural Decision Records (`ADR/`) - 18 quyết định thiết kế quan trọng
- Bảng liệt kê và link tới tất cả các service/repo khác trong hệ thống

## 2. Kiến trúc tổng thể & danh sách service

UITGo sử dụng kiến trúc **microservices** với các pattern:
- **API Gateway Pattern**: Single entry point cho tất cả clients
- **Database-per-Service**: Mỗi service có database riêng (PostgreSQL, MongoDB)
- **CQRS**: Trip service tách thành command (write) và query (read)
- **Event-Driven Architecture**: Kafka cho real-time event streaming
- **Sharding**: Driver-stream được shard theo region (HCM, HN)
- **Caching**: Redis cho read-heavy paths và Redis Geo cho spatial queries

### Bảng liệt kê toàn bộ components

| Component | Repo / Folder | Technology / Description |
|-----------|---------------|--------------------------|
| **API Gateway** | [`gateway-service`](https://github.com/UITgo/gateway-service) | NestJS, expose `/api/v1/*`, JWT authentication, request routing |
| **Auth Service** | [`auth-service`](../auth-service) | NestJS, AWS Cognito integration, JWT token generation |
| **User Service** | [`user-service`](../user-service) | NestJS + MongoDB, user profile management, gRPC service |
| **Trip Command** | [`trip-service/trip-command-service`](../trip-service/trip-command-service) | NestJS + PostgreSQL, CQRS write side, region sharding |
| **Trip Query** | [`trip-service/trip-query-service`](../trip-service/trip-query-service) | NestJS + PostgreSQL + Redis, CQRS read side, caching |
| **Driver Stream** | [`driver-stream`](../driver-stream) | Go + Redis Geo + Kafka, driver location & status, trip assignment |
| **Infrastructure** | [`infra`](../infra) | Docker Compose, k6 load testing scripts, infrastructure configs |
| **Proto Contracts** | [`proto`](../proto) | gRPC/Protobuf definitions cho inter-service communication |
| **IaC** | [`iac`](../iac) | Terraform/Bicep cho AWS infrastructure |
| **Frontend User** | `fe-user` | Frontend cho passenger app  |
| **Frontend Driver** | `fe-driver` | Frontend cho driver app  |
| **Architecture Docs** | `architecture` (repo này) | Tài liệu: README, ARCHITECTURE.md, REPORT.md, ADR/ |

## 3. Tài liệu kiến trúc & báo cáo

### [`ARCHITECTURE.md`](./ARCHITECTURE.md)
Mô tả chi tiết kiến trúc hệ thống UITGo, bao gồm:
- Kiến trúc tổng thể (microservices, communication patterns)
- Kiến trúc Module A (trip flow, driver-stream flow, CQRS, sharding)
- Technology stack và lý do lựa chọn
- Database schema và data flow

### [`REPORT.md`](./REPORT.md)
Báo cáo chuyên sâu Module A (3-5 trang), bao gồm:
- Phân tích kiến trúc hệ thống
- In-depth analysis của Module A (CQRS, Redis cache, Redis Geo, sharding, k6)
- Architectural decisions và trade-offs
- Challenges và lessons learned
- Results và future development

### [`ADR/`](./ADR/)
Thư mục chứa 18 Architectural Decision Records, mỗi ADR document một quyết định thiết kế quan trọng:

- **ADR-001**: Architecture Style - Microservices
- **ADR-002**: Communication - REST, gRPC, Kafka
- **ADR-003**: Driver Location - Redis Geo vs DynamoDB
- **ADR-004**: Authentication - Gateway vs Per-Service JWT
- **ADR-005**: Trip Read Path - Cache Redis, CQRS
- **ADR-006**: Driver Stream - Sharding by Region
- **ADR-007**: Database Choice - PostgreSQL vs MongoDB
- **ADR-008**: Message Queue - Kafka vs RabbitMQ
- **ADR-009**: Caching Strategy - Redis vs Memcached
- **ADR-010**: API Gateway Pattern
- **ADR-011**: Container Orchestration - Docker Compose
- **ADR-012**: Error Handling and Retry Strategy
- **ADR-013**: Monitoring and Observability Strategy
- **ADR-014**: Data Consistency - Eventual Consistency
- **ADR-015**: Testing Strategy - Unit, Integration, Load Testing
- **ADR-016**: Security - JWT vs OAuth2
- **ADR-017**: Rate Limiting Strategy
- **ADR-018**: Deployment Strategy - Blue-Green vs Rolling

Mỗi ADR theo format chuẩn: Title, Status, Date, Context, Decision, Consequences.

## 4. Cách bắt đầu đọc & chạy hệ thống

### Bước 1: Đọc tài liệu kiến trúc

Để hiểu tổng quan về hệ thống, hãy đọc theo thứ tự:

1. **README này** - Entry point, tổng quan về các components
2. **[`ARCHITECTURE.md`](./ARCHITECTURE.md)** - Kiến trúc chi tiết, technology stack, data flow
3. **[`REPORT.md`](./REPORT.md)** - Báo cáo chuyên sâu Module A, challenges, results
4. **[`ADR/`](./ADR/)** - Các quyết định thiết kế (đọc theo thứ tự ADR-001 → ADR-018)

### Bước 2: Clone các service repositories

Sau khi hiểu kiến trúc, clone các service repositories cần thiết:

```bash
# Clone các service chính
git clone https://github.com/UITgo/gateway-service
git clone https://github.com/UITgo/auth-service
git clone https://github.com/UITgo/user-service
git clone https://github.com/UITgo/trip-service
git clone https://github.com/UITgo/driver-stream

# Clone infrastructure
git clone https://github.com/UITgo/infra
git clone https://github.com/UITgo/proto
```

Vì nhóm tổ chức multi-repo github.

### Bước 3: Chạy hệ thống local với Docker Compose

Để demo hệ thống local, vào repo [`infra`](../infra) và chạy:

```bash
cd infra
docker compose up
```

Lệnh này sẽ khởi động:
- **Infrastructure**: PostgreSQL, MongoDB, Redis, Kafka, OSRM
- **Services**: gateway-service, auth-service, user-service, trip-command-service, trip-query-service, driver-stream-hcm, driver-stream-hn

Sau khi các service đã start, kiểm tra health:

```bash
# Gateway
curl http://localhost:3004/healthz

# Trip Command
curl http://localhost:3002/healthz

# Trip Query
curl http://localhost:3003/healthz
```

Xem hướng dẫn chi tiết trong [`infra/README.md`](../infra/README.md).

### Bước 4: Test API

**1. Đăng nhập để lấy JWT token:**

```bash
curl -X POST http://localhost:3004/api/v1/sessions \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "password"
  }'
```

**2. Tạo trip:**

```bash
curl -X POST http://localhost:3004/api/v1/trips \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_JWT_TOKEN>" \
  -d '{
    "origin": { "lat": 10.8231, "lng": 106.6297 },
    "destination": { "lat": 10.7626, "lng": 106.6602 },
    "cityCode": "HCM",
    "note": "Test trip"
  }'
```

**3. Lấy thông tin trip (sử dụng Redis cache):**

```bash
curl -X GET http://localhost:3004/api/v1/trips/<tripId> \
  -H "Authorization: Bearer <YOUR_JWT_TOKEN>"
```

Xem thêm examples trong README của từng service.

### Load Testing với k6

Các script k6 nằm trong [`infra/k6/`](../infra/k6/):

- `trips_create.js`: Test latency của create trip flow
- `drivers_update_location.js`: Test throughput của location updates
- `trips_read_cached.js`: Test read-heavy scenario với cache

Chạy một script:

```bash
cd infra/k6
k6 run trips_create.js
```

## 5. Yêu cầu môi trường

- **Node.js**: >= 20.x
- **Go**: >= 1.22
- **Docker**: >= 20.x
- **Docker Compose**: >= 2.x
- **k6**: >= 0.47.0 (để chạy load test)

## 6. Cấu trúc repository

```
UITGo/
├── architecture/          # Repo này - tài liệu kiến trúc
│   ├── README.md          # Entry point (file này)
│   ├── ARCHITECTURE.md    # Kiến trúc chi tiết
│   ├── REPORT.md          # Báo cáo cuôi kỳ
│   └── ADR/               # Architectural Decision Records
├── gateway-service/        # API Gateway
├── auth-service/          # Authentication service
├── user-service/          # User profile service
├── trip-service/
│   ├── trip-command-service/  # Trip write (CQRS)
│   └── trip-query-service/    # Trip read (CQRS + cache)
├── driver-stream/         # Driver location & status
├── infra/                 # Docker Compose, k6
├── proto/                 # gRPC proto files
└── iac/                   # Infrastructure as Code (Terraform)
```

## 7. Liên kết nhanh

- **Kiến trúc**: [`ARCHITECTURE.md`](./ARCHITECTURE.md)
- **Báo cáo**: [`REPORT.md`](./REPORT.md)
- **ADRs**: [`ADR/`](./ADR/)
- **Infrastructure**: [`infra/README.md`](../infra/README.md)
- **Gateway Service**: [`gateway-service/README.md`](../gateway-service/README.md)
- **Auth Service**: [`auth-service/README.md`](../auth-service/README.md)
- **User Service**: [`user-service/README.md`](../user-service/README.md)
- **Trip Command**: [`trip-service/trip-command-service/README.md`](../trip-service/trip-command-service/README.md)
- **Trip Query**: [`trip-service/trip-query-service/README.md`](../trip-service/trip-query-service/README.md)
- **Driver Stream**: [`driver-stream/README.md`](../driver-stream/README.md)

---


