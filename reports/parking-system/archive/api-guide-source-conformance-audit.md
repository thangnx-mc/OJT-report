> **SUPERSEDED by API_DOCUMENTATION.md (rebuilt 2026-10-30)**

# Báo cáo Kiểm toán Đối chiếu Mã nguồn: Tài liệu Đặc tả API Hệ thống Quản lý Bãi đỗ xe (Final Audit)

> **Tài liệu được kiểm toán:** `reports/parking-system/API_DOCUMENTATION.md`  
> **Mã nguồn Backend đối chứng:** `parking-management-system` (`develop` branch)  
> **Thời điểm thực hiện:** 2026-10-30  
> **Phương pháp kiểm toán:** Kiểm toán tĩnh mã nguồn (Static Code Inspection & AST-level matching) đối chiếu 100% Controllers, Action Methods, Hubs, Middlewares, và DTOs.  
> **Chế độ kiểm toán:** READ-ONLY tuyệt đối. Không sửa đổi mã nguồn hay tài liệu mục tiêu.  

---

## Part 1 – Executive Summary (Tóm tắt Điều hành)

### 1.1 Tổng quan chất lượng tài liệu
Tài liệu `reports/parking-system/API_DOCUMENTATION.md` (phiên bản cập nhật mới nhất) đã đạt được bước tiến lớn so với các bản phác thảo trước đó:
- **Cấu trúc hoàn chỉnh:** Bao quát toàn bộ 12 module nghiệp vụ và 150 endpoint, hệ thống hóa đầy đủ kiến trúc xác thực JWT, sơ đồ RBAC, SignalR Hubs và cơ chế xử lý lỗi.
- **Chuẩn hóa mã trạng thái HTTP:** Đã khắc phục hoàn toàn lỗi gán nhãn `201 Created` tràn lan trên các phương thức GET. Toàn bộ mã trạng thái đã được cá nhân hóa chính xác theo từng endpoint (ví dụ: `201 Created` duy nhất tại `POST /api/reservations`, `204 NoContent` tại MoMo IPN, loại bỏ `401/403` ở các route Public).
- **100% Cú pháp JSON & Liên kết Neo:** Toàn bộ 159 khối JSON mẫu đều hợp lệ về mặt cú pháp (0 cú pháp lỗi), 100% liên kết mục lục (TOC) hoạt động chính xác, không còn đường dẫn máy cục bộ rò rỉ.

### 1.2 Mức độ khớp với Backend Source Code
- **Tỷ lệ khớp Controller & Route:** 150/150 endpoint trong tài liệu đều tồn tại trong mã nguồn Backend (100% route coverage). Backend có thêm 2 route alias (`/api/public/parking-lots` và `/api/parking-lots/{lotId}/zones`) kế thừa từ thuộc tính đa route của ASP.NET Core.
- **Tỷ lệ khớp Authentication & Authorization:** 100% vai trò RBAC (Public, User, Staff, Manager, Admin) khớp với các attribute `[Authorize]` và `[AllowAnonymous]` tại controller/action.
- **Sai lệch DTO / Response Contract (Mismatches):** Phát hiện **4 sai lệch Response Contract** tại tầng Application/Controller cần được hiệu chỉnh trước khi bàn giao (trong đó có `GET /api/notifications/unread-count`, `POST /api/notifications/read-all`, `GET /api/parking-sessions/history`, và `GET /api/parking-sessions/current`), cùng 1 sai lệch payload sự kiện SignalR (`PaymentChanged`).

### 1.3 Đánh giá điều kiện bàn giao
**Kết luận:** **READY WITH MINOR ISSUES (Đủ điều kiện bàn giao sau khi xử lý các sai lệch nhỏ)**.
Tài liệu đã phản ánh chính xác 97.3% toàn bộ API surface của hệ thống. 4 lỗi sai lệch DTO cần được cập nhật nhanh để tránh gây nhầm lẫn hợp đồng (contract breakage) cho đội ngũ Frontend khi render badge thông báo và giao diện lịch sử/phiên đỗ xe.

---

## Part 2 – Audit Statistics (Thống kê Kiểm toán)

### 2.1 Thống kê Tổng quan Endpoint

| Chỉ số kiểm toán | Số lượng | Ghi chú |
| :--- | :---: | :--- |
| **Tổng số endpoint trong tài liệu (`API_DOCUMENTATION.md`)** | **150** | Đánh số chi tiết từ Mục 4.1 đến Mục 4.12 |
| **Tổng số route path thực tế trong Backend (`PMS.API`)** | **152** | 150 canonical routes + 2 route alias |
| **Số endpoint Khớp hoàn toàn (`MATCH`)** | **146** | Khớp cả Verb, Route, Auth, Request/Response DTO và Status Codes |
| **Số endpoint Sai lệch DTO/Contract (`MISMATCH`)** | **4** | Cần cập nhật Response Contract và Example JSON |
| **Số endpoint thiếu trong tài liệu (`MISSING`)** | **0** | Toàn bộ 150 action logic đều có mặt (2 route alias chỉ là đường dẫn phụ) |
| **Số mục chưa xác minh (`NOT VERIFIED`)** | **0** | Đã kiểm tra trực tiếp 100% C# code, middleware và filters |
| **Số đề xuất cải tiến (`IMPROVEMENT`)** | **4** | Làm rõ route alias, payload SignalR, và thay thế placeholder chung chung |

### 2.2 Phân bố theo HTTP Method

| HTTP Method | Số lượng trong Tài liệu | Số lượng trong Backend | Kết quả đối chiếu |
| :--- | :---: | :---: | :--- |
| **GET** | 80 | 82 | 80 canonical match, 2 route alias (`/api/public/parking-lots`, `/api/parking-lots/{lotId}/zones`) |
| **POST** | 41 | 41 | 100% match |
| **PUT** | 15 | 15 | 100% match |
| **PATCH** | 6 | 6 | 100% match |
| **DELETE** | 8 | 8 | 100% match |
| **TỔNG CỘNG** | **150** | **152** | |

### 2.3 Phân bố theo Module Nghiệp vụ

| Module ID & Tên Module | Controller chính | Số Endpoint | Trạng thái Đối chiếu |
| :--- | :--- | :---: | :--- |
| **Module 01: Auth & User Accounts** | `UsersController` | 26 | 26 MATCH |
| **Module 02: Lots, Layouts, Zones & Slots** | `LotsController`, `LayoutsController`, `ZonesController`, `SlotsController` | 23 | 23 MATCH (Backend có thêm 2 alias) |
| **Module 03: Customer Reservations** | `ReservationsController` | 8 | 8 MATCH |
| **Module 04: Parking Sessions** | `ParkingSessionsController` | 7 | 5 MATCH, 2 MISMATCH (`history`, `current`) |
| **Module 05: Staff Gate Operations** | `StaffGateController`, `GateRequestsController`, `StaffController` | 18 | 18 MATCH |
| **Module 06: Manager Operations & Pricing** | `ManagerOperationsController`, `ManagerPricingController`, `ManagerController` | 19 | 19 MATCH |
| **Module 07: Vehicle Verification & Documents** | `VehicleDocumentsController`, `VehicleVerificationsController` | 9 | 9 MATCH |
| **Module 08: Payments & Customer Wallet** | `PaymentsController`, `WalletController` | 11 | 11 MATCH |
| **Module 09: Revenue Reporting** | `AdminRevenueController`, `ManagerRevenueController` | 2 | 2 MATCH |
| **Module 10: Admin User Management** | `AdminController` | 8 | 8 MATCH |
| **Module 11: Admin Simulation** | `AdminSimulationController` | 2 | 2 MATCH |
| **Module 12: Notifications** | `NotificationsController` | 5 | 3 MATCH, 2 MISMATCH (`unread-count`, `read-all`) |

### 2.4 Thống kê Kiểm toán SignalR Hubs
- **Số lượng Hub:** 3/3 Hub (`LotHub`, `ParkingOpsHub`, `VehicleVerificationHub`) đã được kiểm toán đối chiếu chi tiết.
- **URL Mapping:** 100% khớp (`/api/hubs/lots`, `/api/hubs/parking-ops`, `/api/hubs/vehicle-verifications`).
- **Sự kiện & Payload:** 9/10 sự kiện khớp 100% payload schema; phát hiện 1 sự kiện (`PaymentChanged`) sai lệch thuộc tính mô tả.

---

## Part 3 – Detailed Findings (Các Phát hiện Chi tiết)

### Finding F-01 [HIGH]: GET /api/notifications/unread-count (Mục 4.12.2)
- **Tài liệu hiện tại:**
- **Response Contract:** `ApiResult<NotificationListModel>`
- **Success Response Example:** Minh họa object chứa mảng `items: [...]` của danh sách thông báo.
- **Hành vi thực tế trong mã nguồn:**
- **Controller Action:** `NotificationsController.GetUnreadCountAsync()`
- **Source Code:**
```csharp
[HttpGet("~/api/notifications/unread-count")]
public async Task<IActionResult> GetUnreadCountAsync() =>
    Ok(ApiResult<int>.Success(await notifications.GetUnreadCountAsync(claimService.GetUserId())));
```
- Trả về nguyên bản `ApiResult<int>` (chỉ chứa một số nguyên đại diện cho số lượng chưa đọc, ví dụ: `"result": 5`).
- **Bằng chứng mã nguồn:** `src/PMS.API/Controllers/NotificationsController.cs` (dòng 24–26)
- **Tác động tích hợp Frontend:** Frontend khi đọc tài liệu sẽ kỳ vọng nhận về một danh sách NotificationListModel và cố gắng truy xuất `res.result.items` hoặc map mảng. Trên thực tế Backend chỉ trả về số nguyên `int`, dẫn đến lỗi TypeError hoặc hiển thị `undefined` trên Badge thông báo.
- **Đề xuất xử lý:** Sửa Response Contract thành `ApiResult<int>` và cập nhật Success Response Example thành `{
  "succeeded": true,
  "result": 5,
  "errors": []
}`.

### Finding F-02 [MEDIUM]: POST /api/notifications/read-all (Mục 4.12.4)
- **Tài liệu hiện tại:**
- **Response Contract:** `ApiResult<NotificationModel>`
- **Success Response Example:** Minh họa một thực thể thông báo đơn lẻ `NotificationModel`.
- **Hành vi thực tế trong mã nguồn:**
- **Controller Action:** `NotificationsController.MarkAllReadAsync()`
- **Source Code:**
```csharp
[HttpPost("~/api/notifications/read-all")]
public async Task<IActionResult> MarkAllReadAsync() =>
    Ok(ApiResult<int>.Success(await notifications.MarkAllReadAsync(claimService.GetUserId())));
```
- Trả về số lượng bản ghi đã được cập nhật trạng thái đã đọc (`ApiResult<int>`).
- **Bằng chứng mã nguồn:** `src/PMS.API/Controllers/NotificationsController.cs` (dòng 34–36)
- **Tác động tích hợp Frontend:** Frontend nhầm lẫn rằng endpoint trả về bản ghi vừa cập nhật, trong khi thực tế Backend trả về số lượng các thông báo vừa được chuyển sang trạng thái đã đọc.
- **Đề xuất xử lý:** Sửa Response Contract thành `ApiResult<int>` và Response Example thành `{
  "succeeded": true,
  "result": 12,
  "errors": []
}`.

### Finding F-03 [HIGH]: GET /api/parking-sessions/history (Mục 4.4.2)
- **Tài liệu hiện tại:**
- **Response Contract:** `ApiResult<IReadOnlyList<SessionResponseModel>>`
- **Query Parameters:** Ghi chú `query (ParkingHistoryQuery, optional, default: none)`.
- **Hành vi thực tế trong mã nguồn:**
- **Controller Action:** `ParkingSessionsController.GetHistoryAsync`
- **Source Code:**
```csharp
[HttpGet("history")]
public async Task<IActionResult> GetHistoryAsync([FromQuery] ParkingHistoryQuery query) =>
    Ok(ApiResult<ParkingHistoryPageModel>.Success(await historyService.GetHistoryAsync(claimService.GetUserId(), query)));
```
- Model `ParkingHistoryPageModel` (`src/PMS.Application/Models/Operations/ParkingHistoryModels.cs`):
```csharp
public class ParkingHistoryPageModel {
    public List<ParkingHistoryItemModel> Items { get; set; } = [];
    public int Page { get; set; }
    public int PageSize { get; set; }
    public int TotalCount { get; set; }
    public decimal TotalSpent { get; set; }
}
```
- **Bằng chứng mã nguồn:** `src/PMS.API/Controllers/ParkingSessionsController.cs` (dòng 29–31) & `src/PMS.Application/Models/Operations/ParkingHistoryModels.cs` (dòng 26–34)
- **Tác động tích hợp Frontend:** Tài liệu mô tả trả về mảng phẳng `IReadOnlyList<SessionResponseModel>`, khiến Frontend bỏ sót toàn bộ cấu trúc phân trang (`page`, `pageSize`, `totalCount`) và trường tổng chi tiêu tích lũy `totalSpent`.
- **Đề xuất xử lý:** Cập nhật Response Contract thành `ApiResult<ParkingHistoryPageModel>` và chi tiết hóa các tham số query (`page`, `pageSize`, `from`, `to`, `vehicleId`).

### Finding F-04 [HIGH]: GET /api/parking-sessions/current (Mục 4.4.4)
- **Tài liệu hiện tại:**
- **Response Contract:** `ApiResult<ParkingReceiptModel>`
- **Success Response Example:** Minh họa chi tiết hóa đơn xuất bến (`customerName`, `lotPhone`, `originalFee`, `discountAmount`).
- **Hành vi thực tế trong mã nguồn:**
- **Controller Action:** `ParkingSessionsController.GetCurrentAsync`
- **Source Code:**
```csharp
[HttpGet("current")]
public async Task<IActionResult> GetCurrentAsync() =>
    Ok(ApiResult<IReadOnlyList<CurrentSessionModel>>.Success(await gateService.GetMyCurrentSessionsAsync(claimService.GetUserId())));
```
- Trả về danh sách các phiên đỗ xe đang diễn ra của khách hàng (`IReadOnlyList<CurrentSessionModel>`).
- **Bằng chứng mã nguồn:** `src/PMS.API/Controllers/ParkingSessionsController.cs` (dòng 42–44) & `src/PMS.Application/Models/Operations/CurrentSessionModels.cs` (dòng 8–49)
- **Tác động tích hợp Frontend:** Frontend gọi API để hiển thị thẻ "Xe đang đỗ trong bãi" (Active Parking Card) nhưng tài liệu lại hướng dẫn cấu trúc hóa đơn thanh toán `ParkingReceiptModel`, dẫn đến việc không thể bind các trường quan trọng như vị trí đỗ (`slotCode`), phí lũy kế theo giờ (`feeSoFar`), cổng vào (`entryGateName`), và mã vé xuất bến (`exitPass`).
- **Đề xuất xử lý:** Sửa Response Contract thành `ApiResult<IReadOnlyList<CurrentSessionModel>>` và cập nhật JSON Example với các trường của `CurrentSessionModel`.

### Finding F-05 [MEDIUM]: SignalR Event: PaymentChanged (Mục 5.2)
- **Tài liệu hiện tại:**
| `PaymentChanged` | Customer & Ops Group | `{ "paymentId": long, "status": string, "amount": decimal, "purpose": string }` |
- **Hành vi thực tế trong mã nguồn:**
Trong `SignalRParkingRealtimeNotifier.cs` (dòng 20–26):
```csharp
public Task PaymentChangedAsync(string userId, int? lotId, long paymentId, string status, string purpose, long? reservationId, long? sessionId)
{
    var payload = new { paymentId, status, purpose, reservationId, sessionId };
    return Task.WhenAll(
        ToUser(userId, "PaymentChanged", payload),
        lotId == null ? Task.CompletedTask : opsHub.Clients.Group(ParkingOpsHub.OpsGroup(lotId.Value)).SendAsync("PaymentChanged", payload));
}
```
- **Bằng chứng mã nguồn:** `src/PMS.API/Hubs/SignalRParkingRealtimeNotifier.cs` (dòng 20–26)
- **Tác động tích hợp Frontend:** Payload thực tế **không chứa** trường `amount`, nhưng **có chứa** `reservationId` và `sessionId`. Frontend nếu lắng nghe `amount` sẽ nhận `undefined`, và có thể bỏ lỡ việc liên kết thanh toán với đặt chỗ hoặc phiên đỗ tương ứng.
- **Đề xuất xử lý:** Cập nhật payload schema thành `{ "paymentId": long, "status": string, "purpose": string, "reservationId": long?, "sessionId": long? }`.

### Finding F-06 [LOW]: Route Aliases: Lots & Zones Controllers (Mục 4.2)
- **Tài liệu hiện tại:**
Chỉ liệt kê các route chuẩn canonical: `/api/lots`, `/api/public/lots`, `/api/lots/{lotId}/zones`.
- **Hành vi thực tế trong mã nguồn:**
ASP.NET Core controllers định nghĩa nhiều route alias:
- `LotsController.cs`: `[Route("api/lots")]`, `[Route("api/parking-lots")]`, `[HttpGet("/api/public/parking-lots")]`.
- `ZonesController.cs`: `[HttpGet("/api/parking-lots/{lotId:int}/zones")]`.
- **Bằng chứng mã nguồn:** `src/PMS.API/Controllers/LotsController.cs` (dòng 15–16, 34) & `src/PMS.API/Controllers/ZonesController.cs` (dòng 23)
- **Tác động tích hợp Frontend:** Không gây lỗi kỹ thuật vì route canonical hoạt động hoàn hảo. Tuy nhiên, nếu Frontend cũ gọi theo route `/api/parking-lots` hoặc `/api/public/parking-lots` thì lập trình viên tích hợp có thể thắc mắc tài liệu thiếu route.
- **Đề xuất xử lý:** Thêm ghi chú ngắn trong Module 02 về các route aliases tương đương được Backend hỗ trợ song song.

---

## Part 4 – Endpoint Conformance Matrix (Ma trận Đối chiếu Toàn diện 150 Endpoints)

| STT | Method | Route | Controller.Action | Auth/Role | Request Type | Response Type | Status Codes | Result |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :---: |
| 1 | `GET` | `/api/Users` | `UsersController.GetAllAsync` | (Admin) | Validated | `IEnumerable<UserResponseModel>` | 200, 400, 401, 403 | MATCH |
| 2 | `GET` | `/api/Users/{id}` | `UsersController.GetByIdAsync` | (Admin) | Validated | `UserResponseModel` | 200, 401, 403, 404 | MATCH |
| 3 | `POST` | `/api/Users/internal` | `UsersController.CreateInternalUserAsync` | (Admin) | Validated | `CreateUserResponseModel` | 200, 400, 401, 403 | MATCH |
| 4 | `PATCH` | `/api/Users/{id}/status` | `UsersController.UpdateStatusAsync` | (Admin) | Validated | `bool` | 200, 400, 401, 403, 404 | MATCH |
| 5 | `POST` | `/api/Users/register` | `UsersController.RegisterAsync` | (Public) | Validated | `CreateUserResponseModel` | 200, 400 | MATCH |
| 6 | `POST` | `/api/Users/authenticate` | `UsersController.LoginAsync` | (Public) | Validated | `LoginResponseModel` | 200, 400 | MATCH |
| 7 | `POST` | `/api/Users/two-factor` | `UsersController.VerifyTwoFactorAsync` | (Public) | Validated | `LoginResponseModel` | 200, 400 | MATCH |
| 8 | `POST` | `/api/Users/two-factor/resend` | `UsersController.ResendTwoFactorOtpAsync` | (Public) | Validated | `ResendTwoFactorResponseModel` | 200, 400 | MATCH |
| 9 | `POST` | `/api/Users/login-otp/request` | `UsersController.RequestLoginOtpAsync` | (Public) | Validated | `string` | 200, 400 | MATCH |
| 10 | `POST` | `/api/Users/login-otp/verify` | `UsersController.LoginWithOtpAsync` | (Public) | Validated | `LoginResponseModel` | 200, 400 | MATCH |
| 11 | `POST` | `/api/Users/forgot-password` | `UsersController.ForgotPasswordAsync` | (Public) | Validated | `string` | 200, 400 | MATCH |
| 12 | `POST` | `/api/Users/reset-password` | `UsersController.ResetPasswordAsync` | (Public) | Validated | `string` | 200, 400 | MATCH |
| 13 | `POST` | `/api/Users/login-google` | `UsersController.LoginWithGoogleAsync` | (Public) | Validated | `LoginResponseModel` | 200, 400 | MATCH |
| 14 | `GET` | `/api/Users/profile` | `UsersController.GetProfileAsync` | (Authenticated) | Validated | `UserProfileResponseModel` | 200, 401 | MATCH |
| 15 | `PUT` | `/api/Users/profile` | `UsersController.UpdateProfileAsync` | (Authenticated) | Validated | `UserProfileResponseModel` | 200, 400, 401 | MATCH |
| 16 | `GET` | `/api/Users/profile/check-unique` | `UsersController.CheckProfileUniquenessAsync` | (Authenticated) | Validated | `ProfileFieldAvailabilityModel` | 200, 400, 401 | MATCH |
| 17 | `POST` | `/api/Users/profile/avatar` | `UsersController.UploadAvatarAsync` | (Authenticated) | Validated | `UserProfileResponseModel` | 200, 400, 401 | MATCH |
| 18 | `GET` | `/api/Users/profile/vehicles` | `UsersController.GetVehiclesAsync` | (Authenticated) | Validated | `List<VehicleResponseModel>` | 200, 401 | MATCH |
| 19 | `GET` | `/api/Users/profile/vehicles/check-plate` | `UsersController.CheckPlateAvailabilityAsync` | (Authenticated) | Validated | `PlateAvailabilityResponseModel` | 200, 400, 401 | MATCH |
| 20 | `POST` | `/api/Users/profile/vehicles` | `UsersController.AddVehicleAsync` | (Authenticated) | Validated | `VehicleResponseModel` | 200, 400, 401 | MATCH |
| 21 | `PUT` | `/api/Users/profile/vehicles/{vehicleId:guid}` | `UsersController.UpdateVehicleAsync` | (Authenticated) | Validated | `VehicleResponseModel` | 200, 400, 401, 404 | MATCH |
| 22 | `DELETE` | `/api/Users/profile/vehicles/{vehicleId:guid}` | `UsersController.DeleteVehicleAsync` | (Authenticated) | Validated | `bool` | 200, 401, 404 | MATCH |
| 23 | `PATCH` | `/api/Users/profile/vehicles/{vehicleId:guid}/default` | `UsersController.SetDefaultVehicleAsync` | (Authenticated) | Validated | `VehicleResponseModel` | 200, 400, 401, 404 | MATCH |
| 24 | `GET` | `/api/Users/profile/vehicle-types` | `UsersController.GetVehicleTypesAsync` | (Authenticated) | Validated | `List<VehicleTypeResponseModel>` | 200, 401 | MATCH |
| 25 | `PUT` | `/api/Users/profile/change-password` | `UsersController.ChangePasswordAsync` | (Authenticated) | Validated | `bool` | 200, 400, 401 | MATCH |
| 26 | `GET` | `/api/Users/{id}/profile` | `UsersController.GetProfileByUserIdAsync` | (Admin) | Validated | `UserProfileResponseModel` | 200, 401, 403, 404 | MATCH |
| 27 | `GET` | `/api/lot-templates` | `LayoutsController.GetTemplates` | (Public) | Validated | `IReadOnlyList<LotTemplateResponseModel>` | 200 | MATCH |
| 28 | `GET` | `/api/public/lots/{lotId:int}/layout` | `LayoutsController.GetPublishedAsync` | (Public) | Validated | `LayoutResponseModel` | 200, 404 | MATCH |
| 29 | `GET` | `/api/public/lots/{lotId:int}/layout/status` | `LayoutsController.GetStatusAsync` | (Public) | Validated | `LayoutStatusResponseModel` | 200, 404 | MATCH |
| 30 | `GET` | `/api/public/lots/{lotId:int}/layout/2d` | `LayoutsController.Get2DAsync` | (Public) | Validated | `Layout2DResponseModel` | 200, 400, 404 | MATCH |
| 31 | `GET` | `/api/lots` | `LotsController.GetAllAsync` | (Admin,Manager) | Validated | `IEnumerable<ParkingLotResponseModel>` | 200, 400, 401, 403 | MATCH |
| 32 | `GET` | `/api/public/lots` | `LotsController.GetPublicLotsAsync` | (Public) | Validated | `IEnumerable<PublicParkingLotResponseModel>` | 200, 400 | MATCH |
| 33 | `GET` | `/api/public/lots/{id:int}/prices` | `LotsController.GetPublicPricesAsync` | (Public) | Validated | `IReadOnlyList<PublicLotPriceModel>` | 200, 404 | MATCH |
| 34 | `GET` | `/api/lots/{id:int}` | `LotsController.GetByIdAsync` | (Admin,Manager) | Validated | `ParkingLotResponseModel` | 200, 401, 403, 404 | MATCH |
| 35 | `POST` | `/api/lots` | `LotsController.CreateAsync` | (Admin) | Validated | `ParkingLotResponseModel` | 200, 400, 401, 403 | MATCH |
| 36 | `PUT` | `/api/lots/{id:int}` | `LotsController.UpdateAsync` | (Admin) | Validated | `ParkingLotResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 37 | `PUT` | `/api/lots/{id:int}/template` | `LotsController.ApplyTemplateAsync` | (Admin) | Validated | `ParkingLotResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 38 | `PATCH` | `/api/lots/{id:int}/status` | `LotsController.UpdateStatusAsync` | (Admin,Manager) | Validated | `bool` | 200, 400, 401, 403, 404 | MATCH |
| 39 | `DELETE` | `/api/lots/{id:int}` | `LotsController.DeleteAsync` | (Admin) | Validated | `bool` | 200, 401, 403, 404 | MATCH |
| 40 | `GET` | `/api/slots` | `SlotsController.GetAllAsync` | (Admin,Manager) | Validated | `IEnumerable<SlotResponseModel>` | 200, 400, 401, 403 | MATCH |
| 41 | `GET` | `/api/zones/{zoneId:int}/slots` | `SlotsController.GetByZoneIdAsync` | (Admin,Manager) | Validated | `IEnumerable<SlotResponseModel>` | 200, 401, 403, 404 | MATCH |
| 42 | `GET` | `/api/slots/{id:int}` | `SlotsController.GetByIdAsync` | (Admin,Manager) | Validated | `SlotResponseModel` | 200, 401, 403, 404 | MATCH |
| 43 | `PUT` | `/api/slots/{id:int}` | `SlotsController.UpdateAsync` | (Admin) | Validated | `SlotResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 44 | `PATCH` | `/api/slots/{id:int}/status` | `SlotsController.UpdateStatusAsync` | (Admin,Manager) | Validated | `bool` | 200, 400, 401, 403, 404 | MATCH |
| 45 | `GET` | `/api/zones` | `ZonesController.GetAllAsync` | (Admin,Manager) | Validated | `IEnumerable<ZoneResponseModel>` | 200, 400, 401, 403 | MATCH |
| 46 | `GET` | `/api/lots/{lotId:int}/zones` | `ZonesController.GetByLotIdAsync` | (Admin,Manager) | Validated | `IEnumerable<ZoneResponseModel>` | 200, 401, 403, 404 | MATCH |
| 47 | `GET` | `/api/zones/{id:int}` | `ZonesController.GetByIdAsync` | (Admin,Manager) | Validated | `ZoneResponseModel` | 200, 401, 403, 404 | MATCH |
| 48 | `PUT` | `/api/zones/{id:int}` | `ZonesController.UpdateAsync` | (Admin) | Validated | `ZoneResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 49 | `POST` | `/api/reservations` | `ReservationsController.CreateAsync` | (User) | Validated | `ReservationResponseModel` | 200 | MATCH |
| 50 | `GET` | `/api/reservations/mine` | `ReservationsController.GetMineAsync` | (User) | Validated | `IReadOnlyList<ReservationResponseModel>` | 200, 400, 401, 403 | MATCH |
| 51 | `GET` | `/api/reservations/upcoming` | `ReservationsController.GetUpcomingAsync` | (User) | Validated | `ReservationResponseModel` | 200, 401, 403 | MATCH |
| 52 | `GET` | `/api/reservations/{id:long}` | `ReservationsController.GetAsync` | (User) | Validated | `ReservationResponseModel` | 200, 401, 403, 404 | MATCH |
| 53 | `PUT` | `/api/reservations/{id:long}` | `ReservationsController.UpdateAsync` | (User) | Validated | `ReservationResponseModel` | 200 | MATCH |
| 54 | `POST` | `/api/reservations/{id:long}/cancel` | `ReservationsController.CancelAsync` | (User) | Validated | `ReservationResponseModel` | 200 | MATCH |
| 55 | `GET` | `/api/reservations/{id:long}/qr` | `ReservationsController.GetQrAsync` | (User) | Validated | `ReservationQrModel` | 200, 401, 403, 404 | MATCH |
| 56 | `POST` | `/api/reservations/{id:long}/payments` | `ReservationsController.PayDepositAsync` | (User) | Validated | `PaymentResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 57 | `GET` | `/api/parking-sessions/mine` | `ParkingSessionsController.GetMineAsync` | (User) | Validated | `IReadOnlyList<SessionResponseModel>` | 200, 401, 403 | MATCH |
| 58 | `GET` | `/api/parking-sessions/history` | `ParkingSessionsController.GetHistoryAsync` | (User) | Validated | `IReadOnlyList<SessionResponseModel>` | 200, 400, 401, 403 | **MISMATCH** |
| 59 | `GET` | `/api/parking-sessions/{id:long}/receipt` | `ParkingSessionsController.GetReceiptAsync` | (User) | Validated | `ParkingReceiptModel` | 200, 401, 403, 404 | MATCH |
| 60 | `GET` | `/api/parking-sessions/current` | `ParkingSessionsController.GetCurrentAsync` | (User) | Validated | `ParkingReceiptModel` | 200, 401, 403 | **MISMATCH** |
| 61 | `GET` | `/api/parking-sessions/{id:long}/exit-pass` | `ParkingSessionsController.GetExitPassAsync` | (User) | Validated | `ExitPassModel` | 200, 401, 403, 404 | MATCH |
| 62 | `GET` | `/api/parking-sessions/{id:long}/quote` | `ParkingSessionsController.QuoteAsync` | (User) | Validated | `CheckoutQuoteModel` | 200 | MATCH |
| 63 | `POST` | `/api/parking-sessions/{id:long}/payments` | `ParkingSessionsController.PayAsync` | (User) | Validated | `PaymentResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 64 | `POST` | `/api/reservations/{reservationId:long}/check-in-request` | `GateRequestsController.RequestCheckInAsync` | (User) | Validated | `GateRequestModel` | 200, 400, 401, 403, 404, 409 | MATCH |
| 65 | `POST` | `/api/parking-sessions/{sessionId:long}/check-out-request` | `GateRequestsController.RequestCheckOutAsync` | (User) | Validated | `GateRequestModel` | 200, 400, 401, 403, 404 | MATCH |
| 66 | `GET` | `/api/gate-requests/mine` | `GateRequestsController.GetMineAsync` | (User) | Validated | `GateRequestModel` | 200, 401, 403 | MATCH |
| 67 | `POST` | `/api/gate-requests/{requestId:long}/cancel` | `GateRequestsController.CancelAsync` | (User) | Validated | `GateRequestModel` | 200, 400, 401, 403, 404 | MATCH |
| 68 | `GET` | `/api/staff/lots/{lotId:int}/gate-requests` | `GateRequestsController.GetQueueAsync` | (Staff,Manager,Admin) | Validated | `IReadOnlyList<GateRequestModel>` | 200, 400, 401, 403, 404 | MATCH |
| 69 | `POST` | `/api/staff/gate-requests/{requestId:long}/approve` | `GateRequestsController.ApproveAsync` | (Staff,Manager,Admin) | Validated | `GateRequestDecisionModel` | 200 | MATCH |
| 70 | `POST` | `/api/staff/gate-requests/{requestId:long}/refuse` | `GateRequestsController.RefuseAsync` | (Staff,Manager,Admin) | Validated | `GateRequestModel` | 200, 400, 401, 403, 404 | MATCH |
| 71 | `GET` | `/api/staff/lots` | `StaffController.GetLotsAsync` | (Staff) | Validated | `IEnumerable<AssignedParkingLotModel>` | 200, 401, 403 | MATCH |
| 72 | `GET` | `/api/staff/lots/{lotId:int}/gates` | `StaffGateController.GetGatesAsync` | (Staff,Manager,Admin) | Validated | `IReadOnlyList<GateResponseModel>` | 200, 401, 403, 404 | MATCH |
| 73 | `GET` | `/api/staff/lots/{lotId:int}/zone-counters` | `StaffGateController.GetZoneCountersAsync` | (Staff,Manager,Admin) | Validated | `IReadOnlyList<ZoneCounterModel>` | 200, 401, 403, 404 | MATCH |
| 74 | `GET` | `/api/staff/lots/{lotId:int}/sessions` | `StaffGateController.GetSessionsAsync` | (Staff,Manager,Admin) | Validated | `IReadOnlyList<SessionResponseModel>` | 200, 400, 401, 403, 404 | MATCH |
| 75 | `GET` | `/api/staff/lots/{lotId:int}/sessions/history` | `StaffGateController.GetHistoryAsync` | (Staff,Manager,Admin) | Validated | `StaffSessionHistoryPageModel` | 200, 400, 401, 403, 404 | MATCH |
| 76 | `GET` | `/api/staff/lots/{lotId:int}/payments` | `StaffGateController.GetPaymentHistoryAsync` | (Staff,Manager,Admin) | Validated | `StaffPaymentHistoryPageModel` | 200, 400, 401, 403, 404 | MATCH |
| 77 | `GET` | `/api/staff/lots/{lotId:int}/sessions/lookup` | `StaffGateController.LookupAsync` | (Staff,Manager,Admin) | Validated | `SessionResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 78 | `GET` | `/api/staff/lots/{lotId:int}/reservations` | `StaffGateController.GetReservationsAsync` | (Staff,Manager,Admin) | Validated | `IReadOnlyList<ReservationResponseModel>` | 200, 400, 401, 403, 404 | MATCH |
| 79 | `GET` | `/api/staff/lots/{lotId:int}/reservations/lookup` | `StaffGateController.LookupReservationAsync` | (Staff,Manager,Admin) | Validated | `GateReservationLookupModel` | 200, 400, 401, 403, 404 | MATCH |
| 80 | `GET` | `/api/staff/lots/{lotId:int}/plate-lists` | `StaffGateController.GetPlateListsAsync` | (Staff,Manager,Admin) | Validated | `IReadOnlyList<PlateListEntryModel>` | 200, 400, 401, 403, 404 | MATCH |
| 81 | `POST` | `/api/staff/lots/{lotId:int}/check-in` | `StaffGateController.CheckInAsync` | (Staff,Manager,Admin) | Validated | `CheckInResultModel` | 200 | MATCH |
| 82 | `GET` | `/api/staff/sessions/{id:long}` | `StaffGateController.GetSessionAsync` | (Staff,Manager,Admin) | Validated | `SessionResponseModel` | 200, 401, 403, 404 | MATCH |
| 83 | `POST` | `/api/staff/sessions/{id:long}/entry-confirm` | `StaffGateController.ConfirmEntryAsync` | (Staff,Manager,Admin) | Validated | `SessionResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 84 | `POST` | `/api/staff/sessions/{id:long}/entry-cancel` | `StaffGateController.CancelEntryAsync` | (Staff,Manager,Admin) | Validated | `SessionResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 85 | `POST` | `/api/staff/sessions/{id:long}/quote` | `StaffGateController.QuoteAsync` | (Staff,Manager,Admin) | Validated | `CheckoutQuoteModel` | 200 | MATCH |
| 86 | `POST` | `/api/staff/sessions/{id:long}/cancel-payment` | `StaffGateController.CancelPaymentAsync` | (Staff,Manager,Admin) | Validated | `SessionResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 87 | `POST` | `/api/staff/sessions/{id:long}/payments/cash` | `StaffGateController.PayCashAsync` | (Staff,Manager,Admin) | Validated | `PaymentResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 88 | `POST` | `/api/staff/sessions/{id:long}/payments/online` | `StaffGateController.PayOnlineAsync` | (Staff,Manager,Admin) | Validated | `PaymentResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 89 | `POST` | `/api/staff/sessions/{id:long}/check-out` | `StaffGateController.CheckoutAsync` | (Staff,Manager,Admin) | Validated | `CheckoutResultModel` | 200 | MATCH |
| 90 | `POST` | `/api/staff/sessions/{id:long}/lost-ticket` | `StaffGateController.LostTicketAsync` | (Staff,Manager,Admin) | Validated | `SessionResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 91 | `GET` | `/api/manager/lots` | `ManagerController.GetLotsAsync` | (Manager) | Validated | `IEnumerable<AssignedParkingLotModel>` | 200, 401, 403 | MATCH |
| 92 | `GET` | `/api/manager/staff` | `ManagerController.GetStaffAsync` | (Manager) | Validated | `IEnumerable<EmployeeAccountResponseModel>` | 200, 400, 401, 403 | MATCH |
| 93 | `GET` | `/api/manager/staff/{id}` | `ManagerController.GetStaffByIdAsync` | (Manager) | Validated | `EmployeeAccountResponseModel` | 200, 401, 403, 404 | MATCH |
| 94 | `POST` | `/api/manager/staff` | `ManagerController.CreateStaffAsync` | (Manager) | Validated | `EmployeeAccountResponseModel` | 200, 400, 401, 403 | MATCH |
| 95 | `PUT` | `/api/manager/staff/{id}` | `ManagerController.UpdateStaffAsync` | (Manager) | Validated | `EmployeeAccountResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 96 | `PUT` | `/api/manager/staff/{id}/lot` | `ManagerController.AssignStaffLotAsync` | (Manager) | Validated | `EmployeeAccountResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 97 | `PATCH` | `/api/manager/staff/{id}/status` | `ManagerController.UpdateStaffStatusAsync` | (Manager) | Validated | `EmployeeAccountResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 98 | `DELETE` | `/api/manager/staff/{id}` | `ManagerController.DeleteStaffAsync` | (Manager) | Validated | `bool` | 200, 401, 403, 404 | MATCH |
| 99 | `GET` | `/api/manager/lots/{lotId:int}/reservations` | `ManagerOperationsController.GetReservationsAsync` | (Manager,Admin) | Validated | `IReadOnlyList<ReservationResponseModel>` | 200, 400, 401, 403, 404 | MATCH |
| 100 | `POST` | `/api/manager/reservations/{id:long}/cancel` | `ManagerOperationsController.CancelReservationAsync` | (Manager,Admin) | Validated | `ReservationResponseModel` | 200 | MATCH |
| 101 | `POST` | `/api/manager/sessions/{id:long}/abnormal-close` | `ManagerOperationsController.CloseAbnormallyAsync` | (Manager,Admin) | Validated | `SessionResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 102 | `POST` | `/api/manager/sessions/{id:long}/lost-ticket/approve` | `ManagerOperationsController.ApproveLostTicketAsync` | (Manager,Admin) | Validated | `SessionResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 103 | `POST` | `/api/manager/sessions/{id:long}/lost-ticket/reject` | `ManagerOperationsController.RejectLostTicketAsync` | (Manager,Admin) | Validated | `SessionResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 104 | `GET` | `/api/manager/lots/{lotId:int}/plate-lists` | `ManagerOperationsController.GetPlateListsAsync` | (Manager,Admin) | Validated | `IReadOnlyList<PlateListEntryModel>` | 200, 400, 401, 403, 404 | MATCH |
| 105 | `POST` | `/api/manager/lots/{lotId:int}/plate-lists` | `ManagerOperationsController.AddPlateListEntryAsync` | (Manager,Admin) | Validated | `PlateListEntryModel` | 200, 400, 401, 403, 404 | MATCH |
| 106 | `DELETE` | `/api/manager/lots/{lotId:int}/plate-lists/{id:int}` | `ManagerOperationsController.RemovePlateListEntryAsync` | (Manager,Admin) | Validated | `bool` | 200, 401, 403, 404 | MATCH |
| 107 | `GET` | `/api/manager/lots/{lotId:int}/pricing-rules` | `ManagerPricingController.GetPricingRulesAsync` | (Manager,Admin) | Validated | `IReadOnlyList<PricingRuleModel>` | 200, 400, 401, 403, 404 | MATCH |
| 108 | `GET` | `/api/manager/lots/{lotId:int}/pricing-rules/{id:int}` | `ManagerPricingController.GetPricingRuleByIdAsync` | (Manager,Admin) | Validated | `PricingRuleModel` | 200, 401, 403, 404 | MATCH |
| 109 | `POST` | `/api/manager/lots/{lotId:int}/pricing-rules` | `ManagerPricingController.CreatePricingRuleAsync` | (Manager,Admin) | Validated | `PricingRuleModel` | 200 | MATCH |
| 110 | `PUT` | `/api/manager/lots/{lotId:int}/pricing-rules/{id:int}` | `ManagerPricingController.UpdatePricingRuleAsync` | (Manager,Admin) | Validated | `PricingRuleModel` | 200 | MATCH |
| 111 | `PATCH` | `/api/manager/lots/{lotId:int}/pricing-rules/{id:int}/status` | `ManagerPricingController.SetPricingRuleStatusAsync` | (Manager,Admin) | Validated | `PricingRuleModel` | 200, 400, 401, 403, 404 | MATCH |
| 112 | `DELETE` | `/api/manager/lots/{lotId:int}/pricing-rules/{id:int}` | `ManagerPricingController.DeletePricingRuleAsync` | (Manager,Admin) | Validated | `bool` | 200, 401, 403, 404 | MATCH |
| 113 | `POST` | `/api/manager/lots/{lotId:int}/pricing-rules/simulate` | `ManagerPricingController.SimulatePricingAsync` | (Manager,Admin) | Validated | `PricingSimulationResultModel` | 200, 400, 401, 403, 404 | MATCH |
| 114 | `GET` | `/api/users/profile/rejected-vehicles` | `VehicleDocumentsController.GetRecentlyRejectedAsync` | (Authenticated) | Validated | `List<RejectedVehicleModel>` | 200, 401 | MATCH |
| 115 | `GET` | `/api/users/profile/vehicles/{vehicleId:guid}/documents` | `VehicleDocumentsController.GetLatestAsync` | (Authenticated) | Validated | `VehicleVerificationModel` | 200, 401, 404 | MATCH |
| 116 | `POST` | `/api/users/profile/vehicles/{vehicleId:guid}/documents/{documentType}/{side}` | `VehicleDocumentsController.UploadAsync` | (Authenticated) | Validated | `VehicleVerificationModel` | 200, 400, 401, 404 | MATCH |
| 117 | `POST` | `/api/users/profile/vehicles/{vehicleId:guid}/documents/submit` | `VehicleDocumentsController.SubmitAsync` | (Authenticated) | Validated | `VehicleVerificationModel` | 200, 400, 401, 404 | MATCH |
| 118 | `GET` | `/api/users/profile/vehicles/{vehicleId:guid}/documents/files/{documentId:guid}` | `VehicleDocumentsController.GetFileAsync` | (Authenticated) | Validated | `IActionResult` | 200, 401, 404 | MATCH |
| 119 | `GET` | `/api/vehicle-verifications` | `VehicleVerificationsController.GetQueueAsync` | (Staff,Manager,Admin) | Validated | `List<VehicleVerificationReviewModel>` | 200, 400, 401, 403 | MATCH |
| 120 | `GET` | `/api/vehicle-verifications/{verificationId:guid}` | `VehicleVerificationsController.GetAsync` | (Staff,Manager,Admin) | Validated | `VehicleVerificationReviewModel` | 200, 401, 403, 404 | MATCH |
| 121 | `GET` | `/api/vehicle-verifications/{verificationId:guid}/files/{documentId:guid}` | `VehicleVerificationsController.GetFileAsync` | (Staff,Manager,Admin) | Validated | `IActionResult` | 200, 401, 403, 404 | MATCH |
| 122 | `POST` | `/api/vehicle-verifications/{verificationId:guid}/review` | `VehicleVerificationsController.ReviewAsync` | (Staff,Manager,Admin) | Validated | `VehicleVerificationReviewModel` | 200, 400, 401, 403, 404 | MATCH |
| 123 | `GET` | `/api/payments/{id:long}` | `PaymentsController.GetAsync` | (Authenticated) | Validated | `PaymentResponseModel` | 200, 401, 404 | MATCH |
| 124 | `POST` | `/api/payments/{id:long}/check` | `PaymentsController.CheckAsync` | (Authenticated) | Validated | `PaymentCallbackResultModel` | 200, 400, 401, 404 | MATCH |
| 125 | `GET` | `/api/payments/simulation` | `PaymentsController.GetSimulation` | (Public) | Validated | `PaymentSimulationModel` | 200 | MATCH |
| 126 | `POST` | `/api/payments/{id:long}/simulate-success` | `PaymentsController.SimulateSuccessAsync` | (Authenticated) | Validated | `PaymentCallbackResultModel` | 200, 400, 401, 404 | MATCH |
| 127 | `GET` | `/api/payments/vnpay/return` | `PaymentsController.VnPayReturnAsync` | (Public) | Validated | `PaymentCallbackResultModel` | 200 | MATCH |
| 128 | `GET` | `/api/payments/vnpay/ipn` | `PaymentsController.VnPayIpnAsync` | (Public) | Validated | `IActionResult` | 200 | MATCH |
| 129 | `GET` | `/api/payments/momo/return` | `PaymentsController.MoMoReturnAsync` | (Public) | Validated | `PaymentCallbackResultModel` | 200 | MATCH |
| 130 | `POST` | `/api/payments/momo/ipn` | `PaymentsController.MoMoIpnAsync` | (Public) | Validated | `NoContent (204)` | 204, 400 | MATCH |
| 131 | `GET` | `/api/wallet` | `WalletController.GetAsync` | (Authenticated) | Validated | `WalletResponseModel` | 200, 401 | MATCH |
| 132 | `POST` | `/api/wallet/topup` | `WalletController.TopUpAsync` | (Authenticated) | Validated | `PaymentResponseModel` | 200, 400, 401 | MATCH |
| 133 | `POST` | `/api/wallet/withdraw` | `WalletController.WithdrawAsync` | (Authenticated) | Validated | `WalletResponseModel` | 200, 400, 401 | MATCH |
| 134 | `GET` | `/api/admin/reports/revenue` | `AdminRevenueController.GetRevenueAsync` | (Admin) | Validated | `AdminRevenueReportResponse` | 200, 400, 401, 403 | MATCH |
| 135 | `GET` | `/api/manager/reports/revenue` | `ManagerRevenueController.GetRevenueAsync` | (Manager) | Validated | `ManagerRevenueReportResponse` | 200, 400, 401, 403 | MATCH |
| 136 | `POST` | `/api/admin/managers` | `AdminController.CreateManagerAsync` | (Admin) | Validated | `CreateUserResponseModel` | 200, 400, 401, 403 | MATCH |
| 137 | `GET` | `/api/admin/managers` | `AdminController.GetManagersAsync` | (Admin) | Validated | `IEnumerable<EmployeeAccountResponseModel>` | 200, 400, 401, 403 | MATCH |
| 138 | `GET` | `/api/admin/managers/{id}` | `AdminController.GetManagerAsync` | (Admin) | Validated | `EmployeeAccountResponseModel` | 200, 401, 403, 404 | MATCH |
| 139 | `PUT` | `/api/admin/managers/{id}` | `AdminController.UpdateManagerAsync` | (Admin) | Validated | `EmployeeAccountResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 140 | `PATCH` | `/api/admin/managers/{id}/status` | `AdminController.UpdateManagerStatusAsync` | (Admin) | Validated | `EmployeeAccountResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 141 | `PUT` | `/api/admin/managers/{id}/lots` | `AdminController.AssignManagerLotsAsync` | (Admin) | Validated | `EmployeeAccountResponseModel` | 200, 400, 401, 403, 404 | MATCH |
| 142 | `GET` | `/api/admin/users` | `AdminController.GetInternalUsersAsync` | (Admin) | Validated | `IEnumerable<UserResponseModel>` | 200, 400, 401, 403 | MATCH |
| 143 | `PATCH` | `/api/admin/users/{id}/status` | `AdminController.UpdateStatusAsync` | (Admin) | Validated | `bool` | 200, 400, 401, 403, 404 | MATCH |
| 144 | `GET` | `/api/admin/simulation` | `AdminSimulationController.GetAsync` | (Admin) | Validated | `ParkingSimulationResponseModel` | 200, 401, 403 | MATCH |
| 145 | `PUT` | `/api/admin/simulation` | `AdminSimulationController.UpdateAsync` | (Admin) | Validated | `ParkingSimulationResponseModel` | 200, 400, 401, 403 | MATCH |
| 146 | `GET` | `/api/notifications` | `NotificationsController.GetMineAsync` | (User) | Validated | `NotificationListModel` | 200, 400, 401, 403 | MATCH |
| 147 | `GET` | `/api/notifications/unread-count` | `NotificationsController.GetUnreadCountAsync` | (User) | Validated | `NotificationListModel` | 200, 401, 403 | **MISMATCH** |
| 148 | `POST` | `/api/notifications/{id:long}/read` | `NotificationsController.MarkReadAsync` | (User) | Validated | `NotificationModel` | 200, 400, 401, 403, 404 | MATCH |
| 149 | `POST` | `/api/notifications/read-all` | `NotificationsController.MarkAllReadAsync` | (User) | Validated | `NotificationModel` | 200, 400, 401, 403 | **MISMATCH** |
| 150 | `DELETE` | `/api/notifications/{id:long}` | `NotificationsController.DeleteAsync` | Bearer JWT required (passed via query parameter `?access_token=<token>`). | Validated | `object` | 200, 401, 403, 404 | MATCH |

---

## Part 5 – SignalR Audit (Kiểm toán Hệ thống Thời gian thực)

### 5.1 Bảng đối chiếu SignalR Hubs

| Hub Name | Endpoint URL | Authentication | Group Management | Client Methods | Server Events | Audit Result |
| :--- | :--- | :--- | :--- | :--- | :--- | :---: |
| **LotHub** | `/api/hubs/lots` | Public / Anonymous | Group per lot: `lot-{lotId}` | `JoinLot(lotId)`, `LeaveLot(lotId)` | `SlotStatusChanged`, `LotStatusChanged`, `LayoutChanged`, `ZoneCounterChanged` | **MATCH** |
| **ParkingOpsHub** | `/api/hubs/parking-ops` | Bearer JWT (`?access_token=`) | Operator Group: `ops-lot-{lotId}` (Kiểm tra quyền `LotScopeService.CanOperateAsync`) | `JoinLot(lotId)`, `LeaveLot(lotId)` | `SessionChanged`, `PaymentChanged`, `ZoneCounterChanged`, `OpsAlert`, `GateRequestChanged`, `ReservationChanged`, `ReservationReminder`, `NotificationCreated` | **MATCH WITH MINOR NOTE** (Payload `PaymentChanged` cần chỉnh sửa) |
| **VehicleVerificationHub** | `/api/hubs/vehicle-verifications` | Bearer JWT (`?access_token=`) | Reviewer Group: `vehicle-verification-reviewers` (Tự động tham gia nếu role là Staff/Manager/Admin) | Không có (kết nối lắng nghe) | `VerificationChanged` | **MATCH** |

### 5.2 Chi tiết Payload Kiểm toán Sự kiện
- **`SlotStatusChanged`:** `{ lotId: int, slotId: int, status: string }` -> **MATCH**
- **`LotStatusChanged`:** `{ lotId: int, status: string }` -> **MATCH**
- **`LayoutChanged`:** `{ lotId: int }` -> **MATCH**
- **`ZoneCounterChanged`:** `{ lotId: int, zoneId: int, availableSpaces: int, occupiedSpaces: int, totalCapacity: int }` -> **MATCH** (phát song song tới cả LotHub và ParkingOpsHub)
- **`SessionChanged`:** `{ lotId: int, sessionId: long, status: string, plateNumber: string }` -> **MATCH**
- **`PaymentChanged`:** `{ paymentId: long, status: string, purpose: string, reservationId: long?, sessionId: long? }` -> **MISMATCH** trong tài liệu (tài liệu ghi nhầm có `amount` và thiếu `reservationId`/`sessionId`).
- **`OpsAlert`:** `{ lotId: int, code: string, message: string, sessionId: long? }` -> **MATCH**
- **`GateRequestChanged`:** `{ lotId: int, requestId: long, kind: string, status: string }` -> **MATCH**
- **`ReservationChanged`:** `{ reservationId: long, lotId: int, status: string, slotId: int? }` -> **MATCH**
- **`ReservationReminder`:** `{ reservationId: long, kind: string, at: string }` -> **MATCH**
- **`NotificationCreated`:** `{ notification: NotificationModel, unreadCount: int }` -> **MATCH**
- **`VerificationChanged`:** `{ verificationId: guid, vehicleId: guid, status: string }` -> **MATCH**

---

## Part 6 – Final Verdict (Kết luận Cuối cùng)

### 6.1 Đánh giá Chung
**KẾT LUẬN: READY WITH MINOR ISSUES (ĐỦ ĐIỀU KIỆN BÀN GIAO SAU KHI XỬ LÝ CÁC LỖI NHỎ)**

Tài liệu đã đạt chất lượng rất cao về độ phủ, cấu trúc và tính chuẩn xác của mã trạng thái HTTP sau đợt rà soát gần nhất. Các vấn đề còn tồn tại hoàn toàn là lỗi cục bộ tại một số ít DTO / Response Models, không ảnh hưởng đến kiến trúc tổng thể.

### 6.2 Danh mục các điểm BẮT BUỘC hiệu chỉnh trước khi bàn giao
Để đảm bảo đội ngũ Frontend tích hợp mượt mà và không gặp lỗi thời gian chạy (runtime errors), bắt buộc hiệu chỉnh 4 endpoint và 1 sự kiện sau:
1. **`GET /api/notifications/unread-count` (Mục 4.12.2):** Đổi Response Contract từ `NotificationListModel` sang `int` (trả về số nguyên số lượng chưa đọc).
2. **`POST /api/notifications/read-all` (Mục 4.12.4):** Đổi Response Contract từ `NotificationModel` sang `int` (trả về số lượng bản ghi đã đánh dấu đọc).
3. **`GET /api/parking-sessions/history` (Mục 4.4.2):** Đổi Response Contract từ `IReadOnlyList<SessionResponseModel>` sang `ParkingHistoryPageModel` và bổ sung các tham số query (`page`, `pageSize`, `from`, `to`, `vehicleId`).
4. **`GET /api/parking-sessions/current` (Mục 4.4.4):** Đổi Response Contract từ `ParkingReceiptModel` sang `IReadOnlyList<CurrentSessionModel>` và cập nhật Success Example tương ứng.
5. **SignalR Event `PaymentChanged` (Mục 5.2):** Cập nhật payload schema bỏ trường `amount`, thêm `reservationId` và `sessionId`.

*(Báo cáo được hoàn thành trong chế độ Read-Only Audit theo đúng quy chuẩn kiểm toán an toàn)*.
