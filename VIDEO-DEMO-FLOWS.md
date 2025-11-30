# UITGo - Video Demo Flows & Script (Module A: Scalability & Performance)

Tài liệu này hướng dẫn chi tiết cách quay video demo cho đồ án SE360 - UITGo Module A, bao gồm flow test end-to-end và script quay video.

---

## 1. Chuẩn bị môi trường demo

### 1.1. Các repo cần clone

**Bắt buộc:**
- `infra` - Docker Compose configuration
- `gateway-service` - API Gateway
- `auth-service` - Authentication
- `user-service` - User profile management
- `trip-service` - Trip operations (gồm `trip-command-service` và `trip-query-service`)
- `driver-stream` - Driver realtime service (Go)
- `architecture` - Tài liệu (file này)

**Tùy chọn (nếu có):**
- `driver-service` - Driver profile CRUD
- `payment-service` - Payment (chưa implement)
- `notification-service` - Notification (chưa implement)
- `proto` - gRPC proto definitions

### 1.2. Build và chạy hệ thống

**Bước 1: Build Docker images**

```bash
# Từ thư mục root UITGo
cd infra
docker compose build
```

**Bước 2: Khởi động tất cả services**

```bash
docker compose up -d
```

**Bước 3: Kiểm tra services đã up**

```bash
# Kiểm tra containers
docker ps

# Kiểm tra health của từng service
curl http://localhost:3004/healthz  # Gateway
curl http://localhost:3000/healthz  # Auth (nếu có)
curl http://localhost:3001/healthz  # User
curl http://localhost:3002/healthz  # Trip Command
curl http://localhost:3003/healthz  # Trip Query
curl http://localhost:8081/healthz  # Driver Stream HCM
curl http://localhost:8082/healthz  # Driver Stream HN
```

**Expected output:** Tất cả services trả về `{"status":"ok"}` hoặc tương tự.

**Bước 4: Kiểm tra logs**

```bash
# Xem logs của tất cả services
docker compose logs -f

# Hoặc xem log của từng service
docker logs gateway-service -f
docker logs trip-command-service -f
docker logs driver-stream-hcm -f
```

### 1.3. Lấy JWT token để demo

**Endpoint login:** `POST /api/v1/sessions`

**Request sample (Postman hoặc curl):**

```bash
curl -X POST http://localhost:3004/api/v1/sessions \
  -H "Content-Type: application/json" \
  -d '{
    "email": "passenger@example.com",
    "password": "password123"
  }'
```

**Response sample:**

```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "idToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "user": {
    "email": "passenger@example.com",
    "isDriver": false,
    "isEmailVerified": true
  }
}
```

**Cách lấy token nhanh:**

1. **Postman:**
   - Tạo request `POST http://localhost:3004/api/v1/sessions`
   - Body: JSON với `email` và `password`
   - Copy `accessToken` từ response → dán vào header `Authorization: Bearer <token>` cho các request sau

2. **Terminal (jq required):**
   ```bash
   TOKEN=$(curl -s -X POST http://localhost:3004/api/v1/sessions \
     -H "Content-Type: application/json" \
     -d '{"email":"passenger@example.com","password":"password123"}' \
     | jq -r '.accessToken')
   echo $TOKEN
   ```

**Lưu ý:** 
- Nếu chưa có user, cần đăng ký trước qua `POST /api/v1/sessions` (register) hoặc seed data vào MongoDB.
- Token có thời hạn (default 1 ngày), nếu hết hạn thì login lại.

---

## 2. Flow demo cho Passenger & Driver (User stories)

### Flow P1 – Passenger đặt chuyến & xem giá cước

**User Story:** Passenger muốn xem giá cước trước khi đặt xe và tạo trip.

**Mapping:** User Story Passenger 2 - "Đặt chuyến"

**Các bước chi tiết:**

**Bước 1: Login Passenger**

- **Endpoint:** `POST /api/v1/sessions`
- **Request:**
```json
{
  "email": "passenger@example.com",
  "password": "password123"
}
```
- **Response:** Lấy `accessToken` → dùng cho các request sau

**Bước 2: Xem giá cước (Quote)**

- **Endpoint:** `POST /api/v1/trips/quote`
- **Headers:**
  - `Authorization: Bearer <accessToken>`
  - `Content-Type: application/json`
- **Request:**
```json
{
  "origin": {
    "lat": 10.8231,
    "lng": 106.6297
  },
  "destination": {
    "lat": 10.7626,
    "lng": 106.6602
  },
  "serviceType": "bike"
}
```
- **Expected Response:**
```json
{
  "distanceKm": 8.5,
  "durationMin": 17,
  "etaPickupMin": 5,
  "fare": {
    "base": 10000,
    "distance": 59500,
    "time": 8500,
    "total": 78000
  }
}
```
- **Giải thích:** Hệ thống tính khoảng cách (haversine), thời gian dự kiến, và giá cước (base + distance + time).

**Bước 3: Tạo trip**

- **Endpoint:** `POST /api/v1/trips`
- **Headers:**
  - `Authorization: Bearer <accessToken>`
  - `Content-Type: application/json`
- **Request:**
```json
{
  "origin": {
    "lat": 10.8231,
    "lng": 106.6297
  },
  "destination": {
    "lat": 10.7626,
    "lng": 106.6602
  },
  "cityCode": "HCM",
  "note": "Demo trip"
}
```
- **Expected Response:**
```json
{
  "id": "trip_abc123",
  "passengerId": "user_123",
  "status": "DRIVER_SEARCHING",
  "cityCode": "HCM",
  "originLat": 10.8231,
  "originLng": 106.6297,
  "destLat": 10.7626,
  "destLng": 106.6602,
  "quoteFareTotal": 78000,
  "quoteDistanceKm": 8.5,
  "quoteDurationMin": 17,
  "tracking": {
    "sse": "/v1/trips/trip_abc123/events"
  },
  "createdAt": "2024-01-15T10:30:00Z"
}
```

**Luồng xử lý nội bộ (show logs):**

1. **Gateway (`gateway-service`):**
   - Verify JWT → extract `userId`, `role`
   - Gắn headers: `X-User-Id: user_123`, `X-User-Role: PASSENGER`
   - Forward tới `trip-command-service:3002`

2. **Trip Command Service:**
   - Gọi gRPC `UserService.GetProfile({ user_id: "user_123" })`
   - Verify `role == 'PASSENGER'`
   - Tính quote (haversine distance)
   - Lưu Trip vào Postgres (status: `DRIVER_SEARCHING`)
   - Chọn shard driver-stream: `getDriverStreamUrl("HCM")` → `http://driver-stream-hcm:8080`
   - Gọi HTTP `GET /v1/drivers/nearby?lat=10.8231&lng=106.6297&radius=3000&limit=20`
   - Nếu có candidates → gọi `POST /v1/assign/prepare` với `{ tripId, candidates, ttlSeconds: 15 }`

3. **Driver Stream HCM:**
   - Query Redis Geo `GEORADIUS geo:drivers ...` để tìm drivers trong 3km
   - Filter chỉ drivers ONLINE
   - Lưu candidates vào Redis: `SADD trip:{tripId}:candidates {driverId1} {driverId2} ...`
   - Push SSE event `trip_offer` tới từng driver candidate

**Logs cần show trên video:**
- Gateway log: `POST /api/v1/trips - 201 - 150ms - user=user_123`
- Trip Command log: `Finding nearby drivers for trip trip_abc123 via shard HCM`
- Driver Stream log: `Found 5 nearby drivers for trip trip_abc123`

---

### Flow D1 – Driver bật Online & cập nhật vị trí

**User Story:** Driver muốn bật trạng thái online và cập nhật vị trí để nhận trip offers.

**Mapping:** User Story Driver 2 - "Bật online", Driver 4 - "Cập nhật vị trí"

**Các bước chi tiết:**

**Bước 1: Login Driver**

- **Endpoint:** `POST /api/v1/sessions`
- **Request:**
```json
{
  "email": "driver@example.com",
  "password": "password123"
}
```
- **Response:** Lấy `accessToken` (driver phải có `isDriver: true` và `role: 'DRIVER'` trong JWT)

**Bước 2: Bật trạng thái ONLINE**

- **Endpoint:** `POST /api/v1/drivers/{driverId}/status`
- **Headers:**
  - `Authorization: Bearer <accessToken>`
  - `Content-Type: application/json`
- **Request:**
```json
{
  "status": "ONLINE"
}
```
- **Expected Response:**
```json
{
  "driverId": "driver_001",
  "status": "ONLINE",
  "expiresInSec": 60
}
```

**Luồng xử lý:**
- Gateway forward tới `driver-stream-hcm:8080` (hoặc `driver-stream-hn:8080` tùy region)
- Driver Stream update Redis:
  - `HSET presence:driver:driver_001 status ONLINE last_seen {timestamp}`
  - `EXPIRE presence:driver:driver_001 60` (TTL 60s)

**Bước 3: Cập nhật vị trí (lặp lại nhiều lần)**

- **Endpoint:** `PUT /api/v1/drivers/{driverId}/location`
- **Headers:**
  - `Authorization: Bearer <accessToken>`
  - `Content-Type: application/json`
- **Request:**
```json
{
  "lat": 10.8231,
  "lng": 106.6297,
  "speed": 45.5,
  "heading": 180,
  "ts": 1705312200000
}
```
- **Expected Response:**
```json
{
  "ingested": true
}
```

**Luồng xử lý:**
1. Driver Stream update Redis Geo:
   - `GEOADD geo:drivers {lng} {lat} {driverId}`
   - `HSET presence:driver:{driverId} lat {lat} lng {lng} speed {speed} heading {heading} last_seen {timestamp}`
   - `EXPIRE presence:driver:{driverId} 60`

2. Publish event lên Kafka:
   - Topic: `driver.location.hcm` (hoặc `driver.location.hn`)
   - Payload:
   ```json
   {
     "event": "driver.location",
     "driverId": "driver_001",
     "lat": 10.8231,
     "lng": 106.6297,
     "speed": 45.5,
     "heading": 180,
     "ts": 1705312200000
   }
   ```

**Logs cần show:**
- Driver Stream log: `PUT /v1/drivers/driver_001/location - 202`
- Kafka producer log (nếu có): `Published driver.location event for driver_001`

**Demo tip:** Trong Postman, có thể dùng "Send" lặp lại nhiều lần với lat/lng khác nhau để simulate driver di chuyển.

---

### Flow D2 – Driver nhận trip offer qua SSE và accept

**User Story:** Driver nhận thông báo trip offer và accept trip.

**Mapping:** User Story Driver 3 - "Nhận trip offer", Driver 5 - "Accept trip"

**Các bước chi tiết:**

**Bước 1: Driver subscribe SSE events**

- **Endpoint:** `GET /api/v1/drivers/{driverId}/events`
- **Headers:**
  - `Authorization: Bearer <accessToken>`
- **Response:** Server-Sent Events stream

**Expected SSE messages:**

```
event: ping
data: ok

event: trip_offer
data: {"tripId":"trip_abc123","ttlSeconds":15,"candidates":["driver_001","driver_002"]}
```

**Luồng xử lý:**
- Gateway pipe SSE stream từ `driver-stream:8080/v1/drivers/{id}/events`
- Driver Stream maintain channel per driver, push `trip_offer` event khi có trip mới

**Bước 2: Passenger tạo trip (từ Flow P1)**

- Khi passenger tạo trip, driver-stream gọi `POST /v1/assign/prepare`
- Push SSE event `trip_offer` tới tất cả driver candidates
- Driver mobile app nhận được event và hiển thị thông báo

**Bước 3: Driver accept trip**

- **Endpoint:** `POST /api/v1/trips/{tripId}/accept`
- **Headers:**
  - `Authorization: Bearer <accessToken>`
  - `Content-Type: application/json`
- **Request:** (body rỗng hoặc `{}`)

**Expected Response:**
```json
{
  "ok": true
}
```

**Luồng xử lý:**

1. **Gateway:**
   - Verify JWT, check `role == 'DRIVER'`
   - Forward tới `trip-command-service:3002`

2. **Trip Command Service:**
   - Load trip từ Postgres
   - Chọn shard driver-stream dựa trên `trip.cityCode`
   - Gọi HTTP `POST /v1/assign/claim` với `{ tripId, driverId }`

3. **Driver Stream:**
   - Chạy Lua script `claim.lua` (atomic operation):
     - Kiểm tra TTL còn hạn
     - Kiểm tra chưa có ai claim
     - Kiểm tra driver có trong candidates
     - Kiểm tra driver status là ONLINE
     - Set `claimed = driverId`
   - Trả về `{ status: "ACCEPTED" }`

4. **Trip Command Service:**
   - Update Postgres: `Trip.status = 'EN_ROUTE_TO_PICKUP'`, `Trip.driverId = driverId`
   - Emit SSE event `STATUS_CHANGED` cho passenger
   - Tạo `TripEvent` type `DriverAccepted`

**Logs cần show:**
- Trip Command: `Driver driver_001 attempting to claim trip trip_abc123`
- Driver Stream: `Trip trip_abc123 successfully claimed by driver driver_001`
- Trip Command: `Trip trip_abc123 status updated to EN_ROUTE_TO_PICKUP`

---

### Flow P2 – Passenger xem trip status (với Redis cache)

**User Story:** Passenger muốn xem trạng thái trip của mình.

**Mapping:** User Story Passenger 3 - "Theo dõi trip"

**Các bước chi tiết:**

**Bước 1: Get trip by ID (lần đầu - cache miss)**

- **Endpoint:** `GET /api/v1/trips/{tripId}`
- **Headers:**
  - `Authorization: Bearer <accessToken>`

**Expected Response:**
```json
{
  "id": "trip_abc123",
  "passengerId": "user_123",
  "driverId": "driver_001",
  "status": "EN_ROUTE_TO_PICKUP",
  "originLat": 10.8231,
  "originLng": 106.6297,
  "destLat": 10.7626,
  "destLng": 106.6602,
  "quoteFareTotal": 78000,
  "rating": null,
  "events": [
    {
      "type": "DriverAccepted",
      "payload": { "driverId": "driver_001" },
      "at": "2024-01-15T10:35:00Z"
    }
  ]
}
```

**Luồng xử lý (Trip Query Service):**
1. Check Redis cache: `GET trip:{tripId}`
2. Cache miss → Query Postgres (read replica)
3. Set cache: `SET trip:{tripId} {json} EX 60` (TTL 60s)
4. Trả về trip data

**Bước 2: Get trip by ID (lần 2 - cache hit)**

- Cùng endpoint, cùng tripId
- **Luồng xử lý:**
  1. Check Redis cache: `GET trip:{tripId}`
  2. Cache hit → return ngay (không query Postgres)
  3. Latency thấp hơn nhiều (< 50ms vs ~100ms)

**Logs cần show:**
- Trip Query Service (cache miss): `Cache miss for trip trip_abc123`
- Trip Query Service (cache hit): `Cache hit for trip trip_abc123`
- Redis log (nếu có): `GET trip:trip_abc123`

**Demo tip:** 
- Request lần 1 → show latency ~100ms (cache miss)
- Request lần 2 ngay sau đó → show latency < 50ms (cache hit)
- Request sau 60s → cache expired → lại cache miss

---

## 3. Script quay video cho Module A (Scalability & Performance)

### 3.1. Phần giới thiệu (30–60 giây)

**Gợi ý nội dung để nói:**

> "Xin chào, tôi là [Tên], system architect của dự án UITGo - một hệ thống gọi xe tương tự Grab/Gojek. Hôm nay tôi sẽ demo Module A - Scalability & Performance, tập trung vào việc tối ưu hệ thống để xử lý được lượng lớn requests và đảm bảo latency thấp.
>
> UITGo được xây dựng theo kiến trúc microservices với các service chính:
> - **Gateway Service** (NestJS) - API Gateway với JWT authentication
> - **Auth Service** (NestJS) - Tích hợp AWS Cognito
> - **User Service** (NestJS + MongoDB) - Quản lý profile user
> - **Trip Command Service** (NestJS + Postgres) - Xử lý write operations
> - **Trip Query Service** (NestJS + Postgres + Redis) - Xử lý read operations với cache
> - **Driver Stream** (Go + Redis Geo + Kafka) - Quản lý driver realtime với sharding theo region
>
> Các công nghệ chính: NestJS, Go, Redis (Geo + Cache), Kafka, Postgres, MongoDB, Docker Compose.
>
> Trong video này, tôi sẽ demo:
> 1. Luồng nghiệp vụ chính (Passenger đặt xe, Driver accept)
> 2. Load test với k6 để chứng minh performance improvements
> 3. So sánh latency trước/sau khi bật Redis cache và CQRS"

**Màn hình cần bật:**
- Docker Desktop hoặc terminal với `docker ps` để show containers đang chạy
- Hoặc Postman với collection UITGo đã setup sẵn

---

### 3.2. Demo luồng nghiệp vụ chính (5–7 phút)

**Thứ tự demo:**

#### 3.2.1. Đăng ký / Login Passenger & Driver (1 phút)

**Màn hình:** Postman

**Thao tác:**
1. Show request `POST /api/v1/sessions` với body login passenger
2. Copy `accessToken` từ response
3. Tương tự login driver

**Câu thoại:**
> "Đầu tiên, tôi sẽ login passenger và driver để lấy JWT tokens. Gateway sẽ verify JWT và gắn `X-User-Id`, `X-User-Role` vào headers nội bộ trước khi forward tới backend services."

---

#### 3.2.2. Driver bật ONLINE + cập nhật vị trí (1 phút)

**Màn hình:** Postman + Docker logs (driver-stream-hcm)

**Thao tác:**
1. `POST /api/v1/drivers/driver_001/status` với `{ status: "ONLINE" }`
2. `PUT /api/v1/drivers/driver_001/location` với lat/lng (lặp lại 2-3 lần với vị trí khác nhau)

**Mở Docker logs:**
```bash
docker logs driver-stream-hcm -f
```

**Câu thoại:**
> "Bây giờ driver bật trạng thái ONLINE và cập nhật vị trí. Driver Stream service sẽ:
> - Update Redis Geo index để lưu vị trí driver
> - Publish event lên Kafka topic `driver.location.hcm` để các service khác có thể subscribe
> - Maintain presence trong Redis với TTL 60 giây (driver phải update định kỳ)
>
> Như các bạn thấy trong logs, mỗi lần update location, Redis Geo được update và Kafka event được publish."

---

#### 3.2.3. Passenger tạo trip → show logs (2 phút)

**Màn hình:** Postman + Docker logs (gateway, trip-command-service, driver-stream-hcm)

**Thao tác:**
1. `POST /api/v1/trips/quote` để xem giá
2. `POST /api/v1/trips` với origin/destination

**Mở Docker logs (3 terminals):**
```bash
# Terminal 1: Gateway
docker logs gateway-service -f

# Terminal 2: Trip Command
docker logs trip-command-service -f

# Terminal 3: Driver Stream HCM
docker logs driver-stream-hcm -f
```

**Câu thoại:**
> "Passenger tạo trip. Hãy xem luồng xử lý:
>
> **Gateway:** Nhận request, verify JWT, extract userId và role, gắn headers `X-User-Id`, `X-User-Role`, forward tới trip-command-service.
>
> **Trip Command Service:**
> - Gọi gRPC `UserService.GetProfile` để verify user là PASSENGER
> - Tính quote (khoảng cách, thời gian, giá cước)
> - Lưu Trip vào Postgres với status `DRIVER_SEARCHING`
> - Chọn shard driver-stream dựa trên `cityCode` (HCM → driver-stream-hcm:8080, HN → driver-stream-hn:8080)
> - Gọi HTTP `GET /v1/drivers/nearby` để tìm drivers trong bán kính 3km
> - Nếu có candidates, gọi `POST /v1/assign/prepare` để prepare assign
>
> **Driver Stream:**
> - Query Redis Geo `GEORADIUS` để tìm drivers gần nhất
> - Filter chỉ drivers ONLINE
> - Lưu candidates vào Redis set `trip:{tripId}:candidates`
> - Push SSE event `trip_offer` tới từng driver candidate
>
> Đây là kiến trúc microservices với inter-service communication qua gRPC và HTTP, và sharding driver-stream theo region để scale."

**Response cần highlight:**
- `id`: Trip ID
- `status`: `DRIVER_SEARCHING`
- `cityCode`: `HCM` (để show sharding)
- `tracking.sse`: URL để passenger subscribe events

---

#### 3.2.4. Driver nhận offer (SSE) và accept trip (1.5 phút)

**Màn hình:** Postman (SSE request) + Docker logs (trip-command, driver-stream)

**Thao tác:**
1. Mở tab mới trong Postman: `GET /api/v1/drivers/driver_001/events` (SSE stream)
2. Tạo trip mới từ passenger → driver nhận SSE event `trip_offer`
3. `POST /api/v1/trips/{tripId}/accept` từ driver

**Mở Docker logs:**
```bash
docker logs trip-command-service -f
docker logs driver-stream-hcm -f
```

**Câu thoại:**
> "Driver đang subscribe SSE events. Khi passenger tạo trip mới, driver-stream push event `trip_offer` qua SSE.
>
> Driver accept trip:
> - Trip Command Service gọi `POST /v1/assign/claim` tới driver-stream
> - Driver Stream chạy Lua script `claim.lua` để atomic claim:
>   - Kiểm tra TTL còn hạn
>   - Kiểm tra chưa có ai claim
>   - Kiểm tra driver có trong candidates
>   - Kiểm tra driver status là ONLINE
>   - Set `claimed = driverId` (atomic operation)
> - Trip Command update Postgres: status → `EN_ROUTE_TO_PICKUP`, gắn `driverId`
> - Emit SSE event `STATUS_CHANGED` cho passenger
>
> Lua script đảm bảo chỉ 1 driver có thể claim được trip (race condition safe)."

---

#### 3.2.5. Passenger xem trip (show Redis cache) (1 phút)

**Màn hình:** Postman + Docker logs (trip-query-service)

**Thao tác:**
1. `GET /api/v1/trips/{tripId}` lần đầu (cache miss)
2. `GET /api/v1/trips/{tripId}` lần 2 ngay sau đó (cache hit)

**Mở Docker logs:**
```bash
docker logs trip-query-service -f
```

**Câu thoại:**
> "Passenger xem trip. Trip Query Service sử dụng Redis cache:
> - Lần 1: Cache miss → query Postgres → set cache với TTL 60s → latency ~100ms
> - Lần 2: Cache hit → return từ Redis → latency < 50ms
>
> Đây là CQRS pattern: Trip Command Service xử lý writes, Trip Query Service xử lý reads với cache để tối ưu performance."

---

### 3.3. Demo load test & tuning (3–5 phút)

**Màn hình:** Terminal với k6 output

**Các k6 scripts có sẵn:**

#### 3.3.1. Scenario 1: Create Trip Load Test

**File:** `infra/k6/trips_create.js`

**Câu lệnh:**
```bash
cd infra/k6
k6 run --env TOKEN=<your_jwt_token> --env BASE_URL=http://localhost:3004 trips_create.js
```

**Config:**
- VUs: 30 (default)
- Duration: 1m (default)
- Threshold: p95 < 200ms

**Endpoint target:** `POST /api/v1/trips` qua gateway

**Ý nghĩa:**
- Test luồng Create Trip + Find Driver với nhiều requests đồng thời
- Kiểm chứng hệ thống có thể xử lý nhiều passengers đặt xe cùng lúc
- Verify latency của luồng phức tạp (gRPC call, DB write, HTTP call tới driver-stream)

**Câu thoại:**
> "Bây giờ tôi sẽ chạy load test cho luồng Create Trip. Script này simulate 30 virtual users tạo trip trong 1 phút.
>
> Endpoint target: `POST /api/v1/trips` qua gateway.
>
> Mục tiêu: p95 latency < 200ms.
>
> Đây là luồng phức tạp bao gồm:
> - Gateway verify JWT
> - Trip Command gọi gRPC UserService
> - Lưu DB Postgres
> - Gọi driver-stream để tìm drivers
> - Driver-stream query Redis Geo
>
> Kết quả sẽ cho thấy hệ thống có thể scale để xử lý nhiều requests đồng thời."

---

#### 3.3.2. Scenario 2: Driver Location Update Throughput

**File:** `infra/k6/drivers_update_location.js`

**Câu lệnh:**
```bash
k6 run --env TOKEN=<your_jwt_token> --env BASE_URL=http://localhost:3004 drivers_update_location.js
```

**Config:**
- VUs: 50 (default)
- Duration: 1m (default)
- Threshold: p95 < 50ms

**Endpoint target:** `PUT /api/v1/drivers/:id/location` qua gateway

**Ý nghĩa:**
- Test throughput của location updates (drivers cập nhật vị trí rất thường xuyên)
- Kiểm chứng driver-stream + Redis Geo có thể xử lý high-frequency updates
- Verify latency thấp (< 50ms) để không block driver app

**Câu thoại:**
> "Load test cho Driver Location Update. Script này simulate 50 drivers cập nhật vị trí mỗi 0.5 giây.
>
> Endpoint target: `PUT /api/v1/drivers/:id/location`.
>
> Mục tiêu: p95 latency < 50ms (rất thấp vì đây là high-frequency operation).
>
> Driver Stream service phải:
> - Update Redis Geo index (atomic operation)
> - Publish Kafka event
> - Maintain presence trong Redis
>
> Kết quả sẽ cho thấy Redis Geo có thể xử lý được throughput cao."

---

#### 3.3.3. Scenario 3: GET Trip với Cache (Before/After)

**File:** 
- `infra/k6/trips_read_cached_before.js` (không có cache)
- `infra/k6/trips_read_cached_after.js` (có cache)

**Câu lệnh:**

**Before (không cache):**
```bash
# Đảm bảo Redis cache bị disable trong trip-query-service
k6 run --env TOKEN=<token> --env TRIP_ID=<trip_id> trips_read_cached_before.js
```

**After (có cache):**
```bash
# Đảm bảo Redis cache được enable
k6 run --env TOKEN=<token> --env TRIP_ID=<trip_id> trips_read_cached_after.js
```

**Config:**
- VUs: 50 (default)
- Duration: 1m (default)
- Threshold: 
  - Before: p95 < 100ms (expect higher)
  - After: p95 < 50ms (expect lower)

**Endpoint target:** `GET /api/v1/trips/:id` qua gateway → trip-query-service

**Ý nghĩa:**
- So sánh latency trước/sau khi bật Redis cache
- Chứng minh cache giảm load lên Postgres và cải thiện latency
- Verify CQRS pattern (trip-query-service tách riêng với cache)

**Câu thoại:**
> "Load test so sánh GET Trip trước/sau khi bật Redis cache.
>
> **Before (không cache):**
> - Mọi request đều query Postgres
> - Expected p95 latency ~100ms
> - High load lên database
>
> **After (có cache):**
> - Request đầu tiên query Postgres và set cache
> - Các request sau hit cache từ Redis
> - Expected p95 latency < 50ms
> - Giảm load lên Postgres đáng kể
>
> Đây là lợi ích của CQRS pattern: tách read service riêng với cache để optimize read-heavy workloads."

**Demo tip:**
- Chạy before → ghi lại metrics (p95, p99, requests/sec)
- Bật cache → chạy after → so sánh metrics
- Show improvement: latency giảm ~50%, throughput tăng

---

### 3.4. Kết luận video (1 phút)

**Gợi ý lời kết:**

> "Tóm lại, trong Module A - Scalability & Performance, tôi đã implement:
>
> 1. **Microservices Architecture + API Gateway:**
>    - Tách services theo domain (Auth, User, Trip, Driver)
>    - Gateway tập trung authentication và routing
>    - Inter-service communication qua gRPC và HTTP
>
> 2. **CQRS Pattern:**
>    - Trip Command Service: xử lý writes (create, update, cancel)
>    - Trip Query Service: xử lý reads với Redis cache
>    - Tách read/write để optimize performance
>
> 3. **Redis Geo + Kafka:**
>    - Driver Stream sử dụng Redis Geo để query drivers gần nhất
>    - Publish events lên Kafka để các service khác subscribe
>    - High throughput, low latency
>
> 4. **Sharding by Region:**
>    - Driver Stream được shard theo region (HCM, HN)
>    - Mỗi region có Redis instance riêng
>    - Scale horizontally khi mở rộng sang nhiều region
>
> **Trade-offs đã chấp nhận:**
> - Complexity: nhiều services, nhiều databases, nhiều protocols
> - Eventual consistency: cache có thể stale trong 60s
> - Operational overhead: cần monitor nhiều services
>
> **Nhưng đổi lại:**
> - Scalability: có thể scale từng service độc lập
> - Performance: latency thấp nhờ cache và sharding
> - Availability: một service down không ảnh hưởng toàn bộ hệ thống
>
> UITGo là một portfolio-defining project cho thấy tôi có thể thiết kế và implement hệ thống distributed phức tạp.
>
> **Hướng phát triển tiếp:**
> - Multi-region deployment thật (AWS multi-AZ)
> - Read replicas cho Postgres
> - DynamoDB cho driver location (thay Redis Geo nếu cần scale hơn nữa)
> - Event-driven architecture với Kafka streams
> - Monitoring và observability (Prometheus, Grafana)
>
> Cảm ơn các bạn đã xem!"

---

## 4. Checklist khi quay video

### 4.1. Trước khi bắt đầu quay

**Docker & Services:**
- [ ] Tất cả containers đã up: `docker ps` (check gateway, auth, user, trip-command, trip-query, driver-stream-hcm, driver-stream-hn, postgres, mongo, redis, kafka)
- [ ] Tất cả services healthy: test healthcheck URLs
- [ ] Logs sạch (hoặc đã clear logs cũ)

**Postman:**
- [ ] Collection UITGo đã setup sẵn với:
  - [ ] Request login passenger
  - [ ] Request login driver
  - [ ] Request create trip
  - [ ] Request get trip
  - [ ] Request driver status
  - [ ] Request driver location
  - [ ] Request driver events (SSE)
  - [ ] Request accept trip
- [ ] JWT tokens đã lấy sẵn (hoặc biết cách lấy nhanh)
- [ ] Environment variables đã set: `BASE_URL=http://localhost:3004`

**Terminal Windows:**
- [ ] Terminal 1: `docker logs gateway-service -f`
- [ ] Terminal 2: `docker logs trip-command-service -f`
- [ ] Terminal 3: `docker logs trip-query-service -f`
- [ ] Terminal 4: `docker logs driver-stream-hcm -f`
- [ ] Terminal 5: `docker logs driver-stream-hn -f` (nếu demo HN)
- [ ] Terminal 6: Sẵn sàng chạy k6 scripts

**k6 Scripts:**
- [ ] Đã update JWT token trong scripts (hoặc dùng `--env TOKEN=...`)
- [ ] Đã có trip ID sẵn cho read test (hoặc biết cách tạo nhanh)
- [ ] Đã test chạy thử 1 script để đảm bảo không lỗi

**Database:**
- [ ] Có ít nhất 1 user passenger trong MongoDB (hoặc biết cách đăng ký nhanh)
- [ ] Có ít nhất 1 user driver trong MongoDB
- [ ] Users đã verify email (hoặc bypass verify trong code)

**Redis:**
- [ ] Redis đang chạy và accessible
- [ ] Có thể connect: `docker exec -it redis redis-cli ping`

**Kafka:**
- [ ] Kafka đang chạy
- [ ] Topics có thể tạo được (auto-create)

### 4.2. Trong khi quay

**Màn hình cần bật:**
- [ ] Postman (collection UITGo)
- [ ] Docker logs terminals (ít nhất 3-4 terminals)
- [ ] Terminal để chạy k6 (khi đến phần load test)
- [ ] Browser hoặc DB viewer (nếu cần show data trong DB)

**Audio:**
- [ ] Microphone test (không bị nhiễu)
- [ ] Screen recording software đã bật (OBS, QuickTime, etc.)

**Timing:**
- [ ] Đã practice flow trước để biết timing
- [ ] Có script nói sẵn (file này) để không bị quên

### 4.3. Sau khi quay

**Kiểm tra:**
- [ ] Video đã record đầy đủ
- [ ] Audio rõ ràng
- [ ] Màn hình không bị mờ
- [ ] Logs hiển thị rõ trong video

**Backup:**
- [ ] Save video file
- [ ] Save script này (đã update nếu có)

---

## 5. Troubleshooting

### 5.1. Services không start

**Lỗi:** `docker compose up` bị lỗi

**Giải pháp:**
- Check ports đã được sử dụng: `lsof -i :3004` (macOS/Linux) hoặc `netstat -ano | findstr :3004` (Windows)
- Check Docker resources: `docker stats`
- Xem logs chi tiết: `docker compose logs <service-name>`

### 5.2. JWT token không work

**Lỗi:** `401 Unauthorized`

**Giải pháp:**
- Token đã hết hạn → login lại
- Token format sai → check có `Bearer ` prefix
- JWT_SECRET không match giữa gateway và auth-service → check env variables

### 5.3. Driver không nhận được trip offer

**Lỗi:** SSE không có event `trip_offer`

**Giải pháp:**
- Check driver status là ONLINE: `GET /api/v1/drivers/{id}/presence`
- Check driver location đã update: Redis Geo có driver không
- Check trip cityCode match với driver-stream shard
- Check driver-stream logs: có gọi `/v1/assign/prepare` không

### 5.4. k6 script fail

**Lỗi:** `401` hoặc `500` errors

**Giải pháp:**
- Check JWT token đã update trong script
- Check BASE_URL đúng: `http://localhost:3004`
- Check services đang chạy: `docker ps`
- Check gateway logs: có nhận request không

### 5.5. Redis cache không work

**Lỗi:** Trip Query Service không cache

**Giải pháp:**
- Check Redis connection: `REDIS_URL` env variable
- Check RedisModule đã import trong `trip-query-service`
- Check RedisService đã inject và sử dụng
- Check Redis đang chạy: `docker ps | grep redis`

---

## 6. Tài liệu tham khảo

- **Service Flows chi tiết:** `architecture/SERVICE-FLOWS.md`
- **k6 Scripts:** `infra/k6/README.md`
- **Docker Compose:** `infra/docker-compose.yml`
- **Architecture ADRs:** `architecture/ADR/`

---

**Chúc bạn quay video thành công! 🎬**

