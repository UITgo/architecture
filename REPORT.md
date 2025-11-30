# Báo Cáo Module A - UITGo

## 1. Tổng quan kiến trúc hệ thống

UITGo là một hệ thống gọi xe được xây dựng theo kiến trúc microservices, trong đó mỗi service quản lý một bounded context riêng biệt. Hệ thống bao gồm các service chính: gateway-service (API Gateway), auth-service (xác thực), user-service (quản lý profile), trip-command-service và trip-query-service (quản lý trip theo CQRS), driver-stream (quản lý trạng thái và vị trí tài xế).

Hệ thống sử dụng các thành phần hạ tầng: PostgreSQL cho trip data, MongoDB cho user data, Redis cho Redis Geo và cache, Kafka cho event streaming, và OSRM cho routing. Giao tiếp giữa các service sử dụng REST (HTTP) cho external API, gRPC cho internal communication, và Kafka cho event streaming.

Các pattern đã áp dụng bao gồm: API Gateway Pattern (gateway-service xử lý authentication và routing), Database-per-Service (mỗi service có database riêng), CQRS cho Trip service (tách command và query), Sharding driver-stream theo region (HCM, HN), Event-Driven Architecture với Kafka, và Caching Strategy với Redis.

## 2. Phân tích Module chuyên sâu

### 2.1. Luồng đặt chuyến & tìm tài xế (Latency-sensitive, Read-heavy)

Luồng này được thiết kế để tối ưu cho latency và xử lý read-heavy traffic.

**Flow chi tiết:**

1. **Client → Gateway**: Client gửi `POST /api/v1/trips` với body chứa `origin`, `destination`, `cityCode`. Gateway (`gateway-service/src/trips-proxy.controller.ts`) verify JWT token, extract `userId` và `role`, gắn vào headers `X-User-Id`, `X-User-Role`, sau đó route đến `trip-command-service`.

2. **trip-command-service.create()**: 
   - Verify user qua gRPC `GetProfile` đến `user-service` để đảm bảo user có role `PASSENGER`
   - Tính fare sử dụng OSRM hoặc fake calculation
   - Lưu trip vào PostgreSQL với `status = 'DRIVER_SEARCHING'` và `cityCode` để routing
   - Chọn shard driver-stream dựa trên `cityCode` (HCM → driver-stream-hcm, HN → driver-stream-hn)
   - Gọi `GetNearbyDrivers` qua HTTP đến shard tương ứng, sử dụng Redis Geo (`GEORADIUS`) để tìm tài xế trong bán kính 3km
   - Gọi `PrepareAssign` để lưu candidates vào Redis và push SSE event đến các driver candidates
   - Lưu assignments vào PostgreSQL và emit SSE event

**Tối ưu đã implement:**

- **CQRS**: Tách trip-command-service (write) và trip-query-service (read) để scale độc lập
- **Redis Geo**: Sử dụng Redis Geo thay vì query database quan hệ, giảm latency từ ~100ms xuống ~10-20ms
- **Sharding**: Driver-stream được shard theo region, giảm số lượng tài xế cần query và latency
- **Redis Cache**: trip-query-service cache Trip data với TTL 60s, giảm latency cho read operations từ ~50-100ms xuống <10ms

**Kết quả:** Latency của create trip flow đạt <200ms (p95) theo k6 test, với phần lớn thời gian dành cho Redis Geo query và database write.

### 2.2. Luồng cập nhật vị trí tài xế (Write-heavy, Throughput-sensitive)

Luồng này được thiết kế để xử lý high-throughput location updates.

**Flow chi tiết:**

1. **Client → Gateway**: Driver app gửi `PUT /api/v1/drivers/:id/location` với body chứa `lat`, `lng`, `speed`, `heading`. Gateway route đến `driver-stream`.

2. **driver-stream xử lý**:
   - Update Redis Geo: `GEOADD geo:drivers <lng> <lat> <driverId>` để update location trong Redis Geo set
   - Update metadata: `HSET presence:driver:{id} lat <lat> lng <lng> speed <speed> heading <heading> last_seen <timestamp>` với TTL 60s
   - Publish Kafka: Publish event vào topic `driver.location.{region}` với payload chứa location data

**Tối ưu đã implement:**

- **Redis Geo**: Sử dụng Redis Geo để lưu location, không cần query database, giảm latency xuống <50ms
- **Kafka**: Publish events async, không block request, cho phép multiple consumers subscribe
- **Sharding**: Mỗi region có Kafka topic riêng (`driver.location.hcm`, `driver.location.hn`), giảm contention

**Kết quả:** Throughput của location updates đạt <50ms (p95) theo k6 test, với khả năng xử lý hàng nghìn updates mỗi giây.

## 3. Tổng hợp các quyết định thiết kế và Trade-off

### 3.1. Microservices Architecture (ADR-001)

**Quyết định:** Chọn microservices thay vì monolithic architecture.

**Lý do:**
- Mỗi service có thể scale độc lập (ví dụ: driver-stream cần scale cao hơn cho location updates)
- Có thể deploy và update từng service riêng biệt
- Mỗi service có thể chọn công nghệ phù hợp (NestJS cho business logic, Go cho driver-stream)

**Trade-off:**
- **Tích cực**: Scalability, flexibility, technology diversity
- **Tiêu cực**: Tăng complexity (network calls, distributed transactions), khó debug (cần distributed tracing)
- **Chi phí**: Cần nhiều infrastructure hơn (multiple databases, message brokers)

### 3.2. Communication: REST + gRPC + Kafka (ADR-002)

**Quyết định:** Sử dụng REST cho external API, gRPC cho internal communication, Kafka cho event streaming.

**Lý do:**
- REST: Standard, dễ debug, phù hợp cho external API
- gRPC: Binary protocol, hiệu năng cao, type-safe với proto files, phù hợp cho internal communication
- Kafka: Decoupling, hỗ trợ multiple consumers, durable storage

**Trade-off:**
- **Tích cực**: Hiệu năng cao (gRPC), decoupling (Kafka), dễ debug (REST)
- **Tiêu cực**: Tăng complexity (cần maintain proto files, Kafka infrastructure)
- **Chi phí**: Cần infrastructure cho Kafka, nhưng đổi lại có thể scale tốt

### 3.3. Driver Location: Redis Geo + Kafka (ADR-003)

**Quyết định:** Sử dụng Redis Geo để lưu driver location, publish events qua Kafka, để DynamoDB cho tương lai.

**Lý do:**
- Redis Geo: Hiệu năng cao cho spatial queries, real-time updates, không cần query database
- Kafka: Decoupling, cho phép các service khác subscribe để xử lý events
- DynamoDB: Có thể sử dụng trong tương lai cho location history (time-series data)

**Trade-off:**
- **Tích cực**: Hiệu năng cao (Redis Geo <10ms), real-time (Kafka events)
- **Tiêu cực**: Redis không persistent (cần backup), Kafka cần infrastructure
- **Chi phí**: Cần Redis và Kafka infrastructure, nhưng đổi lại có thể xử lý high-throughput

### 3.4. Authentication: Gateway-level JWT (ADR-004)

**Quyết định:** Đặt JWT verification ở gateway, gắn `X-User-Id` và `X-User-Role` vào headers cho downstream services.

**Lý do:**
- Centralized authentication: Gateway xử lý một lần, các service không cần verify lại
- Giảm latency: Không cần verify JWT ở mỗi service
- Security: Gateway có thể implement rate limiting, IP whitelisting

**Trade-off:**
- **Tích cực**: Giảm latency, centralized security
- **Tiêu cực**: Gateway trở thành single point of failure (cần high availability)
- **Chi phí**: Cần đảm bảo gateway có high availability

### 3.5. CQRS + Redis Cache (ADR-005)

**Quyết định:** Tách trip service thành command (write) và query (read), sử dụng Redis cache cho read-path.

**Lý do:**
- CQRS: Cho phép scale read và write độc lập, tối ưu từng path
- Redis Cache: Giảm latency cho read operations, giảm tải database

**Trade-off:**
- **Tích cực**: Scalability (scale read và write độc lập), performance (cache giảm latency)
- **Tiêu cực**: Tăng complexity (cần maintain 2 services, cache invalidation)
- **Chi phí**: Cần Redis infrastructure, nhưng đổi lại có thể scale tốt hơn
- **Consistency**: Cache có thể stale (TTL 60s), nhưng acceptable cho read operations

### 3.6. Sharding driver-stream theo Region (ADR-006)

**Quyết định:** Shard driver-stream theo region (HCM, HN), mỗi region có container riêng, Redis DB index riêng, Kafka topic riêng.

**Lý do:**
- Scale theo địa lý: Mỗi region có thể scale độc lập
- Giảm latency: Tài xế HCM không cần query tài xế HN
- Isolation: Lỗi ở một region không ảnh hưởng region khác

**Trade-off:**
- **Tích cực**: Scalability, performance, isolation
- **Tiêu cực**: Tăng complexity (cần routing logic, maintain multiple containers)
- **Chi phí**: Cần nhiều containers hơn, nhưng đổi lại có thể scale tốt hơn

## 4. Thách thức & Bài học kinh nghiệm

### 4.1. Thách thức khi implement

**gRPC nội bộ:**
- Khó khăn: Cần maintain proto files, generate code, handle versioning
- Giải pháp: Sử dụng shared proto repository, generate code trong CI/CD
- Bài học: gRPC tốt cho internal communication, nhưng cần tooling tốt

**Docker Compose multi-service:**
- Khó khăn: Debug khi có nhiều services, network issues, dependency management
- Giải pháp: Sử dụng health checks, logging tập trung, dependency ordering
- Bài học: Cần observability tốt (logging, metrics, tracing) để debug distributed system

**Redis/Kafka:**
- Khó khăn: Redis không persistent (cần backup), Kafka cần Zookeeper, configuration phức tạp
- Giải pháp: Sử dụng Redis persistence (AOF/RDB), Kafka với health checks
- Bài học: Cần hiểu rõ behavior của Redis và Kafka (TTL, partitions, consumers)

**k6 Load Testing:**
- Khó khăn: Viết scripts phức tạp, interpret results
- Giải pháp: Sử dụng thresholds, custom metrics, visualize results
- Bài học: Load testing quan trọng để verify performance, cần test thường xuyên

### 4.2. Bài học rút ra

**Thiết kế hệ thống:**
- CQRS phù hợp cho read-heavy và write-heavy scenarios khác nhau
- Sharding theo region phù hợp cho geo-distributed systems
- Cache cần có TTL và invalidation strategy

**Observability:**
- Cần logging tập trung để debug distributed system
- Cần metrics để monitor performance (latency, throughput, error rate)
- Cần tracing để track request flow qua multiple services

**Testing:**
- Load testing quan trọng để verify performance
- Cần test với realistic data và scenarios
- Cần test failure scenarios (service down, network issues)

## 5. Kết quả & Hướng phát triển

### 5.1. Kết quả đạt được

**Các flow đã chạy được:**
- Create Trip: Client có thể tạo trip, hệ thống tự động tìm tài xế gần nhất
- Find Driver: Sử dụng Redis Geo để tìm tài xế trong bán kính, latency <200ms (p95)
- Update Location: Driver có thể update location, hệ thống publish events qua Kafka, latency <50ms (p95)
- Accept Trip: Driver có thể accept trip, sử dụng Lua script để atomic claim

**Kết quả load test (k6):**
- Create Trip: p95 <200ms với 30 VUs
- Update Location: p95 <50ms với 50 VUs
- Read Trip (với cache): p95 <100ms với 50 VUs
- ![result](https://github.com/UITgo/architecture/blob/main/asset/result.png)

**Tối ưu đã implement:**
- CQRS: Tách command và query, scale độc lập
- Redis Cache: Giảm latency cho read operations
- Redis Geo: Tối ưu spatial queries
- Sharding: Scale theo region
- Kafka: Event-driven architecture

### 5.2. Hướng phát triển

**Hoàn thiện multi-region:**
- Triển khai multi-region thật trên AWS (ap-southeast-1a cho HCM, ap-southeast-1b cho HN)
- Sử dụng Route 53 để route requests đến region gần nhất
- Replicate data giữa các regions

**Chuyển sang AWS:**
- Deploy lên ECS/EKS
- Sử dụng RDS PostgreSQL với read replicas
- Sử dụng ElastiCache Redis
- Sử dụng MSK Kafka
- Sử dụng AWS Location Service thay cho OSRM

**Observability:**
- Thêm metrics (CloudWatch, Prometheus)
- Thêm tracing (AWS X-Ray, Jaeger)
- Thêm logging tập trung (CloudWatch Logs, ELK)

**Refine k6:**
- Thêm more realistic scenarios
- Test failure scenarios
- Visualize results với Grafana

**Triển khai read replica DB:**
- Tách read replica cho trip-query-service
- Sử dụng connection pooling
- Monitor replication lag

**DynamoDB cho location history:**
- Sử dụng DynamoDB để lưu location history (time-series data)
- Sử dụng TTL để auto-delete old data
- Query location history để analytics

**Cải thiện cache:**
- Implement cache invalidation strategy
- Sử dụng cache warming
- Monitor cache hit rate


## 6. Phân chia công việc

**Leader: Phạm Thị Kiều Diễm**

| STT | Hạng mục công việc                                                                                   | Phụ trách |
|-----|------------------------------------------------------------------------------------------------------|----------|
| 1   | Triển khai ECS lên AWS, cấu hình CI/CD, hoàn thành user-service, auth-service                                                                                                 | Thy      |
| 2   | Code và Gọi API từ FE tới các service driver-service, trip-service                                                                                                 | Hân      |
| 3   | Code module chuyên sâu A và các serivces   `api-gateway`, `infra`                                    | Diễm     |
| 4   | Kiểm chứng (testing, verify) module chuyên sâu                                                       | Diễm     |
| 5   | Tối ưu module chuyên sâu theo kết quả kiểm chứng                                                     | Diễm     |
| 6   | Hoàn thành tài liệu báo cáo và cập nhật GitHub                                                       | Hân      |
