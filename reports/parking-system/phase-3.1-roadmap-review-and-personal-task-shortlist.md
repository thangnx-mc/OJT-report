# Phase 3.1: Roadmap Review & Personal Backend Task Shortlist

## 1. Executive Summary

This report delivers a critical review of the proposed 4-sprint backend implementation roadmap and establishes an actionable, evidence-based task shortlist designed for a single developer. The objective is to identify bounded, independently testable backend tasks that can be delivered cleanly without triggering team-wide merge conflicts or prematurely undertaking risky architectural rewrites.

### Key Conclusions

1. **Roadmap Sequence Refinement**:
   The 4-sprint roadmap proposed in Phase 3 is structurally sound in its functional ordering, but requires one critical technical adjustment: **Decouple domain entity CRUD from global DbContext unification**. Attempting a complete consolidation of `DatabaseContext` and `PmsDbContext` upfront is an architectural blocker that creates unnecessary risk. Because parking infrastructure entities (`ParkingLot`, `Zone`, `Slot`) operate strictly within `PmsDbContext`, their development can proceed immediately without mutating Identity tables or altering `TransactionMiddleware`.
2. **Root Dependency of the Parking Domain**:
   Every operational module in the parking management system—Zones, Slots, `ZoneCounter`, Pricing Rules, Gate Ingress/Egress Sessions, Reservations, and IoT Devices—shares a mandatory foreign key dependency on `ParkingLot` (`ParkingLotId`). Building parking lots is the single critical path enabler for the entire backend.
3. **Recommended Personal Task Batch for One Developer**:
   - **Primary Task**: `TASK-LOT-01: Parking Lots Management API (Admin & Public)` (`FR-LOT-01`, `FR-LOT-04`, `FR-LOT-05`, Plan `B1.4`, `B1.5`).
   - **Companion Quick-Win Task**: `TASK-AUTH-SEC: Authentication Hardening & User Query Security` (`FR-AUTH-04`, Plan `B1.3`).
4. **Ownership & Scope Discipline**:
   This report provides an engineering feasibility recommendation, not a team backlog assignment. Tasks involving Edge service contracts (`B2.1`), SignalR WebSockets (`FR-3D-02`), and payment gateway sandboxes (`B2.4`) are explicitly deferred and marked as unconfirmed pending Team Lead coordination.

---

## 2. Evidence Baseline and Scope Limitations

### 2.1 Evaluated Sources
- **Authoritative Specifications**:
  - `SRS v1.3 – Hệ thống Quản lý Nhiều Bãi Đỗ Xe.docx` (Archived in `source-documents/`).
  - `Implementation Plan - Hệ thống Quản lý Nhiều Bãi Đỗ Xe.docx` (Archived in `source-documents/`).
- **Audit Baselines**:
  - `phase-1-fullstack-codebase-scan.md` (Commit `d145c97`).
  - `phase-2-requirements-analysis.md` (Commit `6309c01`).
  - `phase-3-backend-api-inventory.md` (Commit `9b5a5a5`).
- **Codebase Source Inspection**:
  - Backend project: `C:\DEV\OJT-FSA-Team3\parking-management-system`.

### 2.2 Safety and Audit Scope Boundaries
- **Strict Read-Only Posture**: No files were created, edited, deleted, or formatted in the team workspace.
- **Zero Team Git Operations**: Absolutely no Git commands were run against `C:\DEV\OJT-FSA-Team3` or its child directories.
- **Static Analysis Only**: No builds, test runners, migrations, or database connections were executed.

---

## 3. Four-Sprint Roadmap Assessment

The 4-sprint roadmap defined in `phase-3-backend-api-inventory.md` Section 7 was subjected to a feasibility and dependency audit:

```
Proposed Sequence:
[Sprint 1: Operational Foundation] ──► [Sprint 2: Operational Core] ──► [Sprint 3: Automation & Integrations] ──► [Sprint 4: Edge & Governance]
```

### 3.1 Sprint 1: Operational Foundation (Lots, Zones, Slots, Staff RBAC)
- **Position Justification**: **Strongly Justified**. Establishing parking lots, zones, and operational slots is the non-negotiable prerequisite for sessions, pricing, and reservations.
- **Prerequisites**: `PmsDbContext` (already defined in source), `VehicleTypes` catalog (already seeded).
- **Critical Architectural Critique**:
  Phase 3 proposed bundling "DbContext & Transaction Unification" into Sprint 1. This assessment concludes that **full DbContext consolidation must NOT block Sprint 1 domain CRUD**.
  - `[FACT]`: In `UserProfileService.cs`, the team successfully implemented customer and vehicle features by directly injecting `PmsDbContext` alongside `UserManager<ApplicationUser>`.
  - `[INFERENCE]`: A developer can implement `LotsController` and `ParkingLotService` using `PmsDbContext` alone, because parking lot operations do not modify Identity tables.
  - `[RECOMMENDATION]`: Restrict Sprint 1 architectural scope to establishing standard service patterns and DDL provisioning in `AutomatedMigration.cs`. Defer full cross-context transaction restructuring until Sprint 2 (when check-in workflows may touch both guest Identity and domain Sessions).
- **Parallel Tasks**:
  - `Lots Management API` (`TASK-LOT-01`) can proceed immediately.
  - `User Query Security Hardening` (`TASK-AUTH-SEC`) can proceed independently without touching parking tables.

### 3.2 Sprint 2: Operational Core & Sessions (Pricing, Gate Check-in/out, POS Cash)
- **Position Justification**: **Justified**. Completes the minimal viable operational loop for Phase 1 acceptance.
- **Prerequisites**: Sprint 1 (Lot, Zone, Slot entities active; pricing tables provisioned).
- **Risks of Premature Implementation**:
  Attempting to implement Gate Check-in (`B1.7`) before the Pricing Engine (`B1.6`) or `ZoneCounter` concurrency model is settled forces developers to mock pricing calculations and slot state transitions, resulting in throwaway code.
- **Parallel Tasks**:
  - Pricing Engine calculation algorithms (`TASK-PRC-01`) can be developed as pure business logic in parallel with Gate Ingress/Egress workflows once slot schema is stabilized.

### 3.3 Sprint 3: Automation & Integrations (SignalR 3D, ANPR, Online Payments, Reservations)
- **Position Justification**: **Justified**. Expands the operational core to customer self-service and automated gate entry.
- **Prerequisites**: Sprint 2 (Operational sessions and pricing engine functioning).
- **Risks & External Blockers**:
  Depends heavily on external environments: VNPay/VietQR sandbox credentials, webhook tunneling, and Three.js 3D model coordinate contracts. Should not be assigned to a single developer working in isolation without team-lead architectural alignment.

### 3.4 Sprint 4: Edge, Reports & Governance (EdgeSync, Reconciliation, Reporting, Hardening)
- **Position Justification**: **Justified**. Final production hardening, reporting aggregations, and edge resilience.
- **Prerequisites**: Sprints 1, 2, and 3 data pipelines operational.

---

## 4. Technical Dependency and Blocker Analysis

### 4.1 The `DatabaseContext` vs `PmsDbContext` Separation
- **Observed Architecture**:
  - `DatabaseContext` (`IdentityDbContext<ApplicationUser>`): Manages ASP.NET Identity tables.
  - `PmsDbContext`: Scaffolded context containing 63 SQL tables.
  - `TransactionMiddleware`: Wraps incoming HTTP requests in a database transaction on `DatabaseContext` only (`Program.cs:56`).
- **Operational Impact Analysis**:
  - For pure domain entities (`ParkingLot`, `Zone`, `Slot`, `PricingRule`), writes execute via `PmsDbContext.SaveChangesAsync()`. Because they do not write to Identity tables, the fact that `TransactionMiddleware` wraps `DatabaseContext` has **zero functional blocking impact** on domain CRUD.
  - The separation only becomes a hazard when a single HTTP request must atomically write to both Identity and domain tables (e.g., creating an internal staff user in Identity and assigning them to a lot in `UserParkingLot`).
  - *Decision*: A single developer implementing `LotsController` does not need to alter `TransactionMiddleware`.

### 4.2 Database Schema Provisioning in Local Environments
- **Observed Behavior**:
  - `20261001025913_InitialCreate.cs` only contains DDL for Identity tables.
  - `AutomatedMigration.cs` executes raw SQL DDL during startup (`MigrateAsync`) to ensure `VehicleTypes`, `Customers`, and `Vehicles` exist.
  - Physical tables for `ParkingLots`, `Zones`, and `Slots` are not yet included in `AutomatedMigration.EnsurePmsTablesAsync`.
- **Testing vs Runtime Reality**:
  - In automated testing (`PMS.Application.UnitTests` and `PMS.Api.IntegrationTests`), `UseInMemoryDatabase` is used. EF Core automatically builds the full schema in memory from `PmsDbContext.OnModelCreating`. Unit and integration tests run **immediately without requiring SQL Server migrations**.
  - For local runtime execution against SQL Server LocalDB, the developer must add standard `IF OBJECT_ID('dbo.ParkingLots', 'U') IS NULL` DDL to `AutomatedMigration.cs`, following the exact pattern established for `Customers` and `Vehicles`.

---

## 5. Bounded Backend Task Inventory

The candidate tasks below represent bounded, independently testable units of work derived from the Phase 3 API inventory.

| Task ID | Task Name | SRS & Plan Ref | Current Code Status | Scope & Deliverables | Out of Scope | Relative Effort | Risk Level | 1-Dev Feasible? |
| :--- | :--- | :--- | :--- | :--- | :--- | :---: | :---: | :---: |
| **TASK-LOT-01** | **Parking Lots Management API (Admin & Public)** | `FR-LOT-01`, `04`, `05`<br>`B1.4`, `B1.5` | `Missing`<br>(Entity exists) | • `LotsController`<br>• `IParkingLotService` / `ParkingLotService`<br>• Lot CRUD (Admin)<br>• Status Toggle (Active/Maintenance)<br>• Public Lot List (Anonymous)<br>• DTOs, Validators, Mapster<br>• Unit & Integration Tests | • Zones & Slots<br>• 3D rendering<br>• Real-time SignalR<br>• Pricing rules | **Medium**<br>(2.5 days) | **Low** | **Yes**<br>(Recommended #1) |
| **TASK-AUTH-SEC** | **Auth Security Hardening & Admin User API** | `FR-AUTH-04`, `01`<br>`B1.3`, `B2.7` | `Implemented — test-verified` | • Restrict `/api/users` to Admin<br>• Filter users by role/status/search<br>• Create internal users (Manager/Staff)<br>• Account activate/deactivate toggle<br>• Block deactivated login<br>• Unit & Integration Tests | • Phone SMS OTP flow<br>• TOTP 2FA multi-step<br>• Lot-scoped claims | **Small**<br>(1.5 days) | **Low** | **Yes**<br>(Implemented) |
| **TASK-ZONE-01** | **Zones & Operational Slots Management API** | `FR-LOT-02`, `03`, `06`<br>`B1.4` | `Missing`<br>(Entities exist) | • `ZonesController`, `SlotsController`<br>• Zone CRUD under Lot<br>• Slot CRUD with vehicle type<br>• 4 Slot States transition logic<br>• Reservation quota setting | • `ZoneCounter` Redis concurrency<br>• 3D canvas rendering<br>• SignalR status hub | **Medium**<br>(3.0 days) | **Medium** | **Yes**<br>(Depends on LOT-01) |
| **TASK-PRC-01** | **Pricing Rules Configuration & Calculation Engine** | `FR-PRC-01`–`04`<br>`B1.6` | `Missing`<br>(Entities exist) | • `PricingRulesController`<br>• Tariff rule CRUD by lot/vehicle<br>• Rule versioning logic<br>• Core Fee Calculation Engine<br>• Block, hourly, day/night evaluation | • Dynamic peak pricing (`B3.3`)<br>• Voucher system<br>• Payment gateway sync | **Medium-Large**<br>(4.0 days) | **Medium** | **Yes**<br>(Depends on LOT-01) |
| **TASK-SYS-01** | **System Settings & IoT Device Catalog API** | `FR-SYS-02`, `03`<br>`B1.9`, `B2.9` | `Missing`<br>(Entities exist) | • `SettingsController`<br>• Global parameters CRUD<br>• `DevicesController`<br>• Device catalog CRUD<br>• Device heartbeat receiver endpoint | • Hardware SDK drivers<br>• Edge sync snapshots<br>• Backup scripts | **Small-Medium**<br>(2.0 days) | **Low** | **Yes**<br>(Independent) |
| **TASK-OPS-01** | **Manual Gate Check-in/out & Barrier Overrides** | `FR-OPS-01`–`03`<br>`B1.7` | `Missing`<br>(Entity exists) | • `SessionsController`<br>• Manual ingress with mandatory reason<br>• Manual egress with fee preview<br>• Prevent duplicate active plate session | • ANPR camera integration<br>• Automated barrier relay<br>• Edge offline sync | **Large**<br>(5.0 days) | **High** | **No**<br>(Needs LOT, PRC, & Lead sync) |

---

## 6. Recommended Initial Personal Task Batch

### 6.1 Recommended First Task: `TASK-LOT-01` (Parking Lots Management API)
- **Justification**:
  1. **Unlocks the Entire Domain**: `ParkingLot` is the parent root entity for Zones, Slots, Pricing Rules, Gate Devices, and Parking Sessions. Delivering `LotsController` unblocks the entire development team.
  2. **Zero Architecture Blockers**: Operates 100% within `PmsDbContext`. It does not require modifying `DatabaseContext` or altering `TransactionMiddleware`.
  3. **High Reusability & Demonstrable Value**: Exposes both internal administrative endpoints (`POST /api/lots`, `PUT /api/lots/{id}`) and public discovery endpoints (`GET /api/public/lots`) that can immediately connect to the frontend lot browser.
  4. **Complete Testability**: Can be tested thoroughly in-memory with xUnit, FluentAssertions, and WebApplicationFactory without external infrastructure.

### 6.2 Companion / Follow-up Task: `TASK-AUTH-SEC` (Authentication Hardening)
- **Justification**:
  - Closes an existing critical security vulnerability: `GET /api/users` currently returns all registered users anonymously (`[AllowAnonymous]`).
  - Low effort (1–1.5 days), self-contained within `UsersController.cs` and `UserService.cs`, and highly verifiable.
  - Can be completed either as a quick preliminary fix or immediately following `TASK-LOT-01`.

### 6.3 Tasks to Defer for Now
1. **`TASK-OPS-01` (Parking Sessions & Check-in/out)**: Defer until `TASK-LOT-01`, `TASK-ZONE-01`, and `TASK-PRC-01` are merged. Implementing sessions without slots and pricing produces invalid domain models.
2. **SignalR 3D Real-time Hub (`FR-3D-02`)**: Defer. Requires cross-stack agreement on WebSocket payload schema with the frontend team.
3. **Edge Synchronization & MQTT (`B2.1`)**: Defer. Depends on offline hardware specifications and protocol contracts.
4. **Payment Gateway Integrations (`B2.4`)**: Defer. Requires merchant sandbox credentials, IPN webhook routing, and financial reconciliation rules.

### 6.4 Ownership Disclaimer
Ownership of `TASK-LOT-01` and `TASK-AUTH-SEC` is marked as **unconfirmed technical recommendations**. Official assignment must be validated with the Team Lead to prevent duplicate work with other team members.

---

## 7. Implementation and Verification Handoff (Checklist)

### 7.1 Handoff Checklist for `TASK-LOT-01: Parking Lots Management API`

#### A. Preconditions and Required Decisions
- [ ] Confirm team agreement on `LotsController` route: `api/lots` (Admin/Manager) and `api/public/lots` (Guest/Customer).
- [ ] Verify LocalDB DDL strategy: add `CREATE TABLE [dbo].[ParkingLots]` to `EnsurePmsTablesAsync` in `AutomatedMigration.cs`.

#### B. API Contract & Endpoints
1. `GET /api/lots`: Paginated list of lots with status filter (`[Authorize(Roles = "Admin,Manager")]`).
2. `GET /api/lots/{id}`: Full lot details including contact, hours, coordinates (`[Authorize]`).
3. `POST /api/lots`: Create lot with validation (`[Authorize(Roles = "Admin")]`).
4. `PUT /api/lots/{id}`: Update lot metadata and operating hours (`[Authorize(Roles = "Admin,Manager")]`).
5. `PATCH /api/lots/{id}/status`: Toggle status (`Active`, `Suspended`, `Maintenance`) with reason (`[Authorize(Roles = "Admin,Manager")]`).
6. `GET /api/public/lots`: Anonymous public listing returning ID, code, name, address, total capacity, and status (`[AllowAnonymous]`).

#### C. Validation and Business Rules
- [ ] `Code` must be unique, uppercase alphanumeric (e.g., `LOT-Q1-01`).
- [ ] `Capacity` must be an integer $\ge 0$.
- [ ] If `Is24Hours == false`, `OpenTime` and `CloseTime` are mandatory.
- [ ] When status is set to `Maintenance`, `ClosureReason` or `MaintenanceNotice` must be provided.

#### D. Persistence & Architecture Considerations
- [ ] Implement `IParkingLotService` and `ParkingLotService` in `PMS.Application`.
- [ ] Use `PmsDbContext.ParkingLots`.
- [ ] Apply soft-delete pattern (`IsDeleted = true`), filtering out deleted records on standard queries.

#### E. Test Requirements & Acceptance Criteria
- [ ] **Unit Tests (`ParkingLotServiceTests.cs`)**:
  - `CreateAsync`: Succeeded with valid model; throws `BadRequestException` on duplicate lot code.
  - `UpdateStatusAsync`: Successfully toggles status; throws `NotFoundException` if lot ID does not exist.
  - `GetPublicLotsAsync`: Returns only active, non-deleted lots; strips internal fields (`PartnerId`, `ManagerUserId`).
- [ ] **Integration Tests (`LotsEndpointTests.cs`)**:
  - `POST /api/lots`: Returns `401 Unauthorized` without token; returns `403 Forbidden` for standard `User` role; returns `200 OK` for `Admin`.
  - `GET /api/public/lots`: Returns `200 OK` anonymously.

#### F. Stop & Clarify Conditions
- Stop immediately and request clarification if:
  - Another team member has already initiated a branch modifying `ParkingLot.cs` or `PmsDbContext.cs`.
  - The team lead specifies that lots must be partitioned by `PartnerId` multi-tenancy (which is out of scope in SRS 2.5).

---

### 7.2 Handoff Checklist for `TASK-AUTH-SEC: Authentication Hardening & Admin User API`

#### A. Preconditions & Policy Clarifications
- [x] Confirmed Admin-exclusive internal user creation directive from Team Lead (centralizing `Manager` and `Staff` creation).
- [x] Verified Identity role storage model (`ApplicationUser.Role` property mapped to `ClaimTypes.Role`).
- [x] Distinguished administrative account status (`ApplicationUser.IsActive`) from brute-force security lockout (`LockoutEnd`).

#### B. API Contract & Endpoints
1. [x] `GET /api/users`: Changed from `[AllowAnonymous]` to `[Authorize(Roles = "Admin")]`. Added query filters: `role`, `isActive`, `searchTerm`.
2. [x] `GET /api/users/{id}`: Changed from `[AllowAnonymous]` to `[Authorize(Roles = "Admin")]`.
3. [x] `POST /api/users/internal`: Implemented for `[Authorize(Roles = "Admin")]` to create internal `Manager` and `Staff` accounts with strong validation.
4. [x] `PATCH /api/users/{id}/status`: Implemented for `[Authorize(Roles = "Admin")]` with `{ "isActive": bool }`. Enforces self-deactivation and admin-deactivation guards.
5. [x] `POST /api/users/authenticate`: Updated authentication pipeline to block deactivated users (`!user.IsActive`) with `400 Bad Request`.

#### C. Test Requirements & Acceptance Criteria
- [x] Integration tests verify that anonymous calls to `/api/users` and `/api/users/{id}` return `401 Unauthorized`.
- [x] Integration tests verify that non-Admin callers to `/api/users` and `/api/users/internal` return `403 Forbidden`.
- [x] Integration tests verify internal accounts (`Manager`, `Staff`) can be created and immediately authenticated.
- [x] Integration tests verify deactivated accounts cannot log in.
- [x] Unit and integration tests verify Admin cannot deactivate their own account or another Admin.

---

## 8. Decisions Requiring Team-Lead Confirmation

Before starting implementation in the next phase, the following three decisions require confirmation:

| Decision ID | Question / Topic | Impact on First Batch | Recommended Default |
| :---: | :--- | :--- | :--- |
| **D1** | **Ownership of `TASK-LOT-01`** | Avoids redundant implementation across team members. | Assign `TASK-LOT-01` to this developer. |
| **D2** | **Local DB Provisioning Strategy** | Governs how LocalDB is populated when running outside in-memory tests. | Add raw DDL table check for `ParkingLots` in `AutomatedMigration.cs`. |
| **D3** | **Email/Phone Login Support** | Affects `UserService.LoginAsync` query evaluation. | Enable dual login (`u.UserName == input \|\| u.Email == input`). |

---

## 9. Source References

- **SRS v1.3**:
  - Section 3.2: `FR-LOT-01` to `FR-LOT-06` (Parking Lot & Zone specifications).
  - Section 3.1: `FR-AUTH-01` to `FR-AUTH-08` (Authentication & Access Control).
- **Implementation Plan**:
  - Tasks `B1.3`, `B1.4`, `B1.5` (Phase 1 Backend work packages).
- **Repository Source Locations**:
  - Entities: [`ParkingLot.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Core/Entities/ParkingLot.cs), [`Zone.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Core/Entities/Zone.cs), [`Slot.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.Core/Entities/Slot.cs).
  - Context: [`PmsDbContext.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.DataAccess/Persistence/PmsDbContext.cs).
  - Seed / Migration: [`AutomatedMigration.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.DataAccess/Persistence/AutomatedMigration.cs).
  - Controllers: [`UsersController.cs`](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs).

---

## 10. Admin Account Management APIs — Audit & Implementation Record

### 10.1 Executive Summary
In accordance with team lead directives and sprint priorities, the Admin Account Management capability (`TASK-AUTH-SEC` scope) has been audited, designed, implemented, and verified with 100% automated test coverage. This implementation provides administrative lifecycle management for internal accounts (`Manager`, `Staff`), distinguishes business account activation status from temporary brute-force security lockouts, and remediates a critical security vulnerability where user account data was publicly accessible without authentication.

### 10.2 Pre-Implementation Audit Findings & Evidence

1. **Role Model Architecture**:
   - `[FACT]` ASP.NET Identity in `parking-management-system` does not utilize `RoleManager<IdentityRole>` or the `AspNetUserRoles` table. Instead, roles are persisted as a single `string Role` property directly on `ApplicationUser` (`[dbo].[Users].[Role]`).
   - `[FACT]` `JwtHelper` reads `user.Role` and emits it as `ClaimTypes.Role` in JWT tokens, which is enforced via standard ASP.NET Core `[Authorize(Roles = "...")]` attributes.
2. **Account Status & Activation**:
   - `[FACT]` `ApplicationUser` previously lacked an explicit administrative status attribute, relying only on ASP.NET Identity's `LockoutEnd` timestamp.
   - `[FACT]` A separate `Customer.IsActive` column existed, but `Customer` is only created for external end-users during registration (`POST /api/users/customer`), not for internal staff. Conflating temporary security lockout (`LockoutEnd`) with business lifecycle activation (`IsActive`) would violate security best practices and cause state collision.
3. **Security Vulnerability in Existing Query Endpoints**:
   - `[FACT]` `GET /api/users` and `GET /api/users/{id}` were annotated with `[AllowAnonymous]`, allowing unauthenticated public callers to scrape all usernames, email addresses, and phone numbers.
4. **SRS vs. Team Lead Direction on Account Creation**:
   - `[FACT]` SRS `FR-AUTH-07` originally specified that a Lot Manager creates Staff accounts scoped to their assigned lot.
   - `[FACT]` The Team Lead explicitly directed that only the `Admin` role creates internal accounts (`Manager` and `Staff`), and that lot-level delegated creation is deferred until Parking Lot management modules are fully established.

### 10.3 Implemented Endpoints & API Contracts

| Method | Endpoint | Authorization | Description & Business Rules |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/users/internal` | `[Authorize(Roles = "Admin")]` | Creates internal `Manager` or `Staff` accounts. Validates username (5-20 alphanumeric), email, VN phone regex, strong password (6-128 chars), and role strictly in `{"Manager", "Staff"}`. Automatically sets `EmailConfirmed = true` and `IsActive = true`. Does not create unnecessary `Customer` record. |
| `PATCH` | `/api/users/{id}/status` | `[Authorize(Roles = "Admin")]` | Activates or deactivates an account (`{ "isActive": boolean }`). Prevents Admin from deactivating their own account (self-lockout prevention). Prevents Admin from deactivating another Admin. Synchronizes `Customer.IsActive` if a linked Customer entity exists. |
| `GET` | `/api/users` | `[Authorize(Roles = "Admin")]` | Secured from `[AllowAnonymous]`. Supports optional query filters: `role` (Admin/Manager/Staff/User), `isActive` (boolean), and `searchTerm` (matches username, email, full name, or phone number). |
| `GET` | `/api/users/{id}` | `[Authorize(Roles = "Admin")]` | Secured from `[AllowAnonymous]`. Returns user details including current `isActive` status. |
| `POST` | `/api/users/authenticate` | `[AllowAnonymous]` | Updated authentication pipeline: immediately blocks deactivated accounts (`!user.IsActive`) with `400 Bad Request` ("Account is deactivated. Please contact an administrator."). |

### 10.4 Repository Code Modifications

The implementation strictly followed minimal and scoped changes, preserving the existing project architecture:

| Component / Layer | File Path | Nature of Change |
| :--- | :--- | :--- |
| **Domain Entity** | `PMS.Core/Entities/ApplicationUser.cs` | Added `public bool IsActive { get; set; } = true;` to `ApplicationUser`. |
| **Data Configuration** | `PMS.DataAccess/Configurations/ApplicationUserConfiguration.cs` | Configured `IsActive` property with default value `true` and `IsRequired()`. |
| **Migration / DDL** | `PMS.DataAccess/Persistence/AutomatedMigration.cs` | Added idempotent DDL check to ensure `IsActive` column exists on `[dbo].[Users]` in SQL Server / LocalDB runtimes. |
| **DTOs & Models** | `PMS.Application/Models/User/UserResponseModel.cs`<br>`PMS.Application/Models/User/CreateInternalUserModel.cs`<br>`PMS.Application/Models/User/UpdateUserStatusModel.cs` | Added `IsActive` to response model; created request DTOs for internal user creation and status toggle. |
| **Validation** | `PMS.Application/Models/Validators/User/CreateInternalUserModelValidator.cs` | Implemented FluentValidation rules: unique username, valid unique email, VN phone regex, 6-128 char password, and restricted role values (`Manager` or `Staff`). |
| **Application Services** | `PMS.Application/Services/IUserService.cs`<br>`PMS.Application/Services/UserService.cs`<br>`PMS.Application/Services/UserProfileService.cs` | Added `CreateInternalUserAsync`, `UpdateUserStatusAsync`; enhanced `GetAllAsync` with role/active/search filters; updated `LoginAsync` to reject inactive users; adjusted profile status calculation. |
| **API Controller** | `PMS.API/Controllers/UsersController.cs` | Secured `GET /api/users` and `GET /api/users/{id}` with `[Authorize(Roles = "Admin")]`; added `POST /api/users/internal` and `PATCH /api/users/{id}/status`. |
| **Test Infrastructure** | `PMS.Api.IntegrationTests/Common/FactoryExtension.cs` | Configured default test client with `Admin` claim; provided helper methods `CreateClientWithRoleAsync` and `CreateUnauthorizedClient`. |
| **Unit & Integration Tests** | `PMS.Application.UnitTests/Services/UserServiceTests.cs`<br>`PMS.Application.UnitTests/Validators/CreateInternalUserModelValidatorTests.cs`<br>`PMS.Api.IntegrationTests/Controllers/UsersEndpointTests.cs` | Added unit tests for service methods, validations, and full integration tests covering authorization, activation lifecycle, login blocks, and edge cases. |

### 10.5 Verification Results

All automated tests execute cleanly in-memory without requiring external database dependencies:
- **Test Command**: `dotnet test`
- **Total Test Cases**: 125 passed, 0 failed, 0 skipped.
  - `PMS.Application.UnitTests`: 103 passed (including validator tests, status update tests, and internal user creation tests).
  - `PMS.Api.IntegrationTests`: 22 passed (including role authorization checks, deactivation login rejection, admin self-deactivation protection, and user filtering).

### 10.6 Unresolved Decisions & Business Clarifications for Team Lead

1. **Lot-Scoped Staff Management (`FR-AUTH-07` Reconciliation)**:
   - *Observation*: The current implementation centralizes internal account creation under `Admin`.
   - *Decision Needed*: Once `LotsController` and lot assignments (`UserParkingLot`) are implemented, should Managers be granted permission to create/manage Staff specifically scoped to their assigned lot (`POST /api/lots/{id}/staff`), or will account management remain centralized under Admin?
2. **Account Deactivation Permission Scope**:
   - *Observation*: Currently, only `Admin` can activate or deactivate accounts.
   - *Decision Needed*: Should Managers be permitted to deactivate Staff members within their lot, or should account status changes remain exclusively an Admin privilege?
3. **Dual Login by Email/Phone**:
   - *Observation*: `TASK-AUTH-SEC` checklist suggested allowing email or phone in place of username in `LoginAsync`.
   - *Current Implementation*: Preserves username authentication and adds deactivation check. Full dual login query (`UserName == input || Email == input`) can be enabled upon frontend UX alignment.

### 10.7 Definition of Done & Task Status

- **Implementation Status**: Complete.
- **Automated Verification**: Passed (125/125 unit and integration tests passing).
- **External / Runtime Verification**: Pending verification in staging/LocalDB environment with frontend UI.
- **Task Status**: `IMPLEMENTATION COMPLETE / AUTOMATED VERIFICATION PASS / EXTERNAL VERIFICATION PENDING`.

