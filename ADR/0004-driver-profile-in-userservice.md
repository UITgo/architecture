# ADR 0004 – Move Driver Profile CRUD to UserService

**Date:** 2025-10-22  
**Status:** Accepted  
**Owner:** Diễm – Backend Architect  

## Context
Ban đầu, nhóm đặt toàn bộ CRUD của tài xế (drivers, vehicles) trong `driver-service` (NestJS).  
Tuy nhiên, trong kiến trúc tổng thể UIT-Go:
- `driver-service` (Go) được định nghĩa là service realtime – quản lý trạng thái ONLINE/OFFLINE, vị trí và tìm kiếm gần.
- `user-service` (NestJS + Mongo) chịu trách nhiệm quản lý hồ sơ người dùng (passenger + driver).

Nếu giữ CRUD ở `driver-service`, sẽ **phạm nguyên tắc Database-per-Service**  
và gây trùng dữ liệu tài xế giữa hai service.

## Decision
Chuyển toàn bộ CRUD hồ sơ/xe (`drivers`, `vehicles`) từ `driver-service` sang `user-service`.  
DriverService (Go) chỉ giữ các API realtime:  
`POST /v1/drivers/{id}/status`, `PUT /v1/drivers/{id}/location`, `GET /v1/drivers/nearby`, `POST /v1/assign/*`.

## Alternatives Considered
1. **Giữ nguyên CRUD trong driver-service (NestJS)**  
   - ✅ Nhanh, không cần sửa nhiều code.  
   - ❌ Trùng dữ liệu, sai ranh giới domain, khó scale khi realtime tách sang Go.
2. **Tách riêng DriverProfileService riêng biệt**  
   - ✅ Rõ ràng, dễ quản lý.  
   - ❌ Tốn thêm deploy, tăng số lượng service nhỏ không cần thiết.
3. **Đưa về UserService** *(chọn phương án này)*  
   - ✅ Đúng domain, dễ tích hợp hồ sơ người dùng.  
   - ✅ Giảm duplication dữ liệu.  
   - ⚠️ Cần cập nhật route gateway & database migration.

## Consequences
- ✅ Hồ sơ/xe được quản lý thống nhất trong `user-service`.  
- ✅ `driver-service` (Go) nhẹ, chỉ xử lý realtime.  
- ⚠️ Cần cập nhật Gateway route và CI/CD pipeline.  
- ⚠️ Tạm mất 1–2 ngày refactor mã NestJS.

## Related ADRs
- ADR 0002 – Redis GEO vs PostGIS for nearby search  
- ADR 0003 – Outbox & Debezium for trip events
