# Parking Management System (PMS) Backend API Scan Notes

**Repository:** `PMS solution workspace (parking-management-system)`  
**Target Solution:** `PMS.sln` (.NET 9 / ASP.NET Core Web API, EF Core 9, PostgreSQL/SQL Server, Hangfire, SignalR)  
**Deliverable Directory:** `reports/parking-system`  
**Last Updated:** 2026-10-30  
**Status:** Complete - All Deliverables Verified and Generated  

---

## 1. Solution & Project Inventory

- **PMS.sln Projects:**
  - `PMS.API`: Controllers, Hubs, Middlewares, Filters, Background Jobs, Program.cs startup.
  - `PMS.Application`: Business services, DTOs (Models), FluentValidation rules, Pricing engine (FeeCalculator), State machines.
  - `PMS.Core`: Domain entities, common exceptions.
  - `PMS.DataAccess`: DbContext (`PmsDbContext`), migrations, entity configurations, identity stores.
  - `PMS.Shared`: Shared utilities and claim extraction services.

---

## 2. API Surface Verification

- **Total Concrete Controllers:** 22 (excluding `ApiController` base)
- **Total Canonical HTTP Actions:** 150
- **Total Route Aliases:** 2 method-level (`/api/public/parking-lots`, `/api/parking-lots/{lotId}/zones`) + 1 class prefix alias (`/api/lots` and `/api/parking-lots`)
- **Total SignalR Hubs:** 3 (`LotHub` at `/api/hubs/lots`, `ParkingOpsHub` at `/api/hubs/parking-ops`, `VehicleVerificationHub` at `/api/hubs/vehicle-verifications`)

### Controller Breakdown

| # | Controller | Route Base | Role / Auth | Actions |
|---|---|---|---|:---:|
| 1 | `AdminController` | `api/admin` | Admin | 8 |
| 2 | `AdminRevenueController` | `api/admin/reports/revenue` | Admin | 1 |
| 3 | `AdminSimulationController` | `api/admin/simulation` | Admin | 2 |
| 4 | `GateRequestsController` | `api/GateRequests` (overridden actions) | Mixed (User, Staff/Manager/Admin) | 7 |
| 5 | `LayoutsController` | `api/public/lots/{lotId:int}/layout` | Public / AllowAnonymous | 4 |
| 6 | `LotsController` | `api/lots`, `api/parking-lots` | Mixed (Public, Admin, Manager) | 9 |
| 7 | `ManagerController` | `api/manager` | Manager | 8 |
| 8 | `ManagerOperationsController` | `api/manager` | Manager, Admin | 8 |
| 9 | `ManagerPricingController` | `api/manager/lots/{lotId:int}/pricing-rules` | Manager, Admin | 7 |
| 10 | `ManagerRevenueController` | `api/manager/reports/revenue` | Manager | 1 |
| 11 | `NotificationsController` | `api/Notifications` (overridden actions) | User | 5 |
| 12 | `ParkingSessionsController` | `api/parking-sessions` | User | 7 |
| 13 | `PaymentsController` | `api/payments` | Mixed (Authorize, AllowAnonymous) | 8 |
| 14 | `ReservationsController` | `api/reservations` | User | 8 |
| 15 | `SlotsController` | `api/slots` | Admin, Manager | 5 |
| 16 | `StaffController` | `api/staff` | Staff | 1 |
| 17 | `StaffGateController` | `api/staff` | Staff, Manager, Admin | 18 |
| 18 | `UsersController` | `api/Users` | Mixed (Public, User, Admin) | 26 |
| 19 | `VehicleDocumentsController` | `api/users/profile/vehicles/{vehicleId:guid}/documents` | Authorize (User owner) | 5 |
| 20 | `VehicleVerificationsController` | `api/vehicle-verifications` | Staff, Manager, Admin | 4 |
| 21 | `WalletController` | `api/wallet` | Authorize (User) | 3 |
| 22 | `ZonesController` | `api/zones` | Admin, Manager | 4 |
| **Total** | | | | **150** |

---

## 3. SignalR Hub Inventory

1. **`LotHub` (`/api/hubs/lots`)**
   - Public (no authentication required)
   - Group subscription: `JoinLot(lotId)` -> group `lot-{lotId}`, `LeaveLot(lotId)`
   - Server-to-client events:
     - `SlotStatusChanged` -> `{ lotId: int, slotId: int, status: string }`
     - `LotStatusChanged` -> `{ lotId: int, status: string }`
     - `LayoutChanged` -> `{ lotId: int }`
     - `ZoneCounterChanged` -> `{ lotId: int, zoneId: int, availableSpaces: int, occupiedSpaces: int, totalCapacity: int }`

2. **`ParkingOpsHub` (`/api/hubs/parking-ops`)**
   - Authenticated: `[Authorize]`, JWT passed via query parameter `?access_token=<JWT>`
   - Client methods:
     - `JoinLot(lotId)` -> checks `lotScope.CanOperateAsync(userId, role, lotId)`; adds to group `ops-lot-{lotId}`; throws `HubException` if unassigned.
     - `LeaveLot(lotId)`
   - Server-to-client events:
     - `ReservationChanged` -> `{ reservationId: long, lotId: int, status: string, slotId: int? }` (sent to `Clients.User(userId)` AND `ops-lot-{lotId}`)
     - `ReservationReminder` -> `{ reservationId: long, kind: string, at: DateTime }` (sent to `Clients.User(userId)`)
     - `PaymentChanged` -> `{ paymentId: long, status: string, purpose: string, reservationId: long?, sessionId: long? }` (sent to `Clients.User(userId)` AND `ops-lot-{lotId}`)
     - `SessionChanged` -> `{ lotId: int, sessionId: long, status: string, plateNumber: string }` (sent to `ops-lot-{lotId}`)
     - `ZoneCounterChanged` -> `{ lotId: int, zoneId: int, availableSpaces: int, occupiedSpaces: int, totalCapacity: int }` (sent to `ops-lot-{lotId}` and `LotHub`)
     - `OpsAlert` -> `{ lotId: int, code: string, message: string, sessionId: long? }` (sent to `ops-lot-{lotId}`)
     - `GateRequestChanged` -> `{ lotId: int, requestId: long, kind: string, status: string }` (sent to `Clients.User(userId)` AND `ops-lot-{lotId}`)
     - `NotificationCreated` -> `{ notification: NotificationModel, unreadCount: int }` (sent to `Clients.User(userId)`)

3. **`VehicleVerificationHub` (`/api/hubs/vehicle-verifications`)**
   - Authenticated: `[Authorize]`, JWT passed via `?access_token=<JWT>`
   - Automatic group joining in `OnConnectedAsync`: Staff/Manager/Admin added to group `vehicle-verification-reviewers`
   - Server-to-client events:
     - `VerificationChanged` -> `{ verificationId: Guid, vehicleId: Guid, status: string }` (sent to `Clients.User(userId)` AND `vehicle-verification-reviewers`)

---

## 4. Key Security & Architectural Findings

- **JWT Configuration:**
  - Lifetime: 7 days (`Expires = DateTime.UtcNow.AddDays(7)`)
  - Algorithm: HMAC-SHA256 (`HmacSha256Signature`)
  - Claims: `nameid` (userId), `unique_name` (username), `email`, `phone` (`ClaimTypes.MobilePhone`), `role`.
  - Challenge Token: 2FA challenge tokens use `purpose: "2FA_Challenge"`, lifetime 5 minutes. Rejected by API `JwtBearerEvents` if presented as an API access token.
  - Refresh Tokens / Revocation: No refresh token endpoints or revocation lists exist; JWT is fully stateless.
- **Two-Factor Authentication (2FA):**
  - Implemented as Email OTP (6-digit numeric code), NOT TOTP / authenticator app.
  - Code validity: 5 minutes.
  - Attempt limit: 5 attempts maximum before code is consumed and locked.
  - Resend cooldown: 60 seconds.
  - Enable/Disable endpoint: None in API.
- **Rate Limiting Policies:**
  1. `public-layout`: 120 requests/minute per client IP (QueueLimit = 0, returns HTTP 429). Covers `LayoutsController` (`/layout`, `/status`, `/2d`, `/lot-templates`) and public lot endpoints in `LotsController` (`/api/public/lots`, `/api/public/lots/{id}/prices`).
  2. `password-reset`: 5 requests/15 minutes per client IP (QueueLimit = 0, returns HTTP 429). Covers `POST /api/Users/forgot-password` and `POST /api/Users/reset-password`.
- **CORS:** `AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader()`
- **Exception Mapping:**
  - `NotFoundException`, `ResourceNotFoundException` -> 404 Not Found
  - `BadRequestException` -> 400 Bad Request
  - `ForbiddenException` -> 403 Forbidden
  - `ConflictException` -> 409 Conflict
  - `UnprocessableRequestException` -> 422 Unprocessable Entity
  - Others -> 500 Internal Server Error
  - Response shape: `ApiResult<string>.Failure(errors)` with `{ succeeded: false, result: null, errors: [...] }`
- **Model Validation:** FluentValidation auto-validation + `ValidateModelAttribute` returning HTTP 400 with `ApiResult<string>.Failure(errors)`.

---

## 5. Domain Rules & Business Logic

- **Pricing Engine (`FeeCalculator`):**
  - Charge models: `Block`, `Hourly`, `PerEntry`, `DayNight`.
  - Block length default: 60 minutes.
  - Rounding rule: Rounded UP to the next multiple of 1,000 VND (`RoundingStep = 1000m`).
  - First Block: Optional `FirstBlockMinutes` and `FirstBlockPrice` (first day only).
  - Daily cap: `MaxDailyFee` applied per calendar day (split at local midnight in the lot's time zone).
  - Note: Team modified BR-09/BR-22: no free period. The first block is owed immediately upon entry.
  - Settlement: `AmountDue = Max(0, (Fee + LostTicketFee + ExitWindowLateFee) - (DepositApplied + AlreadyPaid))`. Residual deposit is refunded (`DepositRefund`), but prior payments are not refunded at exit.
- **Reservation Policy (`ParkingOptions.Reservations`):**
  - Max advance lead time: 30 days (`MaxLeadDays = 30`).
  - Deposit payment deadline: 10 minutes (`DepositMinutes = 10`) after booking.
  - Slot hold time after arrival:
    - Without deposit: 15 minutes (`HoldMinutes = 15`) before `NoShow`.
    - With deposit: 30 minutes (`DepositHoldMinutes = 30`) before `NoShow`.
  - Early arrival: Check-in opens 15 minutes (`EarlyArrivalMinutes = 15`) prior to arrival time.
  - Cancellation & modification deadline: 30 minutes (`CancelRefundMinutes = 30`) prior to arrival.
    - >= 30 mins: 100% refund of deposit (`DepositAmount - RefundAmount`).
    - < 30 mins: 0% refund, modifications locked.
    - Lot cancellation: 100% refund.
  - Account limits: Max 2 active bookings per account (`MaxActivePerAccount = 2`), max 1 per vehicle (`MaxActivePerVehicle = 1`).
  - Permitted vehicle types: Only `CAR_4` and `CAR_7` (`ReservableVehicleTypeCodes`).
- **Gate & Operations (`ParkingOptions.Gate`):**
  - Exit window: 15 minutes (`ExitWindowMinutes = 15`) after prepayment.
  - Exit window late fee: 10,000 VND (`ExitWindowLateFee = 10000`).
  - Dynamic QR callback wait time: 60 seconds (`PendingPaymentCallbackSeconds = 60`).
  - ANPR confidence threshold: 0.85 (`AnprConfidenceThreshold = 0.85`).
  - Entry confirm timeout: 2 minutes (`EntryConfirmTimeoutMinutes = 2`).
  - Overdue vehicle: 24 hours (`OverdueHours = 24`).
  - Abandoned vehicle: 7 days (`AbandonedDays = 7`).
- **Wallet Rules (`ParkingOptions.Wallet`):**
  - Minimum top-up: 10,000 VND (`MinTopUp = 10000`).
  - Daily online top-up limit: 10,000,000 VND (`DailyTopUpLimit = 10000000`) per calendar day (Vietnam time).

---

## 6. Verification of User Prompt Items A - O

- [x] **A. Auth Route & 2FA Mismatch:** Verified. Canonical routes are `/api/Users/authenticate`, `/api/Users/two-factor`, `/api/Users/two-factor/resend`. 2FA is Email OTP (6-digit, 5-min expiry, max 5 attempts), NOT TOTP. No 2FA toggle endpoint exists in API.
- [x] **B. Staff Gate Routes & Types:** Verified. Real routes: `POST /api/staff/lots/{lotId}/check-in` returning `CheckInResultModel`, `POST /api/staff/sessions/{id}/entry-confirm` returning `SessionResponseModel`.
- [x] **C. RBAC Wallet Access:** Verified. `WalletController` is decorated with `[Authorize]`, allowing Customer/User full access to get wallet telemetry, top up, and withdraw.
- [x] **D. Missing / Empty Request Bodies:** Verified. `register` and `authenticate` bind from body (`CreateUserModel`, `LoginUserModel`). All other empty-body POSTs confirmed as taking no body (`Content-Length: 0`).
- [x] **E. Gate Requests Mine Contract:** Verified. `GET ~/api/gate-requests/mine` returns `ApiResult<IReadOnlyList<GateRequestModel>>`, not a single object.
- [x] **F. Schema vs Description Mismatches:** Verified & cleaned up: no `address` in profile update; no `sensor bindings` in slots; no `coordinates/capacity` in lot create/update; no `capacity multipliers/base price` in vehicle types; zone update has `maxReservationSlots: int?`; avatar upload accepts JPEG, PNG, GIF, WebP up to 2 MB (not PDF).
- [x] **G. Invalid 'Active' Status in Examples & Missing Enums:** Verified. Full domain string constants and valid state enums inventoried.
- [x] **H. Pagination Contracts:** Verified. Endpoints return `page`, `pageSize`, `totalCount` (+ `totalSpent` for parking history). Notifications use `unreadOnly: bool` and `take: int`.
- [x] **I. SessionResponseModel vs CurrentSessionModel:** Verified. `SessionResponseModel` is the canonical session entity DTO. `CurrentSessionModel` wraps `SessionResponseModel` inside property `session` alongside active telemetry properties.
- [x] **J. Coherent Dataset:** Verified. Unified dataset established across the documentation.
- [x] **K. Undocumented / Unclear Behaviors:** Verified: object-level authorization (`EnsureCanSeeAsync`, `lotScope`), simulate-success requirements, 7-day JWT, password policy, rate limit coverage, CORS, time zones, gateway algorithms, return endpoint contracts.
- [x] **L. Business Concepts:** Verified: exit window, late fees, callback deadlines, hold seconds, deposit refund tiers, pricing rule lifecycle.
- [x] **M. Duplicate Responsibilities:** Verified: `/api/admin/users` vs `/api/Users`, `/api/Users/internal`, status endpoints.
- [x] **N. Formatting & GitHub Anchor Slugification:** Verified. Double-hyphen slugs for `&`, clean paths without type constraints, centralized schemas.
- [x] **O. SignalR Semantics:** Verified: `Clients.User` vs `OpsGroup` vs `LotHub`, `HubException`, customer connectivity, fixed reconnect retry array `[0, 2000, 5000, 10000, 30000]` ms.

---

## 7. Execution Plan for Deliverables

- [x] Backup `API_DOCUMENTATION.md` to `API_DOCUMENTATION.old.md`
- [x] Create `_scan-notes.md`
- [x] Part 01: Overview & Conventions (`_parts/01-overview-conventions.md`)
- [x] Part 02: Authentication, Authorization & RBAC (`_parts/02-auth-rbac.md`)
- [x] Part 03: End-to-End Flows (Mermaid Sequences) (`_parts/03-e2e-flows.md`)
- [x] Part 04: Domain Model, Enums & State Machines (`_parts/04-domain-state-machines.md`)
- [x] Part 05: Business Rules & Pricing Engine (`_parts/05-business-rules.md`)
- [x] Part 06: Shared Schemas Reference (`_parts/06-shared-schemas.md`)
- [x] Part 07: API Reference - Modules 01 to 04 (`_parts/07-api-ref-modules-1-4.md`)
- [x] Part 08: API Reference - Modules 05 to 08 (`_parts/08-api-ref-modules-5-8.md`)
- [x] Part 09: API Reference - Modules 09 to 12 (`_parts/09-api-ref-modules-9-12.md`)
- [x] Part 10: Real-Time SignalR API (`_parts/10-realtime-signalr.md`)
- [x] Part 11: Error Catalogue, Validation & Rate Limiting (`_parts/11-errors-validation.md`)
- [x] Part 12: Integration Notes (Gate Hardware, Gateways) (`_parts/12-integration-notes.md`)
- [x] Part 13: Glossary & Maintainer Appendix (`_parts/13-glossary-appendix.md`)
- [x] Assemble parts into final `API_DOCUMENTATION.md` and remove `_parts` directory
- [x] Create `api-documentation-change-log.md`
- [x] Create `open-questions.md`
