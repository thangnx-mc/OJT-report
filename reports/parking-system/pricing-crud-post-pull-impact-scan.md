# Báo Cáo Đánh Giá Tác Động Sau Khi Pull Code (Post-Pull Impact Scan)
## Đề Tài: Quản Lý Cấu Hình Bảng Giá (Pricing & Tariff CRUD) — Parking Management System

---

## 1. Tóm Tắt Điều Hành (Executive Summary)

- **Ngày thực hiện:** 23/10/2026
- **Thời điểm quét:** Sau đợt Pull code của Team lúc 12:49:47 - 12:49:58 (GMT+7).
- **Phân loại tác động tổng thể (Overall Impact Verdict):**
  > ### `NO MATERIAL IMPACT FOUND` (KHÔNG CÓ TÁC ĐỘNG TIÊU CỰC ĐẾN PHẠM VI CÔNG VIỆC)
- **Tóm tắt phát hiện chính:**
  - `[FACT]` Đợt cập nhật mới nhất từ Git của Team bao gồm **23 tệp tin** (14 tệp Backend, 9 tệp Frontend). Toàn bộ nội dung cập nhật thuộc về phân hệ **Thông báo khách hàng trong ứng dụng (In-App Customer Notification Subsystem - Bell Notification)** được kích hoạt qua SignalR và Hangfire Jobs.
  - `[FACT]` **Không có bất kỳ thay đổi nào** đối với các Entity cốt lõi của phân hệ định giá (`PricingRule`, `PricingRuleHistory`, `PricingTimeBand`, `ParkingOptions`).
  - `[FACT]` **Không có bất kỳ thay đổi nào** đối với công thức tính phí (`FeeCalculator.cs`), cơ chế cấp phát bảng giá thời gian thực (`TariffProvider.cs`), phân quyền hạn chế theo bãi (`LotScopeService.cs`), hay giao diện quản lý giá (`ManagerPricingPage.tsx`, `useManagerLot.ts`).
  - `[FACT]` Tệp `GateService.cs` chỉ được bổ sung dependency tùy chọn `IUserNotificationPublisher notifications = null` để đẩy chuông thông báo khi xe Check-in và Check-out (`SessionCheckedIn`, `SessionCheckedOut`). Toàn bộ logic tra cứu biểu giá (`tariffs.ForAsync`), tính tiền (`QuoteAsync`), kết toán (`FeeCalculator.Settle`), và lưu phiên bản bảng giá vào phiên gửi xe (`session.PricingRuleId`, `session.PricingRuleVersion`) hoàn toàn giữ nguyên 100%.
  - `[INFERENCE]` Kế hoạch triển khai kỹ thuật đã lập tại tài liệu `pricing-tariff-crud-readiness.md` vẫn **hoàn toàn chính xác, đầy đủ và khả thi 100%**, sẵn sàng để bước vào giai đoạn hiện thực hóa (Implementation) mà không cần phải điều chỉnh thiết kế kiến trúc hay giải pháp kỹ thuật.

---

## 2. Phạm Vi, Môi Trường & Phương Pháp Điều Tra

### 2.1. Phạm vi workspace
- **Team Workspace:** `C:\DEV\OJT-FSA-Team3` (Read-only hoàn toàn, cấm mọi lệnh Git).
  - Backend: `C:\DEV\OJT-FSA-Team3\parking-management-system` (.NET 8 Web API, EF Core).
  - Frontend: `C:\DEV\OJT-FSA-Team3\parking-management-system-frontend` (React 18, Vite, TypeScript, TanStack Query).
- **Reporting Workspace:** `C:\DEV\OJT-report` (Remote: `https://github.com/thangnx-mc/OJT-report.git`, Branch: `main`).

### 2.2. Phương pháp điều tra an toàn (Read-Only Safety Protocol)
- `[FACT]` Tuân thủ tuyệt đối quy tắc an toàn Git: Không thực hiện bất kỳ lệnh Git nào (kể cả lệnh đọc `git status`, `git log`) bên trong `C:\DEV\OJT-FSA-Team3`.
- `[FACT]` Toàn bộ việc phát hiện tập tin thay đổi được thực hiện qua việc quét thời gian sửa đổi tập tin hệ thống (`os.path.getmtime`), lọc bỏ các thư mục tự sinh (`node_modules`, `bin`, `obj`, `.git`, `.vs`).
- `[FACT]` Phân tích tĩnh mã nguồn (Static Code Analysis), AST inspection và regex pattern matching trên từng file bị tác động để đối chiếu với kiến trúc định giá.

---

## 3. Danh Mục Chi Tiết 23 Tệp Tin Được Cập Nhật

Đợt pull diễn ra đồng bộ vào khoảng thời gian **12:49:47 - 12:49:58**:

| STT | Thời gian | Không gian | Đường dẫn tập tin | Loại tập tin | Mục đích & Trách nhiệm |
|:---:|:---:|:---:|:---|:---:|:---|
| 1 | 12:49:47 | Backend | `src/PMS.Core/Entities/UserNotification.cs` | **Mới** | Entity lưu thông báo chuông của người dùng (Id, UserId, Type, Title, Message, RefType, RefId, EventAt, DataJson, ReadAt) |
| 2 | 12:49:47 | Backend | `src/PMS.Application/Models/Notifications/NotificationModels.cs` | **Mới** | DTOs cho Notification (NotificationModel, NotificationListModel, NotificationTypes) |
| 3 | 12:49:47 | Backend | `src/PMS.Application/Services/INotificationService.cs` | **Mới** | Interface dịch vụ đọc/đánh dấu thông báo |
| 4 | 12:49:47 | Backend | `src/PMS.Application/Services/Impl/NotificationService.cs` | **Mới** | Hiện thực hóa dịch vụ NotificationService truy vấn EF Core |
| 5 | 12:49:47 | Backend | `src/PMS.Application/Services/Impl/HangfireUserNotificationPublisher.cs` | **Mới** | Publisher kích hoạt Hangfire background job tạo thông báo không đồng bộ |
| 6 | 12:49:47 | Backend | `src/PMS.Application/Jobs/NotificationJobs.cs` | **Mới** | Hangfire jobs tạo thông báo xe duyệt/từ chối, check-in, check-out và dọn dẹp định kỳ (Purge) |
| 7 | 12:49:47 | Backend | `src/PMS.API/Controllers/NotificationsController.cs` | **Mới** | Controller phục vụ chuông thông báo khách hàng (5 endpoints) |
| 8 | 12:49:47 | Backend | `src/PMS.API/Hubs/SignalRUserNotificationNotifier.cs` | **Mới** | Bộ phát SignalR đẩy sự kiện real-time `NotificationCreated` tới client |
| 9 | 12:49:47 | Backend | `src/PMS.DataAccess/Persistence/PmsDbContext.Notifications.cs` | **Mới** | Partial DbContext khai báo `DbSet<UserNotification>` và cấu hình Fluent API |
| 10 | 12:49:47 | Backend | `src/PMS.DataAccess/Persistence/PmsDbContext.Layouts.cs` | **Sửa** | Gọi bổ sung `ConfigureNotifications(modelBuilder)` trong partial OnModelCreating |
| 11 | 12:49:47 | Backend | `src/PMS.DataAccess/Persistence/AutomatedMigration.cs` | **Sửa** | Bổ sung hàm migration tự động idempotent `EnsureNotificationsTableAsync` tạo bảng `[dbo].[UserNotifications]` |
| 12 | 12:49:47 | Backend | `src/PMS.API/ApiDependencyInjection.cs` | **Sửa** | Đăng ký DI cho Hangfire, SignalR Notifier, NotificationService và Recurring Job Purge |
| 13 | 12:49:47 | Backend | `src/PMS.Application/Services/Impl/GateService.cs` | **Sửa** | Gọi `notifications?.SessionCheckedIn` và `notifications?.SessionCheckedOut` |
| 14 | 12:49:47 | Backend | `src/PMS.Application/Services/Impl/VehicleDocumentService.cs` | **Sửa** | Gọi `notifications?.VehicleReviewed` khi Staff duyệt/từ chối giấy tờ xe |
| 15 | 12:49:58 | Frontend | `src/api/notification.service.ts` | **Mới** | API client gọi Backend `/api/notifications` |
| 16 | 12:49:58 | Frontend | `src/hooks/useNotifications.ts` | **Mới** | React Query hook quản lý state danh sách thông báo và lắng nghe SignalR |
| 17 | 12:49:58 | Frontend | `src/components/customer/NotificationBell.tsx` | **Mới** | Component nút Chuông thông báo hiển thị số đếm chưa đọc trên Header |
| 18 | 12:49:58 | Frontend | `src/components/customer/dashboard/NotificationCard.tsx` | **Mới** | Widget hiển thị thông báo gần đây trên Dashboard của Customer |
| 19 | 12:49:58 | Frontend | `src/api/queryKeys.ts` | **Sửa** | Thêm namespace queryKeys `notifications: { all, list, unreadCount }` |
| 20 | 12:49:58 | Frontend | `src/api/parkingOpsRealtime.ts` | **Sửa** | Bổ sung xử lý sự kiện `NotificationCreated` nhận từ SignalR hub |
| 21 | 12:49:58 | Frontend | `src/components/customer/CustomerShell.tsx` | **Sửa** | Gắn component `<NotificationBell />` vào thanh điều hướng Header |
| 22 | 12:49:58 | Frontend | `src/pages/customer/CustomerDashboardPage.tsx` | **Sửa** | Gắn `<NotificationCard />` vào Dashboard |
| 23 | 12:49:58 | Frontend | `src/styles/customer.css` | **Sửa** | Bổ sung CSS định kiểu cho thẻ thông báo và chuông |

---

## 4. Phân Tích Tác Động Trực Tiếp (Direct Impact Analysis)

### 4.1. Thực thể Bảng giá & Biểu giá (Pricing Domain Entities)
- `[FACT]` `src/PMS.Core/Entities/PricingRule.cs`: **Hoàn toàn không bị sửa đổi**.
- `[FACT]` `src/PMS.Core/Entities/PricingRuleHistory.cs`: **Hoàn toàn không bị sửa đổi**.
- `[FACT]` `src/PMS.Core/Entities/PricingTimeBand.cs`: **Hoàn toàn không bị sửa đổi**.
- `[FACT]` `src/PMS.Application/Options/ParkingOptions.cs`: **Hoàn toàn không bị sửa đổi**.
- **Kết luận:** Cấu trúc dữ liệu phân hệ Định giá (Rule, TimeBand, History, Options) ở mức Core Domain giữ nguyên tính toàn vẹn 100%.

### 4.2. Động cơ tính phí & Cung cấp biểu giá (Fee Engine & Tariff Provider)
- `[FACT]` `src/PMS.Infrastructure/Services/FeeCalculator.cs`: **Không bị tác động**. Thuật toán tính toán Block-based, Grace period, Day/Night time bands, Maximum daily fee, Overstay fine và Settlement giữ nguyên vẹn.
- `[FACT]` `src/PMS.Infrastructure/Services/TariffProvider.cs`: **Không bị tác động**. Logic nạp cấu hình bảng giá có hiệu lực (`ForAsync`) với fallback về `ParkingOptions` giữ nguyên cơ chế in-memory caching hiện tại.
- **Kết luận:** Động cơ tính tiền cốt lõi không có bất kỳ xung đột hay biến động nào.

### 4.3. Giao diện Người dùng Bảng giá (Manager Pricing Frontend)
- `[FACT]` `src/pages/manager/ManagerPricingPage.tsx`: **Không bị sửa đổi**.
- `[FACT]` `src/types/pricing.dto.ts`: **Không bị sửa đổi**.
- `[FACT]` `src/api/managerOps.mock.ts`: **Không bị sửa đổi**.
- `[FACT]` `src/hooks/useManagerLot.ts`: **Không bị sửa đổi**.
- **Kết luận:** Giao diện Manager Pricing và các hợp đồng DTO phía Frontend vẫn giữ nguyên trạng thái sẵn sàng để tích hợp API thật.

---

## 5. Phân Tích Tác Động Gián Tiếp (Indirect Impact Analysis)

### 5.1. Dịch vụ Cổng xe (`GateService.cs`)
- `[FACT]` Tại `GateService.cs`, Team chỉ bổ sung tham số tùy chọn ở constructor:
  ```csharp
  public GateService(
      PmsDbContext db,
      IOptions<ParkingOptions> options,
      TariffProvider tariffs,
      ILotScopeService lotScope,
      IPaymentService paymentService,
      IUserNotificationPublisher notifications = null) : IGateService
  ```
- `[FACT]` Việc chèn thông báo tại sự kiện Check-in (Dòng 333) và Check-out (Dòng 486, 588):
  ```csharp
  notifications?.SessionCheckedIn(session.Id);
  notifications?.SessionCheckedOut(session.Id);
  ```
- `[FACT]` Kiểm tra logic định giá trong `GateService.cs`:
  - Dòng 246: `var tariff = await tariffs.ForAsync(lotId, reservation?.ZoneId, vehicleTypeId, Now);` (Giữ nguyên).
  - Dòng 263-264: Gán `PricingRuleId = tariff.RuleId`, `PricingRuleVersion = tariff.Version` vào Session (Giữ nguyên).
  - Dòng 742: `var tariff = await tariffs.ForAsync(...)` trong `QuoteAsync` (Giữ nguyên).
  - Dòng 746: `FeeCalculator.Calculate(tariff, Utc(session.EntryTime), Now, timeZone);` (Giữ nguyên).
  - Dòng 755: `FeeCalculator.Settle(...)` (Giữ nguyên).
- **Kết luận:** Việc phát sinh notification sau khi Check-in / Check-out hoàn toàn độc lập với việc tính tiền. Bảng giá được áp dụng vào session trước khi sự kiện thông báo được gửi đi.

### 5.2. Hạ tầng Cơ sở dữ liệu (`AutomatedMigration.cs` & `PmsDbContext`)
- `[FACT]` `AutomatedMigration.cs` được thêm phương thức `EnsureNotificationsTableAsync(PmsDbContext pmsContext)`. Hàm này thực thi câu lệnh SQL idempotent:
  ```sql
  IF OBJECT_ID(N'[dbo].[UserNotifications]', N'U') IS NULL
  CREATE TABLE [dbo].[UserNotifications](
      [Id] [bigint] IDENTITY(1,1) NOT NULL,
      [UserId] [nvarchar](450) NOT NULL,
      [Type] [nvarchar](50) NOT NULL,
      ...
  ```
- `[FACT]` Bảng `dbo.UserNotifications` hoàn toàn độc lập, không có Foreign Key trỏ đến `PricingRules` hay can thiệp vào cấu trúc bảng định giá.
- `[FACT]` `PmsDbContext.Layouts.cs` sửa đổi phương thức `OnModelCreatingPartial` chỉ để gọi thêm `ConfigureNotifications(modelBuilder)`.
- **Kết luận:** Không phát sinh thay đổi Schema hay xung đột khóa ngoại nào liên quan đến Pricing.

### 5.3. Dependency Injection & Background Jobs (`ApiDependencyInjection.cs`)
- `[FACT]` `ApiDependencyInjection.cs` bổ sung:
  - `services.AddScoped<INotificationService, NotificationService>();`
  - `services.AddScoped<IUserNotificationPublisher, HangfireUserNotificationPublisher>();`
  - `services.AddScoped<IUserNotificationNotifier, SignalRUserNotificationNotifier>();`
  - Cấu hình Hangfire Server & Dashboard `/hangfire`.
  - Đăng ký Recurring Job xóa thông báo cũ hàng ngày `jobs.AddOrUpdate<NotificationJobs>(...)`.
- `[INFERENCE]` Khi chúng ta đăng ký `services.AddScoped<IPricingRuleService, PricingRuleService>();` trong lát cắt triển khai sắp tới, vị trí đăng ký dịch vụ của Pricing không bị xung đột với các dịch vụ Notification hay Hangfire.

### 5.4. Frontend Query Keys & Real-Time Hub
- `[FACT]` Trong `src/api/queryKeys.ts`, Team bổ sung khóa `notifications`:
  ```typescript
  notifications: {
    all: ['notifications'] as const,
    list: () => [...queryKeys.notifications.all, 'list'] as const,
    unreadCount: () => [...queryKeys.notifications.all, 'unread-count'] as const,
  },
  ```
- `[FACT]` Trong `src/api/parkingOpsRealtime.ts`, sự kiện `NotificationCreated` được thêm vào union type `ParkingOpsEvent` và danh sách sự kiện lắng nghe của SignalR Hub.
- `[INFERENCE]` Khi triển khai lát cắt Pricing Frontend, chúng ta sẽ thêm namespace `pricing` vào `queryKeys.ts` hoàn toàn độc lập, không xung đột.

---

## 6. Phân Tích Phạm Vi Quản Lý Bãi & Bảo Mật (Lot Scoping & Security)

- `[FACT]` Controller mới `NotificationsController.cs` được bảo vệ bằng thuộc tính `[Authorize(Roles = "User")]` (dành riêng cho khách hàng cá nhân).
- `[FACT]` Dịch vụ phân quyền bãi đỗ của Quản lý (`src/PMS.Application/Services/Impl/LotScopeService.cs`) và interface `ILotScopeService.cs` **hoàn toàn không bị sửa đổi**.
- `[FACT]` Cơ chế thẩm định quyền hạn của Manager qua `lotScopeService.EnsureCanManageAsync(lotId)` đề xuất trong kế hoạch triển khai không bị ảnh hưởng.

---

## 7. Bảng Đối Chiếu Hiện Trạng So Với Baseline Task 0

| Chỉ số / Hạng mục | Baseline Task 0 (Trước 12:49) | Hiện trạng (Sau Pull 12:50) | Biến động | Đánh giá tác động |
|:---|:---:|:---:|:---:|:---|
| **Số lượng Controller PMS.API** | 16 controllers | 17 controllers | +1 (`NotificationsController.cs`) | Không ảnh hưởng (thuộc nhánh `/api/notifications`) |
| **Số lượng Entities PMS.Core** | 35 entities | 36 entities | +1 (`UserNotification.cs`) | Không ảnh hưởng (độc lập bảng thông báo) |
| **Số bảng trong DbContext** | Không có bảng Notification | Có `DbSet<UserNotification>` | +1 bảng `dbo.UserNotifications` | Không xung đột với các bảng Pricing |
| **Logic tính tiền GateService** | Gọi `tariffs.ForAsync` & `FeeCalculator` | Gọi `tariffs.ForAsync` & `FeeCalculator` | 0 thay đổi | Bảo toàn 100% logic định giá và kết toán |
| **Manager Scope Security** | `ILotScopeService` kiểm tra quản lý bãi | `ILotScopeService` kiểm tra quản lý bãi | 0 thay đổi | Cơ chế bảo mật bãi của Manager giữ nguyên |
| **Trang Frontend Manager Pricing**| Dùng demo mock data | Dùng demo mock data | 0 thay đổi | Sẵn sàng tích hợp API thật theo kế hoạch |

---

## 8. Thẩm Định Kế Hoạch Triển Khai (Readiness Plan Validation)

Chúng ta rà soát lại 5 lát cắt mỏng (Thin Slices) đã thiết lập trong `pricing-tariff-crud-readiness.md`:

1. **Lát cắt 1 — Domain & Application Service:**
   - Tạo `PricingRuleModels.cs`, `IPricingRuleService.cs`, `PricingRuleService.cs`.
   - *Thẩm định:* **Hợp lệ 100%**. Không có thay đổi nào trong EF DbContext hay Core Entity làm ảnh hưởng logic nghiệp vụ.
2. **Lát cắt 2 — Backend Controller & Security:**
   - Tạo `ManagerPricingController.cs` với route `api/manager/lots/{lotId:int}/pricing-rules`.
   - *Thẩm định:* **Hợp lệ 100%**. Tuyến đường không trùng lặp hay va chạm với bất kỳ controller nào trong số 17 controller hiện tại.
3. **Lát cắt 3 — Tích hợp Làm tươi Cache `TariffProvider`:**
   - Thêm phương thức `InvalidateCache(int lotId)` vào `TariffProvider.cs`.
   - *Thẩm định:* **Hợp lệ 100%**. `TariffProvider.cs` không bị team sửa đổi trong đợt pull vừa qua.
4. **Lát cắt 4 — Frontend API Client & Hook Integration:**
   - Tạo `pricing.service.ts`, `usePricingRules.ts`, cập nhật `queryKeys.ts` và thay thế mock data trong `ManagerPricingPage.tsx`.
   - *Thẩm định:* **Hợp lệ 100%**. Frontend component không có xung đột mã nguồn.
5. **Lát cắt 5 — Kiểm Thử & Xác Nhận:**
   - Viết Integration Tests và Manual Verification qua giao diện.
   - *Thẩm định:* **Hợp lệ 100%**.

---

## 9. Phân Loại Bằng Chứng Kỹ Thuật (Evidence Classification)

- `[FACT]` Đợt cập nhật của Team bổ sung 23 tệp tin phục vụ chức năng In-App Notification và chuông thông báo người dùng.
- `[FACT]` Không có bất kỳ tệp nào liên quan trực tiếp đến phân hệ Định giá (`PricingRule`, `FeeCalculator`, `TariffProvider`) bị sửa đổi trong đợt cập nhật này.
- `[FACT]` `GateService.cs` chỉ thêm lời gọi tùy chọn tới `IUserNotificationPublisher` sau khi quá trình định giá và thanh toán đã hoàn tất.
- `[INFERENCE]` Thay đổi trong đợt pull là hoàn toàn trực giao (orthogonal) với nghiệp vụ cấu hình bảng giá của Manager.
- `[OBSERVATION]` Các dịch vụ Hangfire và SignalR được cấu hình bổ sung trong `ApiDependencyInjection.cs` hoạt động độc lập và không can thiệp vào pipeline Dependency Injection của các dịch vụ nghiệp vụ khác.

---

## 10. Kết Luận & Khuyến Nghị Hành Động (Actionable Next Steps)

- **Kết luận:** Lời khẳng định của Team rằng *"bản cập nhật không ảnh hưởng đến task Pricing CRUD"* đã được **chứng minh là CHÍNH XÁC** bằng bằng chứng mã nguồn tĩnh thực tế.
- **Khuyến nghị:**
  1. Giữ nguyên toàn bộ thiết kế kiến trúc và kế hoạch triển khai tại `pricing-tariff-crud-readiness.md`.
  2. Bắt đầu ngay **Lát cắt 1 (Slice 1)**: Xây dựng DTOs và Service nghiệp vụ cho Pricing CRUD ở Backend.
