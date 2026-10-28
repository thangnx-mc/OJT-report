# Báo Cáo Rà Soát Codebase & Thiết Kế API Ghi Nhận Doanh Thu Bãi Đỗ Xe (Parking System Revenue API Specification & Architecture Design)

---

## 1. Mục Đích và Yêu Cầu Cập Nhật Nghiệp Vụ (Purpose & Business Requirements Update)

### 1.1. Bối cảnh nhiệm vụ
Nhiệm vụ được giao từ Trưởng nhóm:
> *“Làm API khi user thanh toán xong thì tiền cập nhật vào doanh thu của bãi đỗ, thằng manager xem dashboard thấy bãi đã có doanh thu.”*

Dựa trên các trao đổi và xác nhận nghiệp vụ mới nhất, yêu cầu thiết kế cho hệ thống Báo cáo Doanh thu (Revenue Reporting API) được cập nhật với hai định hướng kiến trúc cốt lõi:
1. **Phân tách hoàn toàn hai API theo vai trò (Role-Based API Separation):** Manager và Admin sử dụng hai API riêng biệt, không dùng chung một endpoint rồi phân nhánh kết quả theo role.
   - **Manager API:** Chỉ cung cấp doanh thu của bãi xe mà Manager đang trực tiếp quản lý (`ParkingLots.ManagerUserId == currentUserId`). Manager không được tự ý truyền `parkingLotId` của bãi không thuộc quyền quản lý (trả về lỗi HTTP 403 Forbidden). Quan hệ giữa Manager và bãi xe trong thực tế là **1 : N** (một Manager có thể quản lý nhiều bãi).
   - **Admin API:** Cung cấp doanh thu toàn hệ thống, tổng hợp tất cả các bãi đang hoạt động (`!p.IsDeleted`) và hỗ trợ phân rã hoặc lọc theo từng bãi cụ thể.
   - **Quy chuẩn định tuyến (Routing Convention):** Tên route tuân thủ nghiêm ngặt quy chuẩn của dự án (`api/manager/...` và `api/admin/...`).
2. **Quy tắc doanh thu chuẩn hóa ("Today's Revenue" = Phí gửi xe của Slot/Session):**
   - Doanh thu slot trong Manager Console **chỉ tính từ phí gửi xe của phiên đỗ (`ParkingSession.FinalFee`)**.
   - **Tuyệt đối không tính:** tiền cọc đặt chỗ rời rạc, tiền nạp ví nội bộ, tiền hoàn cọc, hoặc các khoản thu phạt/dịch vụ khác ngoài phí gửi xe.
   - **Không phân biệt phương thức thanh toán:** Tổng phí gửi xe của slot được ghi nhận trọn vẹn bất kể khách thanh toán bằng cọc, ví, tiền mặt hay cổng thanh toán trực tuyến.
   - *Ví dụ chuẩn hóa:* Khách có 50.000 VND tiền cọc được cấn trừ vào phí gửi xe tổng cộng 80.000 VND và thanh toán thêm 30.000 VND tại lối ra. Doanh thu slot của bãi phải ghi nhận đủ **80.000 VND** (không phải 30.000 VND và không phải 130.000 VND).

---

## 2. Xác Minh Baseline Codebase Thực Tế (Current Repository Baseline & Verification)

### 2.1. Backend Repository (`parking-management-system`)
- **Đường dẫn thư mục:** `C:\DEV\OJT-FSA-Team3\parking-management-system`
- **Nhánh Git hiện tại:** `develop` @ commit `2cf5eb1` (`feat: prevent duplicate phone number and citizen ID on profile update`)
- **Khảo sát Routing Convention:**
  - `src/PMS.API/Controllers/AdminController.cs#L14-15`: `[Route("api/admin")]`, `[Authorize(Roles = "Admin")]`.
  - `src/PMS.API/Controllers/ManagerController.cs#L14-15`: `[Route("api/manager")]`, `[Authorize(Roles = "Manager")]`.
  - `src/PMS.API/Controllers/ManagerOperationsController.cs#L11-12`: `[Route("api/manager")]`.
  - `src/PMS.API/Controllers/ManagerPricingController.cs#L15-16`: `[Route("api/manager/lots/{lotId:int}/pricing-rules")]`.
  - `src/PMS.API/Controllers/StaffController.cs#L13`: `[Route("api/staff")]`.
  - `src/PMS.API/Controllers/LotsController.cs#L15-16`: `[Route("api/lots")]`, `[Route("api/parking-lots")]`.
  $\rightarrow$ **Quy ước:** Toàn bộ controller phân quyền nội bộ trong backend sử dụng tiền tố theo vai trò không có versioning: `/api/admin/...` và `/api/manager/...`.

### 2.2. Frontend Repository (`parking-management-system-frontend`)
- **Đường dẫn thư mục:** `C:\DEV\OJT-FSA-Team3\parking-management-system-frontend`
- **Nhánh Git hiện tại:** `develop` @ commit `f0d6a00`
- **Hiện trạng tiêu thụ báo cáo:**
  - `src/pages/management/ManagerConsolePage.tsx#L60-69`: Hiển thị thẻ KPI **"Today's revenue"** với số tiền tổng (`vnd(revenue)`) và breakdown 2 kênh: `VietQR / bank` (`vnd(qr)`) và `Cash` (`vnd(cash)`).
  - `src/pages/admin/RevenueReportPage.tsx`: Màn hình báo cáo doanh thu quản trị chuyên sâu của Admin (KPI cards, biểu đồ xu hướng theo ngày, biểu đồ so sánh giữa các bãi xe, bảng phân rã chi tiết).
  - `src/config/env.ts#L23,75`: Định nghĩa `reports: '/v1/reports'` và fallback `VITE_MOCK_SERVICES=capacity,reports`.

---

## 3. Phân Tích Chuyên Sâu: Nguồn Dữ Liệu Doanh Thu Slot Trong Mã Nguồn

### 3.1. Thực thể lưu trữ phí gửi xe chuẩn xác: `ParkingSession`
Trong cơ sở dữ liệu và mã nguồn backend:
- `src/PMS.Core/Entities/ParkingSession.cs#L68-74`:
  - `OriginalFee`: Tổng phí đỗ xe tính toán theo thời lượng lưu bãi và biểu giá (`[decimal](18,2)`).
  - `DiscountAmount`: Giá trị giảm giá hoặc khuyến mãi (`[decimal](18,2)`).
  - `FinalFee`: **Tổng phí gửi xe cuối cùng mà phiên đỗ xe ghi nhận thu của khách hàng** (`[decimal](18,2)`).
  - `Status`: Trạng thái phiên đỗ (`Active`, `PendingPayment`, `Paid`, `Completed`, `AbnormalClosed`, `EmergencyExit`).
- `src/PMS.Core/Entities/ParkingSession.Operations.cs#L34-39`:
  - `DepositApplied`: Số tiền cọc từ đặt chỗ trước được cấn trừ vào phí phiên đỗ theo quy định FR-RSV-07 (`[decimal](18,2)`).
  - `PaidAmount`: Số tiền thực tế khách đã trả thêm tại làn ra / kiosk / app (`[decimal](18,2)`).

### 3.2. Truy vết toàn bộ Flow Tính Phí, Cấn Trừ Cọc và Checkout
1. **Tính toán cấn trừ cọc tại thời điểm Checkout (`QuoteCheckoutAsync`):**
   Tại `src/PMS.Application/Services/Impl/GateService.cs#L753-784`, dịch vụ cổng gọi bộ tính phí `FeeCalculator.Settle`:
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
2. **Thanh toán phần phí còn thiếu tại Lối ra / Kiosk / App:**
   Tại `src/PMS.Application/Services/Impl/PaymentService.cs#L560-568` (`SettleSession`):
   ```csharp
   session.PaidAmount += payment.Amount;
   session.Status = SessionStatus.Paid;
   session.UpdatedAt = Now;
   ```
   **Bất biến toán học luôn được bảo đảm:**
   $$\text{session.FinalFee} = \text{session.DepositApplied} + \text{session.PaidAmount}$$
   *Minh chứng thực tế:* Với phiên đỗ có tổng phí 80.000 VND, tiền cọc 50.000 VND:
   - `session.DepositApplied` = 50.000 VND.
   - `session.PaidAmount` = 30.000 VND.
   - `session.FinalFee` = **80.000 VND**.
3. **Xe qua cần Barie rời bãi (`CheckOutAsync`):**
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

### 3.3. Kết luận: Bảng `dbo.Payments` không phải là nguồn tính doanh thu slot
- Bảng `dbo.Payments` lưu trữ dòng tiền giao dịch tài chính rời rạc:
  - Bản ghi cọc (`Purpose = 'Deposit'`, 50.000 VND) được thanh toán lúc đặt chỗ (có thể từ vài ngày trước).
  - Bản ghi phí gửi xe (`Purpose = 'ParkingFee'`, 30.000 VND) được thanh toán lúc rời bãi hôm nay.
- Nếu chỉ truy vấn `Payments` hôm nay với `Purpose = 'ParkingFee'`: bãi xe chỉ thấy 30.000 VND $\rightarrow$ **mất 50.000 VND doanh thu slot**.
- Nếu cộng cả `Purpose = 'Deposit'`: các giao dịch cọc của khách hủy đặt chỗ hoặc No-Show (không hề sử dụng slot đỗ xe nào) sẽ bị tính nhầm vào "doanh thu slot".
- **Do đó: `dbo.ParkingSessions.FinalFee` là Single Source of Truth duy nhất phản ánh chính xác 100% doanh thu phí gửi xe của slot.**

---

## 4. Phân Tách Chi Tiết Hai API Theo Vai Trò (Manager vs Admin)

### 4.1. Bảng So Sánh Kiến Trúc Hai API

| Tiêu chí kỹ thuật | Manager Revenue API | Admin Revenue API |
| :--- | :--- | :--- |
| **Route quy chuẩn** | `GET /api/manager/reports/revenue` | `GET /api/admin/reports/revenue` |
| **Phân quyền (Authorize)** | `[Authorize(Roles = "Manager")]` | `[Authorize(Roles = "Admin")]` |
| **Mô hình sở hữu bãi** | **1 : N** (Một Manager có thể quản lý 0, 1 hoặc nhiều bãi xe, chứng minh tại [`EmployeeAccountService.cs#L217-218`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Services/Impl/EmployeeAccountService.cs#L217-L218) và [`AdminController.cs#L25-29`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminController.cs#L25-L29)) | Không giới hạn người quản lý |
| **Phạm vi truy vấn dữ liệu** | Giới hạn nghiêm ngặt trong các bãi do Manager gọi API quản lý: `pmsContext.ParkingLots.Where(p => p.ManagerUserId == currentUserId && !p.IsDeleted)` | Toàn hệ thống: `pmsContext.ParkingLots.Where(p => !p.IsDeleted)` |
| **Xử lý tham số `parkingLotId`** | **Tùy chọn**, nhưng **bắt buộc kiểm tra bảo mật**: Nếu Manager truyền `parkingLotId` mà mình không quản lý $\rightarrow$ Ném `ForbiddenException("You do not manage this parking lot.")` (HTTP 403). Nếu không truyền $\rightarrow$ Tổng hợp tất cả các bãi Manager đang quản lý. | **Tùy chọn**: Có thể truyền bất kỳ `parkingLotId` nào. Nếu bãi không tồn tại hoặc đã xóa $\rightarrow$ Ném `NotFoundException` (HTTP 404). Nếu không truyền $\rightarrow$ Tổng hợp toàn bộ bãi trong hệ thống. |
| **Mục đích sử dụng chính** | Thẻ KPI Today's Revenue trên Manager Console, giám sát vận hành ca trực của Manager. | Báo cáo tài chính vĩ mô, biểu đồ so sánh doanh thu liên bãi của Quản trị viên hệ thống. |

### 4.2. Hợp đồng Response DTO Đề Xuất

#### A. Response Hợp đồng Manager API (`ManagerRevenueReportResponse`)
```json
{
  "success": true,
  "data": {
    "managerUserId": "mgr-uuid-001",
    "startDate": "2026-10-28",
    "endDate": "2026-10-28",
    "totalRevenue": 80000.0,
    "totalSessions": 1,
    "byLot": [
      {
        "parkingLotId": 1,
        "parkingLotName": "Bãi xe Landmark 81",
        "totalRevenue": 80000.0,
        "totalSessions": 1
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
  }
}
```

#### B. Response Hợp đồng Admin API (`AdminRevenueReportResponse`)
```json
{
  "success": true,
  "data": {
    "startDate": "2026-10-28",
    "endDate": "2026-10-28",
    "systemTotalRevenue": 1500000.0,
    "systemTotalSessions": 45,
    "activeLotsCount": 4,
    "byLot": [
      {
        "parkingLotId": 1,
        "parkingLotName": "Bãi xe Landmark 81",
        "managerUserId": "mgr-uuid-001",
        "totalRevenue": 80000.0,
        "totalSessions": 1
      },
      {
        "parkingLotId": 2,
        "parkingLotName": "Bãi xe Bitexco",
        "managerUserId": "mgr-uuid-002",
        "totalRevenue": 1420000.0,
        "totalSessions": 44
      }
    ],
    "paymentMethods": [
      {
        "method": "QR_BANKING",
        "amount": 1000000.0,
        "percentage": 66.67
      },
      {
        "method": "CASH",
        "amount": 500000.0,
        "percentage": 33.33
      }
    ]
  }
}
```

---

## 5. Công Thức Tính Doanh Thu & Xử Lý Các Trường Hợp Ngoại Lệ

### 5.1. Công thức Doanh thu Slot
Với danh sách bãi xe được phân quyền (`ScopedLotIds`) và khoảng thời gian truy vấn `[FromUtc, ToUtc)`:

$$\text{Slot Revenue} = \sum_{\substack{s \in \text{ParkingSessions} \\ s.\text{ParkingLotId} \in \text{ScopedLotIds} \\ s.\text{Status} = \text{'Completed'} \\ s.\text{ExitTime} \ge \text{FromUtc} \land s.\text{ExitTime} < \text{ToUtc}}} s.\text{FinalFee}$$

*(Trường hợp phiên thanh toán trước `Paid` qua App trong bãi đang chờ qua barrier: nếu tính vào doanh thu tức thời thì điều kiện là `s.Status IN ('Completed', 'Paid')` và mốc thời gian là `s.ExitTime ?? s.UpdatedAt`).*

### 5.2. Bảng Đối Soát Toàn Diện 7 Trường Hợp Ngoại Lệ (Edge Cases)

| STT | Trường hợp nghiệp vụ | Trạng thái kỹ thuật & Giá trị thực tế | Doanh thu slot ghi nhận | Giải thích cơ chế nghiệp vụ |
| :---: | :--- | :--- | :---: | :--- |
| **1** | **Khách có cọc bù trừ phí gửi xe** (Ví dụ: Tổng phí 80k, cọc 50k, trả thêm 30k) | `session.OriginalFee = 80000`<br>`session.DepositApplied = 50000`<br>`session.PaidAmount = 30000`<br>`session.FinalFee = 80000` | **80.000 VND** | Cọc 50k được chuyển hóa thành một phần phí gửi xe (`DepositApplied`). Doanh thu slot đúng bằng `FinalFee` (80.000 VND), không phải 30k và không phải 130k. |
| **2** | **Cọc lớn hơn phí gửi xe, hoàn tiền thừa** (Ví dụ: Cọc 50k, phí gửi 40k, hoàn cọc 10k) | `session.DepositApplied = 40000`<br>`session.PaidAmount = 0`<br>`session.FinalFee = 40000`<br>`DepositRefund = 10000` (hoàn cọc qua payment) | **40.000 VND** | Bãi xe chỉ ghi nhận doanh thu slot theo dịch vụ thực tế đã cung cấp (`FinalFee` = 40.000 VND). 10.000 VND hoàn lại cho khách không làm ảnh hưởng `FinalFee`. |
| **3** | **Khách hủy đặt chỗ hoặc không đến (Cancelled / No-Show)** | `Reservation.Status` = `Cancelled` hoặc `NoShow`. Không có `ParkingSession` nào hoàn tất. | **0 VND** | Theo yêu cầu: *"Không tính tiền cọc, tiền nạp ví, hoàn cọc hoặc các khoản thu không phải phí gửi xe"*. Tiền cọc giữ lại nếu có là khoản phạt vi phạm đặt chỗ, không phải doanh thu phí gửi xe của slot. |
| **4** | **Xe đang đỗ hoặc đang chờ trả tiền (Active / PendingPayment)** | `session.Status IN ('Active', 'PendingPayment')`. Chưa hoàn tất thanh toán/chưa rời bãi. | **0 VND** | Doanh thu chưa thực hiện (unrealized revenue). Chỉ ghi nhận khi phiên đã hoàn tất dịch vụ. |
| **5** | **Đóng bất thường / Thoát khẩn cấp (AbnormalClosed / EmergencyExit)** | Quản lý đóng phiên treo (`CloseAbnormallyAsync`) hoặc thoát khẩn cấp: phí được miễn (`Fee waived`), `FinalFee = 0`. | **0 VND** | Không phát sinh doanh thu thu từ khách hàng. |
| **6** | **Miễn phí (Grace Period / Whitelist)** | `session.ZeroFeeReason` = `GracePeriod` (ra trong 15 phút) hoặc `Whitelist`. `FinalFee = 0`. | **0 VND** | Miễn phí gửi xe theo chính sách bãi. |
| **7** | **Mất vé hoặc trễ hạn cửa ra (Lost ticket / Exit late fee)** | `settlement.LostTicketFee` hoặc `ExitWindowLateFee` được cộng gộp trực tiếp vào `session.FinalFee` ([`GateService.cs#L778`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Services/Impl/GateService.cs#L778)). | **Toàn bộ `FinalFee`** | Phí phạt mất thẻ / trễ giờ ra gắn liền với phiên gửi xe đó và là một phần cấu thành tổng phí slot mà khách phải thanh toán. |

### 5.3. Cơ chế Phân Rã Phương Thức Thanh Toán (Payment Method Breakdown)
Để bảo đảm tổng các phương thức thanh toán khớp chính xác với `totalRevenue` (80.000 VND):
1. Phần trả thêm tại làn (`PaidAmount`): Map theo phương thức trên bản ghi `Payment` gắn với `SessionId` (Cash $\rightarrow$ `CASH`, Online $\rightarrow$ `QR_BANKING`, Wallet $\rightarrow$ `RFID_WALLET`).
2. Phần cọc bù trừ (`DepositApplied`): Map theo phương thức trên bản ghi `Payment` gắn với `ReservationId` (Online $\rightarrow$ `QR_BANKING`, Wallet $\rightarrow$ `RFID_WALLET`).
$$\sum \text{Phần Cọc} + \sum \text{Phần Trả thêm} = \sum \text{FinalFee} = \text{totalRevenue}$$
Đảm bảo biểu đồ cơ cấu thanh toán trên Dashboard khớp 100% không bị lệch sổ.

---

## 6. Kế Hoạch Triển Khai Tiếp Theo (Implementation Plan)

### 6.1. Danh sách file dự kiến tạo mới / chỉnh sửa
1. `src/PMS.Application/Models/Reports/RevenueReportModels.cs`: Cập nhật DTO request/response phân tách riêng cho Manager và Admin.
2. `src/PMS.Application/Services/IRevenueReportService.cs`: Khai báo 2 phương thức riêng biệt `GetManagerRevenueAsync` và `GetAdminRevenueAsync`.
3. `src/PMS.Application/Services/Impl/RevenueReportService.cs`: Chuyển đổi toàn bộ nguồn truy vấn từ `dbo.Payments` sang `dbo.ParkingSessions.FinalFee`, tích hợp kiểm tra bảo mật bãi của Manager.
4. `src/PMS.API/Controllers/ManagerRevenueController.cs`: Controller riêng cho Manager tại route `[Route("api/manager/reports/revenue")]` với `[Authorize(Roles = "Manager")]`.
5. `src/PMS.API/Controllers/AdminRevenueController.cs`: Controller riêng cho Admin tại route `[Route("api/admin/reports/revenue")]` với `[Authorize(Roles = "Admin")]`.
6. Unit Tests & Integration Tests:
   - `tests/PMS.Application.UnitTests/Operations/RevenueReportServiceTests.cs`: Kiểm thử toàn bộ công thức `FinalFee`, bù trừ cọc, hoàn cọc thừa, No-Show và phân quyền.
   - `tests/PMS.Api.IntegrationTests/Tests/ReportsEndpointTests.cs`: Kiểm thử 2 endpoint độc lập, kiểm tra chặn 403 Forbidden khi Manager truy cập bãi xe không phụ trách.

---

## 7. Bảng Đối Soát Tệp Tin Mã Nguồn Thực Tế (Source Traceability Matrix)

| Hạng mục nghiệp vụ | Tệp tin mã nguồn thực tế | Dòng code kiểm chứng |
| :--- | :--- | :--- |
| **Tổng phí slot (`FinalFee`)** | `src/PMS.Core/Entities/ParkingSession.cs`<br>`src/PMS.Core/Entities/ParkingSession.Operations.cs` | Line 68–74 (`OriginalFee`, `FinalFee`)<br>Line 34–39 (`DepositApplied`, `PaidAmount`) |
| **Tính phí & bù trừ cọc** | `src/PMS.Application/Helpers/Parking/FeeCalculator.cs`<br>`src/PMS.Application/Services/Impl/GateService.cs` | `FeeCalculator.cs#L123-135` (`Settle`)<br>`GateService.cs#L777-779` (`session.FinalFee = settlement...`) |
| **Thu tiền & cập nhật PaidAmount** | `src/PMS.Application/Services/Impl/PaymentService.cs` | Line 560–568 (`session.PaidAmount += payment.Amount;`) |
| **Hoàn tất phiên & hoàn cọc thừa** | `src/PMS.Application/Services/Impl/GateService.cs` | Line 450–488 (`session.Status = Completed; RefundDepositAsync(...)`) |
| **Phân quyền bãi của Manager (1:N)** | `src/PMS.Application/Services/Impl/EmployeeAccountService.cs`<br>`src/PMS.Application/Services/Impl/LotScopeService.cs` | `EmployeeAccountService.cs#L217-218`<br>`LotScopeService.cs#L29-35` |
| **Tiền lệ tính tổng chi tiêu slot** | `src/PMS.Application/Services/Impl/ParkingHistoryService.cs` | Line 48 (`await sessions.SumAsync(s => s.FinalFee);`) |
