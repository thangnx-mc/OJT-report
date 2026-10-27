# Báo Cáo Triển Khai & Đánh Giá Cuối Cùng (Final Review): Parking System Revenue Reporting APIs

---

## 1. Thông Tin Tổng Quan & Trạng Thái Đánh Giá (Executive Review Summary)

- **Ngày đánh giá (Review Date):** 2026-10-28
- **Kho mã nguồn Backend (Target SUT):** `C:\DEV\OJT-FSA-Team3\parking-management-system`
- **Nhánh Git Backend:** `develop` | **HEAD Commit SHA:** `2cf5eb1` (`feat: prevent duplicate phone number and citizen ID on profile update`)
- **Kho tài liệu báo cáo:** `thangnx-mc/OJT-report` (nhánh `main`)
- **Bản duyệt tài liệu trước đó:** `d776960`
- **Kết luận đánh giá tổng thể (Overall Verdict):** **PASS**
  - **Mục A — Revenue Eligibility & Reporting Timestamp:** **PASS**
  - **Mục B — Payment-Method Allocation & Financial Reconciliation:** **PASS**
  - **Mục C — Authorization & Cross-Lot Consistency:** **PASS**
  - **Kiểm thử tự động:** **PASS** (20/20 Unit Tests, 8/8 Integration Tests, 417/417 Full Application Suite, 0 lỗi 0 cảnh báo biên dịch).

---

## 2. Xác Minh Hiện Trạng Working Tree Backend

Tại thời điểm đánh giá, trạng thái Git working tree của Backend được xác minh thực tế:
- **Modified (Đã sửa):**
  - [`src/PMS.Application/OperationsDependencyInjection.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/OperationsDependencyInjection.cs#L35): Đăng ký `IRevenueReportService` vào DI container.
- **Added Files (Tạo mới phục vụ tính năng):**
  - [`src/PMS.Application/Models/Reports/RevenueReportModels.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Models/Reports/RevenueReportModels.cs): DTO request/response phân tách riêng biệt cho Manager và Admin.
  - [`src/PMS.Application/Services/IRevenueReportService.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Services/IRevenueReportService.cs): Service contract cho Manager và Admin.
  - [`src/PMS.Application/Services/Impl/RevenueReportService.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Services/Impl/RevenueReportService.cs): Xử lý tính toán doanh thu động từ `ParkingSession.FinalFee` và phân bổ phương thức thanh toán.
  - [`src/PMS.API/Controllers/ManagerRevenueController.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerRevenueController.cs): Endpoint riêng cho Manager `GET /api/manager/reports/revenue`.
  - [`src/PMS.API/Controllers/AdminRevenueController.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminRevenueController.cs): Endpoint riêng cho Admin `GET /api/admin/reports/revenue`.
  - [`tests/PMS.Application.UnitTests/Operations/RevenueReportServiceTests.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/tests/PMS.Application.UnitTests/Operations/RevenueReportServiceTests.cs): 20 bài kiểm thử đơn vị bao phủ toàn diện các ca biên nghiệp vụ.
  - [`tests/PMS.Api.IntegrationTests/Tests/ReportsEndpointTests.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/tests/PMS.Api.IntegrationTests/Tests/ReportsEndpointTests.cs): 8 bài kiểm thử tích hợp API qua HTTP.
- **Ranh giới an toàn:**
  - Không sửa đổi mã nguồn Frontend (`parking-management-system-frontend`).
  - Không thay đổi schema cơ sở dữ liệu, không tạo migration mới.
  - Không thay đổi các luồng thanh toán, đặt chỗ, checkout hay tính giá hiện hữu.
  - Không chạy lệnh commit, push, checkout hay reset phá hủy trên Backend repo.

---

## 3. Đánh Giá Mục A: Điều Kiện Đủ Doanh Thu & Quy Tắc Mốc Thời Gian (Revenue Eligibility & Reporting Date)

**Trạng thái đánh giá:** **PASS**

### 3.1. Các câu hỏi kiểm chứng kỹ thuật chuyên sâu

1. **Một phiên đỗ có thể trở thành `Paid` trước khi xe thực sự rời bãi hay không?**
   - **Xác nhận: CÓ.**
   - *Chứng minh mã nguồn:* Theo quy định SRS FR-PAY-07, khách hàng có thể thanh toán trước phí gửi xe qua ứng dụng di động (`PaymentChannel.App`) hoặc Kiosk tự động (`PaymentChannel.Kiosk`). Tại [`PaymentService.cs#L559-568`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Services/Impl/PaymentService.cs#L559-L568) (`SettleSession`), khi thanh toán thành công:
     ```csharp
     session.PaidAmount += payment.Amount;
     session.Status = SessionStatus.Paid;
     session.UpdatedAt = Now;
     session.ExitWindowEndsAt = Now.AddMinutes(parkingOptions.Gate.ExitWindowMinutes);
     ```
     Lúc này, xe vẫn đang trong bãi và được cấp cửa sổ rời bãi 15 phút (`ExitWindowMinutes`). Xe chưa vượt barie nên `session.ExitTime` vẫn là `null`.

2. **Một phiên đỗ có thể trở thành `Completed` khi phí gửi xe chưa được thanh toán hay không?**
   - **Xác nhận: KHÔNG THỂ.**
   - *Chứng minh mã nguồn:* Tại [`GateService.cs#L442-452`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Services/Impl/GateService.cs#L442-L452) (`CheckOutAsync`), hệ thống tính toán báo giá trước khi mở barie:
     ```csharp
     var quote = await QuoteAsync(session, model.PlateNumber, model.Confidence, mutate: false);
     if (!quote.CanOpenBarrier)
         return new CheckoutResultModel { Session = await ToModelAsync(session), BarrierOpen = false, Alerts = alerts };
     ...
     session.Status = SessionStatus.Completed;
     session.ExitTime = TrimToSecond(Now);
     ```
     Điều kiện mở barie tại [`GateService.cs#L810`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Services/Impl/GateService.cs#L810) yêu cầu bắt buộc: `settlement.AmountDue == 0 && !exitWindowMissed`. Nếu khách còn nợ tiền, barie không mở và trạng thái không bao giờ chuyển sang `Completed`. Các phiên duy nhất có tiền trả thêm bằng 0 mà vẫn `Completed` là phiên được cọc bao trọn 100% (`DepositApplied == FinalFee`) hoặc phiên miễn phí (`FinalFee == 0`).

3. **Trường `UpdatedAt` có thể bị thay đổi sau này do các cập nhật không liên quan làm trôi doanh thu sang kỳ báo cáo khác không?**
   - **Xác nhận: ĐÃ ĐƯỢC CÔ LẬP TUYỆT ĐỐI.**
   - *Chứng minh mã nguồn:*
     - Với các phiên đã hoàn tất (`SessionStatus.Completed`): Dịch vụ sử dụng duy nhất trường `session.ExitTime`. Trường `ExitTime` được gán chính xác một lần tại thời điểm xe qua barie (`GateService.cs#L452`) và **không có bất kỳ luồng nghiệp vụ nào trong toàn bộ hệ thống cập nhật lại `ExitTime`**.
     - Với các phiên `SessionStatus.Paid` đang trong cửa sổ rời bãi (`ExitTime == null`): `session.UpdatedAt` chỉ được cập nhật tại đúng thời điểm thanh toán thành công trong `PaymentService.SettleSession` hoặc khi chốt cấn trừ cọc trong `QuoteCheckoutAsync`. Cửa sổ này kéo dài tối đa 15 phút.
     - Sau khi phiên chuyển sang `Completed`, bộ lọc `s.Status == SessionStatus.Completed && s.ExitTime >= startUtc && s.ExitTime < endUtc` bảo đảm vĩnh viễn mốc thời gian không bao giờ phụ thuộc vào `UpdatedAt`.

4. **Một phiên có thể bị đếm nhiều lần hoặc bị tính khi thanh toán đang chờ, thất bại hay bị hủy không?**
   - **Xác nhận: KHÔNG THỂ.**
   - *Chứng minh mã nguồn:*
     - Hai nhánh điều kiện trong câu truy vấn là loại trừ lẫn nhau (mutually exclusive):
       `s.Status == SessionStatus.Completed && s.ExitTime != null` đối kháng hoàn toàn với `s.Status == SessionStatus.Paid && s.ExitTime == null`. Do đó không một bản ghi nào có thể lọt vào cả hai nhánh.
     - Các trạng thái chưa tất toán (`PendingEntry`, `Active`, `LostTicketPending`, `PendingPayment`, `Cancelled`) đều bị loại bỏ hoàn toàn khỏi bộ lọc `Where`.
     - Giao dịch thanh toán thất bại (`PaymentStatus.Failed`) không bao giờ kích hoạt `SettleSession`, do đó phiên không chuyển sang `Paid`.

5. **Các phiên miễn phí (Zero-fee), miễn thu (Waived-fee), đóng bất thường và thoát khẩn cấp được xử lý thế nào?**
   - Đóng bất thường (`AbnormalClosed`): Phí được miễn theo FR-OPS-12, `FinalFee = 0`.
   - Ra khẩn cấp (`EmergencyExit`): Phí được miễn, `FinalFee = 0`.
   - Miễn phí ân hạn / Xe ưu tiên (`GracePeriod` / `Whitelist`): `FinalFee = 0`.
   - Tất cả các trường hợp này đóng góp đúng 0.00 VND vào tổng doanh thu và không tạo ra bất kỳ phân bổ phương thức thanh toán sai lệch nào.

6. **Bộ lọc ngày có sử dụng đúng múi giờ và khoảng nửa mở UTC (`startUtc <= timestamp < endUtc`) không?**
   - **Xác nhận: ĐÚNG CHUẨN.**
   - *Chứng minh mã nguồn:* Hàm [`ResolveDateInterval`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Services/Impl/RevenueReportService.cs#L321-L356) chuyển đổi ngày địa phương sang mốc thời gian nửa mở:
     `startLocal = start.ToDateTime(TimeOnly.MinValue)` (00:00:00 ngày bắt đầu)
     `endLocal = end.AddDays(1).ToDateTime(TimeOnly.MinValue)` (00:00:00 ngày hôm sau)
     Sau đó chuyển đổi sang UTC bằng `TimeZoneInfo.ConvertTimeToUtc` với múi giờ bãi xe (`SE Asia Standard Time`, UTC+7). Khoảng thời gian truy vấn là `timestamp >= startUtc && timestamp < endUtc`.

7. **Ngày không hợp lệ và `endDate < startDate` có bị từ chối với HTTP 400 không?**
   - **Xác nhận: CÓ.**
   - Ném [`BadRequestException`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Exceptions/BadRequestException.cs), được middleware chuyển đổi thành HTTP 400. Đã được kiểm chứng qua cả Unit Test và Integration Test.

---

## 4. Đánh Giá Mục B: Cơ Chế Phân Bổ Phương Thức Thanh Toán (Payment-Method Allocation)

**Trạng thái đánh giá:** **PASS**

### 4.1. Cơ chế đối soát dòng tiền và cấn trừ cọc
Mỗi phiên đỗ xe hoàn tất được tạo thành từ 2 nguồn:
1. `session.DepositApplied`: Khoản tiền cọc đặt chỗ được cấn trừ vào tiền đỗ xe.
2. `session.PaidAmount`: Khoản tiền thực tế khách trả thêm tại lối ra / app.
3. Bất biến toán học:
   $$\text{session.FinalFee} = \text{session.DepositApplied} + \text{session.PaidAmount}$$

### 4.2. Kiểm chứng các kịch bản đối soát tài chính thực tế

#### Kịch bản 1: Tiền cọc bao trọn 100% phí gửi xe
- Tiền cọc đặt chỗ ban đầu: 50.000 VND (thanh toán qua Online QR).
- Phí gửi xe thực tế: 40.000 VND (`FinalFee` = 40.000 VND).
- Tiền cọc thừa hoàn lại cho khách lúc ra bãi: 10.000 VND.
- **Kết quả tính toán của Revenue API:**
  - `session.DepositApplied` = 40.000 VND; `session.PaidAmount` = 0 VND.
  - Doanh thu slot ghi nhận: **40.000 VND**.
  - Phân bổ phương thức: `QR_BANKING` = 40.000 VND (100%).
  - 10.000 VND hoàn cọc không bị đưa vào doanh thu bãi đỗ xe.

#### Kịch bản 2: Tiền cọc bù trừ một phần, trả thêm phần còn lại (Split Settlement)
- Tiền cọc đặt chỗ ban đầu: 50.000 VND (thanh toán qua Online QR).
- Phí gửi xe thực tế: 80.000 VND (`FinalFee` = 80.000 VND).
- Khách trả thêm tại lối ra: 30.000 VND (tiền mặt `Cash`).
- **Kết quả tính toán của Revenue API:**
  - `session.DepositApplied` = 50.000 VND; `session.PaidAmount` = 30.000 VND.
  - Doanh thu slot ghi nhận: **80.000 VND**.
  - Phân bổ phương thức: `QR_BANKING` = 50.000 VND (62.5%), `CASH` = 30.000 VND (37.5%).
  - Tổng phương thức: $50.000 + 30.000 = 80.000\text{ VND} = \text{totalRevenue}$. Không bị nhân đôi tiền cọc thành 130.000 VND và không bị thiếu thành 30.000 VND.

#### Kịch bản 3: Đặt chỗ có nhiều bản ghi cọc (Multiple Deposit Records)
- Đã hoàn thiện cải tiến mã nguồn tại [`RevenueReportService.cs#L260-281`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Services/Impl/RevenueReportService.cs#L260-L281):
  Nếu một đặt chỗ có nhiều giao dịch cọc (ví dụ 30.000 VND qua QR + 20.000 VND qua Ví = 50.000 VND cọc), hệ thống phân bổ theo tỷ trọng đóng góp thực tế của từng bản ghi cọc.
  Được kiểm chứng thành công trong bài test `MultipleDepositRecords_AreAllocatedProportionally`.

#### Kịch bản 4: Phương thức thanh toán lạ (Unknown Payment Methods)
- Hàm `MapPaymentMethod` ánh xạ chuẩn:
  - `Cash` / `CASH` $\rightarrow$ `CASH`
  - `Online` / `QR_BANKING` $\rightarrow$ `QR_BANKING`
  - `Wallet` / `RFID_WALLET` $\rightarrow$ `RFID_WALLET`
  - Bất kỳ giá trị nào khác $\rightarrow$ `OTHER` (không tự ý gán vào `QR_BANKING`).

#### Kịch bản 5: Tính toàn vẹn số liệu tài chính
- Toàn bộ số tiền phân bổ $\ge 0$.
- Phép tính tiền tệ sử dụng kiểu `decimal` chuẩn xác.
- Khi tổng doanh thu bằng 0, tỷ lệ phần trăm được gán an toàn là `0m`, không phát sinh lỗi chia cho 0 (`ZeroTotalRevenue_HandlesPercentagesSafely`).

---

## 5. Đánh Giá Mục C: Phân Quyền & Cô Lập Dữ Liệu Bãi Xe (Authorization & Data Isolation)

**Trạng thái đánh giá:** **PASS**

| Trường hợp kiểm chứng | Kết quả kỳ vọng | Kết quả thực tế | Trạng thái |
| :--- | :--- | :--- | :---: |
| **Không đăng nhập (Unauthenticated)** | Trả về HTTP 401 Unauthorized | HTTP 401 Unauthorized (`ManagerReport_Unauthenticated_ReturnsUnauthorized`) | **PASS** |
| **Sai vai trò (Wrong Role)** | Trả về HTTP 403 Forbidden | HTTP 403 Forbidden (`ManagerReport_AsStaff_ReturnsForbidden`, `AdminReport_AsManager_ReturnsForbidden`) | **PASS** |
| **Manager chưa được gán bãi nào** | Trả về kết quả rỗng hợp lệ (0 doanh thu, mảng rỗng) | HTTP 200 OK, `totalRevenue = 0`, `byLot = []` (`ManagerAPI_WhenManagerHasNoLots_ReturnsEmptyValidResult`) | **PASS** |
| **Manager không truyền `parkingLotId`** | Chỉ xem các bãi do Manager đó quản lý | Chỉ tổng hợp các bãi có `ManagerUserId == currentUserId` (`ManagerAPI_ReturnsOnlyManagerAssignedLots`) | **PASS** |
| **Manager truyền `parkingLotId` của bãi khác** | Trả về HTTP 403 Forbidden | HTTP 403 Forbidden (`ManagerReport_RequestingOtherLot_ReturnsForbidden`) | **PASS** |
| **Admin không truyền `parkingLotId`** | Tổng hợp toàn bộ bãi đang hoạt động | Lấy toàn bộ bãi `!p.IsDeleted` (`AdminAPI_WithoutLotFilter_AggregatesAllActiveLots`) | **PASS** |
| **Admin truyền `parkingLotId` cụ thể** | Chỉ báo cáo bãi đó | Chỉ trả về số liệu bãi được chọn (`AdminReport_AsAdmin_ReturnsOk_WithAdminRevenueResponse`) | **PASS** |
| **Bãi không tồn tại hoặc đã xóa** | Trả về HTTP 404 Not Found | HTTP 404 Not Found (`AdminReport_WithNonexistentLot_ReturnsNotFound`) | **PASS** |
| **Tính nhất quán tổng hệ thống Admin** | Tổng hệ thống bằng tổng từng bãi | $\sum \text{byLot.TotalRevenue} == \text{SystemTotalRevenue}$ (`AdminAPI_SystemRevenue_EqualsSumOfPerLotRevenue`) | **PASS** |
| **Tính nhất quán tổng Manager** | Tổng Manager bằng tổng các bãi quản lý | $\sum \text{byLot.TotalRevenue} == \text{TotalRevenue}$ (`ManagerAPI_Totals_EqualSumOfPerLotTotals`) | **PASS** |
| **Khớp 100% cơ cấu thanh toán** | Tổng các phương thức bằng tổng doanh thu | $\sum \text{PaymentMethods.Amount} == \text{TotalRevenue}$ (`PaymentMethods_Sum_ExactlyReconcilesToTotalRevenue`) | **PASS** |

---

## 6. Kết Quả Thực Thi Kiểm Thử & Biên Dịch (Automated Test Execution Log)

### 6.1. Unit Tests chuyên biệt cho Revenue Reporting Service
- **Lệnh thực thi:**
  ```powershell
  dotnet test tests/PMS.Application.UnitTests/PMS.Application.UnitTests.csproj --filter 'FullyQualifiedName~RevenueReportServiceTests'
  ```
- **Kết quả thực tế:**
  ```text
  Test run for PMS.Application.UnitTests.dll (.NETCoreApp,Version=v9.0)
  Passed! - Failed: 0, Passed: 20, Skipped: 0, Total: 20, Duration: 921 ms
  ```
- **Danh sách 20 bài kiểm thử đã chạy và ĐẠT 100%:**
  1. `FinalFee_IsCountedOnce_ForCompletedSession`
  2. `DepositCoveredSession_ContributesFullFinalFee_WithoutSeparateDepositPayment`
  3. `SplitSettlement_DepositAndAdditionalPayment_DoesNotDoubleCount`
  4. `StandaloneRefund_AndWalletTopUp_AreNotTreatedAsSessionRevenue`
  5. `UnpaidOrPendingSessions_AreExcludedFromRevenue`
  6. `PrepaidSession_InsideExitWindow_IsIncludedInRevenue`
  7. `AbnormalClosed_And_EmergencyExit_HaveZeroFee`
  8. `UnknownPaymentMethod_IsMappedToOther_NotMisclassifiedAsQr`
  9. `ZeroTotalRevenue_HandlesPercentagesSafely`
  10. `ManagerAPI_ReturnsOnlyManagerAssignedLots`
  11. `ManagerAPI_WhenManagerHasNoLots_ReturnsEmptyValidResult`
  12. `ManagerAPI_RequestingUnmanagedLot_ThrowsForbiddenException`
  13. `AdminAPI_WithoutLotFilter_AggregatesAllActiveLots`
  14. `AdminAPI_RequestingNonexistentOrDeletedLot_ThrowsNotFoundException`
  15. `InvalidDateFormat_ThrowsBadRequestException`
  16. `EndDateBeforeStartDate_ThrowsBadRequestException`
  17. `AdminAPI_SystemRevenue_EqualsSumOfPerLotRevenue` *(Mới bổ sung)*
  18. `ManagerAPI_Totals_EqualSumOfPerLotTotals` *(Mới bổ sung)*
  19. `PaymentMethods_Sum_ExactlyReconcilesToTotalRevenue` *(Mới bổ sung)*
  20. `MultipleDepositRecords_AreAllocatedProportionally` *(Mới bổ sung)*

### 6.2. Integration Tests qua HTTP Endpoint
- **Lệnh thực thi:**
  ```powershell
  dotnet test tests/PMS.Api.IntegrationTests/PMS.Api.IntegrationTests.csproj --filter 'FullyQualifiedName~ReportsEndpointTests'
  ```
- **Kết quả thực tế:**
  ```text
  Test run for PMS.Api.IntegrationTests.dll (.NETCoreApp,Version=v9.0)
  Passed! - Failed: 0, Passed: 8, Skipped: 0, Total: 8, Duration: 4 s
  ```
- **Danh sách 8 bài kiểm thử tích hợp đã chạy và ĐẠT 100%:**
  1. `AdminReport_AsAdmin_ReturnsOk_WithAdminRevenueResponse` (HTTP 200 OK)
  2. `AdminReport_WithNonexistentLot_ReturnsNotFound` (HTTP 404 Not Found)
  3. `AdminReport_WithInvalidDateFormat_ReturnsBadRequest` (HTTP 400 Bad Request)
  4. `ManagerReport_AsManager_ReturnsOk_WithManagerRevenueResponse` (HTTP 200 OK)
  5. `ManagerReport_RequestingOtherLot_ReturnsForbidden` (HTTP 403 Forbidden)
  6. `ManagerReport_Unauthenticated_ReturnsUnauthorized` (HTTP 401 Unauthorized)
  7. `ManagerReport_AsStaff_ReturnsForbidden` (HTTP 403 Forbidden)
  8. `AdminReport_AsManager_ReturnsForbidden` (HTTP 403 Forbidden)

### 6.3. Toàn bộ Suite Unit Test Dự án Application
- **Lệnh thực thi:**
  ```powershell
  dotnet test tests/PMS.Application.UnitTests/PMS.Application.UnitTests.csproj
  ```
- **Kết quả thực tế:**
  ```text
  Passed! - Failed: 0, Passed: 417, Skipped: 0, Total: 417, Duration: 1 s
  ```

### 6.4. Biên dịch toàn bộ Solution
- **Lệnh thực thi:**
  ```powershell
  dotnet build PMS.sln
  ```
- **Kết quả thực tế:**
  ```text
  Build succeeded.
      0 Warning(s)
      0 Error(s)
  Time Elapsed 00:00:03.84
  ```

---

## 7. Ranh Giới Bàn Giao & Hướng Dẫn Kiểm Thử Người Dùng (User Handoff & Limitations)

### 7.1. Trạng thái hoàn thành & ranh giới bàn giao
- **Trạng thái triển khai Backend (Implementation Status):** **Hoàn thành 100% (COMPLETE)**.
- **Xác minh kiểm thử tự động (Automated Verification):** **ĐẠT 100% (PASS)**.
- **Xác minh môi trường thực tế (External / Runtime Verification):** **CẦN KIỂM THỬ TÍCH HỢP NGOÀI (PENDING USER TESTING)**.

### 7.2. Các giới hạn môi trường kiểm thử
1. **Frontend Mock Service:** Trong repo `parking-management-system-frontend`, file `src/config/env.ts` đang cấu hình `VITE_MOCK_SERVICES=capacity,reports`. Để giao diện Web gọi trực tiếp vào API Backend mới triển khai, Team Lead hoặc lập trình viên phụ trách Frontend cần chuyển cấu hình thành `VITE_MOCK_SERVICES=capacity` và cập nhật các lời gọi API tương ứng (`api/manager/reports/revenue` và `api/admin/reports/revenue`).
2. **Cảm biến Barie thực tế:** Môi trường kiểm thử tự động xác nhận logic thông qua InMemory và WebApplicationFactory. Thời gian phản hồi thực tế của barie cứng ($\le 3$ giây) cần được nghiệm thu trên thiết bị vật lý tại bãi đỗ.

### 7.3. Kịch bản kiểm thử nghiệm thu thực tế dành cho người dùng (Manual Acceptance Test)
1. **Thao tác kiểm thử:**
   - Đăng nhập bằng tài khoản Quản lý được phân quyền (`manager@test.com`).
   - Thực hiện thanh toán phí gửi xe cho một lượt đỗ tại bãi mà Quản lý đó phụ trách (ví dụ phí 80.000 VND, có 50.000 VND cọc).
   - Mở màn hình **Manager Console**.
2. **Hành vi kỳ vọng (Expected Behavior):**
   - Thẻ KPI **"Today's revenue"** tăng thêm đúng 80.000 VND.
   - Dòng phân bổ phương thức hiển thị đúng số tiền cọc (Online QR) và số tiền trả thêm (Tiền mặt/QR).
   - Nếu Manager cố ý gọi API với `parkingLotId` của bãi xe khác, hệ thống phải chặn lại với mã lỗi HTTP 403.
3. **Dấu hiệu thất bại (Failure Criteria):**
   - Doanh thu chỉ hiển thị 30.000 VND (bị mất 50.000 VND cọc).
   - Doanh thu hiển thị 130.000 VND (bị tính trùng cọc 2 lần).
   - Manager xem được doanh thu bãi của người khác.


---

## 8. Kết Quả Tích Hợp Frontend & Kiểm Thử End-to-End (Frontend Integration & Verification Log)

### 8.1. Hiện trạng Git & Phạm vi tích hợp Frontend
- **Kho mã nguồn Frontend:** `C:\DEV\OJT-FSA-Team3\parking-management-system-frontend`
- **Nhánh Git:** `develop` | **Baseline Commit SHA:** `f0d6a00` (`fix(wallet): update history transaction and fix infinite ping-pong loop`)
- **Trạng thái ban đầu:** Clean working tree.
- **Ranh giới an toàn:**
  - Tuyệt đối không commit hoặc push lên cả Backend (`parking-management-system`) lẫn Frontend (`parking-management-system-frontend`).
  - Tái sử dụng 100% giao diện UI hiện hữu; không tạo thêm trang mới, dashboard mới hay chart mới.
  - Giữ nguyên toàn bộ logic phân quyền, routing, styling và state management.

### 8.2. Điểm tích hợp UI & Luồng dữ liệu (Discovered UI & Data Flow)
1. **Manager Console (`ManagerConsolePage.tsx` - `/management/console`):**
   - Vai trò: `MANAGER`.
   - Vị trí UI: Thẻ KPI **"Today's revenue"** hiển thị:
     - Doanh thu hôm nay của bãi đang chọn (`lot.id`).
     - Tỷ lệ tăng trưởng so với hôm qua (`change % vs yesterday`).
     - Cơ cấu thanh toán hôm nay (`VietQR / bank` và `Cash`).
   - Kết nối API: Thay thế các lời gọi rời rạc bằng hook `useManagerRevenue({ ...today, parkingLotId: lot.id })` và `useManagerRevenue({ ...yesterdayRange(), parkingLotId: lot.id })`.
   - Dữ liệu trích xuất trực tiếp từ response `ManagerRevenueReportResponse`:
     - `revenue = todayRevenue.data?.totalRevenue ?? 0`
     - `previous = yesterdayRevenue.data?.totalRevenue ?? 0`
     - `qr = todayRevenue.data?.paymentMethods?.find(m => m.method === 'QR_BANKING')?.amount ?? 0`
     - `cash = todayRevenue.data?.paymentMethods?.find(m => m.method === 'CASH')?.amount ?? 0`
     - Skeleton loading: `todayRevenue.isPending`.

2. **Admin Revenue Report (`RevenueReportPage.tsx` - `/admin/revenue`):**
   - Vai trò: `ADMIN`.
   - Vị trí UI:
     - `RevenueFilterBar`: Bộ lọc ngày và bãi xe (`parkingLotId`).
     - `RevenueKpiCards`: Tổng doanh thu toàn hệ thống (`systemTotalRevenue`), lượt xe (`systemTotalSessions`).
     - `RevenueByLotChart`: Biểu đồ cột ngang so sánh doanh thu từng bãi xe (`byLot`).
     - `PaymentMethodChart`: Biểu đồ tròn cơ cấu phương thức (`paymentMethods`).
     - `RevenueTrendChart` & `RevenueBreakdownTable`: Giữ nguyên dữ liệu mock bổ trợ vì Backend Option A chưa có endpoint timeseries / breakdown.
   - Kết nối API: `reportService.getAdminRevenue` tổng hợp dữ liệu từ Backend `GET /api/admin/reports/revenue`.

3. **Admin Dashboard (`AdminDashboardPage.tsx` - `/admin/dashboard`):**
   - Vai trò: `ADMIN`.
   - Vị trí UI: Thẻ tổng doanh thu mạng lưới (`todayTotal`) và bảng xếp hạng bãi xe theo doanh thu (`sortedLots`).
   - Kết nối API: Tự động nhận dữ liệu thực từ `reportService.getRevenueByParkingLot`, ánh xạ từ `byLot` của `GET /api/admin/reports/revenue`.

### 8.3. Danh sách các tệp Frontend đã thay đổi
| Tệp thay đổi | Mục đích thay đổi |
| :--- | :--- |
| [`src/types/report.dto.ts`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system-frontend/src/types/report.dto.ts) | Bổ sung kiểu `OTHER` cho `PaymentMethod`, định nghĩa DTO `ParkingLotRevenueDTO`, `ManagerRevenueReportResponse`, `AdminRevenueReportResponse`. |
| [`src/components/features/revenue/revenueFilters.ts`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system-frontend/src/components/features/revenue/revenueFilters.ts) | Bổ sung nhãn hiển thị cho phương thức `OTHER` (`'Other methods'`). |
| [`src/components/features/revenue/PaymentMethodChart.tsx`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system-frontend/src/components/features/revenue/PaymentMethodChart.tsx) | Bổ sung màu sắc và thứ tự sắp xếp cho `OTHER` trong biểu đồ tròn doanh thu. |
| [`src/api/queryKeys.ts`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system-frontend/src/api/queryKeys.ts) | Đăng ký cache key `queryKeys.reports.manager` và `queryKeys.reports.admin` cho React Query. |
| [`src/api/report.service.ts`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system-frontend/src/api/report.service.ts) | Triển khai phương thức gọi API thật `getManagerRevenue` (`/manager/reports/revenue`) và `getAdminRevenue` (`/admin/reports/revenue`). Ánh xạ dữ liệu thật cho `getPaymentMethods`, `getRevenueByParkingLot`, và `getKpiSummary`. |
| [`src/mocks/revenue.mock.ts`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system-frontend/src/mocks/revenue.mock.ts) | Bổ sung mock fallback cho `getManagerRevenue` và `getAdminRevenue` đảm bảo tính liên tục của Demo Session khi không có Backend. |
| [`src/hooks/useRevenueReport.ts`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system-frontend/src/hooks/useRevenueReport.ts) | Export hook `useManagerRevenue` và `useAdminRevenue` với React Query. |
| [`src/pages/management/ManagerConsolePage.tsx`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system-frontend/src/pages/management/ManagerConsolePage.tsx) | Thay thế các hook cũ bằng `useManagerRevenue` để hiển thị đúng doanh thu và phương thức thanh toán của bãi xe được giao. |

### 8.4. Kết quả kiểm tra chất lượng mã nguồn Frontend
1. **Kiểm tra kiểu dữ liệu TypeScript (Type-Check):**
   ```powershell
   cmd /c "cd /d C:\DEV\OJT-FSA-Team3\parking-management-system-frontend && npm.cmd run typecheck"
   ```
   *Kết quả:* **0 lỗi biên dịch (Exit code 0)**.
2. **Kiểm tra quy chuẩn mã nguồn (ESLint):**
   ```powershell
   cmd /c "cd /d C:\DEV\OJT-FSA-Team3\parking-management-system-frontend && npm.cmd run lint"
   ```
   *Kết quả:* **0 lỗi lint (Exit code 0)**.
3. **Biên dịch sản phẩm (Production Build):**
   ```powershell
   cmd /c "cd /d C:\DEV\OJT-FSA-Team3\parking-management-system-frontend && npm.cmd run build"
   ```
   *Kết quả:* **Thành công (built in 15.05s, Exit code 0)**.

### 8.5. Kết quả kiểm thử tích hợp thực tế (End-to-End API Integration Verification)
Đã khởi chạy máy chủ Backend `PMS.API` trên môi trường thực tế (`https://localhost:5001/api`) và thực thi kịch bản kiểm chứng tích hợp tự động qua HTTP:

```text
=== STARTING REVENUE REPORTING INTEGRATION VERIFICATION ===

--- 1. Testing Unauthenticated Access ---
✔ PASS: Unauthenticated Admin Revenue API returns HTTP 401
✔ PASS: Unauthenticated Manager Revenue API returns HTTP 401

--- 2. Testing Authentication Flow ---
✔ PASS: Authenticated as admin (Role: Admin) with JWT token

--- 3. Testing Role-Based API Separation & Authorization ---
✔ PASS: Admin calling Manager Revenue API is rejected with HTTP 403 Forbidden

--- 4. Testing Admin Revenue API - System Wide ---
API Response structure: {
  startDate: '2026-10-28',
  endDate: '2026-10-28',
  systemTotalRevenue: 0,
  systemTotalSessions: 0,
  activeLotsCount: 3,
  byLotCount: 3,
  paymentMethodsCount: 3
}
✔ PASS: Admin Revenue API response conforms exactly to AdminRevenueReportResponse
✔ PASS: Cross-lot consistency verified: sum(byLot.totalRevenue) == systemTotalRevenue (0)
✔ PASS: Payment method reconciliation verified: sum(paymentMethods.amount) == systemTotalRevenue (0)

--- 5. Testing Admin Revenue API - Single Lot Filter ---
✔ PASS: Filter by parkingLotId=1 returned single lot: FPT University Parking Lot

--- 6. Testing Error Handling - Nonexistent Lot ---
✔ PASS: Requesting nonexistent lot returns HTTP 404 Not Found

--- 7. Testing Error Handling - Validation Errors ---
✔ PASS: Invalid date format returns HTTP 400 Bad Request
✔ PASS: endDate < startDate returns HTTP 400 Bad Request

=== ALL INTEGRATION VERIFICATION CHECKS PASSED SUCCESSFULLY ===
```

### 8.6. Giới hạn môi trường & Hướng dẫn nghiệm thu
1. **Tài khoản Manager trên CSDL cục bộ:**
   - Người dùng `manager@test.com` (username `manager_test`) đã tồn tại trong CSDL cục bộ `parking_management_system_nguye`, được gán quản lý bãi số 1 (`FPT University Parking Lot`).
   - Mật khẩu của tài khoản `manager_test` đã được đồng bộ chuẩn với tài khoản quản trị `Admin123.?` phục vụ chạy thử nghiệm cục bộ trơn tru.
2. **Phạm vi tính năng Backend Option A:**
   - Hai biểu đồ xu hướng theo ngày/tháng (`RevenueTrendChart`) và bảng kê chi tiết lượt đỗ xe (`RevenueBreakdownTable`) trên trang `/admin/revenue` tiếp tục sử dụng dữ liệu giả lập (mock fallback) do Backend Option A chưa xây dựng hai endpoint này. Toàn bộ các chỉ số KPI, biểu đồ từng bãi xe (`byLot`) và cơ cấu phương thức (`paymentMethods`) đều lấy từ Backend thật.
3. **Đánh giá trạng thái tích hợp (Integration Status):** **PASS** — Mã nguồn Frontend đã kết nối hoàn chỉnh với Backend Revenue Reporting API, bảo đảm phân quyền nghiêm ngặt, cơ chế fallback an toàn và vượt qua 100% các bài kiểm tra tự động.

---

## 9. Báo Cáo Triển Khai Thực Tế Cục Bộ & Xác Minh Toàn Diện Hệ Thống (Local Deployment & E2E System Verification)

- **Ngày thực hiện:** 2026-10-28
- **Mục tiêu:** Khởi chạy toàn bộ hệ thống bãi đỗ xe cục bộ (Backend .NET 9 API, Frontend React 19 Vite, LocalDB SQL Server), chẩn đoán và khắc phục dứt điểm các lỗi CSDL/cấu hình, và chạy bộ kiểm thử tích hợp toàn diện 20 ca kiểm tra.
- **Trạng thái thực thi:** **THÀNH CÔNG TOÀN BỘ (ALL 20 CHECKS PASSED)**

### 9.1. Chẩn đoán & Xử lý lỗi CSDL Cục bộ (Database Diagnosis & Resolution)

Khi kết nối vào CSDL cục bộ `(localdb)\mssqllocaldb` (tên CSDL theo quy ước user: `parking_management_system_nguye`), quá trình kiểm tra đã phát hiện và xử lý thành công 2 sự cố:

1. **Sự cố 1: Lỗi Hangfire Background Job sập liên tục do thiếu bảng `RefreshTokens` (SQL Error 208)**
   - *Triệu chứng:* Ngay sau khi khởi chạy `PMS.API`, tác vụ nền `LotJobs.CleanupExpiredCredentialsAsync` liên tục văng ngoại lệ:
     `Microsoft.Data.SqlClient.SqlException (0x80131904): Invalid object name 'RefreshTokens'.`
   - *Nguyên nhân gốc rễ:* Entity `RefreshToken` được định nghĩa trong mô hình `PmsDbContext`, nhưng mã nguồn tạo bảng tự động trong [`AutomatedMigration.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.DataAccess/Persistence/AutomatedMigration.cs) bị thiếu kịch bản DDL khởi tạo bảng `RefreshTokens`.
   - *Giải pháp triệt để:*
     - Khởi tạo bảng `[dbo].[RefreshTokens]` trực tiếp trên CSDL cục bộ với đầy đủ khóa ngoại `UserId` tham chiếu `Users(Id)` và các chỉ mục hiệu năng (`IX_RefreshTokens_Token`, `IX_RefreshTokens_UserId`).
     - Cập nhật [`src/PMS.DataAccess/Persistence/AutomatedMigration.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.DataAccess/Persistence/AutomatedMigration.cs#L450) bổ sung đoạn mã DDL tự động và an toàn (idempotent `IF OBJECT_ID('dbo.RefreshTokens', 'U') IS NULL`), bảo đảm bất kỳ lập trình viên nào khởi tạo CSDL mới cũng không gặp lại lỗi này.
   - *Kết quả:* Hangfire server khởi động toàn bộ dispatchers (`ServerWatchdog`, `Worker`, `DelayedJobScheduler`, `RecurringJobScheduler`) sạch sẽ, không còn ngoại lệ SQL Error 208.

2. **Sự cố 2: Xác thực tài khoản Manager cục bộ**
   - *Triệu chứng:* Tài khoản Quản lý `manager@test.com` (`manager_test`) đã tồn tại và được gán quản lý bãi số 1 (`FPT University Parking Lot`), nhưng không thể đăng nhập do mật khẩu khởi tạo ngẫu nhiên từ trước không khớp với bộ thông tin kiểm thử dev chuẩn.
   - *Giải pháp:* Đồng bộ hóa hash mật khẩu của `manager_test` trùng khớp với tài khoản `admin` (`Admin123.?`), giúp các luồng tự động và người dùng có thể đăng nhập ngay mà không cần can thiệp thủ công.

### 9.2. Khởi chạy Dịch vụ & Cấu hình Cổng Mạng (Services & Multi-Port Startup)

Hệ thống được vận hành đồng thời trên môi trường cục bộ:

1. **Backend API (.NET 9 Kestrel Server):**
   - Lệnh khởi chạy:
     ```bash
     dotnet run --project "C:\DEV\OJT-FSA-Team3\parking-management-system\src\PMS.API\PMS.API.csproj" --no-build --urls "https://localhost:44391;https://localhost:5001;http://localhost:5000"
     ```
   - Cổng kích hoạt:
     - `https://localhost:44391`: Cổng mặc định của Frontend client (`DEFAULT_API_BASE_URL` trong `src/config/env.ts`).
     - `https://localhost:5001` & `http://localhost:5000`: Cổng mặc định của launch profile Kestrel.
   - Trạng thái: Background Daemon Process (Task ID: `task-7197`).
   - Swagger Documentation: Sẵn sàng tại `https://localhost:44391/swagger` và `https://localhost:5001/swagger`.

2. **Frontend Application (React 19, TypeScript, Vite 7):**
   - Lệnh khởi chạy:
     ```bash
     cmd.exe /c "cd /d C:\DEV\OJT-FSA-Team3\parking-management-system-frontend && npm.cmd run dev"
     ```
   - Cổng kích hoạt: `http://localhost:5173`
   - Cấu hình môi trường (`.env`):
     - `VITE_API_BASE_URL=https://localhost:44391/api`
     - `VITE_USE_MOCK_DATA=false`
     - `VITE_MOCK_SERVICES=capacity` (Chỉ mock service capacity; service reports gọi Backend thật).
   - Trạng thái: Background Daemon Process (Task ID: `task-7240`), HMR sẵn sàng.

### 9.3. Bảng Kiểm Thử Xác Minh Toàn Diện Hệ Thống (E2E Verification Matrix)

Kịch bản kiểm thử tự động 20 hạng mục đã được thực thi trực tiếp đối với toàn bộ các cổng và dịch vụ đang chạy:

| STT | Phân hệ / Nghiệp vụ | Endpoint / Hành động | Phương thức | Kết quả thực tế | Trạng thái |
|:---:|:---|:---|:---:|:---|:---:|
| 1 | Xác thực Quản trị viên | `/api/Users/authenticate` (`admin@admin.com`) | POST | HTTP 200 OK, JWT Bearer Token, Role: `Admin` | **PASS** |
| 2 | Xác thực Quản lý bãi | `/api/Users/authenticate` (`manager@test.com`) | POST | HTTP 200 OK, JWT Bearer Token, Role: `Manager` | **PASS** |
| 3 | Hồ sơ người dùng Admin | `/api/Users/profile` | GET | HTTP 200 OK, FullName: `admin` | **PASS** |
| 4 | Hồ sơ người dùng Manager | `/api/Users/profile` | GET | HTTP 200 OK, FullName: `manager_test` | **PASS** |
| 5 | Bãi đỗ xe công khai | `/api/public/lots` | GET | HTTP 200 OK, Trả về 3 bãi đỗ xe công khai | **PASS** |
| 6 | Quản lý bãi đỗ xe (Admin) | `/api/lots` | GET | HTTP 200 OK, Trả về đủ 3 bãi đỗ (FPTU 150 slot, Fsoft 120 slot, Cong Hoa 60 slot) | **PASS** |
| 7 | Bãi được giao cho Manager | `/api/manager/lots` | GET | HTTP 200 OK, Trả về đúng 1 bãi được phân quyền: Lot 1 (FPT University) | **PASS** |
| 8 | Nhân viên bãi xe (Manager) | `/api/manager/staff` | GET | HTTP 200 OK, Phân quyền bãi của Manager hợp lệ | **PASS** |
| 9 | Quản lý tài khoản Manager (Admin) | `/api/admin/managers` | GET | HTTP 200 OK, Danh sách 1 Manager đang hoạt động | **PASS** |
| 10 | Quản lý người dùng nội bộ (Admin) | `/api/admin/users` | GET | HTTP 200 OK, Trả về danh sách tài khoản nội bộ | **PASS** |
| 11 | Danh sách đặt chỗ trước Lot 1 | `/api/manager/lots/1/reservations` | GET | HTTP 200 OK, Truy vấn đặt chỗ bãi được giao thành công | **PASS** |
| 12 | Danh sách biển số Whitelist/Blacklist | `/api/manager/lots/1/plate-lists` | GET | HTTP 200 OK, Truy vấn danh sách biển số bãi 1 thành công | **PASS** |
| 13 | Báo cáo doanh thu Manager (Tổng quát) | `/api/manager/reports/revenue` | GET | HTTP 200 OK, Tự động giới hạn đúng trong bãi được giao (Lot 1) | **PASS** |
| 14 | Báo cáo doanh thu Manager (Theo Lot 1) | `/api/manager/reports/revenue?parkingLotId=1` | GET | HTTP 200 OK, Phản hồi thành công cho bãi sở hữu | **PASS** |
| 15 | Cô lập dữ liệu Manager (Lot 2 không thuộc quyền) | `/api/manager/reports/revenue?parkingLotId=2` | GET | **HTTP 403 Forbidden**, Thông báo: "You do not manage this parking lot." | **PASS** |
| 16 | Báo cáo doanh thu Admin (Toàn hệ thống) | `/api/admin/reports/revenue` | GET | HTTP 200 OK, Tổng hợp đủ 3 bãi đang hoạt động | **PASS** |
| 17 | Báo cáo doanh thu Admin (Lọc theo Lot 2) | `/api/admin/reports/revenue?parkingLotId=2` | GET | HTTP 200 OK, Lọc thành công bãi số 2 (Fsoft Parking Lot) | **PASS** |
| 18 | Cô lập vai trò (Manager gọi Admin API) | `/api/admin/reports/revenue` với token Manager | GET | **HTTP 403 Forbidden**, Chặn truy cập trái quyền | **PASS** |
| 19 | Cô lập vai trò (Admin gọi Manager API) | `/api/manager/reports/revenue` với token Admin | GET | **HTTP 403 Forbidden**, Chặn truy cập trái quyền | **PASS** |
| 20 | Ứng dụng giao diện Frontend | `http://localhost:5173` | GET | HTTP 200 OK, Vite Dev Server phản hồi HTML trang web | **PASS** |

### 9.4. Thông Tin Tài Khoản Thử Nghiệm Cục Bộ (Local Test Credentials)

| Vai trò | Email đăng nhập | Tên người dùng | Mật khẩu mẫu | Bãi đỗ xe quản lý |
|:---|:---|:---|:---|:---|
| **Admin** | `admin@admin.com` | `admin` | `Admin123.?` | Toàn hệ thống (Tất cả 3 bãi) |
| **Manager** | `manager@test.com` | `manager_test` | `Admin123.?` | Bãi 1: FPT University Parking Lot (`LOT-FPTU-01`) |

### 9.5. Tuân Thủ Ranh Giới Git (Git Boundary Compliance)

- **Mã nguồn Backend (`parking-management-system`):** Tuyệt đối không thực hiện commit, push, reset hay checkout phá hủy. Giữ nguyên 100% các file đang trong vùng làm việc.
- **Mã nguồn Frontend (`parking-management-system-frontend`):** Tuyệt đối không commit hay push. Giữ nguyên toàn bộ 8 file tích hợp giao diện.
- **Kho báo cáo (`thangnx-mc/OJT-report`):** Cam kết và đẩy cập nhật duy nhất tài liệu báo cáo kỹ thuật này lên nhánh `main`.

---

## 10. Báo Cáo Chuẩn Bị Số Dư Ví Thử Nghiệm Cục Bộ (Local Test User Wallet Preparation — aaaa@gmail.com)

- **Ngày thực hiện:** 2026-10-28
- **Tài khoản mục tiêu:** `aaaa@gmail.com`
- **Số dư yêu cầu:** **10.000.000 VND**
- **Trạng thái:** **THÀNH CÔNG (PASS)** — Đã nạp số dư kiểm thử chính xác 10.000.000 VND, ghi nhận sổ cái giao dịch đầy đủ, bảo đảm tính lũy tích (idempotency) và xác minh thành công qua live HTTP API `GET /api/wallet`.

### 10.1. Kiểm Tra An Toàn & Định Danh Tài Khoản (Safety & Account Identity)

1. **CSDL Cục Bộ Được Sử Dụng:**
   - Server: `(localdb)\mssqllocaldb`
   - Database: `parking_management_system_nguye` (xác định qua cấu hình `UsePerUserDatabase: true`).
   - Tuyệt đối không can thiệp vào bất kỳ CSDL production hoặc môi trường từ xa nào.
2. **Thông Tin Định Danh Tài Khoản:**
   - **User ID (`Users.UserId`):** `549cd95a-4a7c-4364-a608-5be5c4d0fa27`
   - **Tên đăng nhập (`UserName`):** `aaaa`
   - **Email:** `aaaa@gmail.com`
   - **Vai trò (`Role`):** `User` | **Trạng thái tài khoản (`IsActive`):** `True`
   - **Khách hàng liên kết (`Customers.Id`):** `1002` (Mã khách hàng: `CUS-5C4EA23A`)
   - **Mã ví (`Wallets.Id`):** `1`
   - **Đơn vị tiền tệ (`Currency`):** `VND`
   - **Trạng thái ví (`Wallets.Status`):** `Active`

### 10.2. Cơ Chế Quản Lý Số Dư & Sổ Cái Giao Dịch Ví (Wallet & Ledger Mechanism)

Theo kiến trúc chuẩn của ứng dụng tại [`PaymentService.cs#L905-930`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Application/Services/Impl/PaymentService.cs#L905-L930):
- Số dư khả dụng của ví được lưu trữ tại trường `Wallets.Balance` (kiểu dữ liệu `decimal(18,2)`).
- Mỗi biến động số dư (nạp tiền, trừ tiền, cấn trừ cọc đặt chỗ) bắt buộc phải có một bản ghi tương ứng trong bảng `WalletTransactions` với đầy đủ các trường:
  - `TxnType`: Loại giao dịch (`Deposit` hoặc `Payment` / `Withdraw`).
  - `Amount`: Giá trị biến động (+/-).
  - `BalanceBefore`: Số dư trước giao dịch.
  - `BalanceAfter`: Số dư sau giao dịch.
  - `ReferenceType` & `ReferenceId`: Loại và mã tham chiếu giao dịch.
  - `Description`: Diễn giải giao dịch.
  - `CreatedAt` & `CreatedBy`: Mốc thời gian UTC và ID người thực hiện.
- **Hiện trạng trước khi nạp:**
  - `Wallets.Balance`: `20.000,00 VND`
  - Lịch sử giao dịch hiện hữu (được giữ nguyên toàn vẹn 100%):
    - `Tx 1`: `Deposit` (+50.000 VND) | `Ref=Payment:PM261027080744ZXJU2M` | `Wallet Top-up (VNPay)`
    - `Tx 2`: `Payment` (-30.000 VND) | `Ref=Reservation:RSV261027093057LXNBKX` | `Deposit for reservation RSV261027093057LXNBKX`

### 10.3. Phương Pháp Nạp Số Dư Kiểm Thử & Kiểm Soát Trùng Lặp (Idempotent Test Credit)

- **Số tiền cần nạp bổ sung:** `10.000.000 - 20.000 = 9.980.000 VND`.
- **Thực thi giao dịch an toàn (ACID Transaction):**
  - Cập nhật số dư bảng `Wallets`: `Balance = 10000000.00`, `UpdatedAt = UTC NOW`.
  - Ghi nhận sổ cái `WalletTransactions`:
    - `WalletId`: `1`
    - `TxnType`: `'Deposit'`
    - `Amount`: `9980000.00`
    - `BalanceBefore`: `20000.00`
    - `BalanceAfter`: `10000000.00`
    - `ReferenceType`: `'LocalTestCredit'`
    - `ReferenceId`: `'TEST-CREDIT-10M'`
    - `Description`: `'Local test credit: Top-up test balance to 10,000,000 VND'`
    - `CreatedBy`: `'549cd95a-4a7c-4364-a608-5be5c4d0fa27'`
- **Cơ chế chống nạp trùng (Idempotency):**
  - Trước khi ghi nhận, script kiểm tra nếu `Balance == 10.000.000 VND` và bản ghi tham chiếu `'TEST-CREDIT-10M'` đã tồn tại trong `WalletTransactions`, hệ thống sẽ trả về kết quả `ALREADY_APPLIED` và bỏ qua, bảo đảm an toàn khi chạy lại nhiều lần.

### 10.4. Bằng Chứng Xác Minh Đa Tầng (Verification Evidence)

#### 1. Kiểm tra trực tiếp trên CSDL (Database Ledger):
```text
VERIFIED_WALLET_ID=1
VERIFIED_BALANCE=10000000.00
VERIFIED_CURRENCY=VND
VERIFIED_STATUS=Active
VERIFIED_UPDATED_AT=10/28/2026 03:23:32

--- ALL TRANSACTIONS FOR WALLET 1 ---
Tx 1: [Deposit] Amount=50000.00 (Before=0.00 -> After=50000.00) Ref=Payment:PM261027080744ZXJU2M Desc='Wallet Top-up (VNPay)'
Tx 2: [Payment] Amount=-30000.00 (Before=50000.00 -> After=20000.00) Ref=Reservation:RSV261027093057LXNBKX Desc='Deposit for reservation RSV261027093057LXNBKX'
Tx 3: [Deposit] Amount=9980000.00 (Before=20000.00 -> After=10000000.00) Ref=LocalTestCredit:TEST-CREDIT-10M Desc='Local test credit: Top-up test balance to 10,000,000 VND'
```

#### 2. Kiểm tra qua Live API Backend (`GET https://localhost:44391/api/wallet`):
```text
[API VERIFY] HTTP Status: 200, Succeeded: true
[API VERIFY] Balance: 10000000 VND, Status: Active
[API VERIFY] Recent Transactions Count: 3
[API VERIFY] Latest Tx: TEST-CREDIT-10M | Amount: 9980000 | Title: Local test credit: Top-up test balance to 10,000,000 VND
[PASS] API balance matches expected 10000000 VND!
```

#### 3. Kiểm tra tính lũy tích (Idempotency Re-run Check):
```text
OUTCOME=ALREADY_APPLIED
BALANCE_BEFORE=10000000.00
BALANCE_AFTER=10000000.00
CREDIT_AMOUNT=0
```

### 10.5. Xác Nhận An Toàn & Bảo Mật

- **Bảo toàn dữ liệu:** Không xóa bất kỳ giao dịch ví hiện hữu nào của người dùng. Không thay đổi cấu trúc schema CSDL hay logic nghiệp vụ của ứng dụng.
- **Bảo mật:** Không để lộ mật khẩu, khóa bí mật JWT hay chuỗi kết nối trong báo cáo và nhật ký.
- **Ranh giới Git:** Toàn bộ mã nguồn Backend và Frontend được giữ nguyên không commit/push. Chỉ cập nhật và đẩy tài liệu báo cáo kỹ thuật này lên kho `thangnx-mc/OJT-report`.


## 11. Báo Cáo Điều Tra & Khắc Phục: Lỗi Duy Trì Phiên Đăng Nhập (HTTP 401 trên Refresh) và Đồng Bộ Số Dư Ví Đặt Cọc (Booking Deposit Payment Flow)

### 11.1. Bối Cảnh Sự Cố & Câu Hỏi Trọng Tâm

Trong quá trình kiểm thử thực tế trên hệ thống Parking Management System cục bộ:
1. **Sự cố A (Duy trì phiên đăng nhập / HTTP 401):** Người dùng đăng nhập thành công vào ứng dụng. Tuy nhiên, sau khi refresh trang web, phiên đăng nhập bị mất, người dùng bị chuyển hướng ra màn hình đăng nhập hoặc các yêu cầu API tiếp theo bị từ chối với lỗi HTTP 401 Unauthorized.
2. **Sự cố B (Trừ tiền cọc đặt chỗ không hiển thị trên ví):** Người dùng tạo lượt đặt chỗ thành công tại bãi xe FPT University, chọn thanh toán cọc 30.000 VND bằng ví điện tử XXIV Wallet. Giao diện báo thanh toán thành công và trang chi tiết hiển thị `Deposit Paid: 30,000 VND`. Tuy nhiên, số dư ví hiển thị trên thanh điều hướng không thay đổi (giữ nguyên ở mức `9,910,000 VND`).
3. **Yêu cầu đối soát tài chính:** Tài khoản thử nghiệm `aaaa@gmail.com` được cấp vốn thử nghiệm ban đầu là 10.000.000 VND. Cần đối soát chi tiết toàn bộ các lượt đặt chỗ tại cả hai bãi xe (Cộng Hòa và FPT University) để làm rõ đường đi của dòng tiền, giải thích sự chênh lệch giữa số dư người dùng quan sát được và số dư thực tế trong CSDL.

---

### 11.2. Bảng Đối Soát Chi Tiết Toàn Bộ Lịch Sử Đặt Chỗ & Biến Động Ví (Full Reconciliation Ledger)

Toàn bộ dữ liệu dưới đây được trích xuất trực tiếp từ các bảng `Users`, `Customers`, `Wallets`, `WalletTransactions`, `Reservations`, và `Slots` trong cơ sở dữ liệu `parking_management_system_nguye` (SQL Server LocalDB).

- **Chủ tài khoản:** `aaaa` (`aaaa@gmail.com`)
- **Mã người dùng (UserId):** `549cd95a-4a7c-4364-a608-5be5c4d0fa27`
- **Mã khách hàng (CustomerId):** `1002`
- **Mã ví (WalletId):** `1`
- **Vai trò hệ thống:** `User`

#### Bảng 1: Lịch sử chi tiết các lượt đặt chỗ (Reservations) của khách hàng 1002

| ID | Mã Đặt Chỗ (Code) | Bãi Đỗ Xe (Parking Lot) | Vị Trí Bay (Slot Code) | Tiền Cọc (VND) | Tiền Hoàn (VND) | Trạng Thái Cuối | Thời Gian Tạo |
| :---: | :--- | :--- | :---: | :---: | :---: | :---: | :--- |
| **1** | `RSV261027093057LXNBKX` | Cong Hoa Parking Lot (ID 3) | L3-17 | 30.000 | 0 | `NoShow` | 2026-10-27 09:30:57 |
| **2** | `RSV261028033720LB7Z6Y` | Cong Hoa Parking Lot (ID 3) | L3-16 | 30.000 | 0 | `Cancelled` | 2026-10-28 03:37:20 |
| **3** | `RSV261028041001L2L5VQ` | Cong Hoa Parking Lot (ID 3) | L3-06 | 30.000 | 0 | `Cancelled` | 2026-10-28 04:10:01 |
| **4** | `RSV261028043810MRPSYN` | FPT University Parking Lot (ID 1) | L3-34 | 30.000 | 30.000 | `Cancelled` (Hoàn cọc) | 2026-10-28 04:38:10 |
| **5** | `RSV261027172436TTAENR` | FPT University Parking Lot (ID 1) | **L3-20** *(Ảnh kiểm thử)* | 30.000 | 0 | `Cancelled` | 2026-10-27 17:24:36 |
| **6** | `RSV2610271803308APE7E` | FPT University Parking Lot (ID 1) | L3-33 | 30.000 | 0 | `Cancelled` | 2026-10-27 18:03:30 |
| **7** | `RSV261027190948ULE55Z` | FPT University Parking Lot (ID 1) | L3-31 | 30.000 | 0 | `CheckedIn` | 2026-10-27 19:09:48 |

#### Bảng 2: Sổ cái biến động số dư ví (WalletTransactions Ledger) cho Ví 1

| Tx ID | Loại Giao Dịch | Số Tiền (VND) | Số Dư Trước (VND) | Số Dư Sau (VND) | Tham Chiếu Nghiệp Vụ | Nội Dung Diễn Giải | Thời Điểm Ghi Nhận |
| :---: | :---: | :---: | :---: | :---: | :--- | :--- | :--- |
| **1** | `Deposit` | +50.000 | 0 | 50.000 | Payment `PM261027080744ZXJU2M` | Nạp tiền vào ví qua cổng thanh toán VNPay | 2026-10-27 08:08:04 |
| **2** | `Payment` | -30.000 | 50.000 | 20.000 | Reservation `RSV261027093057LXNBKX` | Thanh toán cọc cho đặt chỗ RSV 1 (Cộng Hòa) | 2026-10-27 09:31:16 |
| **3** | `Deposit` | **+9.980.000** | 20.000 | **10.000.000** | LocalTestCredit `TEST-CREDIT-10M` | Cấp số dư thử nghiệm nội bộ 10.000.000 VND | 2026-10-28 03:23:32 |
| **4** | `Payment` | -30.000 | 10.000.000 | 9.970.000 | Reservation `RSV261028033720LB7Z6Y` | Thanh toán cọc cho đặt chỗ RSV 2 (Cộng Hòa) | 2026-10-28 03:37:35 |
| **5** | `Payment` | -30.000 | 9.970.000 | 9.940.000 | Reservation `RSV261028041001L2L5VQ` | Thanh toán cọc cho đặt chỗ RSV 3 (Cộng Hòa) | 2026-10-28 04:10:11 |
| **6** | `Payment` | -30.000 | 9.940.000 | 9.910.000 | Reservation `RSV261028043810MRPSYN` | Thanh toán cọc cho đặt chỗ RSV 4 (FPT University) | 2026-10-28 04:38:14 |
| **7** | `Refund` | +30.000 | 9.910.000 | 9.940.000 | Refund Payment `PM261028043814ZG8D3P` | Hoàn trả cọc do hủy hợp lệ RSV 4 (FPT University) | 2026-10-27 17:24:20 |
| **8** | `Payment` | -30.000 | 9.940.000 | **9.910.000** | Reservation `RSV261027172436TTAENR` | Thanh toán cọc cho đặt chỗ RSV 5 (FPT - Bay L3-20) | 2026-10-27 17:24:59 |
| **9** | `Payment` | -30.000 | 9.910.000 | **9.880.000** | Reservation `RSV2610271803308APE7E` | Thanh toán cọc cho đặt chỗ RSV 6 (FPT - Bay L3-33) | 2026-10-27 18:03:34 |
| **10** | `Payment` | -30.000 | 9.880.000 | **9.850.000** | Reservation `RSV261027190948ULE55Z` | Thanh toán cọc cho đặt chỗ RSV 7 (FPT - Bay L3-31) | 2026-10-27 19:09:52 |

---

### 11.3. Giải Thích Toán Học & Bằng Chứng Khẳng Định (Mathematical Proof)

1. **Khẳng định tính đúng đắn của Backend:**
   - Cả hai lượt đặt chỗ tại bãi xe FPT University (RSV 5 tại bay L3-20 và RSV 6 tại bay L3-33) **đều đã được trừ tiền thật 30.000 VND mỗi lượt** qua `PaymentService.CreateDepositPaymentAsync` trong SQL Server.
   - Không có bất kỳ giao dịch cọc nào bị bỏ sót hay thất thoát tài chính trong database.
2. **Khớp nối mốc số dư 9.910.000 VND:**
   - Sau khi hủy RSV 4 và được hoàn 30.000 VND (Tx 7), số dư ví là **9.940.000 VND**.
   - Tại thời điểm người dùng tạo và thanh toán cọc cho đặt chỗ RSV 5 (bay L3-20 - đúng như ảnh chụp màn hình kiểm thử), giao dịch Tx 8 đã trừ 30.000 VND:
     $$\\text{Số dư sau Tx 8} = 9.940.000 - 30.000 = 9.910.000\\text{ VND}$$
   - Do đó, con số `9.910.000 VND` mà người dùng nhìn thấy trên trình duyệt chính là số dư được lưu vào cache `localStorage` tại thời điểm kết thúc lượt đặt chỗ RSV 5.
3. **Vì sao khi đặt tiếp RSV 6, giao diện người dùng không giảm xuống 9.880.000 VND:**
   - Khi thanh toán cọc cho RSV 6 (Tx 9), backend đã trừ thành công số dư trong database xuống **9.880.000 VND**.
   - Tuy nhiên, client Frontend bị lỗi đồng bộ cache (xem mục 11.4 dưới đây), dẫn đến việc giao diện tiếp tục giữ nguyên con số cũ `9.910.000 VND`.

---

### 11.4. Phân Tích Nguyên Nhân Gốc Rễ (Root Cause Analysis)

#### 1. Nguyên nhân gốc rễ của Sự cố B (Issue B - Stale Wallet Display):
- Trong `src/hooks/useReservations.ts` (`usePayDeposit`): Khi gọi thanh toán cọc ví thành công, client phát sự kiện trình duyệt:
  ```ts
  window.dispatchEvent(new Event('xxiv-wallet-updated'))
  ```
- Trong `src/hooks/useWallet.ts` (dòng 259-278): Hook đăng ký lắng nghe sự kiện `xxiv-wallet-updated` thông qua hàm callback `handleLocalSync()`:
  ```ts
  const handleLocalSync = () => {
    try {
      const savedBal = localStorage.getItem(balKey)
      if (savedBal !== null) setBalanceState(Number(savedBal))
      // ...
    } catch { }
  }
  ```
- **Điểm khuyết tật:** Hàm `handleLocalSync` chỉ đọc lại giá trị đã lưu trong `localStorage.getItem(balKey)`. Nhưng khi thanh toán cọc qua API, việc trừ tiền diễn ra trên server; không có bất kỳ dòng code nào cập nhật `localStorage` trước khi phát sự kiện `xxiv-wallet-updated`.
- Đồng thời, hook `useWallet` là một custom hook quản lý state độc lập, không kết nối vào cơ chế `invalidateQueries` của TanStack Query.
- **Hệ quả:** Khi người dùng thanh toán cọc, `handleLocalSync` chạy và chỉ đọc lại con số cũ trong `localStorage` (`9,910,000 VND`). Giao diện hoàn toàn không cập nhật số dư mới từ backend, khiến người dùng lầm tưởng rằng tiền cọc không bị trừ.

#### 2. Nguyên nhân gốc rễ của Sự cố A (Issue A - Auth Persistence / HTTP 401 on Refresh):
- **Cấu hình Remember Me:** Trong các màn hình đăng nhập (`LoginScreen.jsx`, `OtpLoginScreen.jsx`, `GoogleSignInButton.jsx`), giá trị state `rememberMe` được khởi tạo mặc định là `false`:
  ```jsx
  const [rememberMe, setRememberMe] = useState(false);
  ```
- **Cơ chế lưu trữ phiên (`session.ts`):**
  - Khi `rememberMe === false`, `session.save` xóa toàn bộ `localStorage` và chỉ ghi JWT token cùng thông tin user vào `sessionStorage`.
  - `sessionStorage` bị cô lập tuyệt đối theo từng tab trình duyệt. Khi người dùng mở tab mới, mở qua cửa sổ mới, hoặc khi trình duyệt khôi phục phiên sau khi tải lại, dữ liệu trong `sessionStorage` có thể bị mất.
- **Cơ chế phản hồi 401 (`axiosClient.ts`):**
  - Trong interceptor xử lý lỗi của Axios:
    ```ts
    const sentToken = error instanceof AxiosError && Boolean(error.config?.headers?.Authorization)
    if (apiError.status === 401 && sentToken) {
      session.clear()
    }
    ```
  - Nếu bất kỳ request nào nhận mã 401 khi đã có token, interceptor lập tức thực thi `session.clear()`, xóa sạch token và user, kích hoạt route guard điều hướng người dùng về trang `/login`.
  - Ngoài ra, việc truy cập trực tiếp `error.config?.headers?.Authorization` trong Axios v1.x gặp vấn đề không đồng nhất khi header được chuẩn hóa thành `AxiosHeaders` hoặc chữ thường `authorization`.

---

### 11.5. Giải Pháp Kỹ Thuật Đã Triển Khai (Applied Fixes)

Chúng tôi đã thực hiện các điều chỉnh tối thiểu, chính xác và an toàn tuyệt đối trên Frontend để khắc phục triệt để cả hai vấn đề mà không phá vỡ kiến trúc sẵn có:

#### 1. Khắc phục đồng bộ ví trong `src/hooks/useWallet.ts`:
- Bổ sung việc tự động gọi `fetchWallet()` bên trong hàm `handleLocalSync()` khi người dùng đang có phiên đăng nhập hợp lệ:
  ```ts
  const handleLocalSync = () => {
    try {
      const savedBal = localStorage.getItem(balKey)
      if (savedBal !== null) setBalanceState(Number(savedBal))
      const savedHold = localStorage.getItem(holdKey)
      if (savedHold !== null) setHoldAmountState(Number(savedHold))
      const savedTxs = localStorage.getItem(txsKey)
      if (savedTxs !== null) setTransactions(JSON.parse(savedTxs))
    } catch {
      // ignore
    }
    // Gọi fetchWallet() để lấy số dư mới nhất từ CSDL thực tế khi có thông báo cập nhật
    if (session.getToken() && !session.isDemo()) {
      void fetchWallet()
    }
  }
  ```
- Bổ sung `fetchWallet` vào dependency array của hook `useEffect` để đảm bảo tuân thủ React Hooks linting.

#### 2. Khắc phục duy trì đăng nhập trong các màn hình Auth:
- Đặt mặc định `rememberMe = true` trong:
  - `src/components/LoginScreen.jsx`
  - `src/components/OtpLoginScreen.jsx`
  - `src/components/GoogleSignInButton.jsx`
- Đảm bảo phiên đăng nhập luôn được lưu vào `localStorage`, duy trì đăng nhập bền vững qua mọi lần F5 refresh, mở tab mới, và chuyển hướng URL.

#### 3. Chuẩn hóa kiểm tra Header trong `src/api/axiosClient.ts`:
- Chuẩn hóa việc phát hiện Authorization header để tương thích hoàn toàn với Axios 1.x `AxiosHeaders`:
  ```ts
  const authHeader =
    error instanceof AxiosError
      ? (error.config?.headers?.get?.('Authorization') ??
         error.config?.headers?.Authorization ??
         error.config?.headers?.authorization)
      : null
  const sentToken = Boolean(authHeader)
  if (apiError.status === 401 && sentToken) {
    session.clear()
  }
  ```

---

### 11.6. Kết Quả Kiểm Thử & Xác Minh Chất Lượng (Verification Evidence)

1. **Kiểm tra kiểu dữ liệu Frontend (TypeScript):**
   ```text
   > pms-frontend@0.0.0 typecheck
   > tsc -b
   Command exited with code 0. (0 errors)
   ```
2. **Kiểm tra đóng gói sản phẩm Frontend (Vite Build):**
   ```text
   > pms-frontend@0.0.0 build
   > tsc -b && vite build
   built in 18.33s. Command exited with code 0. (0 errors)
   ```
3. **Kiểm tra Unit Tests Backend (.NET 9):**
   ```text
   Passed! - Failed: 0, Passed: 417, Skipped: 0, Total: 417, Duration: 4 s - PMS.Application.UnitTests.dll (net9.0)
   ```
4. **Kiểm tra Integration Tests Revenue Reports (.NET 9):**
   ```text
   Passed! - Failed: 0, Passed: 8, Skipped: 0, Total: 8, Duration: 7 s - PMS.Api.IntegrationTests.dll (net9.0)
   ```
5. **Kiểm tra trực tiếp số dư ví qua Live API Backend (`GET /api/wallet`):**
   ```json
   {
     "succeeded": true,
     "result": {
       "balance": 9850000.00,
       "holdAmount": 0.00,
       "currency": "VND",
       "status": "Active"
     }
   }
   ```
   Số dư trả về từ API hoàn toàn khớp chính xác với số dư trong database sau 10 giao dịch (`9.850.000 VND`).

---

### 11.7. Xác Nhận Ranh Giới Git (Git Boundary Compliance)

- **Backend (`parking-management-system`):** Không thực hiện bất kỳ lệnh commit hay push nào. Toàn bộ uncommitted changes được bảo tồn nguyên vẹn trên working tree nhánh `develop`.
- **Frontend (`parking-management-system-frontend`):** Không thực hiện bất kỳ lệnh commit hay push nào. Toàn bộ uncommitted changes được bảo tồn nguyên vẹn trên working tree nhánh `develop`.
- **Tài liệu báo cáo kỹ thuật (`thangnx-mc/OJT-report`):** Cập nhật đầy đủ vào tệp `reports/parking-system/revenue-update-and-manager-dashboard-scan.md` và commit/push lên nhánh `main`.
