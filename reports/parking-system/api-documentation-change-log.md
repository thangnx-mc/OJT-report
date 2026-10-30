# XXIV Parking Management System — API Documentation Change Log & Audit Trail

> **Target Document:** `C:\DEV\OJT-report\reports\parking-system\API_DOCUMENTATION.md`  
> **Source Code Baseline:** `c:\DEV\OJT-FSA-Team3\parking-management-system` (`PMS.API`, `PMS.Application`, `PMS.Core`, `PMS.DataAccess`)  
> **Audit Date:** 2026-10-30  
> **Author:** Senior Technical Writer + Backend Lead  
> **Status:** Verified Against Source Code Baseline  

---

## 1. Executive Summary

This document details all technical discrepancies, architectural corrections, and schema reconciliations between the legacy documentation (`API_DOCUMENTATION.old.md`) and the verified ASP.NET Core source code baseline. Every change is supported by explicit source code line citations.

---

## 2. Item-by-Item Reconciliation Matrix (Items A through O)

### Item A: Authentication Routes, 2FA Mechanism & Management Scope
- **Old Document Status:**
  - Sections 2.3 and 3.2 documented `/api/Users/login` and `/api/Users/2fa/login`.
  - Described 2FA as TOTP (authenticator app / Google Authenticator) while simultaneously documenting a contradictory "resend OTP" endpoint.
  - Missed explanation of how 2FA is enabled or disabled.
- **Verified Source Truth:**
  - Primary authentication endpoint is `POST /api/Users/authenticate` (`src/PMS.API/Controllers/UsersController.cs:69-75`).
  - Two-factor challenge verification endpoint is `POST /api/Users/two-factor` (`src/PMS.API/Controllers/UsersController.cs:77-83`).
  - Resend endpoint is `POST /api/Users/two-factor/resend` (`src/PMS.API/Controllers/UsersController.cs:85-91`).
  - **2FA is Email OTP, NOT TOTP:** Generates a 6-digit numeric OTP (`TwoFactorTokenHelper.GenerateNumericOtp(6)`) dispatched via SMTP email (`src/PMS.Application/Services/Impl/UserService.cs:248-265` and `src/PMS.Application/Helpers/TwoFactorTokenHelper.cs:16-29`).
  - **Challenge Token:** Issues a 5-minute JWT with claim `purpose: "2FA_Challenge"` (`TwoFactorTokenHelper.cs:53-75`). If presented as an API access token, `ApiDependencyInjection.cs:144-147` intercepts and rejects it.
  - **Security Limits:** Max 5 failed verification attempts before invalidation (`UserService.cs:342-348`); 60-second cooldown between resend requests (`UserService.cs:392-398`).
  - **2FA Management:** No API endpoint exists to enable or disable 2FA. The flag `TwoFactorEnabled` resides on `AspNetUsers` and is managed directly in database configuration or demo seeders (`src/PMS.Core/Entities/User.cs:32`).
- **Correction Applied:** Updated Sections 3.1, 3.2, and Module 01 in `API_DOCUMENTATION.md`.

---

### Item B: Staff Gate Operation Routes and Check-In Contracts
- **Old Document Status:**
  - Section 8.2 documented non-existent routes `POST /api/staff/sessions/check-in` and `POST /confirm-entry`.
  - Claimed check-in returns `SessionResponseModel`.
- **Verified Source Truth:**
  - Canonical gate check-in route is `POST /api/staff/lots/{lotId}/check-in` (`src/PMS.API/Controllers/StaffGateController.cs:57-59`).
  - Check-in return contract is `ApiResult<CheckInResultModel>` (`src/PMS.Application/Models/Operations/GateModels.cs:78-88`), containing `created: bool`, `barrierOpen: bool`, `session: SessionResponseModel` (in `PendingEntry` state), and `alerts: GateAlertModel[]`.
  - Ground loop sensor confirmation route is `POST /api/staff/sessions/{id}/entry-confirm` (`src/PMS.API/Controllers/StaffGateController.cs:66-68`), which transitions the session from `PendingEntry` to `Active`.
  - Turnaround cancellation route is `POST /api/staff/sessions/{id}/entry-cancel` (`src/PMS.API/Controllers/StaffGateController.cs:71-73`).
- **Correction Applied:** Standardized Section 4.2 (Gate Flows), Section 7.5 (Shared Schemas), and Section 8.5 (Module 05 Reference).

---

### Item C: RBAC Matrix Customer Wallet Permissions
- **Old Document Status:**
  - Marked "Customer XXIV Wallet (TopUp/Withdraw)" with 'X' (denied) for every role including Customer (`User`).
- **Verified Source Truth:**
  - `WalletController` is decorated with `[Authorize]` at class level (`src/PMS.API/Controllers/WalletController.cs:12`), granting access to all authenticated users.
  - Endpoints `GET /api/wallet`, `POST /api/wallet/topup`, and `POST /api/wallet/withdraw` operate directly on `claimService.GetUserId()`, providing customer self-service wallet operations.
- **Correction Applied:** Rebuilt the complete RBAC Matrix in Section 3.4 and Section 8.8.

---

### Item D: Missing and Ambiguous Request Bodies
- **Old Document Status:**
  - `POST /api/Users/register` and `POST /api/Users/authenticate` lacked request body schema tables.
  - Multiple POST/PUT/PATCH endpoints omitted body specifications.
- **Verified Source Truth:**
  - `POST /api/Users/register` binds `CreateUserModel` from body (`src/PMS.API/Controllers/UsersController.cs:64-67`).
  - `POST /api/Users/authenticate` binds `LoginUserModel` from body (`src/PMS.API/Controllers/UsersController.cs:72-75`).
  - `POST /api/payments/momo/ipn` binds JSON payload `Dictionary<string, JsonElement>` (`src/PMS.API/Controllers/PaymentsController.cs:112-130`).
  - The following 11 POST endpoints require **NO request body** (empty body, `Content-Length: 0`):
    - `POST ~/api/reservations/{reservationId}/check-in-request` (`src/PMS.API/Controllers/GateRequestsController.cs:22-24`)
    - `POST ~/api/parking-sessions/{sessionId}/check-out-request` (`src/PMS.API/Controllers/GateRequestsController.cs:27-29`)
    - `POST ~/api/gate-requests/{requestId}/cancel` (`src/PMS.API/Controllers/GateRequestsController.cs:36-38`)
    - `POST /api/staff/sessions/{id}/entry-confirm` (`src/PMS.API/Controllers/StaffGateController.cs:66-68`)
    - `POST /api/staff/sessions/{id}/entry-cancel` (`src/PMS.API/Controllers/StaffGateController.cs:71-73`)
    - `POST /api/staff/sessions/{id}/cancel-payment` (`src/PMS.API/Controllers/StaffGateController.cs:81-83`)
    - `POST /api/users/profile/vehicles/{vehicleId}/documents/submit` (`src/PMS.API/Controllers/VehicleDocumentsController.cs:59-64`)
    - `POST /api/payments/{id}/check` (`src/PMS.API/Controllers/PaymentsController.cs:36-41`)
    - `POST /api/payments/{id}/simulate-success` (`src/PMS.API/Controllers/PaymentsController.cs:55-60`)
    - `POST ~/api/notifications/{id}/read` (`src/PMS.API/Controllers/NotificationsController.cs:29-31`)
    - `POST ~/api/notifications/read-all` (`src/PMS.API/Controllers/NotificationsController.cs:34-36`)
- **Correction Applied:** Documented full request body schemas or explicitly noted "None (empty body, `Content-Length: 0`)" across all endpoints.

---

### Item E: Contract Schema for `GET /api/gate-requests/mine`
- **Old Document Status:**
  - Descriptive text stated "list of requests", but the documented response contract was a single `GateRequestModel`.
- **Verified Source Truth:**
  - The controller action returns `ApiResult<IReadOnlyList<GateRequestModel>>` (`src/PMS.API/Controllers/GateRequestsController.cs:31-33`), serializing a JSON array of requests from the last 12 hours.
- **Correction Applied:** Corrected return type and response example to `ApiResult<IReadOnlyList<GateRequestModel>>` in Section 8.5.3.

---

### Item F: Schemas Diverging from Textual Descriptions
- **Old Document Status:**
  - Profile update claimed to support `address`.
  - Slot configuration claimed to support `sensor bindings`.
  - Lot creation claimed to support `geographical coordinates` (latitude/longitude) and `capacity`.
  - Vehicle types claimed to support `capacity multipliers` and `base price`.
  - Zone update claimed to support `reservation share percentage`.
  - Avatar upload claimed to accept "JPEG, PNG, or PDF".
- **Verified Source Truth:**
  - `UpdateUserProfileModel` contains `fullName`, `email`, `phoneNumber`, `avatarUrl`, `citizenId` — NO `address` (`src/PMS.Application/Models/Profile/UpdateUserProfileModel.cs:3-18`).
  - `CreateSlotModel` and `UpdateSlotModel` contain `zoneId`, `vehicleTypeId`, `code`, `status`, `dedicatedVehicleId`, `dedicatedNote`, `isActive` — NO sensor bindings (`src/PMS.Application/Models/Slot/CreateSlotModel.cs:5-18`).
  - `CreateParkingLotModel` contains address and template key — NO lat/long coordinates or capacity; capacity is derived from 3D templates (`src/PMS.Application/Models/ParkingLot/CreateParkingLotModel.cs:5-35`).
  - `VehicleTypeResponseModel` contains `id`, `code`, `name`, `description`, `sortOrder` — NO capacity multipliers or pricing (`src/PMS.Application/Models/Profile/VehicleTypeResponseModel.cs:3-13`).
  - `UpdateZoneModel` has integer property `maxReservationSlots: int?` — NOT a percentage (`src/PMS.Application/Models/Zone/UpdateZoneModel.cs:13`).
  - Avatar upload accepts `image/jpeg`, `image/png`, `image/gif`, `image/webp` up to 2 MB (`src/PMS.Application/Services/Impl/UserProfileService.cs:17-18, 106-108`). PDF is strictly rejected.
- **Correction Applied:** Removed all phantom properties and corrected descriptions across Sections 7 and 8.

---

### Item G: Fictional "Active" Status in Examples & Missing Enums
- **Old Document Status:**
  - Fictitiously used `"Active"` status in examples for reservations, payments, and gate requests.
  - Omitted crucial domain enums.
- **Verified Source Truth:**
  - `ReservationStatus` has NO `"Active"` value (`src/PMS.Application/Models/Operations/ParkingStatuses.cs:15-39`). Valid: `PendingDeposit`, `Confirmed`, `CheckedIn`, `Completed`, `Cancelled`, `LotCancelled`, `Expired`, `NoShow`.
  - `PaymentStatus` has NO `"Active"` value (`ParkingStatuses.cs:95-104`). Valid: `Pending`, `Succeeded`, `Failed`, `Expired`, `RefundPending`, `Refunded`.
  - `GateRequestStatus` has NO `"Active"` value (`GateRequestModels.cs:14-24`). Valid: `Waiting`, `Completed`, `Refused`, `Cancelled`.
  - Fully catalogued all 22 domain enum and status constants.
- **Correction Applied:** Documented full domain enum tables in Section 5.1 and updated all JSON examples.

---

### Item H: Pagination Contracts and Expanded Query Models
- **Old Document Status:**
  - Section 4.4 described generic `pageNumber`, `totalPages`, `hasNextPage`.
  - Omitted full query filter definitions for `ParkingHistoryQuery`.
- **Verified Source Truth:**
  - The API uses three source-verified pagination shapes:
    1. `ParkingHistoryPageModel` (`src/PMS.Application/Models/Operations/ParkingHistoryModels.cs:26-34`): returns `items`, `page`, `pageSize`, `totalCount`, and cumulative `totalSpent`. Full query filter: `page`, `pageSize`, `from`, `to`, `vehicleId`.
    2. `StaffSessionHistoryPageModel` and `StaffPaymentHistoryPageModel` (`src/PMS.Application/Models/Operations/GateModels.cs:332-366`): returns `items`, `page`, `pageSize`, `totalCount`.
    3. Notifications stream (`src/PMS.Application/Models/Notifications/NotificationModels.cs:50-55`): uses `unreadOnly: bool` and `take: int` (default 30, max 100); returns `items` and `unreadCount: int`.
- **Correction Applied:** Replaced Section 2.6 with the three actual pagination patterns.

---

### Item I: `SessionResponseModel` vs. `CurrentSessionModel` Structure
- **Old Document Status:**
  - Inconsistent shapes for session telemetry (`session.sessionCode` vs `code`, `checkInTime` vs `entryTime`).
- **Verified Source Truth:**
  - `SessionResponseModel` is the canonical flat entity DTO with properties `id`, `code`, `parkingLotId`, `status`, `entryTime`, `plateNumber`, `paidAmount`, `finalFee` (`src/PMS.Application/Models/Operations/GateModels.cs:90-135`).
  - `CurrentSessionModel` is an outer card DTO returned exclusively by `GET /api/parking-sessions/current` (`src/PMS.Application/Models/Operations/CurrentSessionModels.cs:8-49`), wrapping `session: SessionResponseModel` alongside live telemetry properties: `elapsedMinutes`, `feeSoFar`, `depositApplied`, `alreadyPaid`, `amountDue`, `tariff`, `pendingPayment`, and `exitPass`.
- **Correction Applied:** Clearly separated both models in Sections 7.4, 8.4, and 8.5.

---

### Item J: Elimination of Auto-Generated Placeholder Garbage
- **Old Document Status:**
  - Contained over 400 instances of `"Id_value"`, `"Message_value"`, `"Kind_value"`.
  - Used `"Nguyen Van A"` as names for lots, zones, floors, and vehicle types.
  - Used `"RES-20261030-01"` as session codes, invoices, and payments.
  - Cancelled responses still showed status `"Active"`.
- **Verified Source Truth:**
  - Established a unified, semantically coherent sample dataset:
    - Facility: ID `1`, Code `"LOT-DIST1-01"`, Name `"XXIV Central Park Plaza"`
    - Zone: ID `101`, Code `"ZONE-A"`, Name `"Zone A - Ground Floor"`
    - Bay: ID `1001`, Code `"A-012"`
    - Vehicle: Plate `"59A-123.45"`, Type `"CAR_4"`, Model `"Toyota Camry"`
    - Customer: `"c8e4f1a0-5b23-4c91-8d3f-7e9b2c011234"` (`"Nguyen Van An"`)
    - Staff: `"e2b1c3d4-0001-4000-8000-000000000001"` (`"Tran Van Binh"`)
    - Manager: `"m1a2b3c4-0001-4000-8000-000000000001"` (`"Le Thi Cam"`)
    - Codes: Reservation `"RSV-20261030-001"`, Session `"SES-20261030-001"`, Payment `"PM-20261030-001"`
- **Correction Applied:** 100% of JSON samples rewritten with realistic, domain-accurate values.

---

### Item K: Security, Object-Level Auth, JWT & Gateway Details
- **Old Document Status:**
  - Silent on object-level access control (can a user query another user's payments or files?).
  - Unclear who can call `simulate-success`.
  - Omitted JWT 7-day lifetime, lack of refresh tokens, password complexity regex, OTP limits, rate limits, time zones, and gateway signatures.
- **Verified Source Truth:**
  - Object-level authorization enforced by `EnsureCanSeeAsync` (`src/PMS.Application/Services/Impl/PaymentService.cs:405-414`): requires caller to be the paying customer or have lot operational scope; throws `NotFoundException` (HTTP 404) to prevent ID enumeration.
  - `VehicleDocumentsController.OpenFileAsync` strictly scopes file downloads to the registered vehicle owner (`VehicleDocumentService.cs:140-155`).
  - `POST /api/payments/{id}/simulate-success` requires `Payments:AllowSimulatedPayments == true` and verifies caller ownership (`PaymentService.cs:380-400`).
  - JWT is valid for 7 days (`JwtHelper.cs:40`), HMAC-SHA256, stateless.
  - Password policy requires 6–128 characters, 1 uppercase, 1 special character (`UserValidatorConfiguration.cs:11-18`).
  - Rate limiting covers public layout channels (120 req/min) and password reset (5 req/15 min) (`ApiDependencyInjection.cs:20-52`).
  - VNPay signs with HMAC-SHA512 lowercase hex; MoMo signs with HMAC-SHA256 lowercase hex (`VnPayGateway.cs:88` and `MoMoGateway.cs:101`).
  - Return endpoints (`/vnpay/return`, `/momo/return`) return JSON envelopes, NOT HTTP redirects (`PaymentsController.cs:63-70, 101-107`).
- **Correction Applied:** Fully documented in Sections 2, 3, 10, and 11.

---

### Item L: Business Rules & Calculation Concepts
- **Old Document Status:**
  - Mentioned business concepts without explaining formulas: exit window, exit window late fee, callback deadline, deposit hold rules, cancellation refund tiers, pricing rule priority.
- **Verified Source Truth:**
  - Exit Window: 15 minutes (`ExitWindowMinutes = 15`) after prepayment (`ParkingOptions.cs:153`).
  - Exit Window Late Fee: 10,000 VND flat penalty (`ExitWindowLateFee = 10000`, `ParkingOptions.cs:156`).
  - Callback Deadline: 60 seconds (`PendingPaymentCallbackSeconds = 60`, `ParkingOptions.cs:159`).
  - Hold Duration: 15 mins without deposit / 30 mins with paid deposit (`ParkingOptions.cs:95-104`).
  - Deposit Payment Window: 10 minutes (`DepositMinutes = 10`, `ParkingOptions.cs:86`).
  - Cancellation Refund: $\ge 30$ mins before arrival yields 100% deposit refund; $< 30$ mins yields 0% refund (`ReservationService.cs:461-464`). Lot cancellation yields 100% refund (`ReservationService.cs:435-445`).
- **Correction Applied:** Documented with formulas and two complete worked mathematical examples in Section 6.

---

### Item M: Clarification of Duplicate Responsibilities
- **Old Document Status:**
  - Confusing overlap between `/api/Users/internal`, `/api/admin/users`, `/api/admin/managers`, and status endpoints.
- **Verified Source Truth:**
  - `/api/admin/managers` (`AdminController.cs:22-68`): Specialized Admin management of Facility Managers and lot assignments.
  - `/api/admin/users` (`AdminController.cs:70-78`): Queries internal employee accounts (`Staff`, `Manager`, `Admin`).
  - `/api/Users` (`UsersController.cs:30-36`): Global directory of all users including customer accounts.
  - `/api/Users/internal` (`UsersController.cs:46-51`): Legacy general employee creation endpoint.
  - `PATCH /api/Users/{id}/status` and `PATCH /api/admin/users/{id}/status`: Both call `userService.UpdateUserStatusAsync` with identical signature.
- **Correction Applied:** Explained in Section 3.4 and Section 8.10.

---

### Item N: Document Architecture, GitHub Anchors & Maintainer Appendix
- **Old Document Status:**
  - Broken Table of Contents anchors containing ampersands (`&`).
  - Route headings contained C# type constraints (`{id:long}`).
  - Massive schema duplication (repeating `SessionResponseModel` 17 times).
  - Internal C# controller and file paths leaked into the main developer body.
- **Verified Source Truth:**
  - Fixed GitHub ToC anchors (converting ` & ` to `--`).
  - Cleaned route paths in all headings (e.g. `GET /api/lots/{id}`).
  - Centralized Shared Schemas section (Section 7) referenced by name.
  - Controller names and file paths consolidated into Maintainer Appendix (Section 13).
- **Correction Applied:** Implemented throughout `API_DOCUMENTATION.md`.

---

### Item O: SignalR Architecture & Reconnection Policy
- **Old Document Status:**
  - Ambiguous group joining requirements; unclarified whether customers connect to `ParkingOpsHub`; incorrectly described retry schedule as exponential backoff.
- **Verified Source Truth:**
  - Customers connect to `ParkingOpsHub` via `?access_token=<JWT>` but do NOT call `JoinLot`. They receive events via `Clients.User(userId)` (`SignalRParkingRealtimeNotifier.cs:50-52`).
  - Invoking `JoinLot` without assignment throws `HubException("You are not assigned to this parking lot.")` (`ParkingOpsHub.cs:26-30`).
  - Staff/Managers call `JoinLot(lotId)` and join group `ops-lot-{lotId}`.
  - "Customer & Ops Group" refers to dual delivery to both customer connection and operations group.
  - Reconnection retry schedule is a **fixed delay array** `[0, 2000, 5000, 10000, 30000]` ms (`0s, 2s, 5s, 10s, 30s`), NOT exponential backoff.
- **Correction Applied:** Fully documented in Section 9.

---

## 3. Discrepancy Statistics Summary

| Category | Discrepancies Identified | Corrections Applied |
|:---|:---:|:---:|
| **Route Path & Verb Mismatches** | 4 | 4 corrected |
| **Response Contract / Schema Errors** | 6 | 6 corrected |
| **Missing Request Bodies** | 13 | 13 corrected / specified |
| **RBAC / Security Matrix Errors** | 3 | 3 corrected |
| **Domain Enum / Status Violations** | 5 | 5 corrected |
| **Broken Anchor Links & Formatting** | 28 | 28 corrected |
| **Synthetic Placeholder Instances Replaced** | 400+ | All replaced with coherent dataset |
| **Total Major Discrepancy Items** | **15 (A through O)** | **15 Resolved** |
