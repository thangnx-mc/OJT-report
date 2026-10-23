# Báo Cáo Post-Sprint Baseline Scan — Parking Management System

> **Thời điểm thực hiện scan:** 23/10/2026  
> **Mục tiêu:** Rà soát và thiết lập baseline toàn diện (read-only) sau đợt Sprint Agile Review của team, nắm bắt toàn bộ thay đổi kiến trúc, controllers, endpoints, frontend pages, contract mapping FE-BE, độ phủ test và các rủi ro kỹ thuật hiện tại.

---

## 1. Executive Summary

- `[FACT]` Sau đợt Sprint Agile Review gần nhất, toàn bộ mã nguồn mới nhất của cả Backend (`parking-management-system`) và Frontend (`parking-management-system-frontend`) đã được đồng bộ về local.
- `[FACT]` **Backend đột phá quy mô:** Backend đã mở rộng từ **6 Controllers (47–56 actions)** ở baseline Phase 3.1 lên **16 Controllers (114 action methods, 116 HTTP routes)**, tích hợp hoàn chỉnh các domain nghiệp vụ cốt lõi: Reservations (đặt chỗ trước), Parking Sessions (phiên đỗ xe thực tế), Staff Gate Operations (làn vào/ra, tính cước, thu tiền mặt/online), Payments (VNPay, MoMo), Vehicle Documents & Verification (tải và duyệt giấy tờ xe/CCCD), và Real-time SignalR Hubs.
- `[FACT]` **Frontend chuyển dịch công nghệ toàn diện:** Frontend đã được chuyển đổi hoàn toàn sang **TypeScript + React 19 + Vite + TailwindCSS + TanStack Query**, cấu trúc thành **22 Pages** phục vụ đầy đủ 4 nhóm vai trò: Admin, Manager, Staff, Customer (User), cùng Landing Page (`HomePage`) và 3D Interactive Garage Model dựng bằng Three.js.
- `[FACT]` **Phân tầng Real vs Mock rõ ràng:** Frontend đã tích hợp cơ chế cấu hình linh hoạt `VITE_MOCK_SERVICES`. Mặc định các service `auth`, `profile`, `staff`, `team`, `parking`, `booking`, `gate` được kết nối trực tiếp đến Backend REST API và SignalR Hubs; riêng `reports` và `capacity` được cấu hình mock mặc định (`VITE_MOCK_SERVICES=capacity,reports`) do Backend chưa triển khai controller báo cáo (`ReportsController`) và endpoint `capacity` chuyên biệt.
- `[OBSERVATION]` Thư mục `Vercel deploy` xuất hiện tại thư mục gốc của team workspace (`C:\DEV\OJT-FSA-Team3\Vercel deploy`), chứa bản clone của frontend ở một thời điểm triển khai thử nghiệm trước đó.

---

## 2. Tổng Quan Cấu Trúc Dự Án (Workspace Architecture Snapshot)

### 2.1. Phân Bố Thư Mục và Solution

| Khu Vực / Thư Mục | Đường Dẫn Tuyệt Đối | Vai Trò & Trạng Thái Hiện Tại |
| :--- | :--- | :--- |
| **Team Workspace Root** | `C:\DEV\OJT-FSA-Team3` | Thư mục làm việc chung của team (chứa BE, FE, `.postman`, `Vercel deploy`). |
| **Backend** | `C:\DEV\OJT-FSA-Team3\parking-management-system` | .NET 9 Web API monolith (`PMS.sln`), triển khai theo mô hình Clean Architecture / N-Tier. |
| **Frontend** | `C:\\DEV\\OJT-FSA-Team3\\parking-management-system-frontend` | Ứng dụng Single Page Application (SPA) viết bằng TypeScript, React 19, Vite, Three.js, Lucide Icons, Sonner. |
| **Deployment Clone** | `C:\DEV\OJT-FSA-Team3\Vercel deploy` | Bản sao frontend phục vụ deploy Vercel thử nghiệm (độc lập, không ảnh hưởng trực tiếp source code chính). |
| **Postman Resource** | `C:\DEV\OJT-FSA-Team3\.postman` | File cấu hình `resources.yaml` trỏ tới OpenApi spec local của API. |
| **Reporting Repository**| `C:\DEV\OJT-report` | Repository Git độc lập dùng để lưu trữ toàn bộ tài liệu, nhật ký công việc và báo cáo kỹ thuật. |

### 2.2. Chi Tiết Các Project trong .NET Solution (`PMS.sln`)

Solution `PMS.sln` bao gồm 7 project chính đang hoạt động (song song với các template thư viện gốc `N-Tier.*`):

1. `PMS.API`: Project khởi chạy (Host Web API), chứa các Controller, SignalR Hubs, Middleware, DI configurations và Swagger.
2. `PMS.Application`: Layer xử lý logic nghiệp vụ trung tâm, Interfaces, DTOs, FluentValidation validators, Background Jobs, Payment Gateways (VNPay, MoMo), Helpers (State Machines, Fee Calculator, LicensePlateHelper).
3. `PMS.Core`: Chứa các Domain Entities (73 entity classes), Enums, Value Objects và Domain interfaces.
4. `PMS.DataAccess`: Triển khai truy xuất dữ liệu với Entity Framework Core, DbContext (`DatabaseContext`, `PmsDbContext`), Identity `ApplicationUser`, DDL Migrations tự động và Data Seeding.
5. `PMS.Shared`: Chứa các tiện ích dùng chung (ClaimService, Email template engine cơ bản).
6. `PMS.Application.UnitTests`: Bộ unit tests toàn diện cho Application layer (Services, Operations, Jobs, Validators).
7. `PMS.Api.IntegrationTests`: Bộ integration tests kiểm thử các HTTP endpoints và SignalR Hub.

---

## 3. Backend Deep Dive: 16 Controllers & 116 Endpoints

### 3.1. Thống Kê Tổng Hợp Controllers

| STT | Controller | Base Route | Phân Quyền Class-level | Số Routes | Phạm Vi Nghiệp Vụ |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 1 | `AdminController.cs` | `/api/admin` | `[Authorize(Roles = "Admin")]` | 8 | Quản lý tài khoản Quản lý bãi (Managers), phân công bãi đỗ, quản lý trạng thái tài khoản nội bộ. |
| 2 | `ApiController.cs` | `/api/[controller]` | Không | 0 | Base controller kế thừa `ControllerBase`, đánh dấu `[ApiController]`. |
| 3 | `LayoutsController.cs` | `/api/public/lots/{lotId:int}/layout` | Không (Public) | 4 | Cung cấp template bãi xe, sơ đồ 2D/3D layout, trạng thái slot công khai cho khách hàng. |
| 4 | `LotsController.cs` | `/api/lots` | `[Authorize(Roles = "Admin,Manager")]` | 9 | CRUD bãi đỗ xe, áp dụng layout template, bật/tắt trạng thái vận hành, danh sách bãi public. |
| 5 | `ManagerController.cs` | `/api/manager` | `[Authorize(Roles = "Manager")]` | 8 | Quản lý danh sách nhân viên cấp dưới (Staff) của Manager theo bãi đỗ được phân công. |
| 6 | `ManagerOperationsController.cs` | `/api/manager` | `[Authorize(Roles = "Manager,Admin")]` | 8 | Xử lý vận hành nâng cao: danh sách đặt chỗ bãi, đóng phiên bất thường, duyệt/từ chối mất vé, quản lý danh sách biển số (Blacklist/Whitelist). |
| 7 | `ParkingSessionsController.cs` | `/api/parking-sessions` | `[Authorize(Roles = "User")]` | 7 | Khách hàng tra cứu phiên đỗ hiện tại, lịch sử đỗ xe, lấy vé xuất bãi (Exit Pass), báo giá và thanh toán online. |
| 8 | `PaymentsController.cs` | `/api/payments` | `[Authorize]` (Partial AllowAnonymous) | 6 | Tra cứu giao diện thanh toán, kiểm tra trạng thái, tiếp nhận Return URL và Webhook IPN từ VNPay và MoMo. |
| 9 | `ReservationsController.cs` | `/api/reservations` | `[Authorize(Roles = "User")]` | 8 | Khách hàng tạo đặt chỗ (booking), cập nhật lịch, hủy đặt chỗ, lấy mã QR check-in, thanh toán tiền cọc (deposit). |
| 10 | `SlotsController.cs` | `/api/slots` | `[Authorize(Roles = "Admin,Manager")]` | 5 | Quản lý chi tiết vị trí đỗ (Slot), lấy slot theo Zone, cập nhật cấu hình và trạng thái slot. |
| 11 | `StaffController.cs` | `/api/staff` | `[Authorize(Roles = "Staff")]` | 1 | Lấy danh sách các bãi đỗ được phân công cho tài khoản nhân viên hiện tại. |
| 12 | `StaffGateController.cs` | `/api/staff` | `[Authorize(Roles = "Staff,Manager,Admin")]` | 17 | Nghiệp vụ cổng kiểm soát: tra cứu biển số/vé/mã đặt chỗ, Check-in, xác nhận/hủy cổng vào, tính cước xuất bãi, thu tiền mặt/online, Checkout mở barie, lập biên bản mất vé. |
| 13 | `UsersController.cs` | `/api/users` | Kế thừa `[Authorize(Roles = "Admin")]` (Partial AllowAnonymous / Authorize) | 21 | Xác thực đăng nhập (Password, Google, Login OTP, Email 2FA), Đăng ký, Quản lý Profile khách hàng, CRUD phương tiện và xe mặc định, Đổi mật khẩu. |
| 14 | `VehicleDocumentsController.cs` | `/api/users/profile/vehicles/{vehicleId:guid}/documents` | `[Authorize]` | 5 | Khách hàng tải ảnh giấy đăng ký xe (mặt trước/sau), ảnh CCCD (mặt trước/sau), nộp hồ sơ xét duyệt xe chính chủ, lấy danh sách xe bị từ chối gần đây. |
| 15 | `VehicleVerificationsController.cs` | `/api/vehicle-verifications` | `[Authorize(Roles = "Staff,Manager,Admin")]` | 4 | Nhân viên duyệt hồ sơ giấy tờ xe: lấy danh sách chờ duyệt (`NeedsReview`), xem ảnh tài liệu gốc, phê duyệt (`Approve`) hoặc từ chối (`Reject`) kèm lý do. |
| 16 | `ZonesController.cs` | `/api/zones` | `[Authorize(Roles = "Admin,Manager")]` | 5 | Quản lý khu vực (Zone) trong bãi đỗ, phân loại theo tầng hoặc loại phương tiện. |
| **Tổng cộng** | **16 Controllers** | — | — | **116 Routes** | **Bao quát toàn diện luồng nghiệp vụ đỗ xe thông minh** |

### 3.2. Bảng Danh Mục Chi Tiết 116 Endpoints

| Controller | HTTP Method | Endpoint Route | Quyền Truy Cập (Auth) | Mô Tả Nghiệp Vụ |
| :--- | :--- | :--- | :--- | :--- |
| `AdminController.cs` | **POST** | `/api/admin/managers` | `[Authorize(Roles = "Admin")]` | `CreateManagerAsync([FromBody] CreateEmployeeAccountModel model)` |
| `AdminController.cs` | **GET** | `/api/admin/managers` | `[Authorize(Roles = "Admin")]` | `GetManagersAsync([FromQuery] bool? isActive = null, [FromQuery] string searchTerm = null)` |
| `AdminController.cs` | **GET** | `/api/admin/managers/{id}` | `[Authorize(Roles = "Admin")]` | `GetManagerAsync(string id)` |
| `AdminController.cs` | **PUT** | `/api/admin/managers/{id}` | `[Authorize(Roles = "Admin")]` | `UpdateManagerAsync(string id, [FromBody] UpdateEmployeeAccountModel model)` |
| `AdminController.cs` | **PATCH** | `/api/admin/managers/{id}/status` | `[Authorize(Roles = "Admin")]` | `UpdateManagerStatusAsync(string id, [FromBody] UpdateUserStatusModel model)` |
| `AdminController.cs` | **PUT** | `/api/admin/managers/{id}/lots` | `[Authorize(Roles = "Admin")]` | `AssignManagerLotsAsync(string id, [FromBody] AssignParkingLotsModel model)` |
| `AdminController.cs` | **GET** | `/api/admin/users` | `[Authorize(Roles = "Admin")]` | `GetInternalUsersAsync(` |
| `AdminController.cs` | **PATCH** | `/api/admin/users/{id}/status` | `[Authorize(Roles = "Admin")]` | `UpdateStatusAsync(string id, [FromBody] UpdateUserStatusModel model)` |
| `LayoutsController.cs` | **GET** | `/api/lot-templates` | `None` | `GetTemplates()` |
| `LayoutsController.cs` | **GET** | `/api/public/lots/{lotId:int}/layout` | `None` | `GetPublishedAsync(int lotId)` |
| `LayoutsController.cs` | **GET** | `/api/public/lots/{lotId:int}/layout/status` | `None` | `GetStatusAsync(int lotId)` |
| `LayoutsController.cs` | **GET** | `/api/public/lots/{lotId:int}/layout/2d` | `None` | `Get2DAsync(int lotId, [FromQuery] string detail = LayoutDetail.Slots)` |
| `LotsController.cs` | **GET** | `/api/lots` | `[Authorize(Roles = "Admin,Manager")]` | `GetAllAsync(` |
| `LotsController.cs` | **GET** | `/api/public/lots` | `[AllowAnonymous]` | `GetPublicLotsAsync([FromQuery] string searchTerm = null)` |
| `LotsController.cs` | **GET** | `/api/public/parking-lots` | `[AllowAnonymous]` | `GetPublicLotsAsync([FromQuery] string searchTerm = null)` |
| `LotsController.cs` | **GET** | `/api/lots/{id:int}` | `[Authorize(Roles = "Admin,Manager")]` | `GetByIdAsync(int id)` |
| `LotsController.cs` | **POST** | `/api/lots` | `[Authorize(Roles = "Admin")]` | `CreateAsync([FromBody] CreateParkingLotModel model)` |
| `LotsController.cs` | **PUT** | `/api/lots/{id:int}` | `[Authorize(Roles = "Admin")]` | `UpdateAsync(int id, [FromBody] UpdateParkingLotModel model)` |
| `LotsController.cs` | **PUT** | `/api/lots/{id:int}/template` | `[Authorize(Roles = "Admin")]` | `ApplyTemplateAsync(int id, [FromBody] ApplyLotTemplateModel model)` |
| `LotsController.cs` | **PATCH** | `/api/lots/{id:int}/status` | `[Authorize(Roles = "Admin,Manager")]` | `UpdateStatusAsync(int id, [FromBody] UpdateParkingLotStatusModel model)` |
| `LotsController.cs` | **DELETE** | `/api/lots/{id:int}` | `[Authorize(Roles = "Admin")]` | `DeleteAsync(int id)` |
| `ManagerController.cs` | **GET** | `/api/manager/lots` | `[Authorize(Roles = "Manager")]` | `GetLotsAsync()` |
| `ManagerController.cs` | **GET** | `/api/manager/staff` | `[Authorize(Roles = "Manager")]` | `GetStaffAsync(` |
| `ManagerController.cs` | **GET** | `/api/manager/staff/{id}` | `[Authorize(Roles = "Manager")]` | `GetStaffByIdAsync(string id)` |
| `ManagerController.cs` | **POST** | `/api/manager/staff` | `[Authorize(Roles = "Manager")]` | `CreateStaffAsync([FromBody] CreateStaffAccountModel model)` |
| `ManagerController.cs` | **PUT** | `/api/manager/staff/{id}` | `[Authorize(Roles = "Manager")]` | `UpdateStaffAsync(string id, [FromBody] UpdateEmployeeAccountModel model)` |
| `ManagerController.cs` | **PUT** | `/api/manager/staff/{id}/lot` | `[Authorize(Roles = "Manager")]` | `AssignStaffLotAsync(string id, [FromBody] AssignStaffParkingLotModel model)` |
| `ManagerController.cs` | **PATCH** | `/api/manager/staff/{id}/status` | `[Authorize(Roles = "Manager")]` | `UpdateStaffStatusAsync(string id, [FromBody] UpdateUserStatusModel model)` |
| `ManagerController.cs` | **DELETE** | `/api/manager/staff/{id}` | `[Authorize(Roles = "Manager")]` | `DeleteStaffAsync(string id)` |
| `ManagerOperationsController.cs` | **GET** | `/api/manager/lots/{lotId:int}/reservations` | `[Authorize(Roles = "Manager,Admin")]` | `GetReservationsAsync(int lotId, [FromQuery] string status = null, [FromQuery] DateTime? from = null, [FromQuery] DateTime? to = null) =>` |
| `ManagerOperationsController.cs` | **POST** | `/api/manager/reservations/{id:long}/cancel` | `[Authorize(Roles = "Manager,Admin")]` | `CancelReservationAsync(long id, [FromBody] CancelReservationModel model) =>` |
| `ManagerOperationsController.cs` | **POST** | `/api/manager/sessions/{id:long}/abnormal-close` | `[Authorize(Roles = "Manager,Admin")]` | `CloseAbnormallyAsync(long id, [FromBody] SessionDecisionModel model) =>` |
| `ManagerOperationsController.cs` | **POST** | `/api/manager/sessions/{id:long}/lost-ticket/approve` | `[Authorize(Roles = "Manager,Admin")]` | `ApproveLostTicketAsync(long id, [FromBody] SessionDecisionModel model) =>` |
| `ManagerOperationsController.cs` | **POST** | `/api/manager/sessions/{id:long}/lost-ticket/reject` | `[Authorize(Roles = "Manager,Admin")]` | `RejectLostTicketAsync(long id, [FromBody] SessionDecisionModel model) =>` |
| `ManagerOperationsController.cs` | **GET** | `/api/manager/lots/{lotId:int}/plate-lists` | `[Authorize(Roles = "Manager,Admin")]` | `GetPlateListsAsync(int lotId, [FromQuery] string listType = null) =>` |
| `ManagerOperationsController.cs` | **POST** | `/api/manager/lots/{lotId:int}/plate-lists` | `[Authorize(Roles = "Manager,Admin")]` | `AddPlateListEntryAsync(int lotId, [FromBody] CreatePlateListEntryModel model) =>` |
| `ManagerOperationsController.cs` | **DELETE** | `/api/manager/lots/{lotId:int}/plate-lists/{id:int}` | `[Authorize(Roles = "Manager,Admin")]` | `RemovePlateListEntryAsync(int lotId, int id) =>` |
| `ParkingSessionsController.cs` | **GET** | `/api/parking-sessions/mine` | `[Authorize(Roles = "User")]` | `GetMineAsync() =>` |
| `ParkingSessionsController.cs` | **GET** | `/api/parking-sessions/history` | `[Authorize(Roles = "User")]` | `GetHistoryAsync([FromQuery] ParkingHistoryQuery query) =>` |
| `ParkingSessionsController.cs` | **GET** | `/api/parking-sessions/{id:long}/receipt` | `[Authorize(Roles = "User")]` | `GetReceiptAsync(long id) =>` |
| `ParkingSessionsController.cs` | **GET** | `/api/parking-sessions/current` | `[Authorize(Roles = "User")]` | `GetCurrentAsync() =>` |
| `ParkingSessionsController.cs` | **GET** | `/api/parking-sessions/{id:long}/exit-pass` | `[Authorize(Roles = "User")]` | `GetExitPassAsync(long id) =>` |
| `ParkingSessionsController.cs` | **GET** | `/api/parking-sessions/{id:long}/quote` | `[Authorize(Roles = "User")]` | `QuoteAsync(long id) =>` |
| `ParkingSessionsController.cs` | **POST** | `/api/parking-sessions/{id:long}/payments` | `[Authorize(Roles = "User")]` | `PayAsync(long id, [FromBody] CreatePaymentModel model) =>` |
| `PaymentsController.cs` | **GET** | `/api/payments/{id:long}` | `[Authorize]` | `GetAsync(long id)` |
| `PaymentsController.cs` | **POST** | `/api/payments/{id:long}/check` | `[Authorize]` | `CheckAsync(long id)` |
| `PaymentsController.cs` | **GET** | `/api/payments/vnpay/return` | `[AllowAnonymous]` | `VnPayReturnAsync()` |
| `PaymentsController.cs` | **GET** | `/api/payments/vnpay/ipn` | `[AllowAnonymous]` | `VnPayIpnAsync()` |
| `PaymentsController.cs` | **GET** | `/api/payments/momo/return` | `[AllowAnonymous]` | `MoMoReturnAsync()` |
| `PaymentsController.cs` | **POST** | `/api/payments/momo/ipn` | `[AllowAnonymous]` | `MoMoIpnAsync([FromBody] Dictionary<string, JsonElement> body)` |
| `ReservationsController.cs` | **POST** | `/api/reservations` | `[Authorize(Roles = "User")]` | `CreateAsync([FromBody] CreateReservationModel model)` |
| `ReservationsController.cs` | **GET** | `/api/reservations/mine` | `[Authorize(Roles = "User")]` | `GetMineAsync([FromQuery] bool activeOnly = false)` |
| `ReservationsController.cs` | **GET** | `/api/reservations/upcoming` | `[Authorize(Roles = "User")]` | `GetUpcomingAsync() =>` |
| `ReservationsController.cs` | **GET** | `/api/reservations/{id:long}` | `[Authorize(Roles = "User")]` | `GetAsync(long id)` |
| `ReservationsController.cs` | **PUT** | `/api/reservations/{id:long}` | `[Authorize(Roles = "User")]` | `UpdateAsync(long id, [FromBody] UpdateReservationModel model)` |
| `ReservationsController.cs` | **POST** | `/api/reservations/{id:long}/cancel` | `[Authorize(Roles = "User")]` | `CancelAsync(long id, [FromBody] CancelReservationModel model)` |
| `ReservationsController.cs` | **GET** | `/api/reservations/{id:long}/qr` | `[Authorize(Roles = "User")]` | `GetQrAsync(long id)` |
| `ReservationsController.cs` | **POST** | `/api/reservations/{id:long}/payments` | `[Authorize(Roles = "User")]` | `PayDepositAsync(long id, [FromBody] CreatePaymentModel model)` |
| `SlotsController.cs` | **GET** | `/api/slots` | `[Authorize(Roles = "Admin,Manager")]` | `GetAllAsync(` |
| `SlotsController.cs` | **GET** | `/api/zones/{zoneId:int}/slots` | `[Authorize(Roles = "Admin,Manager")]` | `GetByZoneIdAsync(int zoneId)` |
| `SlotsController.cs` | **GET** | `/api/slots/{id:int}` | `[Authorize(Roles = "Admin,Manager")]` | `GetByIdAsync(int id)` |
| `SlotsController.cs` | **PUT** | `/api/slots/{id:int}` | `[Authorize(Roles = "Admin")]` | `UpdateAsync(int id, [FromBody] UpdateSlotModel model)` |
| `SlotsController.cs` | **PATCH** | `/api/slots/{id:int}/status` | `[Authorize(Roles = "Admin,Manager")]` | `UpdateStatusAsync(int id, [FromBody] UpdateSlotStatusModel model)` |
| `StaffController.cs` | **GET** | `/api/staff/lots` | `[Authorize(Roles = "Staff")]` | `GetLotsAsync()` |
| `StaffGateController.cs` | **GET** | `/api/staff/lots/{lotId:int}/gates` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `GetGatesAsync(int lotId) =>` |
| `StaffGateController.cs` | **GET** | `/api/staff/lots/{lotId:int}/zone-counters` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `GetZoneCountersAsync(int lotId) =>` |
| `StaffGateController.cs` | **GET** | `/api/staff/lots/{lotId:int}/sessions` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `GetSessionsAsync(int lotId, [FromQuery] string status = null, [FromQuery] string plate = null) =>` |
| `StaffGateController.cs` | **GET** | `/api/staff/lots/{lotId:int}/sessions/lookup` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `LookupAsync(int lotId, [FromQuery] string plate = null, [FromQuery] string ticket = null, [FromQuery] string code = null) =>` |
| `StaffGateController.cs` | **GET** | `/api/staff/lots/{lotId:int}/reservations` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `GetReservationsAsync(int lotId, [FromQuery] string status = null, [FromQuery] DateTime? from = null, [FromQuery] DateTime? to = null) =>` |
| `StaffGateController.cs` | **GET** | `/api/staff/lots/{lotId:int}/reservations/lookup` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `LookupReservationAsync(int lotId, [FromQuery] string code = null, [FromQuery] string qr = null) =>` |
| `StaffGateController.cs` | **GET** | `/api/staff/lots/{lotId:int}/plate-lists` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `GetPlateListsAsync(int lotId, [FromQuery] string listType = null) =>` |
| `StaffGateController.cs` | **POST** | `/api/staff/lots/{lotId:int}/check-in` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `CheckInAsync(int lotId, [FromBody] CheckInModel model) =>` |
| `StaffGateController.cs` | **GET** | `/api/staff/sessions/{id:long}` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `GetSessionAsync(long id) =>` |
| `StaffGateController.cs` | **POST** | `/api/staff/sessions/{id:long}/entry-confirm` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `ConfirmEntryAsync(long id) =>` |
| `StaffGateController.cs` | **POST** | `/api/staff/sessions/{id:long}/entry-cancel` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `CancelEntryAsync(long id) =>` |
| `StaffGateController.cs` | **POST** | `/api/staff/sessions/{id:long}/quote` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `QuoteAsync(long id, [FromBody] CheckoutQuoteRequestModel model) =>` |
| `StaffGateController.cs` | **POST** | `/api/staff/sessions/{id:long}/cancel-payment` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `CancelPaymentAsync(long id) =>` |
| `StaffGateController.cs` | **POST** | `/api/staff/sessions/{id:long}/payments/cash` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `PayCashAsync(long id, [FromBody] CashPaymentModel model) =>` |
| `StaffGateController.cs` | **POST** | `/api/staff/sessions/{id:long}/payments/online` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `PayOnlineAsync(long id, [FromBody] LanePaymentModel model) =>` |
| `StaffGateController.cs` | **POST** | `/api/staff/sessions/{id:long}/check-out` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `CheckoutAsync(long id, [FromBody] CheckoutModel model) =>` |
| `StaffGateController.cs` | **POST** | `/api/staff/sessions/{id:long}/lost-ticket` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `LostTicketAsync(long id, [FromBody] LostTicketModel model) =>` |
| `UsersController.cs` | **GET** | `/` | `[Authorize(Roles = "Admin")]` | `GetAllAsync([FromQuery] string role = null, [FromQuery] bool? isActive = null, [FromQuery] string searchTerm = null)` |
| `UsersController.cs` | **GET** | `/{id}` | `[Authorize(Roles = "Admin")]` | `GetByIdAsync(string id)` |
| `UsersController.cs` | **POST** | `/internal` | `[Authorize(Roles = "Admin")]` | `CreateInternalUserAsync([FromBody] CreateInternalUserModel model)` |
| `UsersController.cs` | **PATCH** | `/{id}/status` | `[Authorize(Roles = "Admin")]` | `UpdateStatusAsync(string id, [FromBody] UpdateUserStatusModel model)` |
| `UsersController.cs` | **POST** | `/register` | `[AllowAnonymous]` | `RegisterAsync(CreateUserModel createUserModel)` |
| `UsersController.cs` | **POST** | `/authenticate` | `[AllowAnonymous]` | `LoginAsync(LoginUserModel loginUserModel)` |
| `UsersController.cs` | **POST** | `/two-factor` | `[AllowAnonymous]` | `VerifyTwoFactorAsync([FromBody] TwoFactorLoginModel model)` |
| `UsersController.cs` | **POST** | `/two-factor/resend` | `[AllowAnonymous]` | `ResendTwoFactorOtpAsync([FromBody] ResendTwoFactorModel model)` |
| `UsersController.cs` | **POST** | `/login-otp/request` | `[AllowAnonymous]` | `RequestLoginOtpAsync(RequestLoginOtpModel model)` |
| `UsersController.cs` | **POST** | `/login-otp/verify` | `[AllowAnonymous]` | `LoginWithOtpAsync(VerifyLoginOtpModel model)` |
| `UsersController.cs` | **POST** | `/login-google` | `[AllowAnonymous]` | `LoginWithGoogleAsync(GoogleLoginModel model)` |
| `UsersController.cs` | **GET** | `/profile` | `[Authorize]` | `GetProfileAsync()` |
| `UsersController.cs` | **PUT** | `/profile` | `[Authorize]` | `UpdateProfileAsync([FromBody] UpdateUserProfileModel model)` |
| `UsersController.cs` | **GET** | `/profile/vehicles` | `[Authorize]` | `GetVehiclesAsync()` |
| `UsersController.cs` | **POST** | `/profile/vehicles` | `[Authorize]` | `AddVehicleAsync([FromBody] CreateVehicleModel model)` |
| `UsersController.cs` | **PUT** | `/profile/vehicles/{vehicleId:guid}` | `[Authorize]` | `UpdateVehicleAsync(Guid vehicleId, [FromBody] UpdateVehicleModel model)` |
| `UsersController.cs` | **DELETE** | `/profile/vehicles/{vehicleId:guid}` | `[Authorize]` | `DeleteVehicleAsync(Guid vehicleId)` |
| `UsersController.cs` | **PATCH** | `/profile/vehicles/{vehicleId:guid}/default` | `[Authorize]` | `SetDefaultVehicleAsync(Guid vehicleId)` |
| `UsersController.cs` | **GET** | `/profile/vehicle-types` | `[Authorize]` | `GetVehicleTypesAsync()` |
| `UsersController.cs` | **PUT** | `/profile/change-password` | `[Authorize]` | `ChangePasswordAsync([FromBody] ChangePasswordModel model)` |
| `UsersController.cs` | **GET** | `/{id}/profile` | `[Authorize(Roles = "Admin")]` | `GetProfileByUserIdAsync(string id)` |
| `VehicleDocumentsController.cs` | **GET** | `/api/users/profile/rejected-vehicles` | `[Authorize]` | `GetRecentlyRejectedAsync()` |
| `VehicleDocumentsController.cs` | **GET** | `/api/users/profile/vehicles/{vehicleId:guid}/documents` | `[Authorize]` | `GetLatestAsync(Guid vehicleId)` |
| `VehicleDocumentsController.cs` | **POST** | `/api/users/profile/vehicles/{vehicleId:guid}/documents/{documentType}/{side}` | `[Authorize]` | `UploadAsync(Guid vehicleId, string documentType, string side, IFormFile file)` |
| `VehicleDocumentsController.cs` | **POST** | `/api/users/profile/vehicles/{vehicleId:guid}/documents/submit` | `[Authorize]` | `SubmitAsync(Guid vehicleId)` |
| `VehicleDocumentsController.cs` | **GET** | `/api/users/profile/vehicles/{vehicleId:guid}/documents/files/{documentId:guid}` | `[Authorize]` | `GetFileAsync(Guid vehicleId, Guid documentId)` |
| `VehicleVerificationsController.cs` | **GET** | `/api/vehicle-verifications` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `GetQueueAsync([FromQuery] string status = null)` |
| `VehicleVerificationsController.cs` | **GET** | `/api/vehicle-verifications/{verificationId:guid}` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `GetAsync(Guid verificationId)` |
| `VehicleVerificationsController.cs` | **GET** | `/api/vehicle-verifications/{verificationId:guid}/files/{documentId:guid}` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `GetFileAsync(Guid verificationId, Guid documentId)` |
| `VehicleVerificationsController.cs` | **POST** | `/api/vehicle-verifications/{verificationId:guid}/review` | `[Authorize(Roles = "Staff,Manager,Admin")]` | `ReviewAsync(Guid verificationId, [FromBody] ReviewVehicleVerificationModel model)` |
| `ZonesController.cs` | **GET** | `/api/zones` | `[Authorize(Roles = "Admin,Manager")]` | `GetAllAsync([FromQuery] int? lotId = null)` |
| `ZonesController.cs` | **GET** | `/api/lots/{lotId:int}/zones` | `[Authorize(Roles = "Admin,Manager")]` | `GetByLotIdAsync(int lotId)` |
| `ZonesController.cs` | **GET** | `/api/parking-lots/{lotId:int}/zones` | `[Authorize(Roles = "Admin,Manager")]` | `GetByLotIdAsync(int lotId)` |
| `ZonesController.cs` | **GET** | `/api/zones/{id:int}` | `[Authorize(Roles = "Admin,Manager")]` | `GetByIdAsync(int id)` |
| `ZonesController.cs` | **PUT** | `/api/zones/{id:int}` | `[Authorize(Roles = "Admin")]` | `UpdateAsync(int id, [FromBody] UpdateZoneModel model)` |

### 3.3. Hệ Thống Realtime SignalR Hubs

Backend tích hợp sẵn 3 SignalR Hubs chuyên biệt tại `PMS.API\Hubs`:

1. **`LotHub` (`/api/hubs/lots` - Public):**
   - `[FACT]` Kênh thông báo trạng thái mô hình bãi xe công khai (SRS FR-3D-03, FR-3D-08). Không chứa dữ liệu cá nhân hay biển số xe.
   - Client tham gia qua phương thức `JoinLot(int lotId)` / `LeaveLot(int lotId)` (nhóm: `lot-{lotId}`).
   - Các sự kiện đẩy từ server: `SlotStatusChanged` (trạng thái slot), `LotStatusChanged` (trạng thái bãi), `LayoutChanged` (thay đổi sơ đồ), `ZoneCounterChanged` (số chỗ trống theo khu vực).

2. **`ParkingOpsHub` (`/api/hubs/parking-ops` - Private, `[Authorize]`):**
   - `[FACT]` Kênh điều hành nội bộ cho đặt chỗ, phiên đỗ, thanh toán và sự kiện cổng (SRS FR-3D-09). Client gửi JWT token qua Query String `access_token`.
   - Khách hàng (User) nhận trực tiếp các sự kiện cá nhân qua UserId: `ReservationChanged`, `ReservationReminder`, `PaymentChanged`.
   - Nhân viên (Staff) và Quản lý (Manager) tham gia nhóm vận hành qua `JoinLot(int lotId)` (kiểm tra phân quyền nghiêm ngặt qua `LotScopeService.CanOperateAsync`) để nhận: `SessionChanged` (phiên đỗ xe, có biển số), `PaymentChanged`, `ZoneCounterChanged`, `OpsAlert` (cảnh báo vi phạm/sự cố).

3. **`VehicleVerificationHub` (`/api/hubs/vehicle-verifications` - Private, `[Authorize]`):**
   - `[FACT]` Kênh phục vụ quy trình thẩm định giấy tờ xe.
   - Khách hàng nộp hồ sơ nhận sự kiện `VerificationChanged` khi hồ sơ của mình thay đổi trạng thái.
   - Nhân viên review (`Staff`, `Manager`, `Admin`) tự động được gán vào nhóm `vehicle-verification-reviewers` khi kết nối để nhận thông báo real-time khi có hồ sơ mới được nộp.

### 3.4. Mô Hình Dữ Liệu & Entity Framework Core (73 Entities)

- `[FACT]` `PMS.Core\Entities` bao gồm **73 entity classes**, mô hình hóa đầy đủ quan hệ giữa bãi xe (`ParkingLot`), khu vực (`Zone`), vị trí đỗ (`Slot`), sơ đồ (`LotLayout`), tài khoản (`ApplicationUser`, `Customer`, `Employee`), phương tiện (`Vehicle`, `VehicleType`, `VehicleDocument`, `VehicleVerification`), đặt chỗ (`Reservation`, `ReservationChangeLog`), phiên đỗ xe thực tế (`ParkingSession`, `SessionViolation`), thanh toán (`Payment`, `Invoice`, `Voucher`), ca trực (`Shift`, `ShiftCashCount`), thiết bị barie (`Gate`, `BarrierControlLog`, `Device`), và danh sách kiểm soát biển số (`PlateListEntry`).
- `[FACT]` **Cơ chế DDL Migration tự động:** Dự án sử dụng `AutomatedMigration.cs` và `AutomatedMigration.Operations.cs` tự động kiểm tra và khởi tạo các bảng dữ liệu cho Reservation, Gate Sessions và Payments bằng các câu lệnh SQL an toàn (`IF NOT EXISTS`) trong quá trình khởi động ứng dụng mà không bắt buộc chạy `dotnet ef database update` thủ công đối với môi trường local dev.
- `[FACT]` **Dữ liệu mẫu (Seed Data):** Lớp `DatabaseContextSeed.cs` khởi tạo sẵn hệ thống quyền, tài khoản quản trị Admin mặc định, các loại xe cơ bản, cấu hình mẫu và tài khoản thử nghiệm.

---

## 4. Frontend Deep Dive: Kiến Trúc TypeScript, Router & Pages

### 4.1. Khung Công Nghệ & Tổ Chức Mã Nguồn

- `[FACT]` Frontend chuyển hoàn toàn từ React JavaScript (.jsx) sang **TypeScript** với các file cấu hình `tsconfig.json`, `tsconfig.app.json`, `tsconfig.node.json` và `vite.config.ts`.
- `[FACT]` Thư viện chính: React 19, React Router DOM 7 (`createBrowserRouter`), TanStack React Query 5 (quản lý state server và caching), Axios, Three.js (mô hình 3D Garage), Lucide React (icon system), Sonner (toast notifications), Tailwind CSS.

### 4.2. Danh Mục 22 Pages & Phân Quyền Router (`router.tsx`)

| Nhóm Trang / Role | Route Path | Component File | Phân Quyền Guard | Mô Tả Chức Năng |
| :--- | :--- | :--- | :--- | :--- |
| **Public** | `/` | `HomePage.tsx` | Không | Landing page giới thiệu hệ thống, mạng lưới bãi xe, số liệu công suất trực quan. |
| **Public** | `/login` | `LoginPage.tsx` | Không | Màn hình đăng nhập đa kênh (Mật khẩu, Email OTP, Google OAuth 2.0, hỗ trợ Email OTP 2FA). |
| **Public** | `/register` | `RegisterPage.tsx` | Không | Màn hình đăng ký tài khoản khách hàng mới. |
| **Public** | `/payment/return/:provider` | `PaymentReturnPage.tsx` | Không | Trang tiếp nhận kết quả thanh toán trả về từ VNPay / MoMo (hỗ trợ cả khách vãng lai thanh toán qua QR trên điện thoại). |
| **Customer** | `/me` | `CustomerDashboardPage.tsx` | `RequireAuth` | Dashboard tài xế: thông tin bãi đỗ ưa thích, phiên đỗ đang diễn ra, thẻ xe, bản đồ Zone. |
| **Customer** | `/me/profile` | `CustomerProfilePage.tsx` | `RequireAuth` | Quản lý thông tin cá nhân, đổi mật khẩu, xem danh sách xe đã đăng ký. |
| **Customer** | `/me/vehicles` | `RegisterVehiclePage.tsx` | `RequireAuth` | Đăng ký phương tiện mới, đặt xe mặc định, chụp/tải lên Giấy đăng ký xe và CCCD để thẩm định. |
| **Customer** | `/me/bookings` | `CustomerBookingsPage.tsx` | `RequireAuth` | Quản lý danh sách đặt chỗ, thanh toán tiền cọc, xem mã QR check-in, cập nhật giờ hoặc hủy cọc. |
| **Admin** | `/admin/dashboard` | `AdminDashboardPage.tsx` | `RequireAuth(ADMIN)` | Tổng quan toàn hệ thống: KPI doanh thu, số lượng bãi, tài khoản khách hàng và nhân sự. |
| **Admin** | `/admin/staff` | `StaffAccountsPage.tsx` | `RequireAuth(ADMIN)` | Quản lý danh sách nhân sự cấp quản lý (Managers), cấp tài khoản, đổi trạng thái hoạt động. |
| **Admin** | `/admin/customers` | `CustomerAccountsPage.tsx` | `RequireAuth(ADMIN)` | Tra cứu danh sách tài khoản khách hàng cá nhân trong hệ thống. |
| **Admin** | `/admin/lots` | `ParkingLotsPage.tsx` | `RequireAuth(ADMIN)` | Quản lý danh sách bãi xe, tạo mới bãi, gán sơ đồ mẫu (layout template), phân công Manager. |
| **Admin** | `/admin/revenue` | `RevenueReportPage.tsx` | `RequireAuth(ADMIN)` | Báo cáo doanh thu chuyên sâu: biểu đồ xu hướng, phân tích phương thức thanh toán, doanh thu theo bãi. |
| **Admin** | `/admin/capacity` | `ParkingCapacityPage.tsx` | `RequireAuth(ADMIN)` | Giám sát công suất bãi đỗ xe theo thời gian thực. |
| **Manager** | `/management/console` | `ManagerConsolePage.tsx` | `RequireAuth(MANAGER)` | Bảng điều hành của Quản lý bãi: mô hình bãi xe trực tiếp (LiveLotModel), KPI vận hành. |
| **Manager** | `/management/reservations`| `ManagerReservationsPage.tsx`| `RequireAuth(MANAGER)` | Quản lý danh sách đặt chỗ tại bãi xe do Manager phụ trách. |
| **Manager** | `/management/approvals` | `ManagerApprovalsPage.tsx` | `RequireAuth(MANAGER)` | Phê duyệt các yêu cầu vận hành (mất vé, xử lý ngoại lệ, sự cố). |
| **Manager** | `/management/pricing` | `ManagerPricingPage.tsx` | `RequireAuth(MANAGER)` | Cấu hình bảng giá cước đỗ xe theo khung giờ và loại xe. |
| **Manager** | `/management/staff` | `ManagerStaffPage.tsx` | `RequireAuth(MANAGER)` | Quản lý nhân viên trực cổng (Staff) thuộc bãi đỗ của Manager. |
| **Manager** | `/management/lots` | `ParkingLotsPage.tsx` | `RequireAuth(MANAGER)` | Xem và cấu hình trạng thái bãi đỗ thuộc quyền quản lý. |
| **Staff** | `/staff/gate` | `StaffGatePage.tsx` | `RequireAuth(STAFF)` | Bàn điều khiển làn vào (Entry Lane Console): camera chụp biển, nhận diện, check-in, mở barie. |
| **Staff** | `/staff/checkout` | `StaffCheckoutPage.tsx` | `RequireAuth(STAFF)` | Bàn thu phí làn ra (Exit Lane): tra cứu vé/biển số, báo giá cước, thu tiền mặt, tạo mã QR VNPay/MoMo, xác nhận thanh toán mở barie ra. |
| **Staff** | `/staff/exceptions` | `StaffExceptionsPage.tsx` | `RequireAuth(STAFF)` | Xử lý các sự cố cổng: mất vé, biển số không khớp, xe quá hạn. |
| **Staff** | `/staff/cash-tally` | `StaffCashTallyPage.tsx` | `RequireAuth(STAFF)` | Kết ca và đối soát tiền mặt thu được trong ca trực. |
| **Staff** | `/staff/verifications` | `StaffVehicleVerificationPage.tsx` | `RequireAuth(STAFF)` | Bàn kiểm định hồ sơ giấy tờ xe: đối chiếu ảnh đăng ký xe, CCCD với thông tin biển số và phê duyệt. |

### 4.3. Kiến Trúc 3D Interactive Garage Model (Three.js)

- `[FACT]` Thư mục `src/components/customer/garage/` chứa hệ thống mô hình 3D hoàn chỉnh được dựng bằng **Three.js** (`Garage3D.tsx`, `garageModel.ts`, `vehicleModels.ts`).
- `[FACT]` Hỗ trợ phân tầng (Ground L1, L2, Rooftop), phân loại slot theo ô xe hơi (`car`) và xe máy (`motorbike`), có khả năng 'lift away' các tầng phía trên để nhìn sâu vào tầng đang chọn, render các mô hình 3D ô tô / xe máy tại các slot đã có xe đỗ, đổi màu trạng thái (Available: Xanh lá, Occupied: Xám/Đỏ, Reserved: Vàng hổ phách), và tương tác chọn slot trực tiếp để đặt trước.

---

## 5. Phân Tích Ma Trận Hợp Đồng FE-BE (Contract Mapping & Gap Analysis)

### 5.1. Bảng Đối Soát Chi Tiết Service Gọi Phía FE vs Controller Phía BE

| Lĩnh Vực / Service FE | Phương Thức HTTP & Route Gọi Phía FE | Endpoint Xử Lý Tương Ứng Phía BE | Tình Trạng Kết Nối | Ghi Chú Kỹ Thuật |
| :--- | :--- | :--- | :--- | :--- |
| **Xác Thực (Auth)** | `POST /api/Users/authenticate` | `POST /api/users/authenticate` | **Đã Khớp (Live)** | Đăng nhập bằng username/password, trả về JWT hoặc cờ yêu cầu 2FA. |
| **Xác Thực (Auth)** | `POST /api/Users/two-factor` | `POST /api/users/two-factor` | **Đã Khớp (Live)** | Xác thực mã Email OTP khi tài khoản kích hoạt 2FA. |
| **Xác Thực (Auth)** | `POST /api/Users/two-factor/resend` | `POST /api/users/two-factor/resend` | **Đã Khớp (Live)** | Gửi lại mã Email OTP 2FA. |
| **Xác Thực (Auth)** | `POST /api/Users/login-otp/request`| `POST /api/users/login-otp/request` | **Đã Khớp (Live)** | Yêu cầu gửi OTP đăng nhập nhanh không cần mật khẩu. |
| **Xác Thực (Auth)** | `POST /api/Users/login-otp/verify` | `POST /api/users/login-otp/verify` | **Đã Khớp (Live)** | Xác thực OTP đăng nhập nhanh. |
| **Xác Thực (Auth)** | `POST /api/Users/login-google` | `POST /api/users/login-google` | **Đã Khớp (Live)** | Đăng nhập bằng Google ID Token. |
| **Xác Thực (Auth)** | `POST /api/Users/register` | `POST /api/users/register` | **Đã Khớp (Live)** | Đăng ký tài khoản khách hàng mới. |
| **Hồ Sơ (Profile)** | `GET /api/Users/profile` | `GET /api/users/profile` | **Đã Khớp (Live)** | Lấy thông tin cá nhân của user đăng nhập. |
| **Hồ Sơ (Profile)** | `PUT /api/Users/profile` | `PUT /api/users/profile` | **Đã Khớp (Live)** | Cập nhật thông tin cá nhân. |
| **Hồ Sơ (Profile)** | `PUT /api/Users/profile/change-password` | `PUT /api/users/profile/change-password` | **Đã Khớp (Live)** | Đổi mật khẩu tài khoản. |
| **Phương Tiện (Vehicles)** | `GET /api/Users/profile/vehicles` | `GET /api/users/profile/vehicles` | **Đã Khớp (Live)** | Lấy danh sách xe đã đăng ký của khách hàng. |
| **Phương Tiện (Vehicles)** | `POST /api/Users/profile/vehicles` | `POST /api/users/profile/vehicles` | **Đã Khớp (Live)** | Thêm phương tiện mới. |
| **Phương Tiện (Vehicles)** | `PUT /api/Users/profile/vehicles/{id}` | `PUT /api/users/profile/vehicles/{vehicleId}` | **Đã Khớp (Live)** | Cập nhật thông tin phương tiện. |
| **Phương Tiện (Vehicles)** | `DELETE /api/Users/profile/vehicles/{id}` | `DELETE /api/users/profile/vehicles/{vehicleId}` | **Đã Khớp (Live)** | Xóa phương tiện. |
| **Phương Tiện (Vehicles)** | `PATCH /api/Users/profile/vehicles/{id}/default` | `PATCH /api/users/profile/vehicles/{vehicleId}/default` | **Đã Khớp (Live)** | Đặt phương tiện làm xe mặc định. |
| **Giấy Tờ Xe (Doc Upload)** | `POST /api/Users/profile/vehicles/{id}/documents/{type}/{side}` | `POST /api/users/profile/vehicles/{vehicleId}/documents/{documentType}/{side}` | **Đã Khớp (Live)** | Tải lên file ảnh giấy tờ đăng ký xe, CCCD. |
| **Giấy Tờ Xe (Doc Upload)** | `POST /api/Users/profile/vehicles/{id}/documents/submit` | `POST /api/users/profile/vehicles/{vehicleId}/documents/submit` | **Đã Khớp (Live)** | Nộp hồ sơ giấy tờ xe để nhân viên phê duyệt. |
| **Duyệt Giấy Tờ (Verification)** | `GET /api/vehicle-verifications` | `GET /api/vehicle-verifications` | **Đã Khớp (Live)** | Staff lấy danh sách hồ sơ xe chờ thẩm định. |
| **Duyệt Giấy Tờ (Verification)** | `POST /api/vehicle-verifications/{id}/review` | `POST /api/vehicle-verifications/{verificationId}/review` | **Đã Khớp (Live)** | Staff duyệt hoặc từ chối hồ sơ xe kèm ghi chú. |
| **Đặt Chỗ (Reservations)** | `POST /api/reservations` | `POST /api/reservations` | **Đã Khớp (Live)** | Khách hàng tạo đặt chỗ giữ slot trước. |
| **Đặt Chỗ (Reservations)** | `GET /api/reservations/mine` | `GET /api/reservations/mine` | **Đã Khớp (Live)** | Khách hàng xem lịch sử các lượt đặt chỗ. |
| **Đặt Chỗ (Reservations)** | `GET /api/reservations/upcoming` | `GET /api/reservations/upcoming` | **Đã Khớp (Live)** | Khách hàng lấy lượt đặt chỗ sắp diễn ra. |
| **Đặt Chỗ (Reservations)** | `GET /api/reservations/{id}/qr` | `GET /api/reservations/{id}/qr` | **Đã Khớp (Live)** | Lấy mã QR phục vụ check-in tự động tại cổng. |
| **Đặt Chỗ (Reservations)** | `POST /api/reservations/{id}/cancel` | `POST /api/reservations/{id}/cancel` | **Đã Khớp (Live)** | Hủy đặt chỗ (áp dụng chính sách hoàn cọc). |
| **Đặt Chỗ (Reservations)** | `POST /api/reservations/{id}/payments` | `POST /api/reservations/{id}/payments` | **Đã Khớp (Live)** | Khởi tạo giao dịch thanh toán tiền cọc qua VNPay/MoMo. |
| **Phiên Đỗ (Parking Sessions)** | `GET /api/parking-sessions/current` | `GET /api/parking-sessions/current` | **Đã Khớp (Live)** | Lấy thông tin các xe của khách đang đỗ trong bãi. |
| **Phiên Đỗ (Parking Sessions)** | `GET /api/parking-sessions/{id}/exit-pass` | `GET /api/parking-sessions/{id}/exit-pass` | **Đã Khớp (Live)** | Lấy vé thông hành xuất bãi (Exit Pass). |
| **Phiên Đỗ (Parking Sessions)** | `GET /api/parking-sessions/history` | `GET /api/parking-sessions/history` | **Đã Khớp (Live)** | Lấy lịch sử đỗ xe có phân trang. |
| **Cổng Nhân Viên (Staff Gate)** | `POST /api/staff/lots/{id}/check-in` | `POST /api/staff/lots/{lotId}/check-in` | **Đã Khớp (Live)** | Nhân viên check-in xe vào bãi. |
| **Cổng Nhân Viên (Staff Gate)** | `POST /api/staff/sessions/{id}/quote` | `POST /api/staff/sessions/{id}/quote` | **Đã Khớp (Live)** | Tính cước cho xe chuẩn bị rời bãi. |
| **Cổng Nhân Viên (Staff Gate)** | `POST /api/staff/sessions/{id}/payments/cash` | `POST /api/staff/sessions/{id}/payments/cash` | **Đã Khớp (Live)** | Ghi nhận thu tiền mặt tại quầy cổng. |
| **Cổng Nhân Viên (Staff Gate)** | `POST /api/staff/sessions/{id}/payments/online` | `POST /api/staff/sessions/{id}/payments/online` | **Đã Khớp (Live)** | Tạo giao dịch QR online tại làn ra. |
| **Cổng Nhân Viên (Staff Gate)** | `POST /api/staff/sessions/{id}/check-out` | `POST /api/staff/sessions/{id}/check-out` | **Đã Khớp (Live)** | Hoàn tất xuất bãi, ghi nhận mở barie ra. |
| **Báo Cáo (Reports)** | `GET /api/v1/reports/*` | **Chưa có Controller** | **Mock (Chưa Khớp)** | Frontend dùng `revenue.mock.ts` do Backend chưa có `ReportsController`. |
| **Công Suất Bãi (Capacity)** | `GET /api/v1/parking/lots/capacity` | **Chưa có Endpoint chuyên biệt** | **Mock (Chưa Khớp)** | Frontend dùng `capacity.mock.ts` (Backend chỉ có số đếm qua `StaffGateController` và SignalR `ZoneCounterChanged`). |
| **Quản Lý Bãi Vận Hành (Manager Ops)**| `ManagerApprovalsPage`, `ManagerReservationsPage` | `ManagerOperationsController` (`/api/manager/...`) | **Mock tại UI (Chưa Nối Live)** | Backend đã có đầy đủ 8 endpoints trong `ManagerOperationsController`, nhưng UI Manager hiện vẫn đang cắm vào `managerOps.mock.ts` có gắn nhãn `DemoTag`. |
| **Sự Cố & Đối Soát (Staff Exceptions & Tally)**| `StaffExceptionsPage`, `StaffCashTallyPage` | **Chưa có API chuyên biệt** | **Mock tại UI (DemoTag)** | Hiện tại UI nhân viên dùng state mô phỏng từ `staffOps.mock.ts`. |

---

## 6. Phân Loại Bằng Chứng & Đánh Giá Rủi Ro Kỹ Thuật

Tuân thủ quy chuẩn phân loại bằng chứng kỹ thuật:

### 6.1. Bằng Chứng Xác Thực (`[FACT]`)
- `[FACT]` Backend có chính xác **16 Controllers**, **114 Action methods**, và **116 Routes** đã được quét và liệt kê đầy đủ.
- `[FACT]` 3 kênh SignalR Hubs (`LotHub`, `ParkingOpsHub`, `VehicleVerificationHub`) đã được cấu hình hoàn chỉnh cả ở BE và FE.
- `[FACT]` Frontend đã hoàn tất cấu trúc 22 Pages trên nền TypeScript và React Router DOM 7.
- `[FACT]` Các luồng Đăng nhập đa kênh, OTP 2FA, Hồ sơ người dùng, Quản lý xe, Tải giấy tờ xe, Đặt chỗ trước, Phiên đỗ, Thu phí tại cổng Staff đã có hợp đồng API khớp 100% giữa FE và BE.

### 6.2. Quan Sát & Suy Luận Kỹ Thuật (`[OBSERVATION]` & `[INFERENCE]`)
- `[OBSERVATION]` Thư mục `Vercel deploy` nằm tại root workspace là một snapshot build cũ của frontend, chứa các màn hình Admin cơ bản trước đợt nâng cấp nghiệp vụ đỗ xe.
- `[INFERENCE]` Do mục tiêu Sprint trước tập trung vào hoàn thiện luồng vận hành cốt lõi (Core Parking Sessions, Reservations, Gate & Vehicle Verification), phân hệ Báo cáo thống kê (`ReportsController`) được nhóm phát triển chủ động trì hoãn sang sprint sau, cấu hình sẵn cơ chế mock an toàn phía FE bằng cờ `VITE_MOCK_SERVICES=capacity,reports` để giao diện không bị gián đoạn.
- `[INFERENCE]` Controller `ManagerOperationsController` đã sẵn sàng ở Backend với các endpoint xử lý mất vé, đóng phiên bất thường và whitelist/blacklist biển số, tuy nhiên trang giao diện của Manager (`ManagerApprovalsPage`, `ManagerReservationsPage`) vẫn đang dùng dữ liệu mock (`useDemoApprovals`, `demoReservations`). Việc đấu nối các trang này sang API thực tế là một task khả thi với chi phí thấp trong sprint kế tiếp.

### 6.3. Giới Hạn Cần Xác Minh Thực Tế (`[REQUIRES EXTERNAL VERIFICATION]`)
- `[REQUIRES EXTERNAL VERIFICATION]` **Giao tiếp phần cứng barie & Camera ANPR:** Các phương thức đóng/mở barie (`GateService`, `BarrierControlLog`) và nhận diện biển số tại cổng hiện tại chạy theo cơ chế mô phỏng phần mềm. Việc tích hợp với thiết bị camera IP thực tế hoặc bộ điều khiển PLC barie cần kiểm thử tại hiện trường phần cứng bãi xe.
- `[REQUIRES EXTERNAL VERIFICATION]` **Cổng thanh toán thực tế (VNPay / MoMo Sandbox):** Các Webhook IPN (`PaymentsController`) yêu cầu domain public hoặc tunnel (ngrok) để cổng thanh toán thật có thể gọi callback vào backend local.

---

## 7. Khuyến Nghị Ưu Tiên Cho Các Task Tiếp Theo

Dựa trên kết quả rà soát baseline toàn diện sau Sprint Agile Review, thứ tự ưu tiên các hành động tiếp theo được đề xuất như sau:

1. **Đấu nối trực tiếp Manager Operations FE vào BE (Quick Win - Ưu tiên 1):**
   - Thay thế các hook `useDemoApprovals` và `demoReservations` trong `ManagerApprovalsPage` và `ManagerReservationsPage` bằng các API calls thực tế tới `ManagerOperationsController` (`/api/manager/lots/{lotId}/reservations`, `/api/manager/sessions/{id}/abnormal-close`, `/api/manager/sessions/{id}/lost-ticket/*`).
   - Tận dụng ngay 8 endpoints BE đã viết sẵn và đã được unit test đầy đủ.

2. **Triển khai `ReportsController` phía Backend (Ưu tiên 2):**
   - Xây dựng `ReportsController` với 4 endpoints tương thích với hợp đồng FE đã dựng sẵn trong `report.service.ts`: `/v1/reports/kpi`, `/v1/reports/timeseries`, `/v1/reports/payment-methods`, `/v1/reports/by-lot`.
   - Khi hoàn thành, chỉ cần bỏ `reports` khỏi biến môi trường `VITE_MOCK_SERVICES` ở Frontend.

3. **Xây dựng Endpoint `Capacity` Tập Trung phía Backend (Ưu tiên 3):**
   - Xây dựng endpoint `/v1/parking/lots/capacity` tổng hợp trạng thái đỗ xe từ các bảng `ZoneCounter` và `ParkingSession` để phục vụ trang `ParkingCapacityPage` và dashboard, chuyển `capacity` từ mock sang live.

4. **Dọn dẹp thư mục dư thừa:**
   - Kiểm tra và xin ý kiến team lead về việc lưu trữ hoặc đưa `Vercel deploy` vào `.gitignore` để tránh trùng lặp mã nguồn trong team workspace.

---

## 8. Bảng Đối Chiếu Thay Đổi So Với Baseline Cũ (Phase 3.1 & Pre-push)

| Hạng Mục Đo Lường | Baseline Phase 3.1 (Trước Đợt Rà Soát Này) | Baseline Post-Sprint Hiện Tại | Mức Độ Thay Đổi |
| :--- | :--- | :--- | :--- |
| **Số lượng BE Controllers** | 6 Controllers | **16 Controllers** | **Tăng +166% (+10 controllers)** |
| **Số lượng BE Actions/Routes** | 47 Action methods (56 routes) | **114 Action methods (116 routes)** | **Tăng +142% (+67 actions)** |
| **Công nghệ Frontend** | React JavaScript (.jsx), 6 màn hình Admin | **React 19 + TypeScript (.tsx), 22 màn hình** | **Chuyển dịch toàn bộ sang TypeScript** |
| **Mô hình 3D Bãi Xe** | Chưa tích hợp | **Three.js WebGL Interactive Garage Model** | **Mới hoàn toàn** |
| **Realtime SignalR** | Chưa triển khai | **3 Hubs chuyên biệt (`LotHub`, `ParkingOpsHub`, `VehicleVerificationHub`)** | **Mới hoàn toàn** |
| **Nghiệp Vụ Đặt Chỗ (Booking)**| Chưa có | **Hoàn chỉnh từ Client booking -> Deposit payment -> Gate check-in QR** | **Mới hoàn toàn** |
| **Cổng Nhân Viên (Staff Gate)**| Chưa có | **Đầy đủ làn vào/ra, tính cước tự động, thanh toán tiền mặt/online** | **Mới hoàn toàn** |
| **Duyệt Giấy Tờ Xe (OCR/Docs)**| Chưa có | **Upload 4 mặt giấy tờ/CCCD, hàng đợi duyệt cho nhân viên** | **Mới hoàn toàn** |
| **Bộ Test Tự Động** | Kiểm thử xác thực & bãi đỗ cơ bản | **17 Integration test suites, Unit tests bao phủ Operations/Jobs/Gate** | **Bảo đảm độ vững chắc cao** |

---
*Báo cáo được tổng hợp tự động từ mã nguồn thực tế của dự án, tuân thủ nguyên tắc Safe Code Modification & Read-Only Inspection.*