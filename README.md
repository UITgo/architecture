# UITGo - Hệ thống Gọi Xe

## Giới thiệu

UITGo là một hệ thống gọi xe được xây dựng theo kiến trúc microservices, hỗ trợ các chức năng đặt chuyến, tìm tài xế gần nhất, và quản lý trạng thái tài xế theo thời gian thực.

**Module A** của đồ án tập trung vào các kỹ thuật nâng cao cho trip service và driver-stream:

- **trip-command-service**: Service xử lý các thao tác ghi (write) cho trip, áp dụng pattern CQRS
- **trip-query-service**: Service xử lý các thao tác đọc (read) cho trip, sử dụng Redis cache để tối ưu hiệu năng
- **driver-stream**: Service quản lý trạng thái và vị trí tài xế, sử dụng Redis Geo và Kafka, được shard theo region (HCM, HN)
- **gateway-service**: API Gateway xử lý authentication và routing
- **k6**: Load testing scripts để đánh giá hiệu năng

## Yêu cầu môi trường

- **Node.js**: >= 20.x
- **Go**: >= 1.22
- **Docker**: >= 20.x
- **Docker Compose**: >= 2.x
- **k6**: >= 0.47.0 (để chạy load test)

## Chạy trên môi trường local

### Bước 1: Clone repository

Repository này chứa toàn bộ code Module A. Các service được tổ chức trong các thư mục:

```
UITGo/
├── gateway-service/          # API Gateway
├── auth-service/             # Authentication service
├── user-service/             # User profile service
├── trip-service/
│   ├── trip-command-service/ # Trip write service (CQRS)
│   └── trip-query-service/   # Trip read service (CQRS)
├── driver-stream/            # Driver location & status service
└── infra/                    # Docker Compose & k6 scripts
```

### Bước 2: Chạy Docker Compose

Từ thư mục root, di chuyển vào thư mục `infra` và chạy:

```bash
cd infra
docker compose up
```

Lệnh này sẽ khởi động các service sau:

**Infrastructure:**
- PostgreSQL (port 5432): Database cho trip data
- MongoDB (port 27017): Database cho user data
- Redis (port 6379): Redis Geo + cache
- Kafka + Zookeeper (ports 9092, 29092): Message broker
- OSRM (port 5000): Routing engine

**Application Services:**
- gateway-service (port 3004): API Gateway
- auth-service (port 3000): Authentication
- user-service (port 3001): User profile
- trip-command-service (port 3002): Trip write operations
- trip-query-service (port 3003): Trip read operations
- driver-stream-hcm (port 8081): Driver stream cho HCM region
- driver-stream-hn (port 8082): Driver stream cho HN region

### Bước 3: Kiểm tra health

Kiểm tra các service đã khởi động thành công:

```bash
# Kiểm tra containers đang chạy
docker ps

# Kiểm tra gateway health
curl http://localhost:3004/healthz

# Kiểm tra trip-command-service
curl http://localhost:3002/healthz

# Kiểm tra trip-query-service
curl http://localhost:3003/healthz

# Xem logs của một service
docker logs <container-name>
# Ví dụ: docker logs driver-stream-hcm
```

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

Lưu token từ response để dùng cho các request sau.

**2. Tạo trip (POST /api/v1/trips):**

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

Response sẽ chứa `tripId` và `tracking.sse` endpoint để theo dõi events.

**3. Cập nhật vị trí tài xế (PUT /api/v1/drivers/:id/location):**

```bash
curl -X PUT http://localhost:3004/api/v1/drivers/driver-001/location \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <YOUR_JWT_TOKEN>" \
  -d '{
    "lat": 10.8231,
    "lng": 106.6297,
    "speed": 45,
    "heading": 90
  }'
```

**4. Lấy thông tin trip (GET /api/v1/trips/:id):**

```bash
curl -X GET http://localhost:3004/api/v1/trips/<tripId> \
  -H "Authorization: Bearer <YOUR_JWT_TOKEN>"
```

Request này sẽ được route đến `trip-query-service` và sử dụng Redis cache nếu có.

## Load Testing với k6

Các script k6 nằm trong thư mục `infra/k6/`:

- `trips_create.js`: Test latency của create trip flow
- `drivers_update_location.js`: Test throughput của location updates
- `trips_read_cached.js`: Test read-heavy scenario với cache

Chạy một script:

```bash
cd infra/k6
k6 run trips_create.js
```

Xem thêm hướng dẫn trong `infra/k6/README.md`.

## Gợi ý deploy trên AWS

**Lưu ý:** Phần này chỉ là định hướng, chưa được triển khai trong Module A.

Để deploy UITGo lên production trên AWS, có thể sử dụng:

- **Compute**: Amazon ECS (Fargate) hoặc Amazon EKS để chạy containers
- **Database**: 
  - Amazon RDS PostgreSQL cho trip data (có thể tách read replica cho trip-query-service)
  - Amazon DocumentDB cho MongoDB (user data)
  - Amazon ElastiCache Redis cho Redis Geo và cache
- **Message Broker**: Amazon MSK (Managed Streaming for Apache Kafka)
- **API Gateway**: AWS API Gateway hoặc Application Load Balancer
- **Routing**: Có thể sử dụng AWS Location Service thay cho OSRM
- **Monitoring**: Amazon CloudWatch, AWS X-Ray cho observability

Các service có thể được deploy vào nhiều Availability Zones để đảm bảo high availability. Driver-stream sharding có thể được mở rộng thành multi-region deployment (ví dụ: ap-southeast-1a cho HCM, ap-southeast-1b cho HN).

## Cấu trúc repository

- `gateway-service/`: API Gateway với JWT authentication
- `auth-service/`: Authentication service tích hợp AWS Cognito
- `user-service/`: User profile service (MongoDB + gRPC)
- `trip-service/trip-command-service/`: Trip write service (CQRS)
- `trip-service/trip-query-service/`: Trip read service (CQRS + Redis cache)
- `driver-stream/`: Driver location & status service (Go + Redis Geo + Kafka)
- `infra/`: Docker Compose configuration và k6 load test scripts
- `proto/`: gRPC proto files
- `ADR/`: Architectural Decision Records

## Tài liệu tham khảo

- `ARCHITECTURE.md`: Mô tả chi tiết kiến trúc hệ thống
- `REPORT.md`: Báo cáo chuyên sâu về Module A
- `ADR/`: Các quyết định thiết kế kiến trúc

