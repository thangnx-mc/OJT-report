# Báo Cáo Triển Khai & Kiểm Toán Kỹ Thuật: Revenue Reporting API (Manager / Admin Role Separation)

---

## 1. Mục Đích và Yêu Cầu Nghiệp Vụ Đã Phê Duyệt (Approved Business Requirements)

### 1.1. Bối cảnh & Yêu cầu từ Trưởng nhóm
Nhiệm vụ: Triển khai API ghi nhận doanh thu bãi đỗ xe khi người dùng thanh toán xong, phục vụ việc hiển thị doanh thu trên Dashboard của Quản lý bãi (Manager Console) và Quản trị viên hệ thống (Admin Revenue Report).

Các yêu cầu nghiệp vụ cốt lõi đã được phê duyệt và hoàn thành triển khai:
1. **Phân tách hoàn toàn hai API độc lập theo vai trò:** Manager và Admin có endpoint riêng, route riêng, quyền hạn riêng và hợp đồng DTO phản hồi riêng biệt, không dùng chung một endpoint rồi phân nhánh role.
2. **Cô lập dữ liệu bãi xe của Manager (Quan hệ 1 : N):**
   - Manager chỉ có quyền xem doanh thu của các bãi xe mà mình đang trực tiếp quản lý (`ParkingLots.ManagerUserId == currentUserId && !p.IsDeleted`).
   - Nếu Manager truyền `parkingLotId` không thuộc quyền quản lý $\rightarrow$ Hệ thống lập tức trả về **HTTP 403 Forbidden**.
   - Nếu Manager chưa được gán bãi nào $\rightarrow$ Trả về kết quả rỗng hợp lệ (0 doanh thu, 0 phiên), tuyệt đối không để lộ dữ liệu bãi của người khác.
   - Hỗ trợ mô hình một Manager quản lý 0, 1 hoặc nhiều bãi xe.
3. **Báo cáo toàn hệ thống cho Admin:**
   - Admin xem được doanh thu toàn hệ thống trên tất cả các bãi đang hoạt động (`!p.IsDeleted`).
   - Admin có thể lọc theo từng `parkingLotId` cụ thể. Nếu bãi không tồn tại hoặc đã xóa $\rightarrow$ Trả về **HTTP 404 Not Found**.
4. **Nguồn doanh thu chuẩn hóa ("Today's Revenue" = Phí gửi xe của Slot/Session):**
   - Doanh thu slot được lấy duy nhất từ **`ParkingSession.FinalFee`** của các phiên đã tất toán/hoàn tất (`Completed` hoặc `Paid`).
   - Tuyệt đối không tính tiền cọc đặt chỗ rời rạc, nạp ví nội bộ, tiền hoàn cọc, hoặc các khoản thu phạt/dịch vụ khác ngoài phí gửi xe.
   - Không phân biệt phương thức thanh toán đối với tổng phí slot.
   - Không bị tính trùng (double-count) khi tiền cọc được cấn trừ vào phí đỗ xe tại thời điểm checkout.
5. **Cơ cấu phương thức thanh toán (Payment-Method Breakdown):**
   - Phản ánh chính xác phần đóng góp của từng phương thức thanh toán cấu thành nên doanh thu của phiên đỗ xe (phần cọc bù trừ `DepositApplied` + phần trả thêm tại checkout `PaidAmount`).
   - Tổng giá trị các phương thức thanh toán luôn khớp 100% với `totalRevenue`.

---

## 2. Xác Minh Baseline Codebase Thực Tế (Repository Verification)

### 2.1. Backend Repository (`parking-management-system`)
- **Đường dẫn thư mục:** `C:\DEV\OJT-FSA-Team3\parking-management-system`
- **Nhánh Git:** `develop`
- **HEAD Commit SHA:** `2cf5eb1` (`feat: prevent duplicate phone number and citizen ID on profile update`)
- **Khảo sát Routing Convention:**
  - `src/PMS.API/Controllers/AdminController.cs#L14-15`: `[Route("api/admin")]`, `[Authorize(Roles = "Admin")]`.
  - `src/PMS.API/Controllers/ManagerController.cs#L14-15`: `[Route("api/manager")]`, `[Authorize(Roles = "Manager")]`.
  - `src/PMS.API/Controllers/ManagerOperationsController.cs#L11-12`: `[Route("api/manager")]`.
  - `src/PMS.API/Controllers/ManagerPricingController.cs#L15-16`: `[Route("api/manager/lots/{lotId:int}/pricing-rules")]`.
  $\rightarrow$ **Quy ước:** Toàn bộ route quản trị và vận hành nội bộ sử dụng định dạng `/api/admin/...` và `/api/manager/...`.

### 2.2. Frontend Repository (`parking-management-system-frontend`)
- **Đường dẫn thư mục:** `C:\DEV\OJT-FSA-Team3\parking-management-system-frontend`
- **Nhánh Git:** `develop` @ commit `f0d6a00`
- **Tình trạng:** Giữ nguyên vẹn, không có file nào bị chỉnh sửa.

---

## 3. Phân Tích Kỹ Thuật: Nguồn Doanh Thu Slot & Vòng Đời Phiên Đỗ

### 3.1. Thực thể lưu trữ doanh thu phí gửi xe: `ParkingSession`
Trong cơ sở dữ liệu và mã nguồn backend:
- `src/PMS.Core/Entities/ParkingSession.cs#L68-74`:
  - `OriginalFee`: Tổng phí tính theo thời gian đỗ xe và biểu giá (`decimal(18,2)`).
  - `FinalFee`: **Tổng phí gửi xe cuối cùng của phiên đỗ xe** (`decimal(18,2)`).
  - `Status`: Trạng thái phiên đỗ (`Completed`, `Paid`, `Active`, `PendingPayment`, `AbnormalClosed`, `EmergencyExit`).
- `src/PMS.Core/Entities/ParkingSession.Operations.cs#L34-39`:
  - `DepositApplied`: Số tiền cọc đặt chỗ được cấn trừ vào phí phiên đỗ (FR-RSV-07).
  - `PaidAmount`: Số tiền thực tế khách đã trả thêm tại làn ra / kiosk / app.

### 3.2. Truy vết Flow Tính Phí, Cấn Trừ Cọc và Tất Toán
1. **Chốt tính phí tại thời điểm Checkout (`QuoteCheckoutAsync`):**
   Tại `src/PMS.Application/Services/Impl/GateService.cs#L753-784`, dịch vụ cổng gọi `FeeCalculator.Settle`:
   ```csharp
   // FeeCalculator.cs dòng 123-135
   var gross = fee + lostTicketFee + late;
   var depositUsed = Math.Min(deposit, Math.Max(0, gross - alreadyPaid));
   var due = Math.Max(0, gross - (deposit + alreadyPaid));
   var depositRefund = deposit - depositUsed;
   return new Settlement(fee, lostTicketFee, late, depositUsed, alreadyPaid, RoundUp(due), depositRefund);
   ```
   Khi chốt tính phí phiên (`mutate = true`):
   ```csharp
   // GateService.cs dòng 777-783
   session.OriginalFee = breakdown.Total;
   session.FinalFee = settlement.Fee + settlement.LostTicketFee + settlement.ExitWindowLateFee;
   session.DepositApplied = settlement.DepositApplied;
   session.UpdatedAt = Now;
   ```
2. **Khách thanh toán phần còn thiếu tại Lối ra / Kiosk / App:**
   Tại `src/PMS.Application/Services/Impl/PaymentService.cs#L560-568` (`SettleSession`):
   ```csharp
   session.PaidAmount += payment.Amount;
   session.Status = SessionStatus.Paid;
   session.UpdatedAt = Now;
   ```
   **Bất biến toán học luôn được bảo đảm:**
   $$\text{session.FinalFee} = \text{session.DepositApplied} + \text{session.PaidAmount}$$
   *Ví dụ thực tế:* Phiên đỗ có tổng phí 80.000 VND, tiền cọc 50.000 VND:
   - `session.DepositApplied` = 50.000 VND.
   - `session.PaidAmount` = 30.000 VND.
   - `session.FinalFee` = **80.000 VND**.
3. **Xe qua Barie rời bãi (`CheckOutAsync`):**
   Tại `src/PMS.Application/Services/Impl/GateService.cs#L450-488`:
   ```csharp
   session.Status = SessionStatus.Completed;
   session.ExitTime = TrimToSecond(Now);
   ```
   Nếu khách có cọc thừa (`quote.DepositRefund > 0`), hệ thống tự động hoàn lại cho khách qua `paymentService.RefundDepositAsync` (BR-03). Phần hoàn này không làm thay đổi `session.FinalFee`.
4. **Tiền lệ chuẩn trong codebase:**
   Dịch vụ tra cứu lịch sử đỗ xe `src/PMS.Application/Services/Impl/ParkingHistoryService.cs#L48` cũng tính tổng chi phí dịch vụ gửi xe bằng cách truy vấn:
   ```csharp
   var totalSpent = totalCount == 0 ? 0 : await sessions.SumAsync(s => s.FinalFee);
   ```

### 3.3. Quy Tắc Xác Định Thời Điểm Ghi Nhận Doanh Thu (Deterministic Reporting Timestamp)
Để ngăn ngừa việc cập nhật `UpdatedAt` ngẫu nhiên làm dịch chuyển phiên đỗ sang ngày khác:
- **Phiên đã hoàn tất (`SessionStatus.Completed`):** `session.ExitTime` là mốc thời gian bất biến và duy nhất, được ghi nhận tại thời điểm xe vượt barie lối ra. Sau khi hoàn tất, không có bất kỳ luồng xử lý nào thay đổi lại `ExitTime`.
- **Phiên đã thanh toán trước (`SessionStatus.Paid`):** Khách trả tiền tại Kiosk/App trong bãi và đang trong 15 phút lái xe ra cửa (`ExitTime` chưa có). Mốc thời gian được xác định từ `session.UpdatedAt` (được gán chính xác tại thời điểm `SettleSession` hoặc `QuoteCheckoutAsync` đưa trạng thái thành `Paid`).
- Khoảng thời gian lọc được tính theo khoảng nửa mở (half-open interval) `[startUtc, endUtc)` theo múi giờ bãi xe (`SE Asia Standard Time` / ICT +7), sử dụng tiện ích chuẩn `ParkingText.FindTimeZone`.

---

## 4. Đặc Tả Kỹ Thuật Hai API Độc Lập Theo Vai Trò

### 4.1. Bảng So Sánh Kiến Trúc Hai API

| Tiêu chí | Manager Revenue API | Admin Revenue API |
| :--- | :--- | :--- |
| **Route** | `GET /api/manager/reports/revenue` | `GET /api/admin/reports/revenue` |
| **Controller** | `ManagerRevenueController.cs` | `AdminRevenueController.cs` |
| **Authorize** | `[Authorize(Roles = "Manager")]` | `[Authorize(Roles = "Admin")]` |
| **Phạm vi bãi xe** | Giới hạn nghiêm ngặt: `ParkingLots.Where(p => p.ManagerUserId == currentUserId && !p.IsDeleted)` | Toàn hệ thống: `ParkingLots.Where(p => !p.IsDeleted)` |
| **Xử lý `parkingLotId`** | Bắt buộc kiểm tra quyền sở hữu. Nếu bãi không thuộc quyền quản lý $\rightarrow$ **HTTP 403 Forbidden**. Nếu Manager không có bãi nào $\rightarrow$ Trả về kết quả rỗng hợp lệ (HTTP 200). | Tùy chọn lọc. Nếu bãi không tồn tại hoặc đã xóa $\rightarrow$ **HTTP 404 Not Found**. |
| **Hợp đồng Response** | `ManagerRevenueReportResponse` | `AdminRevenueReportResponse` |
| **Response Envelope** | `ApiResult<ManagerRevenueReportResponse>` | `ApiResult<AdminRevenueReportResponse>` |

### 4.2. Chi Tiết Hợp Đồng DTO Phản Hồi

#### A. Hợp đồng Manager API (`ManagerRevenueReportResponse`)
```json
{
  "succeeded": true,
  "result": {
    "managerUserId": "mgr-guid-123",
    "startDate": "2026-10-28",
    "endDate": "2026-10-28",
    "totalRevenue": 80000.0,
    "totalSessions": 1,
    "byLot": [
      {
        "parkingLotId": 1,
        "parkingLotName": "Bãi xe Landmark 81",
        "totalRevenue": 80000.0,
        "totalSessions": 1,
        "managerUserId": "mgr-guid-123"
      }
    ],
    "paymentMethods": [
      {
        "method": "QR_BANKING",
        "amount": 50000.0,
        "percentage": 62.5
      },
      {
        "method": "CASH",
        "amount": 30000.0,
        "percentage": 37.5
      }
    ]
  },
  "errors": []
}
```

#### B. Hợp đồng Admin API (`AdminRevenueReportResponse`)
```json
{
  "succeeded": true,
  "result": {
    "startDate": "2026-10-28",
    "endDate": "2026-10-28",
    "systemTotalRevenue": 1500000.0,
    "systemTotalSessions": 45,
    "activeLotsCount": 4,
    "byLot": [
      {
        "parkingLotId": 1,
        "parkingLotName": "Bãi xe Landmark 81",
        "totalRevenue": 80000.0,
        "totalSessions": 1,
        "managerUserId": "mgr-guid-123"
      }
    ],
    "paymentMethods": [
      {
        "method": "QR_BANKING",
        "amount": 1000000.0,
        "percentage": 66.7
      },
      {
        "method": "CASH",
        "amount": 500000.0,
        "percentage": 33.3
      }
    ]
  },
  "errors": []
}
```

---

## 5. Bảng Đối Soát 7 Trường Hợp Biên (Edge Cases)

| STT | Trường hợp nghiệp vụ | Trạng thái kỹ thuật & Giá trị thực tế | Doanh thu ghi nhận | Cơ chế giải quyết trong mã nguồn |
| :---: | :--- | :--- | :---: | :--- |
| **1** | **Khách có cọc bù trừ phí gửi xe** (Ví dụ: Tổng phí 80k, cọc 50k, trả thêm 30k) | `FinalFee = 80000`<br>`DepositApplied = 50000`<br>`PaidAmount = 30000` | **80.000 VND** | Cọc 50k được cấn trừ vào phí (`DepositApplied`). Doanh thu đúng bằng `FinalFee` (80.000 VND), không phải 30k hay 130k. |
| **2** | **Cọc lớn hơn phí gửi xe, hoàn tiền thừa** (Ví dụ: Cọc 50k, phí gửi 40k, hoàn cọc 10k) | `DepositApplied = 40000`<br>`PaidAmount = 0`<br>`FinalFee = 40000`<br>`DepositRefund = 10000` | **40.000 VND** | Bãi xe chỉ ghi nhận doanh thu theo dịch vụ thực tế đã cung cấp (`FinalFee` = 40.000 VND). Tiền hoàn 10k không ảnh hưởng `FinalFee`. |
| **3** | **Khách hủy đặt chỗ hoặc không đến (Cancelled / No-Show)** | `Reservation.Status` = `Cancelled` hoặc `NoShow`. Không có `ParkingSession` nào hoàn tất. | **0 VND** | Tuyệt đối không tính tiền cọc phạt vào doanh thu slot gửi xe của bãi. |
| **4** | **Xe đang đỗ hoặc đang chờ trả tiền (Active / PendingPayment)** | `session.Status IN ('Active', 'PendingPayment')`. Chưa hoàn tất thanh toán/chưa rời bãi. | **0 VND** | Doanh thu chưa thực hiện (unrealized revenue). Bị loại trừ khỏi tập truy vấn. |
| **5** | **Đóng bất thường / Thoát khẩn cấp (AbnormalClosed / EmergencyExit)** | Quản lý đóng phiên treo (`CloseAbnormallyAsync`) hoặc thoát khẩn cấp: phí được miễn (`Fee waived`), `FinalFee = 0`. | **0 VND** | Đóng góp 0.00 VND vào tổng doanh thu. |
| **6** | **Miễn phí (Grace Period / Whitelist)** | `session.ZeroFeeReason` = `GracePeriod` hoặc `Whitelist`. `FinalFee = 0`. | **0 VND** | Miễn phí gửi xe theo chính sách. |
| **7** | **Mất vé hoặc trễ hạn cửa ra (Lost ticket / Exit late fee)** | `settlement.LostTicketFee` hoặc `ExitWindowLateFee` được cộng gộp trực tiếp vào `session.FinalFee` (`GateService.cs#L778`). | **Toàn bộ `FinalFee`** | Phí phạt gắn liền với phiên gửi xe và là một phần cấu thành tổng phí slot mà khách phải thanh toán. |

---

## 6. Cơ Chế Phân Rã Phương Thức Thanh Toán (Payment-Method Breakdown)

Mỗi phiên đỗ xe hoàn tất được phân bổ phương thức thanh toán cấu thành:
1. **Phần trả thêm tại chỗ (`PaidAmount`):**
   Liên kết với bản ghi `Payment` có `SessionId == session.Id` và `Purpose == PaymentPurpose.ParkingFee`:
   - `PaymentMethod.Cash` ("Cash") $\rightarrow$ `CASH`
   - `PaymentMethod.Online` ("Online") $\rightarrow$ `QR_BANKING`
   - `PaymentMethod.Wallet` ("Wallet") $\rightarrow$ `RFID_WALLET`
2. **Phần bù trừ từ tiền cọc (`DepositApplied`):**
   Liên kết với bản ghi `Payment` có `ReservationId == session.ReservationId` và `Purpose == PaymentPurpose.Deposit`:
   - `PaymentMethod.Online` $\rightarrow$ `QR_BANKING`
   - `PaymentMethod.Wallet` $\rightarrow$ `RFID_WALLET`
3. **An toàn dữ liệu:**
   - Giá trị phương thức thanh toán không nhận diện được map sang `OTHER`, không bị ép sai thành `QR_BANKING`.
   - Xử lý chia cho 0 an toàn khi tổng doanh thu bằng 0 (`percentage = 0m`).
   - Đảm bảo đẳng thức tài chính:
     $$\sum \text{QR\_BANKING} + \sum \text{RFID\_WALLET} + \sum \text{CASH} + \sum \text{OTHER} = \text{totalRevenue}$$

---

## 7. Chi Tiết File Mã Nguồn Đã Triển Khai

| STT | File mã nguồn | Loại thay đổi | Trách nhiệm |
| :---: | :--- | :---: | :--- |
| **1** | `src/PMS.Application/Models/Reports/RevenueReportModels.cs` | Tạo mới | Định nghĩa các DTO: `ParkingLotRevenueModel`, `PaymentMethodBreakdownModel`, `ManagerRevenueReportResponse`, `AdminRevenueReportResponse`. |
| **2** | `src/PMS.Application/Services/IRevenueReportService.cs` | Tạo mới | Khai báo 2 phương thức: `GetManagerRevenueAsync` và `GetAdminRevenueAsync`. |
| **3** | `src/PMS.Application/Services/Impl/RevenueReportService.cs` | Tạo mới | Xử lý logic truy vấn dữ liệu từ `ParkingSession.FinalFee`, kiểm tra phân quyền Manager, phân rã phương thức thanh toán. |
| **4** | `src/PMS.API/Controllers/ManagerRevenueController.cs` | Tạo mới | Controller cho Manager tại `GET /api/manager/reports/revenue`, `[Authorize(Roles = "Manager")]`. |
| **5** | `src/PMS.API/Controllers/AdminRevenueController.cs` | Tạo mới | Controller cho Admin tại `GET /api/admin/reports/revenue`, `[Authorize(Roles = "Admin")]`. |
| **6** | `src/PMS.Application/OperationsDependencyInjection.cs` | Sửa đổi | Đăng ký `services.AddScoped<IRevenueReportService, RevenueReportService>();`. |
| **7** | `tests/PMS.Application.UnitTests/Operations/RevenueReportServiceTests.cs` | Tạo mới | 16 bài kiểm thử đơn vị cho toàn bộ các trường hợp biên tài chính, công thức cọc bù trừ, phân quyền Manager/Admin. |
| **8** | `tests/PMS.Api.IntegrationTests/Tests/ReportsEndpointTests.cs` | Tạo mới | 8 bài kiểm thử tích hợp API qua HTTP cho cả 2 endpoint Manager và Admin (bao gồm xác thực token, kiểm tra chặn 403, 404, 400, 401). |

---

## 8. Kết Quả Kiểm Thử Thực Tế (Automated Test Execution Results)

### 8.1. Unit Tests (`PMS.Application.UnitTests`)
- **Lệnh thực thi:** `dotnet test tests/PMS.Application.UnitTests/PMS.Application.UnitTests.csproj --filter 'FullyQualifiedName~RevenueReportServiceTests'`
- **Kết quả:** **16 / 16 PASSED** (100% pass rate, 0 failed, 0 skipped, thời gian: 885 ms).
- **Bộ test toàn bộ dự án Application:** **413 / 413 PASSED** (0 failed).

### 8.2. Integration Tests (`PMS.Api.IntegrationTests`)
- **Lệnh thực thi:** `dotnet test tests/PMS.Api.IntegrationTests/PMS.Api.IntegrationTests.csproj --filter 'FullyQualifiedName~ReportsEndpointTests'`
- **Kết quả:** **8 / 8 PASSED** (100% pass rate, 0 failed, 0 skipped, thời gian: 6 s).
  - `AdminReport_AsAdmin_ReturnsOk_WithAdminRevenueResponse`: 200 OK.
  - `AdminReport_WithNonexistentLot_ReturnsNotFound`: 404 Not Found.
  - `AdminReport_WithInvalidDateFormat_ReturnsBadRequest`: 400 Bad Request.
  - `ManagerReport_AsManager_ReturnsOk_WithManagerRevenueResponse`: 200 OK.
  - `ManagerReport_RequestingOtherLot_ReturnsForbidden`: 403 Forbidden.
  - `ManagerReport_Unauthenticated_ReturnsUnauthorized`: 401 Unauthorized.
  - `ManagerReport_AsStaff_ReturnsForbidden`: 403 Forbidden.
  - `AdminReport_AsManager_ReturnsForbidden`: 403 Forbidden.

### 8.3. Biên dịch Solution (`PMS.sln`)
- **Lệnh thực thi:** `dotnet build PMS.sln`
- **Kết quả:** `Build succeeded. 0 Warning(s), 0 Error(s).`

---

## 9. Bảng Đối Soát Tệp Tin Mã Nguồn Thực Tế (Source Traceability Matrix)

| Hạng mục nghiệp vụ | Tệp tin mã nguồn thực tế | Dòng code kiểm chứng |
| :--- | :--- | :--- |
| **Tổng phí slot (`FinalFee`)** | `src/PMS.Core/Entities/ParkingSession.cs`<br>`src/PMS.Core/Entities/ParkingSession.Operations.cs` | Line 68–74 (`OriginalFee`, `FinalFee`)<br>Line 34–39 (`DepositApplied`, `PaidAmount`) |
| **Tính phí & bù trừ cọc** | `src/PMS.Application/Helpers/Parking/FeeCalculator.cs`<br>`src/PMS.Application/Services/Impl/GateService.cs` | `FeeCalculator.cs#L123-135` (`Settle`)<br>`GateService.cs#L777-779` (`session.FinalFee = settlement...`) |
| **Thu tiền & cập nhật PaidAmount** | `src/PMS.Application/Services/Impl/PaymentService.cs` | Line 560–568 (`session.PaidAmount += payment.Amount;`) |
| **Hoàn tất phiên & hoàn cọc thừa** | `src/PMS.Application/Services/Impl/GateService.cs` | Line 450–488 (`session.Status = Completed; RefundDepositAsync(...)`) |
| **Phân quyền bãi của Manager (1:N)** | `src/PMS.Application/Services/Impl/EmployeeAccountService.cs`<br>`src/PMS.Application/Services/Impl/LotScopeService.cs` | `EmployeeAccountService.cs#L217-218`<br>`LotScopeService.cs#L29-35` |
| **Tiền lệ tính tổng chi tiêu slot** | `src/PMS.Application/Services/Impl/ParkingHistoryService.cs` | Line 48 (`await sessions.SumAsync(s => s.FinalFee);`) |
| **Bảo mật Exception Middleware** | `src/PMS.API/Middleware/ExceptionHandlingMiddleware.cs` | Line 35–43 (Chuyển đổi ForbiddenException $\rightarrow$ 403, NotFoundException $\rightarrow$ 404, BadRequestException $\rightarrow$ 400) |
