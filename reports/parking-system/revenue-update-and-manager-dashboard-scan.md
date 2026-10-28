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
