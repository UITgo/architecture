# UITGo - Chi Tiết Luồng Logic Các Microservice

Tài liệu này mô tả **chi tiết** luồng logic của từng microservice trong hệ thống UITGo, dựa trực tiếp trên code hiện tại.

---

## 1. Gateway Service – API Gateway

### 1.1. Trách nhiệm chính

- Expose API endpoint `/api/v1/...` cho client
- Verify JWT token từ request header `Authorization: Bearer <token>`
- Extract `userId` và `role` từ JWT payload và gắn vào headers nội bộ (`X-User-Id`, `X-User-Role`)
- Route request tới các backend service tương ứng (auth-service, user-service, trip-service, driver-stream)
- Logging request với `X-Request-Id` để trace

### 1.2. Luồng logic chi tiết

#### 1.2.1. Luồng xử lý request nói chung

**File:** `gateway-service/src/main.ts`, `gateway-service/src/app.module.ts`

**Bước 1:** Request từ client đến gateway (port 3004, prefix `/api/v1`)

**Bước 2:** Middleware logging (`main.ts:19-34`)
- Tạo `X-Request-Id` (UUID) nếu chưa có trong header
- Gắn `X-Request-Id` vào response header
- Log request sau khi hoàn thành: `method`, `url`, `statusCode`, `duration`, `userId`, `requestId`

**Bước 3:** JWT Authentication Guard (`jwt-auth.guard.ts`)
- `JwtAuthGuard` được đăng ký global qua `APP_GUARD` trong `app.module.ts:35-37`
- Nếu controller/handler có decorator `@Public()`, bỏ qua authentication
- Nếu không có `@Public()`, gọi `JwtStrategy.validate()` để verify JWT

**Bước 4:** JWT Strategy (`jwt.strategy.ts`)
- Extract JWT từ header `Authorization: Bearer <token>` (dùng `ExtractJwt.fromAuthHeaderAsBearerToken()`)
- Verify JWT với secret từ `JWT_SECRET` env (hoặc default `'dev-secret'`)
- Parse payload và extract:
  - `userId = payload.sub` (subject claim)
  - `role = payload.role`
- Trả về object `{ userId, role }` → được gắn vào `req.user`

**Bước 5:** Controller nhận request với `req.user` đã có `userId` và `role`
- Các controller proxy gắn headers nội bộ:
  - `X-User-Id: req.user.userId`
  - `X-User-Role: req.user.role`
- Forward request tới backend service tương ứng

#### 1.2.2. Proxy từng nhóm endpoint

**File:** `gateway-service/src/auth-proxy.controller.ts`

**Endpoint:** `POST /api/v1/sessions` (login)
- **Public:** Có `@Public()` → không cần JWT
- **Proxy:** `POST http://auth-service:3000/v1/auth/login`
- **Method:** `AuthProxyController.login()`
- **Body:** Forward nguyên body từ client

---

**File:** `gateway-service/src/users-proxy.controller.ts`

**Endpoint:** `POST /api/v1/users` (tạo user)
- **Public:** Có `@Public()` → không cần JWT
- **Proxy:** `POST http://user-service:3001/v1/users`
- **Method:** `UsersProxyController.create()`

**Endpoint:** `GET /api/v1/users/me` (lấy profile của user hiện tại)
- **Protected:** Cần JWT
- **Proxy:** `GET http://user-service:3001/v1/users/me`
- **Method:** `UsersProxyController.me()`
- **Headers nội bộ:**
  - `X-User-Id: req.user.userId`
  - `X-User-Role: req.user.role`

---

**File:** `gateway-service/src/trips-proxy.controller.ts`

**Write Operations → trip-command-service:**

- **`POST /api/v1/trips/quote`**
  - **Protected:** Cần JWT
  - **Proxy:** `POST http://trip-command-service:3002/v1/trips/quote`
  - **Method:** `TripsProxyController.quote()`

- **`POST /api/v1/trips`** (tạo trip)
  - **Protected:** Cần JWT
  - **Proxy:** `POST http://trip-command-service:3002/v1/trips`
  - **Method:** `TripsProxyController.create()`
  - **Headers:** `X-User-Id`, `X-User-Role`

- **`POST /api/v1/trips/:tripId/cancel`**
  - **Proxy:** `POST http://trip-command-service:3002/v1/trips/:tripId/cancel`
  - **Headers:** `X-User-Id`, `X-User-Role`

- **`POST /api/v1/trips/:tripId/accept`**
  - **Proxy:** `POST http://trip-command-service:3002/v1/trips/:tripId/accept`
  - **Headers:** `X-User-Id`, `X-User-Role`

- **`POST /api/v1/trips/:tripId/decline`**
  - **Proxy:** `POST http://trip-command-service:3002/v1/trips/:tripId/decline`

- **`POST /api/v1/trips/:tripId/arrive-pickup`**
  - **Proxy:** `POST http://trip-command-service:3002/v1/trips/:tripId/arrive-pickup`

- **`POST /api/v1/trips/:tripId/start`**
  - **Proxy:** `POST http://trip-command-service:3002/v1/trips/:tripId/start`

- **`POST /api/v1/trips/:tripId/finish`**
  - **Proxy:** `POST http://trip-command-service:3002/v1/trips/:tripId/finish`

**Read Operations → trip-query-service:**

- **`GET /api/v1/trips/:tripId`**
  - **Proxy:** `GET http://trip-query-service:3003/v1/trips/:tripId`
  - **Method:** `TripsProxyController.get()`
  - **Headers:** `X-User-Id`, `X-User-Role`

- **`GET /api/v1/trips/users/:userId/trips`**
  - **Proxy:** `GET http://trip-query-service:3003/v1/trips/users/:userId/trips`
  - **Query params:** `status`, `limit`, `offset`
  - **Method:** `TripsProxyController.getUserTrips()`

---

**File:** `gateway-service/src/drivers-proxy.controller.ts`

- **`GET /api/v1/drivers/nearby`**
  - **Proxy:** `GET http://driver-stream:8080/v1/drivers/nearby`
  - **Query params:** `lat`, `lng`, `radius`, `limit`
  - **Method:** `DriversProxyController.nearby()`

- **`POST /api/v1/drivers/:id/status`**
  - **Proxy:** `POST http://driver-stream:8080/v1/drivers/:id/status`
  - **Body:** `{ status: "ONLINE" | "OFFLINE" }`
  - **Method:** `DriversProxyController.status()`

- **`PUT /api/v1/drivers/:id/location`**
  - **Proxy:** `PUT http://driver-stream:8080/v1/drivers/:id/location`
  - **Body:** `{ lat, lng, speed?, heading?, ts? }`
  - **Method:** `DriversProxyController.location()`

- **`GET /api/v1/drivers/:id/events`** (SSE stream)
  - **Proxy:** `GET http://driver-stream:8080/v1/drivers/:id/events`
  - **Response:** Server-Sent Events (text/event-stream)
  - **Method:** `DriversProxyController.events()` - pipe stream từ driver-stream về client

---

## 2. Auth Service

### 2.1. Trách nhiệm chính

- Đăng ký user (register) với AWS Cognito
- Xác thực OTP (verify email)
- Đăng nhập (login) và trả về JWT tokens (accessToken, idToken, refreshToken)
- Refresh token để lấy access token mới
- Tích hợp với AWS Cognito User Pool (identity-js SDK và AWS SDK v3)

### 2.2. Luồng logic chính

**File:** `auth-service/src/auth/auth.controller.ts`, `auth-service/src/auth/auth.service.ts`

#### 2.2.1. Luồng Register (`POST /v1/auth/register`)

**Method:** `AuthController.register()` → `AuthService.register()`

**Bước 1:** Nhận input từ client (`RegisterDto`: `email`, `phone`, `password`, `isDriver`)

**Bước 2:** Kiểm tra email đã tồn tại trong MongoDB (`userModel.findOne({ email })`)

**Bước 3:** Tạo user trong Cognito User Pool:
- Dùng `CognitoUserPool.signUp()` từ `amazon-cognito-identity-js`
- Attributes: `email`, `phone_number` (nếu có)
- Callback nhận `result.userSub` (Cognito user ID)

**Bước 4:** Lưu user vào MongoDB:
- Hash password bằng `bcrypt.hash(password, 10)`
- Lưu `cognitoSub` (từ `result.userSub`) để đồng bộ ID
- Set `isEmailVerified: false`, `isDriver`

**Bước 5:** Trả về response:
```json
{
  "message": "User registered successfully. Please check your email for verification code.",
  "email": "...",
  "needVerify": true
}
```

---

#### 2.2.2. Luồng Verify OTP (`POST /v1/auth/verify-otp`)

**Method:** `AuthController.verifyOtp()` → `AuthService.verifyOtp()`

**Bước 1:** Nhận input (`VerifyOtpDto`: `email`, `otpcode`)

**Bước 2:** Tìm user trong MongoDB theo email

**Bước 3:** Xác thực OTP với Cognito:
- Tạo `CognitoUser` với `Username: email`
- Gọi `cognitoUser.confirmRegistration(otpcode, true, callback)`

**Bước 4:** Nếu OTP đúng:
- Update `isEmailVerified: true` trong MongoDB
- **Nếu `user.isDriver == true`:** Gọi AWS SDK v3 `AdminAddUserToGroupCommand` để add user vào group `'driver'` trong Cognito (để JWT token có claim `cognito:groups: ['driver']`)

**Bước 5:** Trả về `{ message: 'Email verified successfully', verified: true }`

---

#### 2.2.3. Luồng Login (`POST /v1/auth/login`)

**Method:** `AuthController.login()` → `AuthService.login()`

**Bước 1:** Nhận input (`LoginDto`: `email`, `password`)

**Bước 2:** Kiểm tra user trong MongoDB:
- Tìm user theo email
- Kiểm tra `isEmailVerified == true` (nếu chưa verify thì throw `UnauthorizedException`)

**Bước 3:** Authenticate với Cognito:
- Tạo `CognitoUser` và `AuthenticationDetails`
- Gọi `cognitoUser.authenticateUser(authDetails, callbacks)`

**Bước 4:** Callback `onSuccess` nhận tokens:
- `accessToken = result.getAccessToken().getJwtToken()`
- `idToken = result.getIdToken().getJwtToken()`
- `refreshToken = result.getRefreshToken().getToken()`

**Bước 5:** Trả về response:
```json
{
  "accessToken": "...",
  "idToken": "...",
  "refreshToken": "...",
  "user": {
    "email": "...",
    "isDriver": true/false,
    "isEmailVerified": true
  }
}
```

**JWT Claims quan trọng:**
- `sub`: User ID (Cognito sub hoặc username)
- `email`: Email của user
- `cognito:username`: Username trong Cognito
- `cognito:groups`: Array các group (ví dụ: `['driver']` nếu là tài xế)
- `role`: Role được map từ `cognito:groups` hoặc từ DB (tùy implementation)

---

#### 2.2.4. Luồng Refresh Token (`POST /v1/auth/refresh`)

**Method:** `AuthController.refresh()` → `AuthService.refreshToken()`

**Bước 1:** Nhận input: `{ refreshToken: string, email: string }`

**Bước 2:** Tạo `CognitoUser` và `CognitoRefreshToken`

**Bước 3:** Gọi `cognitoUser.refreshSession(RefreshToken, callback)`

**Bước 4:** Callback nhận session mới:
- `newAccessToken = session.getAccessToken().getJwtToken()`
- `newIdToken = session.getIdToken().getJwtToken()`

**Bước 5:** Trả về:
```json
{
  "accessToken": "...",
  "idToken": "..."
}
```

**Lưu ý:** Có thêm method `refreshTokenV2()` dùng AWS SDK v3 `InitiateAuthCommand` với `AuthFlow: 'REFRESH_TOKEN_AUTH'` (có thể dùng thay thế).

---

#### 2.2.5. Luồng Get Me (`GET /v1/auth/me`)

**Method:** `AuthController.me()` → `AuthService.me()`

**Bước 1:** Nhận header `Authorization: Bearer <token>`

**Bước 2:** Decode JWT (không verify, vì đã verify ở gateway):
- Dùng `jwtDecode<JWTPayload>(token)`

**Bước 3:** Trả về claims:
```json
{
  "sub": "...",
  "email": "...",
  "groups": ["driver"] // từ cognito:groups
}
```

---

## 3. User Service

### 3.1. Trách nhiệm chính

- Quản lý hồ sơ user (passenger/driver) trên MongoDB
- Cung cấp REST API: tạo user, update avatar, lấy profile
- Cung cấp gRPC service `GetProfile` cho các service khác (trip-command-service) gọi để verify user và lấy role

### 3.2. Luồng logic

**File:** `user-service/src/user/user.controller.ts`, `user-service/src/user/user.service.ts`, `user-service/src/user/user.grpc.controller.ts`

#### 3.2.1. REST Endpoints

**`POST /v1/users`** (tạo user)
- **Method:** `UserController.create()` → `UserService.createuser()`
- **Input:** `CreateUserDto`: `{ authId, fullname, role: 'PASSENGER' | 'DRIVER' }`
- **Logic:**
  - Kiểm tra `authId` đã tồn tại chưa (nếu có thì return existing)
  - Tạo user mới trong MongoDB với schema:
    - `authId`: ID từ auth-service (Cognito sub)
    - `fullname`: Tên đầy đủ
    - `role`: `'PASSENGER'` hoặc `'DRIVER'`
    - `avatar`: Mặc định `''`
- **Output:** User document từ MongoDB

**`GET /v1/users/me`** (lấy profile của user hiện tại)
- **Method:** `UserController.me()`
- **Headers:** `X-User-Id` (do gateway set từ JWT)
- **Logic:**
  - Đọc `X-User-Id` từ header
  - Gọi `UserService.getUserbyId(authId)`
  - Trả về user document
- **Output:** User document

**`GET /v1/users/:authId`** (lấy user theo authId)
- **Method:** `UserController.findOne()` → `UserService.getUserbyId()`
- **Logic:** Query MongoDB `findOne({ authId })`
- **Error:** Throw `NotFoundException` nếu không tìm thấy

**`PATCH /v1/users/:authId/avatar`** (upload avatar)
- **Method:** `UserController.updateAvatar()` → `UserService.uploadAvatar()`
- **Input:** Multipart file upload
- **Logic:**
  - Upload file lên S3 (qua `S3Service.uploadfile()`)
  - Update MongoDB: `findOneAndUpdate({ authId }, { avatar: url })`
- **Output:** `{ avatar: url }`

#### 3.2.2. gRPC Service

**Service Name:** `UserService`

**Method:** `GetProfile(GetProfileRequest) returns (GetProfileResponse)`

**File:** `user-service/src/user/user.grpc.controller.ts`

**Proto Definition:** `proto/user.proto`
```protobuf
service UserService {
  rpc GetProfile (GetProfileRequest) returns (GetProfileResponse);
}

message GetProfileRequest {
  string user_id = 1;
}

message GetProfileResponse {
  bool exists = 1;
  string user_id = 2;
  string name = 3;
  string avatar_url = 4;
  string role = 5; // 'PASSENGER' | 'DRIVER'
}
```

**Luồng xử lý:**

**Bước 1:** Nhận request với `user_id` (tương đương `authId`)

**Bước 2:** Gọi `UserService.getUserbyId(user_id)` để query MongoDB

**Bước 3:** Nếu không tìm thấy, trả về:
```json
{
  "exists": false,
  "user_id": "...",
  "name": "",
  "avatar_url": "",
  "role": ""
}
```

**Bước 4:** Nếu tìm thấy, trả về:
```json
{
  "exists": true,
  "user_id": user.authId,
  "name": user.fullname,
  "avatar_url": user.avatar || "",
  "role": user.role // 'PASSENGER' hoặc 'DRIVER'
}
```

**Sử dụng bởi:**
- `trip-command-service`: Gọi `GetProfile` để verify user là `PASSENGER` trước khi tạo trip

#### 3.2.3. Phân biệt role

**Schema:** `user-service/src/user/user.schema.ts`

- Field `role` là enum: `'PASSENGER' | 'DRIVER'`
- Được set khi tạo user (từ `CreateUserDto`)
- Được dùng để:
  - Verify quyền (ví dụ: chỉ `PASSENGER` mới tạo được trip)
  - Filter/query (ví dụ: lấy danh sách driver)

---

## 4. Trip Service – Command & Query (CQRS)

Hệ thống đã tách CQRS thành 2 service riêng biệt:

### 4.1. Trip Command Service (trip-command-service)

**Trách nhiệm:** Xử lý tất cả write operations (create, update, cancel, accept, finish trip)

**File:** `trip-service/trip-command-service/src/trips/trips.controller.ts`, `trip-service/trip-command-service/src/trips/trips.service.ts`

#### 4.1.1. Luồng Create Trip (`POST /v1/trips`)

**Method:** `TripsController.create()` → `TripsService.create()`

**Bước 1:** Nhận request từ gateway
- Headers: `X-User-Id` (passengerId), `X-User-Role`
- Body: `CreateTripDto`: `{ origin: {lat, lng}, destination: {lat, lng}, note?, cityCode? }`

**Bước 2:** Verify user qua gRPC
- Gọi `UserService.GetProfile({ user_id: passengerId })` qua gRPC client
- Kiểm tra:
  - `prof.exists == true` (user tồn tại)
  - `prof.role == 'PASSENGER'` (chỉ passenger mới tạo được trip)
- Nếu không thỏa → throw `ForbiddenException` hoặc `BadRequestException`

**Bước 3:** Tính quote (fare)
- Gọi `TripsService.quote()` nội bộ:
  - Tính khoảng cách bằng `haversine(origin, destination)` (km)
  - Tính duration: `Math.ceil((km / 30) * 60)` (phút, giả sử tốc độ 30km/h)
  - Tính fare:
    - `base = 10000`
    - `distance = Math.ceil(km * 7000)`
    - `time = duration * 500`
    - `total = base + distance + time`

**Bước 4:** Xác định `cityCode`
- Lấy từ `dto.cityCode` hoặc default `'HCM'`
- Dùng để chọn shard driver-stream (HCM → driver-stream-hcm, HN → driver-stream-hn)

**Bước 5:** Tạo trip trong Postgres (Prisma)
- Insert vào bảng `Trip`:
  - `passengerId`, `originLat`, `originLng`, `destLat`, `destLng`
  - `note`, `cityCode`
  - `status: 'DRIVER_SEARCHING'`
  - `quoteDistanceKm`, `quoteDurationMin`, `quoteFareTotal`
- Database: `PRIMARY_DB_URL` (primary database cho writes)

**Bước 6:** Emit SSE event
- Gọi `emitEvent(trip.id, 'TRIP_CREATED', { id, status })` để notify client qua SSE

**Bước 7:** Tìm driver gần nhất và prepare assign
- **Chọn shard driver-stream:**
  - Gọi `getDriverStreamUrl(trip.cityCode)` từ `region-shard.config.ts`
  - Map: `HCM` → `http://driver-stream-hcm:8080`, `HN` → `http://driver-stream-hn:8080`

- **Gọi HTTP `GET /v1/drivers/nearby`:**
  - URL: `${driverStreamBaseUrl}/v1/drivers/nearby`
  - Params: `lat`, `lng`, `radius: 3000` (3km), `limit: 20`
  - Response: `{ drivers: [{ driverId, distance, lat, lng }] }`

- **Nếu có candidates:**
  - Gọi HTTP `POST /v1/assign/prepare`:
    - URL: `${driverStreamBaseUrl}/v1/assign/prepare`
    - Body: `{ tripId, candidates: [driverId1, driverId2, ...], ttlSeconds: 15 }`
    - Driver-stream sẽ:
      - Lưu candidates vào Redis set `trip:{tripId}:candidates`
      - Set deadline `trip:{tripId}:deadline`
      - Push SSE event `trip_offer` tới từng driver candidate

  - Lưu `TripAssignment` vào Postgres:
    - Insert nhiều records: `{ tripId, driverId, state: 'INVITED', ttlSec: 15 }`

  - Tạo `TripEvent`: `{ tripId, type: 'DriverSearchStarted', payload: { candidates } }`

**Bước 8:** Trả về trip
```json
{
  "id": "...",
  "passengerId": "...",
  "status": "DRIVER_SEARCHING",
  "cityCode": "HCM",
  "quoteFareTotal": 50000,
  "tracking": {
    "sse": "/v1/trips/{tripId}/events"
  }
}
```

---

#### 4.1.2. Luồng Accept Trip (`POST /v1/trips/:tripId/accept`)

**Method:** `TripsController.accept()` → `TripsService.accept()`

**Bước 1:** Nhận request từ gateway
- Headers: `X-User-Id` (driverId), `X-User-Role`
- Kiểm tra `role == 'DRIVER'` → throw `ForbiddenException` nếu không phải driver

**Bước 2:** Load trip từ Postgres
- Query `Trip.findUnique({ where: { id: tripId } })`
- Kiểm tra `status` phải là `'DRIVER_SEARCHING'` hoặc `'DRIVER_ASSIGNED'`

**Bước 3:** Chọn shard driver-stream
- Gọi `getDriverStreamUrl(trip.cityCode)` để lấy base URL

**Bước 4:** Claim trip qua driver-stream
- Gọi HTTP `POST /v1/assign/claim`:
  - URL: `${driverStreamBaseUrl}/v1/assign/claim`
  - Body: `{ tripId, driverId }`
- Driver-stream sẽ chạy Lua script `claim.lua` để atomic claim:
  - Kiểm tra TTL còn hạn
  - Kiểm tra chưa có ai claim (`claimed == "0"`)
  - Kiểm tra driver có trong candidates set
  - Kiểm tra driver status là `ONLINE`
  - Set `claimed = driverId`
- Response: `{ status: 'ACCEPTED' }` hoặc `{ status: 'EXPIRED' | 'ALREADY_CLAIMED' | 'NOT_CANDIDATE' | 'DRIVER_OFFLINE' }`

**Bước 5:** Nếu claim thành công (`status == 'ACCEPTED'`):
- Update `TripAssignment`: `{ tripId, driverId, state: 'CLAIMED', respondedAt }`
- Update `Trip`: `{ status: 'EN_ROUTE_TO_PICKUP', driverId }`
- Emit SSE event: `STATUS_CHANGED`
- Tạo `TripEvent`: `{ type: 'DriverAccepted', payload: { driverId } }`

**Bước 6:** Nếu claim thất bại:
- Update `TripAssignment`: `{ state: 'DECLINED' }`
- Tạo `TripEvent`: `{ type: 'DriverDeclined' }`
- Trả về `{ ok: false, reason: status }`

**Bước 7:** Trả về `{ ok: true }`

---

#### 4.1.3. Các luồng update trạng thái khác

**`POST /v1/trips/:tripId/cancel`**
- Method: `TripsService.cancel()`
- Logic:
  - Load trip, kiểm tra status chưa terminal (`COMPLETED`, `CANCELED`)
  - Update: `{ status: 'CANCELED', canceledAt, cancelReasonCode }`
  - Emit SSE event
  - Tạo `TripEvent`

**`POST /v1/trips/:tripId/arrive-pickup`**
- Method: `TripsService.arrive()`
- Update: `{ status: 'ARRIVED' }`

**`POST /v1/trips/:tripId/start`**
- Method: `TripsService.start()`
- Update: `{ status: 'IN_TRIP' }`

**`POST /v1/trips/:tripId/finish`**
- Method: `TripsService.finish()`
- Input: `{ actualDistanceKm, actualDurationMin }`
- Logic:
  - Kiểm tra `status == 'IN_TRIP'`
  - Tính `finalFareTotal = Math.max(10000, quoteFareTotal)`
  - Update: `{ status: 'COMPLETED', actualDistanceKm, actualDurationMin, finalFareTotal }`
  - Emit SSE event
  - Tạo `TripEvent`

**`POST /v1/trips/:tripId/rate`**
- Method: `TripsService.rate()`
- Input: `{ stars, comment? }`
- Logic:
  - Kiểm tra `status == 'COMPLETED'`
  - Upsert `TripRating`: `{ tripId, raterId, driverId, stars, comment }`
  - Tạo `TripEvent`

---

### 4.2. Trip Query Service (trip-query-service)

**Trách nhiệm:** Xử lý tất cả read operations với Redis cache

**File:** `trip-service/trip-query-service/src/trips/trips.controller.ts`, `trip-service/trip-query-service/src/trips/trips.service.ts`

#### 4.2.1. Luồng Get Trip by ID (`GET /v1/trips/:tripId`)

**Method:** `TripsController.get()` → `TripsService.getTripById()`

**Bước 1:** Nhận request từ gateway
- Headers: `X-User-Id`, `X-User-Role` (có thể dùng để check permission)

**Bước 2:** Check Redis cache
- Key: `trip:{tripId}`
- Gọi `RedisService.get(cacheKey)`
- Nếu có data → parse JSON và return ngay (cache hit)

**Bước 3:** Cache miss → Query Postgres
- Database: `READ_DB_URL` (read replica trong production)
- Query: `Trip.findUnique({ where: { id: tripId }, include: { rating: true, events: { orderBy: { at: 'desc' }, take: 10 } } })`
- Nếu không tìm thấy → throw `NotFoundException`

**Bước 4:** Set cache
- Key: `trip:{tripId}`
- Value: JSON string của trip object
- TTL: 60 giây
- Gọi `RedisService.set(cacheKey, JSON.stringify(trip), 60)`

**Bước 5:** Trả về trip object

**Lưu ý:** Cache được invalidate tự động sau 60s, hoặc có thể invalidate thủ công khi trip được update (trip-command-service có thể publish event để query-service invalidate cache).

---

#### 4.2.2. Luồng Get User Trips (`GET /v1/trips/users/:userId/trips`)

**Method:** `TripsController.getUserTrips()` → `TripsService.getUserTrips()`

**Bước 1:** Nhận request với query params: `status?`, `limit?`, `offset?`

**Bước 2:** Build query where clause
- `OR: [{ passengerId: userId }, { driverId: userId }]` (user có thể là passenger hoặc driver)
- Nếu có `status` → thêm `status: status`

**Bước 3:** Query Postgres
- `Trip.findMany({ where, orderBy: { createdAt: 'desc' }, take: limit, skip: offset, include: { rating: true } })`
- `Trip.count({ where })` để lấy total

**Bước 4:** Trả về:
```json
{
  "trips": [...],
  "total": 100,
  "limit": 20,
  "offset": 0,
  "hasMore": true
}
```

**Lưu ý:** Có thể thêm cache cho user trips list (key: `user:{userId}:trips:{status}`) nếu cần optimize.

---

## 5. Driver Stream Service

### 5.1. Trách nhiệm chính

- Quản lý trạng thái online/offline của driver
- Cập nhật và query vị trí realtime của driver (Redis Geo)
- Pipeline assign trip cho driver (prepare, claim với atomic operations)
- Publish driver location events lên Kafka
- Cung cấp SSE stream cho driver mobile app để nhận trip offers

**Tech Stack:** Go, Redis (Geo + Sets + Hashes), Kafka, HTTP REST + gRPC

**File:** `driver-stream/cmd/server/main.go`, `driver-stream/internal/http/server.go`, `driver-stream/internal/redis/store.go`, `driver-stream/internal/grpc/server.go`

### 5.2. Luồng logic chính

#### 5.2.1. Driver Online/Offline (`POST /v1/drivers/:id/status`)

**Method:** `Server.postStatus()` → `Store.SetStatus()`

**Bước 1:** Nhận request
- Body: `{ status: "ONLINE" | "OFFLINE" }`

**Bước 2:** Update Redis
- Key: `presence:driver:{id}`
- Nếu `status == "ONLINE"`:
  - `HSET presence:driver:{id} status ONLINE last_seen {timestamp}`
  - `EXPIRE presence:driver:{id} 60` (TTL 60 giây)
- Nếu `status == "OFFLINE"`:
  - `DEL presence:driver:{id}`
  - `ZREM geo:drivers {id}` (xóa khỏi Geo index)

**Bước 3:** Trả về `{ driverId, status, expiresInSec: 60 }`

**Lưu ý:** TTL 60s nghĩa là driver phải update status định kỳ, nếu không sẽ tự động offline.

---

#### 5.2.2. Cập nhật vị trí (`PUT /v1/drivers/:id/location`)

**Method:** `Server.putLocation()` → `Store.UpsertLocation()`

**Bước 1:** Nhận request
- Body: `{ lat, lng, speed?, heading?, ts? }`

**Bước 2:** Update Redis Geo
- `GEOADD geo:drivers {lng} {lat} {id}` (thêm/update vị trí trong Geo index)
- `HSET presence:driver:{id} lat {lat} lng {lng} last_seen {timestamp} speed {speed} heading {heading}` (lưu metadata)
- `EXPIRE presence:driver:{id} 60` (TTL 60s)

**Bước 3:** Publish event lên Kafka
- Topic: `driver.location` (từ env `KAFKA_TOPIC_LOCATION`)
- Payload:
```json
{
  "event": "driver.location",
  "driverId": "...",
  "lat": ...,
  "lng": ...,
  "speed": ...,
  "heading": ...,
  "ts": ...
}
```

**Bước 4:** Trả về `{ ingested: true }` (status 202)

**Lưu ý:** 
- Redis Geo key: `geo:drivers` (chung cho tất cả drivers)
- Presence key: `presence:driver:{id}` (riêng cho từng driver)

---

#### 5.2.3. Tìm tài xế gần nhất (`GET /v1/drivers/nearby`)

**Method:** `Server.getNearby()` → `Store.Nearby()`

**Bước 1:** Nhận query params: `lat`, `lng`, `radius?` (default 2000m), `limit?` (default 20)

**Bước 2:** Query Redis Geo
- `GEORADIUS geo:drivers {lng} {lat} {radius}m WITHCOORD WITHDIST COUNT {limit} ASC`
- Trả về list drivers trong radius, sorted by distance

**Bước 3:** Filter chỉ lấy drivers ONLINE
- Với mỗi driver trong kết quả:
  - `HGETALL presence:driver:{id}`
  - Kiểm tra `status == "ONLINE"`
  - Nếu offline → skip

**Bước 4:** Trả về:
```json
{
  "count": 5,
  "drivers": [
    {
      "driverId": "...",
      "distance": 123.45,
      "lat": ...,
      "lng": ...,
      "status": "ONLINE",
      "lastSeen": ...
    }
  ]
}
```

---

#### 5.2.4. Pipeline Assign Trip

##### 5.2.4.1. Prepare Assign (`POST /v1/assign/prepare`)

**Method:** `Server.prepareAssign()` → `Store.PrepareAssign()`

**Bước 1:** Nhận request từ trip-command-service
- Body: `{ tripId, candidates: [driverId1, driverId2, ...], ttlSeconds: 15 }`

**Bước 2:** Atomic prepare trong Redis (dùng Pipeline Transaction)
- Reset keys cũ: `DEL trip:{tripId}:candidates trip:{tripId}:claimed trip:{tripId}:deadline`
- Set candidates: `SADD trip:{tripId}:candidates {driverId1} {driverId2} ...` + `EXPIRE 15`
- Set claimed flag: `SET trip:{tripId}:claimed "0"` + `EXPIRE 15` (chưa ai claim)
- Set deadline: `SET trip:{tripId}:deadline {timestamp}` + `EXPIRE 15`

**Bước 3:** Push SSE event tới từng driver candidate
- Với mỗi `driverId` trong `candidates`:
  - Lấy channel từ `Server.getDriverChan(driverId)` (tạo mới nếu chưa có)
  - Marshal payload: `{ tripId, ttlSeconds, candidates }`
  - Push vào channel: `ch <- payload`
  - Driver đang subscribe SSE `/v1/drivers/{id}/events` sẽ nhận được event `trip_offer`

**Bước 4:** Trả về `{ tripId, expiresInSec: 15 }`

---

##### 5.2.4.2. Driver Events (SSE) (`GET /v1/drivers/:id/events`)

**Method:** `Server.driverEvents()`

**Bước 1:** Driver mobile app connect SSE
- Set headers: `Content-Type: text/event-stream`, `Cache-Control: no-cache`, `Connection: keep-alive`

**Bước 2:** Gửi ping ban đầu
- `event: ping\n`
- `data: ok\n\n`

**Bước 3:** Subscribe channel
- Lấy channel từ `Server.getDriverChan(driverId)`
- Loop:
  - Nếu client disconnect → return
  - Nếu có message từ channel → gửi SSE:
    - `event: trip_offer\n`
    - `data: {json}\n\n`

**Lưu ý:** Channel được tạo per driver, buffer size 16. Nếu channel đầy thì bỏ qua (non-blocking).

---

##### 5.2.4.3. Claim Trip (`POST /v1/assign/claim`)

**Method:** `Server.claimAssign()` → `Store.Claim()`

**Bước 1:** Nhận request từ trip-command-service
- Body: `{ tripId, driverId }`

**Bước 2:** Chạy Lua script `claim.lua` (atomic operation)

**Lua Script Logic:**
```lua
-- KEYS: [deadline, claimed, candidates, presence]
-- ARGV: [now_ms, driverId]

-- 1. Kiểm tra TTL còn hạn
local ttl = redis.call('PTTL', KEYS[1])
if ttl <= 0 then return redis.error_reply('EXPIRED') end

-- 2. Kiểm tra chưa có ai claim
local claimed = redis.call('GET', KEYS[2])
if claimed and claimed ~= "0" then return redis.error_reply('ALREADY_CLAIMED') end

-- 3. Kiểm tra driver có trong candidates
local isCand = redis.call('SISMEMBER', KEYS[3], ARGV[2])
if isCand == 0 then return redis.error_reply('NOT_CANDIDATE') end

-- 4. Kiểm tra driver status là ONLINE
local status = redis.call('HGET', KEYS[4], 'status')
if status ~= 'ONLINE' then return redis.error_reply('DRIVER_OFFLINE') end

-- 5. Claim thành công
redis.call('SET', KEYS[2], ARGV[2]) -- set claimed = driverId
redis.call('PEXPIRE', KEYS[2], 600000) -- TTL 10 phút
return 'OK'
```

**Bước 3:** Xử lý kết quả
- Nếu success → trả về `{ tripId, claimedBy: driverId }`
- Nếu error:
  - `EXPIRED` → `409 Conflict { status: "EXPIRED" }`
  - `ALREADY_CLAIMED` → `409 Conflict { status: "ALREADY_CLAIMED" }`
  - `NOT_CANDIDATE` → `403 Forbidden { status: "NOT_CANDIDATE" }`
  - `DRIVER_OFFLINE` → `409 Conflict { status: "DRIVER_OFFLINE" }`

**Lưu ý:** Lua script đảm bảo atomicity, chỉ 1 driver có thể claim được trip (race condition safe).

---

### 5.3. Sharding by Region

**File:** `trip-service/trip-command-service/src/config/region-shard.config.ts`

**Cấu hình:**
```typescript
REGION_SHARD_CONFIG = {
  HCM: { driverStreamBaseUrl: 'http://driver-stream-hcm:8080' },
  HN: { driverStreamBaseUrl: 'http://driver-stream-hn:8080' }
}
```

**Logic:**
- `trip-command-service` lấy `cityCode` từ trip (hoặc từ request body)
- Gọi `getDriverStreamUrl(cityCode)` để map:
  - `HCM` → `driver-stream-hcm:8080`
  - `HN` → `driver-stream-hn:8080`
  - Unknown → fallback về `HCM`

**Mỗi shard driver-stream:**
- Có Redis instance riêng (Geo index riêng)
- Có Kafka topic riêng (nếu cần)
- Chỉ quản lý drivers trong region đó

**Lưu ý:** Trong production, có thể có nhiều shard hơn (ví dụ: `DN`, `CT`, ...) và có thể shard theo lat/lng thay vì cityCode.

---

## 6. Payment Service

**Trạng thái:** Service chưa được implement (chỉ có thư mục trống).

**Dự kiến trách nhiệm:**
- Quản lý ví (wallet) của user (passenger, driver)
- Quản lý số dư (balance)
- Tạo transaction (nạp tiền, trừ tiền, chuyển tiền)
- Query lịch sử giao dịch

**Luồng dự kiến:**
- Khi trip hoàn thành → tạo transaction:
  - Trừ tiền từ passenger wallet
  - Cộng tiền vào driver wallet (trừ platform fee)
  - Cộng tiền vào platform wallet (fee)
- Có thể tích hợp với payment gateway (Stripe, VNPay, ...)

---

## 7. Notification Service

**Trạng thái:** Service chưa được implement (chỉ có thư mục trống).

**Dự kiến trách nhiệm:**
- Gửi thông báo push (FCM, APNS)
- Gửi email (SMTP, SendGrid, ...)
- Gửi SMS (Twilio, ...)
- In-app notifications

**Luồng dự kiến:**
- Subscribe Kafka topics: `trip.created`, `trip.accepted`, `trip.completed`, ...
- Map event → message template
- Gửi qua provider tương ứng (push/email/SMS)
- Có thể có queue nội bộ để retry failed notifications

---

## 8. Driver Service, FE-User, FE-Driver (Overview)

### 8.1. Driver Service

**File:** `driver-service/crud/src/drivers/drivers.controller.ts`

**Trách nhiệm:** Quản lý hồ sơ driver (CRUD operations)

**Endpoints chính:**

- **`POST /v1/drivers`** (tạo hồ sơ driver)
  - Input: `CreateDriverDto` (thông tin cá nhân, giấy phép, ...)
  - Lưu vào Postgres (Prisma)

- **`GET /v1/drivers/:driverId`** (xem hồ sơ)
  - Query từ Postgres

- **`PATCH /v1/drivers/:driverId`** (cập nhật hồ sơ)
  - Update thông tin driver

- **`DELETE /v1/drivers/:driverId`** (xóa - soft delete)
  - Set flag `deletedAt`

- **`GET /v1/admin/drivers`** (admin list)
  - Query với filter `status` (PENDING, APPROVED, REJECTED)

- **`POST /v1/admin/drivers/:driverId/approve`** (admin duyệt)
  - Set `status = 'APPROVED'`

- **`POST /v1/admin/drivers/:driverId/reject`** (admin từ chối)
  - Set `status = 'REJECTED'`

**Lưu ý:** Driver service quản lý hồ sơ tĩnh, còn driver-stream quản lý trạng thái realtime (online/offline, location).

---

### 8.2. FE-User (Frontend cho Passenger)

**High-level flow:**

1. **Đăng nhập:**
   - Gọi `POST /api/v1/sessions` (login)
   - Lưu `accessToken`, `refreshToken` vào localStorage
   - Gắn `Authorization: Bearer <token>` vào các request sau

2. **Đặt xe:**
   - Gọi `POST /api/v1/trips/quote` để xem giá
   - Gọi `POST /api/v1/trips` để tạo trip
   - Subscribe SSE `/api/v1/trips/{tripId}/events` để theo dõi trạng thái:
     - `TRIP_CREATED` → đang tìm tài xế
     - `STATUS_CHANGED` → có tài xế nhận, đang đến đón, đang đi, hoàn thành

3. **Xem lịch sử:**
   - Gọi `GET /api/v1/trips/users/{userId}/trips?status=COMPLETED`

4. **Đánh giá:**
   - Gọi `POST /api/v1/trips/{tripId}/rate`

---

### 8.3. FE-Driver (Frontend cho Driver)

**High-level flow:**

1. **Đăng nhập:**
   - Tương tự FE-User

2. **Bật/tắt online:**
   - Gọi `POST /api/v1/drivers/{id}/status` với `{ status: "ONLINE" | "OFFLINE" }`

3. **Cập nhật vị trí:**
   - Định kỳ (mỗi 5-10 giây) gọi `PUT /api/v1/drivers/{id}/location` với `{ lat, lng, speed?, heading? }`

4. **Nhận trip offer:**
   - Subscribe SSE `GET /api/v1/drivers/{id}/events`
   - Nhận event `trip_offer` khi có trip mới
   - Hiển thị thông tin trip (pickup, destination, fare)

5. **Accept/Decline trip:**
   - Gọi `POST /api/v1/trips/{tripId}/accept` để nhận
   - Gọi `POST /api/v1/trips/{tripId}/decline` để từ chối

6. **Cập nhật trạng thái trip:**
   - `POST /api/v1/trips/{tripId}/arrive-pickup` (đã đến điểm đón)
   - `POST /api/v1/trips/{tripId}/start` (bắt đầu chuyến đi)
   - `POST /api/v1/trips/{tripId}/finish` (hoàn thành)

---

## 9. Infra, Proto (Vai trò)

### 9.1. Infra (`infra/`)

**Vai trò:**
- Cấu hình Docker Compose để chạy toàn bộ hệ thống local
- Chứa k6 load testing scripts
- Chứa proto files (nếu cần generate code)

**Services trong docker-compose:**
- **Databases:**
  - PostgreSQL (port 5432): Trip data
  - MongoDB (port 27017): User data
  - Redis (port 6379): Geo index + cache

- **Message Queue:**
  - Kafka + Zookeeper (ports 9092, 29092): Event streaming

- **Routing:**
  - OSRM (port 5000): Routing engine (có thể dùng để tính route thực tế)

- **Application Services:**
  - gateway-service (3004)
  - auth-service (3000)
  - user-service (3001)
  - trip-command-service (3002)
  - trip-query-service (3003)
  - driver-stream-hcm (8081)
  - driver-stream-hn (8082)

**k6 Scripts:**
- `trips_create.js`: Test create trip latency
- `drivers_update_location.js`: Test location update throughput
- `trips_read_cached.js`: Test read-heavy với cache

---

### 9.2. Proto (`proto/`)

**Vai trò:**
- Định nghĩa gRPC contracts giữa các services
- Đảm bảo type safety và versioning

**Files:**
- **`user.proto`**: UserService gRPC interface
  - Service: `UserService`
  - Method: `GetProfile(GetProfileRequest) returns (GetProfileResponse)`
  - Sử dụng bởi: `trip-command-service` để verify user

- **`driverstream.proto`**: DriverService gRPC interface
  - Service: `DriverService`
  - Methods:
    - `UpdateStatus`
    - `UpdateLocation`
    - `GetNearbyDrivers`
    - `PrepareAssign`
    - `ClaimTrip`
  - Sử dụng bởi: `trip-command-service` (có thể dùng gRPC thay vì HTTP)

- **`trip.proto`**: (nếu có) TripService gRPC interface

**Cách generate code:**
- **Node.js:** Dùng `@grpc/proto-loader` để load dynamic
- **Go:** Dùng `protoc` để generate Go code

**Best practices:**
- Backward compatibility: Không xóa fields, chỉ thêm optional
- Versioning: Tag proto files với version numbers

---

## Summary

Tài liệu này mô tả chi tiết luồng logic của từng microservice trong UITGo. Để xem chi tiết luồng cụ thể:

- **Gateway & Authentication:** Xem section 1 và 2
- **User Management:** Xem section 3
- **Trip Operations (Create, Accept, Update):** Xem section 4.1
- **Trip Queries (Read với Cache):** Xem section 4.2
- **Driver Realtime (Location, Status, Assign):** Xem section 5
- **Driver Profile Management:** Xem section 8.1
- **Frontend Flows:** Xem section 8.2 và 8.3
- **Infrastructure Setup:** Xem section 9.1
- **gRPC Contracts:** Xem section 9.2

