# Phase 3: Backend API Inventory & Implementation Status Audit Report

## 1. Executive Summary

This report establishes a comprehensive, evidence-based inventory of all backend API endpoints and backend processing tasks required for the **Parking System (parking-management-system)**. It contrasts the specifications in the **Software Requirements Specification (SRS v1.3)** and the **Implementation Plan** against the empirical codebase baseline located at `C:\DEV\OJT-FSA-Team3\parking-management-system`.

### Key Audit Findings

1. **Total Backend Operations Audited**: **75 distinct backend API operations and services** were identified and evaluated across 12 functional domains.
2. **Current Implementation Breakdown**:
   - **Implemented — evidence found**: **10 endpoints** (13.3%) — Concentrated exclusively in Customer Profile management (`GET/PUT /api/users/profile`), Vehicle CRUD (`GET/POST/PUT/DELETE/PATCH /api/users/profile/vehicles`), Vehicle Types catalog, and Password Change.
   - **Partially implemented**: **6 operations** (8.0%) — Including User Registration (`POST /api/users/register`, lacks Phone OTP), Authentication (`POST /api/users/authenticate`, lacks 2FA, phone lookup, and refresh tokens), Public User queries (`GET /api/users`, `GET /api/users/{id}`, security risk due to missing authorization), and utility services (`EmailService`, `TemplateService`, implemented without API triggers).
   - **Missing**: **57 operations** (76.0%) — Covering the entire operational core of the parking system: Parking Lots, Zones, Slots, `ZoneCounter`, 3D layout rendering & SignalR streams, Barrier Check-in/out, ANPR integration, Pricing calculation engine, Cash/Online payments, Reservations, Reports, and System/IoT administration.
   - **Unclear / needs verification**: **2 operations** (2.7%) — Schema migration execution status (`B1.2`) and Redis distributed caching for `ZoneCounter` concurrency, which depend on live runtime database and infrastructure inspection.
3. **Architectural Gap — DbContext Bifurcation**:
   The backend architecture maintains two independent Entity Framework DbContexts:
   - `DatabaseContext` (inheriting from `IdentityDbContext<ApplicationUser>`), used for Identity and wrapped by `TransactionMiddleware`.
   - `PmsDbContext` (holding 63 domain entities), injected into `UserProfileService`.
   `TransactionMiddleware` only begins a transaction on `DatabaseContext`; domain writes to `PmsDbContext` execute outside this transaction boundary, creating potential atomicity vulnerabilities during cross-context operations.
4. **Implementation Plan Inversion Confirmed**:
   The development team has completed Phase 2's Customer API (`B2.7`) and Vehicle Profile features ahead of schedule, while the foundational Phase 1 operational core (Lots, Zones, Slots, Sessions, Barriers, Pricing, POS Cash) remains completely unbuilt.

---

## 2. Audit Methodology & Source Baselines

### 2.1 Authoritative Baseline Documents
- **SRS Document**: `SRS v1.3 – Hệ thống Quản lý Nhiều Bãi Đỗ Xe.docx` (Archived at `reports/parking-system/source-documents/`).
- **Implementation Plan**: `Implementation Plan - Hệ thống Quản lý Nhiều Bãi Đỗ Xe.docx` (Archived at `reports/parking-system/source-documents/`).
- **Phase 2 Requirements Analysis**: `reports/parking-system/phase-2-requirements-analysis.md` (Commit `6309c01`).

### 2.2 Target Backend Codebase
- **Root Path**: `C:\DEV\OJT-FSA-Team3\parking-management-system`
- **Inspected Architecture**: Clean / N-Tier architecture composed of:
  - `PMS.API`: Controllers, Middleware, Filters.
  - `PMS.Application`: Services, Models, DTOs, FluentValidation validators, Mapping profiles.
  - `PMS.Core`: Domain Entities, Common base classes, Exceptions.
  - `PMS.DataAccess`: DbContexts (`DatabaseContext`, `PmsDbContext`), Identity, Migrations, Repositories.
  - `PMS.Shared`: ClaimService.
  - `PMS.Api.IntegrationTests` & `PMS.Application.UnitTests`: Test suites.

### 2.3 Status Classification Criteria
Each audited backend operation is evaluated against exact static code evidence and assigned one primary status:
- **`Implemented — evidence found`**: Relevant controller action, routing attributes, DTO mapping, and service implementation exist in source files. (Does not imply runtime database verification).
- **`Partially implemented`**: Endpoint exists, but lacks required operations, validation rules, authorization scoping, or business rules required by SRS v1.3.
- **`Missing`**: Zero controller actions, services, or handler implementations exist in the inspected source.
- **`Unclear / needs verification`**: Implementation status depends on uninspected external runtime, live database schema, or third-party infrastructure.

---

## 3. Master Backend API Inventory Table

*Note on Identifiers*: IDs formatted as `API-[DOMAIN]-[NUM]` are report-generated stable tracking identifiers. Source requirement codes (`FR-*`) and plan tasks (`B*.*`) are preserved verbatim.

| ID | Source Req / Plan Task | Required API or Backend Operation | Existing Endpoint / Route | Status | Code Evidence / Location | Gap / Recommended Next Action |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **API-AUTH-01** | `FR-AUTH-01`<br>`B2.7` | Customer Registration via Phone/Email + OTP | `POST /api/users/register` | **Partially implemented** | `UsersController.cs:34`<br>`UserService.cs:26` | Implements username/email/password only. `user.EmailConfirmed = true` is hardcoded. Phone OTP flow (`FR-AUTH-01`) is absent. Needs OTP generation/verification service. |
| **API-AUTH-02** | `FR-AUTH-01`<br>`B2.7` | Customer Authentication (JWT Token) | `POST /api/users/authenticate` | **Partially implemented** | `UsersController.cs:41`<br>`UserService.cs:62` | Authenticates strictly by `UserName`. Fails if user provides email/phone. Missing refresh token rotation. Needs dual identifier login (username or email/phone). |
| **API-AUTH-03** | `FR-AUTH-02`<br>`B1.3` | Staff Internal Login + 2FA Verification | *None* | **Missing** | *None* | Zero 2FA validation services or forced TOTP verification endpoints exist. Frontend currently mocks 2FA. Needs staff auth endpoint with OTP challenge. |
| **API-AUTH-04** | `FR-AUTH-03`<br>`B1.3` | RBAC Scoping with Lot Assignment (`UserParkingLot`) | *None* | **Missing** | `JwtHelper.cs:32` | JWT contains global role claim only. Lacks `UserParkingLot` claims. Zero lot-scoping authorization handlers or policies exist. Needs Lot-scoped RBAC filter. |
| **API-AUTH-05** | `FR-AUTH-04`<br>`B1.3` | Admin User Management (List/Create/Lock) | `GET /api/users`<br>`GET /api/users/{id}` | **Partially implemented** | `UsersController.cs:20,27`<br>`UserService.cs:87,101` | Endpoints currently marked `[AllowAnonymous]`, creating severe security exposure. Lack pagination, role filtering, and lock/disable actions. Needs `[Authorize(Roles="Admin")]` and account locking API. |
| **API-AUTH-06** | `FR-AUTH-05`<br>`B1.3` | Administrative & Operational Action Audit Logging | *None* | **Missing** | `PMS.Core/Entities/AuditLog.cs` | `AuditLog` entity exists in scaffolding, but no audit interceptor, EF Core change tracker logger, or audit query endpoints exist. |
| **API-AUTH-07** | `FR-AUTH-06`<br>`B1.5` | Public Guest Access without Authentication | Public APIs | **Implemented — evidence found** | `UsersController.cs:19,33,40` | `[AllowAnonymous]` attribute supported and operational on public endpoints. |
| **API-AUTH-08** | `FR-AUTH-07`<br>`B1.3` | Manager Creates & Manages Staff in Lot | *None* | **Missing** | *None* | No lot-scoped staff onboarding endpoints exist. Needs `POST /api/lots/{lotId}/staff`. |
| **API-AUTH-09** | `FR-AUTH-08`<br>`B1.3` | Account Activation via Link & Forced 2FA Setup | *None* | **Missing** | *None* | Zero tokenized activation link endpoints or initial password/2FA setup endpoints exist. |
| **API-CUS-01** | `FR-CUS-01`<br>`B2.7` | Get Authenticated Customer Profile | `GET /api/users/profile` | **Implemented — evidence found** | `UsersController.cs:48`<br>`UserProfileService.cs:27` | Extracts `userId` from claims, fetches customer details, vehicles, and lockout status. |
| **API-CUS-02** | `FR-CUS-01`<br>`B2.7` | Update Customer Profile (Name, Email, Phone, Avatar) | `PUT /api/users/profile` | **Implemented — evidence found** | `UsersController.cs:56`<br>`UserProfileService.cs:71` | Updates customer profile, syncs email/phone with Identity, validates inputs. |
| **API-CUS-03** | `FR-CUS-01`<br>`B2.7` | List Customer Vehicles | `GET /api/users/profile/vehicles` | **Implemented — evidence found** | `UsersController.cs:64`<br>`UserProfileService.cs:112` | Returns active vehicles ordered by `IsDefault` desc, `CreatedAt` desc. |
| **API-CUS-04** | `FR-CUS-01`<br>`B2.7` | Add Customer Vehicle | `POST /api/users/profile/vehicles` | **Implemented — evidence found** | `UsersController.cs:72`<br>`UserProfileService.cs:135` | Normalizes plate, checks duplicate plate per account, assigns default if first vehicle. |
| **API-CUS-05** | `FR-CUS-01`<br>`B2.7` | Update Vehicle Details | `PUT /api/users/profile/vehicles/{id}` | **Implemented — evidence found** | `UsersController.cs:80`<br>`UserProfileService.cs:189` | Updates plate, brand, model, color, type; manages default status toggle. |
| **API-CUS-06** | `FR-CUS-01`<br>`B2.7` | Soft Delete Vehicle | `DELETE /api/users/profile/vehicles/{id}` | **Implemented — evidence found** | `UsersController.cs:88`<br>`UserProfileService.cs:245` | Sets `IsActive = false`, automatically reassigns default flag to next recent vehicle. |
| **API-CUS-07** | `FR-CUS-01`<br>`B2.7` | Set Default Vehicle | `PATCH /api/users/profile/vehicles/{id}/default` | **Implemented — evidence found** | `UsersController.cs:96`<br>`UserProfileService.cs:284` | Unsets existing default vehicles for user and promotes selected vehicle. |
| **API-CUS-08** | `FR-CUS-01`<br>`B2.7` | List Vehicle Types Catalog | `GET /api/users/profile/vehicle-types` | **Implemented — evidence found** | `UsersController.cs:104`<br>`UserProfileService.cs:310` | Returns active vehicle types ordered by `SortOrder`. Seeded in DB. |
| **API-CUS-09** | `FR-CUS-01`<br>`B2.7` | Change Password | `PUT /api/users/profile/change-password` | **Implemented — evidence found** | `UsersController.cs:111`<br>`UserProfileService.cs:327` | Validates current password and updates via `UserManager`. |
| **API-CUS-10** | `FR-AUTH-04`<br>`B1.3` | Admin Get User Profile by ID | `GET /api/users/{id}/profile` | **Implemented — evidence found** | `UsersController.cs:119`<br>`UserProfileService.cs:27` | Protected by `[Authorize(Roles = "Admin")]`. |
| **API-CUS-11** | `FR-CUS-02`<br>`B2.7` | Customer Parking Session History with In/Out Photos | *None* | **Missing** | *None* | Zero endpoints exist for querying historical sessions by customer. Needs `GET /api/customer/sessions/history`. |
| **API-CUS-12** | `FR-CUS-03`<br>`B2.7` | Track Live Parked Vehicle & Pre-Pay | *None* | **Missing** | *None* | Zero endpoints exist for live active session tracking and fee preview. Needs `GET /api/customer/sessions/active`. |
| **API-LOT-01** | `FR-LOT-01`<br>`B1.4` | Parking Lot CRUD (Admin) | *None* | **Missing** | `PMS.Core/Entities/ParkingLot.cs` | Entity exists in scaffolding. Zero controllers, DTOs, or services exist. Needs `LotsController`. |
| **API-LOT-02** | `FR-LOT-02`<br>`B1.4` | Configure Zone, Floor, Row & Vehicle Capacity | *None* | **Missing** | `PMS.Core/Entities/Zone.cs` | Entities exist (`Zone`, `ZoneVehicleCapacity`). Zero API endpoints exist. Needs `ZonesController`. |
| **API-LOT-03** | `FR-LOT-02`<br>`B1.4` | Configure Operational Slots by Vehicle Type | *None* | **Missing** | `PMS.Core/Entities/Slot.cs` | Entity exists (`Slot`). Zero CRUD endpoints exist. Needs `SlotsController`. |
| **API-LOT-04** | `FR-LOT-03`<br>`B1.4` | Manage 4 Slot States (`Trống`, `Giữ chỗ`, `Đang đỗ`, `Bảo trì`) | *None* | **Missing** | *None* | State machine transitions not implemented in backend. Needs slot state transition service. |
| **API-LOT-05** | `FR-LOT-03`<br>`B1.4` | Real-time `ZoneCounter` Tracking for Transient Space | *None* | **Missing** | *None* | `ZoneCounter` entity/table does not exist in codebase. Concurrency model unbuilt. Needs `ZoneCounter` entity & atomic counter service. |
| **API-LOT-06** | `FR-LOT-04`<br>`B1.4` | Toggle Lot Operating Status & Maintenance Notice | *None* | **Missing** | *None* | Zero endpoints exist for lot suspension/activation. Needs `PATCH /api/lots/{id}/status`. |
| **API-LOT-07** | `FR-LOT-05`<br>`B1.5` | Public Lot Listing with Live Occupancy & Pricing | *None* | **Missing** | *None* | Zero public search/occupancy endpoints exist. Needs `GET /api/public/lots`. |
| **API-LOT-08** | `FR-LOT-06`<br>`B1.4` | Configure Reservation Slot Quotas per Zone | *None* | **Missing** | *None* | Zero quota management endpoints exist. Needs `PUT /api/zones/{id}/reservation-quota`. |
| **API-3D-01** | `FR-3D-01`<br>`B1.5` | Get Published 3D Layout JSON / GLB Asset | *None* | **Missing** | *None* | Zero layout retrieval endpoints exist. Needs `GET /api/lots/{id}/layout-3d`. |
| **API-3D-02** | `FR-3D-02`<br>`B1.5` | Real-time Slot State Updates via SignalR Stream | *None* | **Missing** | *None* | SignalR packages, hubs, and streaming endpoints are completely absent. Needs `SlotStatusHub`. |
| **API-3D-03** | `FR-3D-04`<br>`F2.4` | Query Slot Details via 3D Mesh Click | *None* | **Missing** | *None* | Zero endpoints exist for public slot inspection. Needs `GET /api/lots/{id}/slots/{slotId}`. |
| **API-3D-04** | `FR-3D-05`<br>`B3.1` | Save Draft 3D Layout (Manager/Admin) | *None* | **Missing** | *None* | Zero draft layout persistence endpoints exist. Needs `PUT /api/lots/{id}/layout-3d/draft`. |
| **API-3D-05** | `FR-3D-05`<br>`B3.1` | Publish 3D Layout (Version Lifecycle) | *None* | **Missing** | *None* | Zero layout publish endpoints exist. Needs `POST /api/lots/{id}/layout-3d/publish`. |
| **API-3D-06** | `NFR-3D-02`<br>`B3.2` | GLB Asset Upload & CDN Cache Header Management | *None* | **Missing** | *None* | Zero asset upload or ETag caching handlers exist. Needs GLB upload handler ($\le 10$ MB). |
| **API-3D-07** | `FR-3D-07`<br>`B3.4` | Parked Car Location & Pathfinding Guidance Data | *None* | **Missing** | *None* | Zero pathfinding graph or guidance endpoints exist. Needs `GET /api/reservations/{id}/guidance`. |
| **API-3D-08** | `FR-3D-08`<br>`B1.5` | Public 3D Data Privacy Filter (Plate Masking) | *None* | **Missing** | *None* | Zero projection filters exist to strip PII from public 3D streams. |
| **API-OPS-01** | `FR-OPS-01`<br>`B1.7, B2.2` | Gate Check-in Ingress (ANPR / Ticket / Barrier) | *None* | **Missing** | `PMS.Core/Entities/ParkingSession.cs` | Zero check-in endpoints exist. Needs `POST /api/sessions/check-in` with plate check & photo upload. |
| **API-OPS-02** | `FR-OPS-02`<br>`B1.7, B2.2` | Gate Check-out Egress (Plate Match / Exit Window) | *None* | **Missing** | *None* | Zero check-out endpoints exist. Needs `POST /api/sessions/check-out` with fee & payment check. |
| **API-OPS-03** | `FR-OPS-03`<br>`B1.7` | Manual Check-in / Check-out with Mandatory Reason | *None* | **Missing** | *None* | Zero manual barrier override endpoints exist. Needs `POST /api/sessions/manual-entry` with reason validation. |
| **API-OPS-04** | `FR-OPS-04`<br>`B1.7, B2.2` | Gate Plate Discrepancy Verification | *None* | **Missing** | *None* | Zero plate verification handlers exist. Needs discrepancy alert handler. |
| **API-OPS-05** | `FR-OPS-05`<br>`B1.8` | Lost Ticket Case Handling & Manager Approval | *None* | **Missing** | `PMS.Core/Entities/SessionViolation.cs` | Zero lost ticket penalty or approval endpoints exist. Needs `POST /api/lost-tickets` + Maker-Checker. |
| **API-OPS-06** | `FR-OPS-07`<br>`B2.2` | Plate Blacklist & Whitelist Management | *None* | **Missing** | *None* | Zero plate list management endpoints exist. Needs `GET/POST/DELETE /api/plates/blacklist`. |
| **API-OPS-07** | `FR-OPS-08`<br>`B2.9` | Incident Recording with Photo Attachment | *None* | **Missing** | *None* | Zero incident logging endpoints exist. Needs `POST /api/incidents`. |
| **API-OPS-08** | `NFR-14`<br>`B1.7` | Prevent Duplicate Open Sessions for Single Plate | *None* | **Missing** | *None* | Zero concurrency checks or unique constraints exist for active sessions. |
| **API-PRC-01** | `FR-PRC-01`<br>`B1.6` | Configure Pricing Rules by Lot, Zone & Vehicle | *None* | **Missing** | `PMS.Core/Entities/PricingRule.cs` | Entity exists in scaffolding. Zero CRUD endpoints or calculation services exist. Needs `PricingRulesController`. |
| **API-PRC-02** | `FR-PRC-02`<br>`B1.6` | Fee Calculation Models (Blocks, Hourly, Day/Night, Ceiling) | *None* | **Missing** | `PMS.Core/Entities/PricingTimeBand.cs` | Zero tariff calculation algorithms exist in source. Needs `IPricingCalculationService`. |
| **API-PRC-03** | `FR-PRC-03`<br>`B1.6, B2.5` | Grace Period, Overnight Fee & Exit Overdue Billing | *None* | **Missing** | *None* | Zero grace period or compounding penalty algorithms exist. |
| **API-PRC-04** | `FR-PRC-04`<br>`B1.6` | Historical Pricing Rule Versioning | *None* | **Missing** | `PMS.Core/Entities/PricingRuleHistory.cs` | Entity exists in scaffolding. Zero version archiving logic exists. |
| **API-PRC-05** | `FR-PRC-05`<br>`B3.3` | Dynamic Pricing (Peak/Holiday) & Vouchers | *None* | **Missing** | `PMS.Core/Entities/Voucher.cs` | Entities exist (`Voucher`, `Holiday`, `Promotion`). Zero dynamic price adjusters exist. |
| **API-RSV-01** | `FR-RSV-01`<br>`F2.4` | Search Lots & Compare Reservation Prices | *None* | **Missing** | *None* | Zero reservation availability search endpoints exist. Needs `GET /api/reservations/availability`. |
| **API-RSV-02** | `FR-RSV-02`<br>`B2.3` | Create Slot Reservation with Deposit & Hold Timer | *None* | **Missing** | `PMS.Core/Entities/Reservation.cs` | Entity exists. Zero booking endpoints or hold timer services exist. Needs `POST /api/reservations`. |
| **API-RSV-03** | `FR-RSV-03`<br>`B2.3` | Modify / Cancel Reservation & Process Refund | *None* | **Missing** | *None* | Zero cancellation or refund calculation endpoints exist. Needs `PUT/DELETE /api/reservations/{id}`. |
| **API-RSV-04** | `FR-RSV-04`<br>`B2.3` | Generate Reservation Check-in QR Code | *None* | **Missing** | *None* | Zero QR generation services or secure token endpoints exist. Needs `GET /api/reservations/{id}/qr`. |
| **API-RSV-05** | `FR-RSV-05`<br>`B2.3` | Background Auto-Expire Hold & Charge No-Show Fee | *None* | **Missing** | *None* | Zero `IHostedService` or background workers exist for reservation expiration. Needs `ReservationExpirationWorker`. |
| **API-RSV-06** | `FR-RSV-06`<br>`B2.3` | Configure Reservation Policy (Hold Duration, Deposit) | *None* | **Missing** | *None* | Zero policy configuration endpoints exist. Needs `PUT /api/lots/{id}/reservation-policy`. |
| **API-PAY-01** | `FR-PAY-01`<br>`B1.8` | Collect Cash Payment at POS | *None* | **Missing** | `PMS.Core/Entities/Payment.cs` | Zero POS cash collection endpoints exist. Needs `POST /api/payments/cash`. |
| **API-PAY-02** | `FR-PAY-02`<br>`B2.4` | Create Online Payment Intent (VNPay, MoMo, VietQR) | *None* | **Missing** | *None* | Zero payment gateway client services exist. Needs `POST /api/payments/create-intent`. |
| **API-PAY-03** | `FR-PAY-02`<br>`B2.4` | Payment Gateway Webhooks (Signature & Idempotency) | *None* | **Missing** | `PMS.Core/Entities/WebhookEndpoint.cs` | Zero webhook receivers or signature validators exist. Needs `POST /api/webhooks/{gateway}`. |
| **API-PAY-04** | `FR-PAY-03`<br>`B2.4` | Generate Dynamic VietQR at Exit Gate for Guest | *None* | **Missing** | *None* | Zero VietQR EMVCo payload generators exist. Needs `POST /api/payments/guest-qr`. |
| **API-PAY-05** | `FR-PAY-04`<br>`B1.8` | Payment Refund / Adjustment with Maker-Checker | *None* | **Missing** | *None* | Zero adjustment workflows or manager approval endpoints exist. Needs `POST /api/payments/{id}/adjust`. |
| **API-PAY-06** | `FR-PAY-05`<br>`B2.6` | Daily Payment Reconciliation & Discrepancy Resolution | *None* | **Missing** | *None* | Zero reconciliation batch processors exist. Needs `POST /api/payments/reconcile`. |
| **API-PAY-07** | `FR-PAY-06`<br>`B2.8` | Issue Digital Receipt via Email / SMS | *None* | **Missing** | *None* | Zero receipt generation or dispatch endpoints exist. Needs `POST /api/payments/{id}/receipt`. |
| **API-PAY-08** | `FR-PAY-07`<br>`B2.5` | Exit Window Countdown & Overtime Fee Calculation | *None* | **Missing** | *None* | Zero countdown tracking or barrier synchronization endpoints exist. Needs `GET /api/sessions/{id}/exit-window`. |
| **API-RPT-01** | `FR-RPT-01`<br>`B1.10` | Multi-Lot Real-time Overview Dashboard | *None* | **Missing** | *None* | Zero multi-lot aggregation endpoints exist. Needs `GET /api/reports/dashboard`. |
| **API-RPT-02** | `FR-RPT-02`<br>`B1.10` | Revenue Reports (Filtered by Date, Lot, Vehicle, Method) | *None* | **Missing** | *None* | Zero revenue aggregation queries exist. Needs `GET /api/reports/revenue`. |
| **API-RPT-03** | `FR-RPT-03`<br>`B2.9` | Operational Exception Reports (Overstay, Barrier Overrides) | *None* | **Missing** | *None* | Zero exception reporting queries exist. Needs `GET /api/reports/exceptions`. |
| **API-RPT-04** | `FR-RPT-04`<br>`B2.9` | Export Reports to Excel and PDF | *None* | **Missing** | *None* | Zero report document generators exist. Needs `GET /api/reports/export`. |
| **API-SYS-01** | `FR-SYS-01`<br>`B1.9` | Staff Account Management & Gate Assignment | *None* | **Missing** | `PMS.Core/Entities/Employee.cs` | Entities exist (`Employee`, `EmployeeAssignment`). Zero API endpoints exist. Needs `EmployeesController`. |
| **API-SYS-02** | `FR-SYS-02`<br>`B1.9` | Configure Global System Parameters | *None* | **Missing** | `PMS.Core/Entities/SystemSetting.cs` | Entity exists in scaffolding. Zero settings endpoints exist. Needs `SettingsController`. |
| **API-SYS-03** | `FR-SYS-03`<br>`B1.9, B2.9` | IoT Device Catalog & Heartbeat Health Monitor | *None* | **Missing** | `PMS.Core/Entities/Device.cs` | Entities exist (`Device`, `DeviceEvent`). Zero heartbeat receivers exist. Needs `POST /api/devices/heartbeat`. |
| **API-SYS-04** | `FR-SYS-04`<br>`B2.4` | Payment Gateway API Key & Webhook Secret Configuration | *None* | **Missing** | *None* | Zero encrypted credentials configuration endpoints exist. |
| **API-SYS-05** | `FR-SYS-05`<br>`B2.10` | Automated Backup & Disaster Recovery Worker | *None* | **Missing** | `PMS.Core/Entities/BackupJob.cs` | Entity exists in scaffolding. Zero backup routines exist. |
| **API-SYS-06** | `FR-SYS-06`<br>`B2.8` | System Notification Service (Email, SMS, Push) | *Service only* | **Partially implemented** | `EmailService.cs:7`<br>`DevEmailService.cs:6` | Smtp MailKit service implemented with HTML templates. However, SMS and Push dispatchers are absent, and email service has zero API controllers calling it. |
| **API-EDG-01** | `FR-OPS-09`<br>`B2.1` | Edge Service Snapshot Sync API (Sessions, QR, Rates) | *None* | **Missing** | *None* | Zero snapshot synchronization endpoints exist. Needs `GET /api/edge/snapshot`. |
| **API-EDG-02** | `FR-OPS-09`<br>`B2.1` | Edge Ingestion Queue with Idempotency Key & Timestamp Resolution | *None* | **Missing** | *None* | Zero event ingestion or conflict resolution handlers exist. Needs `POST /api/edge/events`. |

---

## 4. Summary by Implementation Status

### 4.1 Statistical Distribution

| Status Category | Count | Percentage | Primary Functional Domains |
| :--- | :---: | :---: | :--- |
| **Implemented — evidence found** | **10** | **13.3%** | Customer Profile, Vehicle Management, Vehicle Types Catalog, Password Change, Admin Profile View |
| **Partially implemented** | **6** | **8.0%** | Customer Registration, Authentication (JWT), Public User Queries, Notification Service (`EmailService`, `TemplateService`) |
| **Missing** | **57** | **76.0%** | Lots, Zones, Slots, `ZoneCounter`, 3D Digital Twin, Sessions, Barriers, ANPR, Pricing Engine, Payments, Reservations, Reports, System Settings, Edge Sync |
| **Unclear / needs verification** | **2** | **2.7%** | Schema Migration Execution (`B1.2`), Redis Distributed Caching for `ZoneCounter` Concurrency (`NFR-01`) |
| **Total Audited Operations** | **75** | **100.0%** | All 12 Functional Domains |

### 4.2 Existing APIs That Can Be Reused

The following 10 actions in [`UsersController.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs) have completed backend implementations and require zero redesign:

1. `GET /api/users/profile` (`UsersController.cs:48`): Reads authenticated customer profile and linked active vehicles.
2. `PUT /api/users/profile` (`UsersController.cs:56`): Updates customer details and synchronizes with Identity user record.
3. `GET /api/users/profile/vehicles` (`UsersController.cs:64`): Lists customer vehicles with vehicle type details.
4. `POST /api/users/profile/vehicles` (`UsersController.cs:72`): Creates vehicle, normalizes license plate string, enforces account-level plate uniqueness, sets default flag.
5. `PUT /api/users/profile/vehicles/{id}` (`UsersController.cs:80`): Updates vehicle attributes and manages default vehicle reassignments.
6. `DELETE /api/users/profile/vehicles/{id}` (`UsersController.cs:88`): Soft deletes vehicle (`IsActive = false`) and falls back default flag to most recent vehicle.
7. `PATCH /api/users/profile/vehicles/{id}/default` (`UsersController.cs:96`): Atomically sets default vehicle.
8. `GET /api/users/profile/vehicle-types` (`UsersController.cs:104`): Reads active vehicle types catalog for dropdown selection.
9. `PUT /api/users/profile/change-password` (`UsersController.cs:111`): Validates current password and updates hash via ASP.NET Identity.
10. `GET /api/users/{id}/profile` (`UsersController.cs:119`): Admin endpoint to inspect any customer profile.

### 4.3 Partially Implemented APIs Requiring Completion

1. **`POST /api/users/register` (`UsersController.cs:34`)**:
   - *Current Behavior*: Accepts email, username, password, phone; sets `user.EmailConfirmed = true`; creates `Customer` row.
   - *SRS v1.3 Gap*: Mandates phone/OTP registration (`FR-AUTH-01`).
   - *Action*: Integrate OTP generation/verification service before activating user account.
2. **`POST /api/users/authenticate` (`UsersController.cs:41`)**:
   - *Current Behavior*: Searches strictly by `u.UserName == loginUserModel.Username`. Generates 7-day JWT.
   - *SRS v1.3 Gap*: Users cannot log in using phone number or email. Lacks 2FA challenge for internal staff (`FR-AUTH-02`). Lacks refresh token issuance (`RefreshToken.cs` is unreferenced).
   - *Action*: Extend query to match username, email, or phone. Implement TOTP verification flow and refresh token rotation.
3. **`GET /api/users` & `GET /api/users/{id}` (`UsersController.cs:20, 27`)**:
   - *Current Behavior*: Decorated with `[AllowAnonymous]`. Returns user IDs, emails, usernames, and phone numbers without authentication.
   - *Security Gap*: Critical data leakage vulnerability.
   - *Action*: Restrict with `[Authorize(Roles = "Admin")]`, add pagination (`PageSize`, `PageIndex`), and search filters.
4. **`IEmailService` & `ITemplateService` (`PMS.Application/Services/Impl/`)**:
   - *Current Behavior*: MailKit SMTP client and token replacement services exist and have passing unit tests.
   - *Gap*: Not registered to any controller workflow; never invoked during user registration or notification.
   - *Action*: Wire into customer registration and system alert dispatch pipelines.

### 4.4 Missing APIs and Backend Operations (By Domain)

- **Domain 3: Parking Infrastructure (`FR-LOT`)**: 100% missing (Zero endpoints for Lots, Zones, Slots, or `ZoneCounter`).
- **Domain 4: 3D Digital Twin (`FR-3D`)**: 100% missing (Zero endpoints for GLB assets, layout draft/publish, or SignalR slot status streams).
- **Domain 5: Parking Sessions & Barrier Operations (`FR-OPS`)**: 100% missing (Zero endpoints for Check-in, Check-out, manual barrier overrides, or lost ticket approvals).
- **Domain 6: Pricing Engine (`FR-PRC`)**: 100% missing (Zero endpoints for tariff rules, fee calculation, overnight charges, or grace periods).
- **Domain 7: Reservations (`FR-RSV`)**: 100% missing (Zero endpoints for slot booking, hold expiration worker, QR generation, or deposit refund).
- **Domain 8: Payment Processing (`FR-PAY`)**: 100% missing (Zero endpoints for POS cash, VNPay/MoMo/VietQR payment intents, webhooks, or daily reconciliation).
- **Domain 9: Reporting & Analytics (`FR-RPT`)**: 100% missing (Zero endpoints for dashboards, revenue aggregations, or Excel/PDF exports).
- **Domain 10: System Administration & IoT (`FR-SYS`)**: 100% missing (Zero endpoints for staff lot assignments, device heartbeats, or system settings).
- **Domain 11: Edge Synchronization (`FR-OPS-09`)**: 100% missing (Zero endpoints for offline sync snapshots or MQTT ingestion).

---

## 5. Architectural Findings & Codebase Scaffolding

### 5.1 DbContext Bifurcation Risk

The backend codebase instantiates two separate Entity Framework Core DbContexts:
1. `DatabaseContext` (`src/PMS.DataAccess/Persistence/DatabaseContext.cs`):
   - Derives from `IdentityDbContext<ApplicationUser>`.
   - Manages tables: `Users`, `Roles`, `UserRoles`, `UserClaims`, `RoleClaims`, `UserLogins`, `UserTokens`.
   - Contains audit tracking (`CreatedBy`, `CreatedOn`, `UpdatedBy`, `UpdatedOn`) for entities implementing `IAuditedEntity`.
2. `PmsDbContext` (`src/PMS.DataAccess/Persistence/PmsDbContext.cs`):
   - Generated domain context mapping 63 SQL tables inherited from `pmsv2.sql`.
   - Manages tables: `Customers`, `Vehicles`, `ParkingLots`, `Zones`, `Slots`, `ParkingSessions`, `Payments`, `PricingRules`, etc.

**Critical Architectural Issue**:
In `Program.cs:56`, the HTTP pipeline registers `TransactionMiddleware`:
```csharp
public async Task Invoke(HttpContext context, DatabaseContext databaseContext)
{
    await using var transaction = await databaseContext.Database.BeginTransactionAsync();
    try {
        await next(context);
        await transaction.CommitAsync();
    } catch {
        await transaction.RollbackAsync();
    }
}
```
`TransactionMiddleware` only manages transactions for `DatabaseContext`. When `UserProfileService` modifies customer and vehicle records, it saves changes via `_pmsContext.SaveChangesAsync()`. Because `PmsDbContext` is not enrolled in the transaction, changes to domain entities commit independently of Identity operations. If an unhandled exception occurs after `_pmsContext.SaveChangesAsync()`, Identity changes roll back while domain entity changes remain committed in the database.

*Recommended Fix for Implementation Phase*:
Consolidate `IdentityDbContext` and `PmsDbContext` into a unified DbContext, or configure an explicit distributed transaction / coordination strategy across both contexts.

### 5.2 Scaffolded Entities vs. Missing Repositories

In `src/PMS.DataAccess/DataAccessDependencyInjection.cs:26-29`:
```csharp
private static void AddRepositories(this IServiceCollection services)
{
    // Register repositories here
}
```
No repository classes are implemented or registered. `UserProfileService` directly consumes `PmsDbContext` and `UserManager<ApplicationUser>`. While direct DbContext consumption is acceptable in thin-slice CQRS, the lack of an operational repository or query abstraction requires writing new controllers directly against `PmsDbContext` or establishing a standard service pattern.

---

## 6. Full Traceability Matrix: SRS → Plan → Backend Source

| SRS Requirement | Implementation Plan Task | Audited Codebase Location | Primary Status | Verification Evidence & Gaps |
| :--- | :--- | :--- | :--- | :--- |
| **FR-AUTH-01** | `B2.7` | `UsersController.cs:34`<br>`UserService.cs:26` | **Partially implemented** | Registration exists; Phone OTP verification absent. |
| **FR-AUTH-02** | `B1.3` | *None* | **Missing** | Internal staff login with forced 2FA absent. |
| **FR-AUTH-03** | `B1.3` | `JwtHelper.cs:32` | **Missing** | Lot-scoped RBAC claims absent from JWT generation. |
| **FR-AUTH-04** | `B1.3` | `UsersController.cs:119` | **Partially implemented** | Admin get profile exists; account CRUD/locking absent. |
| **FR-AUTH-05** | `B1.3` | `PMS.Core/Entities/AuditLog.cs` | **Missing** | Audit entity exists; automatic logging interceptor absent. |
| **FR-AUTH-06** | `B1.5` | `UsersController.cs:19` | **Implemented — evidence found** | Anonymous access supported via attribute. |
| **FR-AUTH-07** | `B1.3` | *None* | **Missing** | Manager staff creation endpoint absent. |
| **FR-AUTH-08** | `B1.3` | *None* | **Missing** | Link activation and initial setup flow absent. |
| **FR-LOT-01** | `B1.4` | `PMS.Core/Entities/ParkingLot.cs` | **Missing** | Lot entity exists; API controller absent. |
| **FR-LOT-02** | `B1.4` | `PMS.Core/Entities/Zone.cs` | **Missing** | Zone/Capacity entities exist; API controllers absent. |
| **FR-LOT-03** | `B1.4` | *None* | **Missing** | 4-state slot engine and `ZoneCounter` absent. |
| **FR-LOT-04** | `B1.4` | *None* | **Missing** | Lot status toggle endpoint absent. |
| **FR-LOT-05** | `B1.5` | *None* | **Missing** | Public lot occupancy & price endpoint absent. |
| **FR-LOT-06** | `B1.4` | *None* | **Missing** | Reservation slot quota endpoint absent. |
| **FR-3D-01** | `B1.5` | *None* | **Missing** | Published 3D layout GET endpoint absent. |
| **FR-3D-02** | `B1.5` | *None* | **Missing** | SignalR WebSocket slot status hub absent. |
| **FR-3D-03** | `F1.9` | *None* | **Missing** | Slot inspection endpoint absent. |
| **FR-3D-04** | `F2.4` | *None* | **Missing** | 3D slot selection booking initiation endpoint absent. |
| **FR-3D-05** | `B3.1` | *None* | **Missing** | 3D Layout Editor Draft/Publish endpoints absent. |
| **FR-3D-06** | `F1.10` | *None* | **Missing** | Fallback uses same unbuilt layout endpoint. |
| **FR-3D-07** | `B3.4` | *None* | **Missing** | Guidance pathfinding data API absent. |
| **FR-3D-08** | `B1.5` | *None* | **Missing** | Plate masking privacy filter absent. |
| **FR-OPS-01** | `B1.7, B2.2` | `PMS.Core/Entities/ParkingSession.cs` | **Missing** | Barrier check-in & ANPR endpoint absent. |
| **FR-OPS-02** | `B1.7, B2.2` | *None* | **Missing** | Barrier check-out & Exit Window endpoint absent. |
| **FR-OPS-03** | `B1.7` | *None* | **Missing** | Manual check-in/out override endpoint absent. |
| **FR-OPS-04** | `B1.7, B2.2` | *None* | **Missing** | Plate mismatch verification handler absent. |
| **FR-OPS-05** | `B1.8` | `PMS.Core/Entities/SessionViolation.cs` | **Missing** | Lost ticket Maker-Checker endpoint absent. |
| **FR-OPS-06** | *N/A* | *N/A* | **N/A** | Deliberately removed in SRS v1.3 (Shift management). |
| **FR-OPS-07** | `B2.2` | *None* | **Missing** | Plate Blacklist/Whitelist CRUD absent. |
| **FR-OPS-08** | `B2.9` | *None* | **Missing** | Incident recording with photo attachment absent. |
| **FR-OPS-09** | `B2.1` | *None* | **Missing** | Edge Service offline sync snapshot absent. |
| **FR-PRC-01** | `B1.6` | `PMS.Core/Entities/PricingRule.cs` | **Missing** | Pricing rule configuration endpoints absent. |
| **FR-PRC-02** | `B1.6` | `PMS.Core/Entities/PricingTimeBand.cs` | **Missing** | Tariff calculation engine absent. |
| **FR-PRC-03** | `B1.6, B2.5` | *None* | **Missing** | Grace period & overnight calculation absent. |
| **FR-PRC-04** | `B1.6` | `PMS.Core/Entities/PricingRuleHistory.cs` | **Missing** | Pricing rule historical versioning absent. |
| **FR-PRC-05** | `B3.3` | `PMS.Core/Entities/Voucher.cs` | **Missing** | Dynamic peak pricing & voucher calculation absent. |
| **FR-RSV-01** | `F2.4` | *None* | **Missing** | Public lot reservation price search absent. |
| **FR-RSV-02** | `B2.3` | `PMS.Core/Entities/Reservation.cs` | **Missing** | Slot booking with deposit hold endpoint absent. |
| **FR-RSV-03** | `B2.3` | *None* | **Missing** | Reservation modification & cancellation absent. |
| **FR-RSV-04** | `B2.3` | *None* | **Missing** | Check-in QR code generator endpoint absent. |
| **FR-RSV-05** | `B2.3` | *None* | **Missing** | Background reservation auto-expire worker absent. |
| **FR-RSV-06** | `B2.3` | *None* | **Missing** | Reservation policy configuration endpoint absent. |
| **FR-PAY-01** | `B1.8` | `PMS.Core/Entities/Payment.cs` | **Missing** | POS cash collection endpoint absent. |
| **FR-PAY-02** | `B2.4` | *None* | **Missing** | Online payment intents (VNPay, MoMo, VietQR) absent. |
| **FR-PAY-03** | `B2.4` | *None* | **Missing** | Dynamic guest QR payment generator absent. |
| **FR-PAY-04** | `B1.8` | *None* | **Missing** | Payment refund/adjustment with Maker-Checker absent. |
| **FR-PAY-05** | `B2.6` | *None* | **Missing** | Payment gateway daily reconciliation batch absent. |
| **FR-PAY-06** | `B2.8` | *None* | **Missing** | Digital receipt dispatch via email/SMS absent. |
| **FR-PAY-07** | `B2.5` | *None* | **Missing** | Exit Window countdown & overtime calculation absent. |
| **FR-CUS-01** | `B2.7` | `UsersController.cs:48-115`<br>`UserProfileService.cs` | **Implemented — evidence found** | Complete Profile and Vehicle CRUD implemented. |
| **FR-CUS-02** | `B2.7` | *None* | **Missing** | Customer session history endpoint absent. |
| **FR-CUS-03** | `B2.7` | *None* | **Missing** | Live parked vehicle tracking & pre-pay absent. |
| **FR-RPT-01** | `B1.10` | *None* | **Missing** | Multi-lot dashboard aggregation query absent. |
| **FR-RPT-02** | `B1.10` | *None* | **Missing** | Filtered revenue report query absent. |
| **FR-RPT-03** | `B2.9` | *None* | **Missing** | Overstay & barrier exception report queries absent. |
| **FR-RPT-04** | `B2.9` | *None* | **Missing** | Excel and PDF document export services absent. |
| **FR-SYS-01** | `B1.9` | `PMS.Core/Entities/Employee.cs` | **Missing** | Staff management & gate assignment endpoints absent. |
| **FR-SYS-02** | `B1.9` | `PMS.Core/Entities/SystemSetting.cs` | **Missing** | Global system configuration parameters absent. |
| **FR-SYS-03** | `B1.9, B2.9` | `PMS.Core/Entities/Device.cs` | **Missing** | Device catalog & heartbeat monitoring absent. |
| **FR-SYS-04** | `B2.4` | *None* | **Missing** | Payment gateway credentials configuration absent. |
| **FR-SYS-05** | `B2.10` | `PMS.Core/Entities/BackupJob.cs` | **Missing** | Automated backup & recovery job absent. |
| **FR-SYS-06** | `B2.8` | `EmailService.cs:7` | **Partially implemented** | Smtp Email built; Push/SMS and alert triggers absent. |

---

## 7. Recommended Implementation Order and Dependencies

To reconcile the project from its current state to a fully functional MVP, backend implementation should proceed in four sequential, dependency-ordered sprints:

```
[Sprint 1: Operational Foundation]
  - DbContext & Transaction Unification
  - Lots, Zones, Slots CRUD (B1.4)
  - ZoneCounter Atomic Service (B1.4)
  - Staff RBAC & Lot Scoping (B1.3)
       │
       ▼
[Sprint 2: Operational Core & Sessions]
  - Pricing Calculation Engine (B1.6)
  - Parking Session Ingress/Egress (B1.7)
  - Manual POS Check-in/out & Cash Collection (B1.8)
  - Public Lot Availability API (B1.5)
       │
       ▼
[Sprint 3: Automation & Integrations]
  - SignalR Hub for Real-time 3D/Slots (B1.5)
  - ANPR Verification & Blacklist (B2.2)
  - Online Payments (VNPay/VietQR) & Exit Window (B2.4, B2.5)
  - Slot Reservations & Expiration Worker (B2.3)
       │
       ▼
[Sprint 4: Edge, Reports & Governance]
  - EdgeSync Snapshot & Offline Ingestion (B2.1)
  - Daily Payment Reconciliation (B2.6)
  - Multi-lot Reports & PDF/Excel Export (B1.10, B2.9)
  - Hardening OWASP & Concurrency Stress Testing (B2.11)
```

### Prioritization Rationale
1. **Sprint 1 (Prerequisites)**: Resolving the DbContext split and creating Lots/Zones/Slots is the foundational blocker for all parking logic. Without lot and slot foreign keys, sessions and pricing rules cannot be stored.
2. **Sprint 2 (Phase 1 MVP Core)**: Delivers physical gate functionality (manual entry, ticket, pricing evaluation, cash collection). Completes the minimal operational loop for Phase 1 acceptance.
3. **Sprint 3 (Automation & Customer Value)**: Connects online payments, reservations, and real-time SignalR streams to the already built customer profile.
4. **Sprint 4 (Enterprise & Resilience)**: Hardens gate autonomy, offline edge synchronization, and financial reconciliation.

---

## 8. Scope Boundaries, Preservation Directives & Assumptions

1. **Mandatory File Preservation**:
   The 63 scaffolded entity classes in `src/PMS.Core/Entities/` (including out-of-scope entities `Subscription*`, `Einvoice`, `Wallet*`, `Shift*`, `Partner`, `LotReview`) **must remain untouched**. Unused scaffolded classes must not be deleted or renamed, preventing build-break cascades in `PmsDbContext.cs`.
2. **Read-Only Posture Maintained**:
   In strict compliance with project rules, zero application source files in `C:\DEV\OJT-FSA-Team3` were modified, created, or deleted during this phase. No tests or database migrations were executed.
3. **Git Boundary Maintained**:
   Zero Git commands were executed against `C:\DEV\OJT-FSA-Team3`. All version control activities are strictly confined to `C:\DEV\OJT-report`.
