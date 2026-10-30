# Comprehensive Project Scan Report for API Documentation Planning

**Project:** Parking Management System (OJT FSA Team 3)  
**Date of Scan:** October 30, 2026  
**Document Status:** Complete Factual Audit & Planning Base  
**Target File:** `reports/parking-system/api-documentation-project-scan.md`  
**Target Repository:** `thangnx-mc/OJT-report` (branch: `main`)  
**Scope:** Strictly Read-Only Project Scan across Backend and Frontend Repositories  

---

## 1. Executive Summary

This report delivers a rigorous, 100% source-verified technical audit of the entire API surface of the Parking Management System. The scan was conducted under strict read-only constraints across both the ASP.NET Core Backend and React/TypeScript Frontend repositories. Its primary purpose is to provide an exact factual foundation for planning and generating production-grade API documentation.

### Key Metrics Summary

| Metric Dimension | Measured Count | Source Reference |
| :--- | :---: | :--- |
| **Total Backend Controllers** | **22** | `src/PMS.API/Controllers/*.cs` (excluding base `ApiController`) |
| **Total SignalR Realtime Hubs** | **3** | `src/PMS.API/Hubs/*.cs` (`LotHub`, `ParkingOpsHub`, `VehicleVerificationHub`) |
| **Total Backend HTTP Endpoint Actions** | **150** | Verified through AST & controller reflection across all 22 controllers |
| **Total Frontend API Client Service Files** | **21** | `src/api/*.service.ts` & `src/api/lot-management.live.ts` |
| **Total Frontend HTTP Service Calls** | **129** | Analyzed across all frontend services via `axiosClient.ts` |
| **Frontend Calls Successfully Mapped to Backend** | **121** | 93.8% direct parity with backend routes |
| **Unique Backend Endpoints Consumed by Frontend** | **114** | Active backend endpoints driving user interfaces |
| **Unconsumed Backend Endpoints (No Direct UI)** | **36** | Internal administration, webhook/IPN callbacks, file downloads |
| **Frontend Calls with Contract Mismatches / Mock Reliance** | **8** | Wallet transactions, capacity, legacy revenue routes, payment return |
| **Total Automated Solution Tests Discovered** | **922** | `PMS.UnitTests` and `PMS.Api.IntegrationTests` |
| **Working Tree Modifications Introduced** | **0** | Clean working tree preserved across all repositories |
| **Git Commits / Pushes Executed** | **0** | Strictly zero mutations performed |

---

## 2. Repository Paths, Branches, and Scan Scope

The scan examined three local repository directories:

1. **Backend Repository:**
   - **Local Path:** `C:\DEV\OJT-FSA-Team3\parking-management-system`
   - **Current Branch:** `develop`
   - **Latest Commit (HEAD):** `1d148c4`
   - **Working Tree State:** Completely clean (`nothing to commit, working tree clean`).
   - **Technology Stack:** .NET 9.0 (ASP.NET Core Web API), Entity Framework Core 9.0, PostgreSQL, Hangfire, SignalR.

2. **Frontend Repository:**
   - **Local Path:** `C:\DEV\OJT-FSA-Team3\parking-management-system-frontend`
   - **Current Branch:** `develop`
   - **Latest Commit (HEAD):** `2087f5d`
   - **Working Tree State:** Completely clean (`nothing to commit, working tree clean`).
   - **Technology Stack:** React 18, TypeScript, Vite, TailwindCSS, Axios, TanStack React Query.

3. **Report Repository:**
   - **Local Path:** `C:\DEV\OJT-report`
   - **Current Branch:** `main`
   - **Remote Tracking:** `origin/main` (`https://github.com/thangnx-mc/OJT-report`)
   - **Latest Commit (HEAD):** `1103b26`
   - **Working Tree State:** Clean with respect to tracked files (prior untracked scratch reports preserved).

---

## 3. Backend Architecture and API Conventions

### 3.1 Base Routing and Controller Inheritance

All HTTP controllers in `PMS.API` inherit from `ApiController`:
```csharp
// Path: src/PMS.API/Controllers/ApiController.cs
[ApiController]
[Route("api/[controller]")]
public abstract class ApiController : ControllerBase
```
- Controllers either use the default token `api/[controller]` (e.g., `api/Wallet`, `api/Lots`, `api/Reservations`) or explicit specialized route prefixes (e.g., `[Route("api/admin")]`, `[Route("api/manager")]`, `[Route("api/staff")]`, `[Route("api/public/lots/{lotId:int}/layout")]`).
- Action-level routes append to the controller base unless prefixed with a leading slash `/`, which overrides the base route.

### 3.2 Response Envelope Pattern (`ApiResult<T>`)

Every standard API endpoint wraps its payload in the generic envelope `ApiResult<T>` (`src/PMS.Application/Models/ApiResult.cs`):
```csharp
public class ApiResult<T>
{
    public bool Succeeded { get; set; }
    public T Result { get; set; }
    public List<string> Errors { get; set; } = new();
}
```

**Success Response Shape (HTTP 200 / 201):**
```json
{
  "succeeded": true,
  "result": { ... },
  "errors": []
}
```

**Failure Response Shape (HTTP 400 / 401 / 403 / 404 / 409 / 422 / 500):**
```json
{
  "succeeded": false,
  "result": null,
  "errors": [
    "Descriptive failure message explaining the rejection reason."
  ]
}
```

### 3.3 Validation Pipeline and Filter Behavior

- Validation is performed using **FluentValidation** (`FluentValidation.AspNetCore` / `FluentValidationAutoValidation`).
- When a model fails validation, `ValidateModelAttribute` (`src/PMS.API/Filters/ValidateModelAttribute.cs`) intercepts execution prior to the controller action and returns:
  ```json
  {
    "succeeded": false,
    "result": null,
    "errors": [
      "The Email field is required.",
      "Password must contain at least 8 characters, one uppercase, one number."
    ]
  }
  ```
- The HTTP status code returned for validation failures is strictly **HTTP 400 Bad Request**.

### 3.4 Rate Limiting Policies

The API enforces rate limiting via ASP.NET Core rate limiting middleware (`src/PMS.API/Middlewares/RateLimitingMiddleware.cs`):
1. **`PasswordResetRateLimit`**: Applied to password reset requests. Allows a maximum of **5 requests per 15 minutes** per client IP. Exceeding requests return **HTTP 429 Too Many Requests**.
2. **`PublicLayoutRateLimit`**: Applied to public lot visual layouts. Allows a maximum of **120 requests per minute** per client IP to safeguard server memory against layout scraping.

### 3.5 Global Exception Handling Middleware

All unhandled exceptions are caught by `ExceptionHandlingMiddleware` (`src/PMS.API/Middlewares/ExceptionHandlingMiddleware.cs`):
- `NotFoundException` / `ResourceNotFoundException` $\rightarrow$ **HTTP 404 Not Found**
- `BadRequestException` $\rightarrow$ **HTTP 400 Bad Request**
- `ForbiddenException` $\rightarrow$ **HTTP 403 Forbidden**
- `UnprocessableRequestException` $\rightarrow$ **HTTP 422 Unprocessable Entity**
- `ConflictException` $\rightarrow$ **HTTP 409 Conflict**
- General unhandled exceptions $\rightarrow$ **HTTP 500 Internal Server Error** with generic safe message in production.

### 3.6 JWT Authentication and RBAC Conventions

- Tokens are signed with HMAC-SHA256 via `JwtHelper.cs` (`src/PMS.Application/Helpers/JwtHelper.cs`).
- Token Lifetime: Default **7 days**.
- Claim mapping includes:
  - `ClaimTypes.NameIdentifier` (User ID GUID)
  - `ClaimTypes.Email` (User Email)
  - `ClaimTypes.Role` (`Admin`, `Manager`, `Staff`, `User`)
  - `purpose` (Optional claim, e.g. `2FA_Challenge`)
- Role authorization is enforced declaratively using `[Authorize(Roles = "...")]`.
- Any endpoint receiving a JWT containing `purpose = "2FA_Challenge"` will reject the request with **HTTP 403 Forbidden** unless the endpoint is specifically designed for 2FA completion (`POST /api/Users/two-factor` or `POST /api/Users/two-factor/resend`).

### 3.7 Background Job Scheduling (Hangfire)

The backend hosts Hangfire at `/hangfire` (`src/PMS.API/Program.cs`) with recurring automated jobs:
1. **`LotReopeningRecurringJob`**: Automatically marks closed lots as open when their closure window elapses.
2. **`PaperDocPurgeRecurringJob`**: Automatically purges temporary uploaded vehicle registration documents for abandoned submissions after 30 days.
3. **`ParkingDeadlineEnforcementJob`**: Sweeps expired reservations and overdue parking sessions to release slots and calculate overtime fees.
4. **`NotificationPurgeRecurringJob`**: Purges read notifications older than 90 days.

---

## 4. Endpoint Inventory Grouped by Module

The 150 backend endpoints are organized into 12 functional modules. Every endpoint below was directly extracted and verified from the controller C# source code.

### Module 1: Authentication & User Accounts (26 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/api/Users` | `UsersController.GetAllAsync` | `Admin` | [FromQuery] string role = null, [FromQuery] bool? isActive = null, [FromQuery] string searchTerm = null | `ApiResult<IEnumerable<UserResponseModel>>` | [UsersController.cs#L33](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L33) |
| `GET` | `/api/Users/{id}` | `UsersController.GetByIdAsync` | `Admin` | string id | `ApiResult<UserResponseModel>` | [UsersController.cs#L41](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L41) |
| `POST` | `/api/Users/internal` | `UsersController.CreateInternalUserAsync` | `Admin` | [FromBody] CreateInternalUserModel model | `ApiResult<CreateUserResponseModel>` | [UsersController.cs#L48](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L48) |
| `PATCH` | `/api/Users/{id}/status` | `UsersController.UpdateStatusAsync` | `Admin` | string id, [FromBody] UpdateUserStatusModel model | `ApiResult<bool>` | [UsersController.cs#L55](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L55) |
| `POST` | `/api/Users/register` | `UsersController.RegisterAsync` | `[AllowAnonymous` | CreateUserModel createUserModel | `ApiResult<CreateUserResponseModel>` | [UsersController.cs#L64](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L64) |
| `POST` | `/api/Users/authenticate` | `UsersController.LoginAsync` | `[AllowAnonymous` | LoginUserModel loginUserModel | `ApiResult<LoginResponseModel>` | [UsersController.cs#L72](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L72) |
| `POST` | `/api/Users/two-factor` | `UsersController.VerifyTwoFactorAsync` | `[AllowAnonymous` | [FromBody] TwoFactorLoginModel model | `ApiResult<LoginResponseModel>` | [UsersController.cs#L80](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L80) |
| `POST` | `/api/Users/two-factor/resend` | `UsersController.ResendTwoFactorOtpAsync` | `[AllowAnonymous` | [FromBody] ResendTwoFactorModel model | `ApiResult<ResendTwoFactorResponseModel>` | [UsersController.cs#L88](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L88) |
| `POST` | `/api/Users/login-otp/request` | `UsersController.RequestLoginOtpAsync` | `[AllowAnonymous` | RequestLoginOtpModel model | `ApiResult<string>` | [UsersController.cs#L96](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L96) |
| `POST` | `/api/Users/login-otp/verify` | `UsersController.LoginWithOtpAsync` | `[AllowAnonymous` | VerifyLoginOtpModel model | `ApiResult<LoginResponseModel>` | [UsersController.cs#L105](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L105) |
| `POST` | `/api/Users/forgot-password` | `UsersController.ForgotPasswordAsync` | `[AllowAnonymous` | [FromBody] ForgotPasswordModel model | `ApiResult<string>` | [UsersController.cs#L114](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L114) |
| `POST` | `/api/Users/reset-password` | `UsersController.ResetPasswordAsync` | `[AllowAnonymous` | [FromBody] ResetPasswordModel model | `ApiResult<string>` | [UsersController.cs#L126](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L126) |
| `POST` | `/api/Users/login-google` | `UsersController.LoginWithGoogleAsync` | `[AllowAnonymous` | GoogleLoginModel model | `ApiResult<LoginResponseModel>` | [UsersController.cs#L135](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L135) |
| `GET` | `/api/Users/profile` | `UsersController.GetProfileAsync` | `Authenticated` | None | `ApiResult<UserProfileResponseModel>` | [UsersController.cs#L143](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L143) |
| `PUT` | `/api/Users/profile` | `UsersController.UpdateProfileAsync` | `Authenticated` | [FromBody] UpdateUserProfileModel model | `ApiResult<UserProfileResponseModel>` | [UsersController.cs#L152](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L152) |
| `GET` | `/api/Users/profile/check-unique` | `UsersController.CheckProfileUniquenessAsync` | `Authenticated` | [FromQuery] string? phone = null, [FromQuery] string? citizenId = null | `ApiResult<ProfileFieldAvailabilityModel>` | [UsersController.cs#L161](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L161) |
| `POST` | `/api/Users/profile/avatar` | `UsersController.UploadAvatarAsync` | `Authenticated` | IFormFile file | `ApiResult<UserProfileResponseModel>` | [UsersController.cs#L171](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L171) |
| `GET` | `/api/Users/profile/vehicles` | `UsersController.GetVehiclesAsync` | `Authenticated` | None | `ApiResult<List<VehicleResponseModel>>` | [UsersController.cs#L180](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L180) |
| `GET` | `/api/Users/profile/vehicles/check-plate` | `UsersController.CheckPlateAvailabilityAsync` | `Authenticated` | [FromQuery] string plate, [FromQuery] Guid? excludeVehicleId = null | `ApiResult<PlateAvailabilityResponseModel>` | [UsersController.cs#L189](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L189) |
| `POST` | `/api/Users/profile/vehicles` | `UsersController.AddVehicleAsync` | `Authenticated` | [FromBody] CreateVehicleModel model | `ApiResult<VehicleResponseModel>` | [UsersController.cs#L198](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L198) |
| `PUT` | `/api/Users/profile/vehicles/{vehicleId:guid}` | `UsersController.UpdateVehicleAsync` | `Authenticated` | Guid vehicleId, [FromBody] UpdateVehicleModel model | `ApiResult<VehicleResponseModel>` | [UsersController.cs#L207](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L207) |
| `DELETE` | `/api/Users/profile/vehicles/{vehicleId:guid}` | `UsersController.DeleteVehicleAsync` | `Authenticated` | Guid vehicleId | `ApiResult<bool>` | [UsersController.cs#L216](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L216) |
| `PATCH` | `/api/Users/profile/vehicles/{vehicleId:guid}/default` | `UsersController.SetDefaultVehicleAsync` | `Authenticated` | Guid vehicleId | `ApiResult<VehicleResponseModel>` | [UsersController.cs#L225](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L225) |
| `GET` | `/api/Users/profile/vehicle-types` | `UsersController.GetVehicleTypesAsync` | `Authenticated` | None | `ApiResult<List<VehicleTypeResponseModel>>` | [UsersController.cs#L234](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L234) |
| `PUT` | `/api/Users/profile/change-password` | `UsersController.ChangePasswordAsync` | `Authenticated` | [FromBody] ChangePasswordModel model | `ApiResult<bool>` | [UsersController.cs#L242](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L242) |
| `GET` | `/api/Users/{id}/profile` | `UsersController.GetProfileByUserIdAsync` | `Admin` | string id | `ApiResult<UserProfileResponseModel>` | [UsersController.cs#L251](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/UsersController.cs#L251) |

### Module 2: Parking Lots, Layouts, Zones & Slots (22 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/api/lot-templates` | `LayoutsController.GetTemplates` | `Inherit / Anonymous` | None | `ApiResult<IReadOnlyList<LotTemplateResponseModel>>` | [LayoutsController.cs#L23](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LayoutsController.cs#L23) |
| `GET` | `/api/public/lots/{lotId:int}/layout` | `LayoutsController.GetPublishedAsync` | `Inherit / Anonymous` | int lotId | `ApiResult<LayoutResponseModel>` | [LayoutsController.cs#L30](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LayoutsController.cs#L30) |
| `GET` | `/api/public/lots/{lotId:int}/layout/status` | `LayoutsController.GetStatusAsync` | `Inherit / Anonymous` | int lotId | `ApiResult<LayoutStatusResponseModel>` | [LayoutsController.cs#L37](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LayoutsController.cs#L37) |
| `GET` | `/api/public/lots/{lotId:int}/layout/2d` | `LayoutsController.Get2DAsync` | `Inherit / Anonymous` | int lotId, [FromQuery] string detail = LayoutDetail.Slots | `ApiResult<Layout2DResponseModel>` | [LayoutsController.cs#L44](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LayoutsController.cs#L44) |
| `GET` | `/api/lots` | `LotsController.GetAllAsync` | `Admin,Manager` | [FromQuery] string searchTerm = null, [FromQuery] string status = null, [FromQuery] bool includeDeleted = false | `ApiResult<IEnumerable<ParkingLotResponseModel>>` | [LotsController.cs#L24](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LotsController.cs#L24) |
| `GET` | `/api/public/lots` | `LotsController.GetPublicLotsAsync` | `[AllowAnonymous` | [FromQuery] string searchTerm = null | `ApiResult<IEnumerable<PublicParkingLotResponseModel>>` | [LotsController.cs#L37](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LotsController.cs#L37) |
| `GET` | `/api/public/lots/{id:int}/prices` | `LotsController.GetPublicPricesAsync` | `[AllowAnonymous` | int id | `ApiResult<IReadOnlyList<PublicLotPriceModel>>` | [LotsController.cs#L50](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LotsController.cs#L50) |
| `GET` | `/api/lots/{id:int}` | `LotsController.GetByIdAsync` | `Admin,Manager` | int id | `ApiResult<ParkingLotResponseModel>` | [LotsController.cs#L57](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LotsController.cs#L57) |
| `POST` | `/api/lots` | `LotsController.CreateAsync` | `Admin` | [FromBody] CreateParkingLotModel model | `ApiResult<ParkingLotResponseModel>` | [LotsController.cs#L65](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LotsController.cs#L65) |
| `PUT` | `/api/lots/{id:int}` | `LotsController.UpdateAsync` | `Admin` | int id, [FromBody] UpdateParkingLotModel model | `ApiResult<ParkingLotResponseModel>` | [LotsController.cs#L75](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LotsController.cs#L75) |
| `PUT` | `/api/lots/{id:int}/template` | `LotsController.ApplyTemplateAsync` | `Admin` | int id, [FromBody] ApplyLotTemplateModel model | `ApiResult<ParkingLotResponseModel>` | [LotsController.cs#L84](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LotsController.cs#L84) |
| `PATCH` | `/api/lots/{id:int}/status` | `LotsController.UpdateStatusAsync` | `Admin,Manager` | int id, [FromBody] UpdateParkingLotStatusModel model | `ApiResult<bool>` | [LotsController.cs#L93](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LotsController.cs#L93) |
| `DELETE` | `/api/lots/{id:int}` | `LotsController.DeleteAsync` | `Admin` | int id | `ApiResult<bool>` | [LotsController.cs#L101](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/LotsController.cs#L101) |
| `GET` | `/api/slots` | `SlotsController.GetAllAsync` | `Admin,Manager` | [FromQuery] int? zoneId = null, [FromQuery] int? vehicleTypeId = null, [FromQuery] string status = null, [FromQuery] bool? isActive = null | `ApiResult<IEnumerable<SlotResponseModel>>` | [SlotsController.cs#L16](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/SlotsController.cs#L16) |
| `GET` | `/api/zones/{zoneId:int}/slots` | `SlotsController.GetByZoneIdAsync` | `Admin,Manager` | int zoneId | `ApiResult<IEnumerable<SlotResponseModel>>` | [SlotsController.cs#L27](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/SlotsController.cs#L27) |
| `GET` | `/api/slots/{id:int}` | `SlotsController.GetByIdAsync` | `Admin,Manager` | int id | `ApiResult<SlotResponseModel>` | [SlotsController.cs#L34](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/SlotsController.cs#L34) |
| `PUT` | `/api/slots/{id:int}` | `SlotsController.UpdateAsync` | `Admin` | int id, [FromBody] UpdateSlotModel model | `ApiResult<SlotResponseModel>` | [SlotsController.cs#L43](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/SlotsController.cs#L43) |
| `PATCH` | `/api/slots/{id:int}/status` | `SlotsController.UpdateStatusAsync` | `Admin,Manager` | int id, [FromBody] UpdateSlotStatusModel model | `ApiResult<bool>` | [SlotsController.cs#L51](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/SlotsController.cs#L51) |
| `GET` | `/api/zones` | `ZonesController.GetAllAsync` | `Admin,Manager` | [FromQuery] int? lotId = null | `ApiResult<IEnumerable<ZoneResponseModel>>` | [ZonesController.cs#L16](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ZonesController.cs#L16) |
| `GET` | `/api/lots/{lotId:int}/zones` | `ZonesController.GetByLotIdAsync` | `Admin,Manager` | int lotId | `ApiResult<IEnumerable<ZoneResponseModel>>` | [ZonesController.cs#L24](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ZonesController.cs#L24) |
| `GET` | `/api/zones/{id:int}` | `ZonesController.GetByIdAsync` | `Admin,Manager` | int id | `ApiResult<ZoneResponseModel>` | [ZonesController.cs#L31](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ZonesController.cs#L31) |
| `PUT` | `/api/zones/{id:int}` | `ZonesController.UpdateAsync` | `Admin` | int id, [FromBody] UpdateZoneModel model | `ApiResult<ZoneResponseModel>` | [ZonesController.cs#L41](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ZonesController.cs#L41) |

### Module 3: Customer Reservations (8 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `POST` | `/api/reservations` | `ReservationsController.CreateAsync` | `User` | [FromBody] CreateReservationModel model | `ApiResult<ReservationResponseModel>` | [ReservationsController.cs#L23](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ReservationsController.cs#L23) |
| `GET` | `/api/reservations/mine` | `ReservationsController.GetMineAsync` | `User` | [FromQuery] bool activeOnly = false | `ApiResult<IReadOnlyList<ReservationResponseModel>>` | [ReservationsController.cs#L30](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ReservationsController.cs#L30) |
| `GET` | `/api/reservations/upcoming` | `ReservationsController.GetUpcomingAsync` | `User` | None | `ApiResult<ReservationResponseModel>` | [ReservationsController.cs#L38](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ReservationsController.cs#L38) |
| `GET` | `/api/reservations/{id:long}` | `ReservationsController.GetAsync` | `User` | long id | `ApiResult<ReservationResponseModel>` | [ReservationsController.cs#L42](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ReservationsController.cs#L42) |
| `PUT` | `/api/reservations/{id:long}` | `ReservationsController.UpdateAsync` | `User` | long id, [FromBody] UpdateReservationModel model | `ApiResult<ReservationResponseModel>` | [ReservationsController.cs#L49](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ReservationsController.cs#L49) |
| `POST` | `/api/reservations/{id:long}/cancel` | `ReservationsController.CancelAsync` | `User` | long id, [FromBody] CancelReservationModel model | `ApiResult<ReservationResponseModel>` | [ReservationsController.cs#L56](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ReservationsController.cs#L56) |
| `GET` | `/api/reservations/{id:long}/qr` | `ReservationsController.GetQrAsync` | `User` | long id | `ApiResult<ReservationQrModel>` | [ReservationsController.cs#L63](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ReservationsController.cs#L63) |
| `POST` | `/api/reservations/{id:long}/payments` | `ReservationsController.PayDepositAsync` | `User` | long id, [FromBody] CreatePaymentModel model | `ApiResult<PaymentResponseModel>` | [ReservationsController.cs#L70](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ReservationsController.cs#L70) |

### Module 4: Customer Parking Sessions (7 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/api/parking-sessions/mine` | `ParkingSessionsController.GetMineAsync` | `User` | None | `ApiResult<IReadOnlyList<SessionResponseModel>>` | [ParkingSessionsController.cs#L22](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ParkingSessionsController.cs#L22) |
| `GET` | `/api/parking-sessions/history` | `ParkingSessionsController.GetHistoryAsync` | `User` | [FromQuery] ParkingHistoryQuery query | `ApiResult<IReadOnlyList<SessionResponseModel>>` | [ParkingSessionsController.cs#L30](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ParkingSessionsController.cs#L30) |
| `GET` | `/api/parking-sessions/{id:long}/receipt` | `ParkingSessionsController.GetReceiptAsync` | `User` | long id | `ApiResult<ParkingReceiptModel>` | [ParkingSessionsController.cs#L35](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ParkingSessionsController.cs#L35) |
| `GET` | `/api/parking-sessions/current` | `ParkingSessionsController.GetCurrentAsync` | `User` | None | `ApiResult<ParkingReceiptModel>` | [ParkingSessionsController.cs#L43](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ParkingSessionsController.cs#L43) |
| `GET` | `/api/parking-sessions/{id:long}/exit-pass` | `ParkingSessionsController.GetExitPassAsync` | `User` | long id | `ApiResult<ExitPassModel>` | [ParkingSessionsController.cs#L48](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ParkingSessionsController.cs#L48) |
| `GET` | `/api/parking-sessions/{id:long}/quote` | `ParkingSessionsController.QuoteAsync` | `User` | long id | `ApiResult<CheckoutQuoteModel>` | [ParkingSessionsController.cs#L53](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ParkingSessionsController.cs#L53) |
| `POST` | `/api/parking-sessions/{id:long}/payments` | `ParkingSessionsController.PayAsync` | `User` | long id, [FromBody] CreatePaymentModel model | `ApiResult<PaymentResponseModel>` | [ParkingSessionsController.cs#L58](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ParkingSessionsController.cs#L58) |

### Module 5: Staff Gate Operations & Gate Requests (27 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `POST` | `/api/reservations/{reservationId:long}/check-in-request` | `GateRequestsController.RequestCheckInAsync` | `User` | long reservationId | `ApiResult<GateRequestModel>` | [GateRequestsController.cs#L24](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/GateRequestsController.cs#L24) |
| `POST` | `/api/parking-sessions/{sessionId:long}/check-out-request` | `GateRequestsController.RequestCheckOutAsync` | `User` | long sessionId | `ApiResult<GateRequestModel>` | [GateRequestsController.cs#L30](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/GateRequestsController.cs#L30) |
| `GET` | `/api/gate-requests/mine` | `GateRequestsController.GetMineAsync` | `User` | None | `ApiResult<GateRequestModel>` | [GateRequestsController.cs#L36](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/GateRequestsController.cs#L36) |
| `POST` | `/api/gate-requests/{requestId:long}/cancel` | `GateRequestsController.CancelAsync` | `User` | long requestId | `ApiResult<GateRequestModel>` | [GateRequestsController.cs#L41](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/GateRequestsController.cs#L41) |
| `GET` | `/api/staff/lots/{lotId:int}/gate-requests` | `GateRequestsController.GetQueueAsync` | `Staff,Manager,Admin` | int lotId, [FromQuery] string kind = null | `ApiResult<IReadOnlyList<GateRequestModel>>` | [GateRequestsController.cs#L49](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/GateRequestsController.cs#L49) |
| `POST` | `/api/staff/gate-requests/{requestId:long}/approve` | `GateRequestsController.ApproveAsync` | `Staff,Manager,Admin` | long requestId, [FromBody] ApproveGateRequestModel model | `ApiResult<GateRequestDecisionModel>` | [GateRequestsController.cs#L55](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/GateRequestsController.cs#L55) |
| `POST` | `/api/staff/gate-requests/{requestId:long}/refuse` | `GateRequestsController.RefuseAsync` | `Staff,Manager,Admin` | long requestId, [FromBody] RefuseGateRequestModel model | `ApiResult<GateRequestModel>` | [GateRequestsController.cs#L61](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/GateRequestsController.cs#L61) |
| `GET` | `/api/staff/lots` | `StaffController.GetLotsAsync` | `Staff` | None | `ApiResult<IEnumerable<AssignedParkingLotModel>>` | [StaffController.cs#L21](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffController.cs#L21) |
| `GET` | `/api/staff/lots/{lotId:int}/gates` | `StaffGateController.GetGatesAsync` | `Staff,Manager,Admin` | int lotId | `ApiResult<IReadOnlyList<GateResponseModel>>` | [StaffGateController.cs#L19](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L19) |
| `GET` | `/api/staff/lots/{lotId:int}/zone-counters` | `StaffGateController.GetZoneCountersAsync` | `Staff,Manager,Admin` | int lotId | `ApiResult<IReadOnlyList<ZoneCounterModel>>` | [StaffGateController.cs#L24](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L24) |
| `GET` | `/api/staff/lots/{lotId:int}/sessions` | `StaffGateController.GetSessionsAsync` | `Staff,Manager,Admin` | int lotId, [FromQuery] string status = null, [FromQuery] string plate = null | `ApiResult<IReadOnlyList<SessionResponseModel>>` | [StaffGateController.cs#L29](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L29) |
| `GET` | `/api/staff/lots/{lotId:int}/sessions/history` | `StaffGateController.GetHistoryAsync` | `Staff,Manager,Admin` | int lotId, [FromQuery] string plate = null, [FromQuery] int page = 1, [FromQuery] int pageSize = 10 | `ApiResult<StaffSessionHistoryPageModel>` | [StaffGateController.cs#L34](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L34) |
| `GET` | `/api/staff/lots/{lotId:int}/payments` | `StaffGateController.GetPaymentHistoryAsync` | `Staff,Manager,Admin` | int lotId, [FromQuery] string method = null, [FromQuery] string plate = null, [FromQuery] int page = 1, [FromQuery] int pageSize = 10 | `ApiResult<StaffPaymentHistoryPageModel>` | [StaffGateController.cs#L39](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L39) |
| `GET` | `/api/staff/lots/{lotId:int}/sessions/lookup` | `StaffGateController.LookupAsync` | `Staff,Manager,Admin` | int lotId, [FromQuery] string plate = null, [FromQuery] string ticket = null, [FromQuery] string code = null | `ApiResult<SessionResponseModel>` | [StaffGateController.cs#L44](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L44) |
| `GET` | `/api/staff/lots/{lotId:int}/reservations` | `StaffGateController.GetReservationsAsync` | `Staff,Manager,Admin` | int lotId, [FromQuery] string status = null, [FromQuery] DateTime? from = null, [FromQuery] DateTime? to = null | `ApiResult<IReadOnlyList<ReservationResponseModel>>` | [StaffGateController.cs#L49](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L49) |
| `GET` | `/api/staff/lots/{lotId:int}/reservations/lookup` | `StaffGateController.LookupReservationAsync` | `Staff,Manager,Admin` | int lotId, [FromQuery] string code = null, [FromQuery] string qr = null, [FromQuery] string plate = null | `ApiResult<GateReservationLookupModel>` | [StaffGateController.cs#L57](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L57) |
| `GET` | `/api/staff/lots/{lotId:int}/plate-lists` | `StaffGateController.GetPlateListsAsync` | `Staff,Manager,Admin` | int lotId, [FromQuery] string listType = null | `ApiResult<IReadOnlyList<PlateListEntryModel>>` | [StaffGateController.cs#L62](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L62) |
| `POST` | `/api/staff/lots/{lotId:int}/check-in` | `StaffGateController.CheckInAsync` | `Staff,Manager,Admin` | int lotId, [FromBody] CheckInModel model | `ApiResult<CheckInResultModel>` | [StaffGateController.cs#L67](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L67) |
| `GET` | `/api/staff/sessions/{id:long}` | `StaffGateController.GetSessionAsync` | `Staff,Manager,Admin` | long id | `ApiResult<SessionResponseModel>` | [StaffGateController.cs#L71](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L71) |
| `POST` | `/api/staff/sessions/{id:long}/entry-confirm` | `StaffGateController.ConfirmEntryAsync` | `Staff,Manager,Admin` | long id | `ApiResult<SessionResponseModel>` | [StaffGateController.cs#L76](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L76) |
| `POST` | `/api/staff/sessions/{id:long}/entry-cancel` | `StaffGateController.CancelEntryAsync` | `Staff,Manager,Admin` | long id | `ApiResult<SessionResponseModel>` | [StaffGateController.cs#L81](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L81) |
| `POST` | `/api/staff/sessions/{id:long}/quote` | `StaffGateController.QuoteAsync` | `Staff,Manager,Admin` | long id, [FromBody] CheckoutQuoteRequestModel model | `ApiResult<CheckoutQuoteModel>` | [StaffGateController.cs#L86](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L86) |
| `POST` | `/api/staff/sessions/{id:long}/cancel-payment` | `StaffGateController.CancelPaymentAsync` | `Staff,Manager,Admin` | long id | `ApiResult<SessionResponseModel>` | [StaffGateController.cs#L90](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L90) |
| `POST` | `/api/staff/sessions/{id:long}/payments/cash` | `StaffGateController.PayCashAsync` | `Staff,Manager,Admin` | long id, [FromBody] CashPaymentModel model | `ApiResult<PaymentResponseModel>` | [StaffGateController.cs#L94](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L94) |
| `POST` | `/api/staff/sessions/{id:long}/payments/online` | `StaffGateController.PayOnlineAsync` | `Staff,Manager,Admin` | long id, [FromBody] LanePaymentModel model | `ApiResult<PaymentResponseModel>` | [StaffGateController.cs#L99](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L99) |
| `POST` | `/api/staff/sessions/{id:long}/check-out` | `StaffGateController.CheckoutAsync` | `Staff,Manager,Admin` | long id, [FromBody] CheckoutModel model | `ApiResult<CheckoutResultModel>` | [StaffGateController.cs#L104](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L104) |
| `POST` | `/api/staff/sessions/{id:long}/lost-ticket` | `StaffGateController.LostTicketAsync` | `Staff,Manager,Admin` | long id, [FromBody] LostTicketModel model | `ApiResult<SessionResponseModel>` | [StaffGateController.cs#L109](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/StaffGateController.cs#L109) |

### Module 6: Manager Operations & Pricing (23 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/api/manager/lots` | `ManagerController.GetLotsAsync` | `Manager` | None | `ApiResult<IEnumerable<AssignedParkingLotModel>>` | [ManagerController.cs#L21](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerController.cs#L21) |
| `GET` | `/api/manager/staff` | `ManagerController.GetStaffAsync` | `Manager` | [FromQuery] int? parkingLotId = null, [FromQuery] bool? isActive = null, [FromQuery] string searchTerm = null | `ApiResult<IEnumerable<EmployeeAccountResponseModel>>` | [ManagerController.cs#L28](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerController.cs#L28) |
| `GET` | `/api/manager/staff/{id}` | `ManagerController.GetStaffByIdAsync` | `Manager` | string id | `ApiResult<EmployeeAccountResponseModel>` | [ManagerController.cs#L38](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerController.cs#L38) |
| `POST` | `/api/manager/staff` | `ManagerController.CreateStaffAsync` | `Manager` | [FromBody] CreateStaffAccountModel model | `ApiResult<EmployeeAccountResponseModel>` | [ManagerController.cs#L45](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerController.cs#L45) |
| `PUT` | `/api/manager/staff/{id}` | `ManagerController.UpdateStaffAsync` | `Manager` | string id, [FromBody] UpdateEmployeeAccountModel model | `ApiResult<EmployeeAccountResponseModel>` | [ManagerController.cs#L52](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerController.cs#L52) |
| `PUT` | `/api/manager/staff/{id}/lot` | `ManagerController.AssignStaffLotAsync` | `Manager` | string id, [FromBody] AssignStaffParkingLotModel model | `ApiResult<EmployeeAccountResponseModel>` | [ManagerController.cs#L59](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerController.cs#L59) |
| `PATCH` | `/api/manager/staff/{id}/status` | `ManagerController.UpdateStaffStatusAsync` | `Manager` | string id, [FromBody] UpdateUserStatusModel model | `ApiResult<EmployeeAccountResponseModel>` | [ManagerController.cs#L66](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerController.cs#L66) |
| `DELETE` | `/api/manager/staff/{id}` | `ManagerController.DeleteStaffAsync` | `Manager` | string id | `ApiResult<bool>` | [ManagerController.cs#L74](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerController.cs#L74) |
| `GET` | `/api/manager/lots/{lotId:int}/reservations` | `ManagerOperationsController.GetReservationsAsync` | `Manager,Admin` | int lotId, [FromQuery] string status = null, [FromQuery] DateTime? from = null, [FromQuery] DateTime? to = null | `ApiResult<IReadOnlyList<ReservationResponseModel>>` | [ManagerOperationsController.cs#L16](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerOperationsController.cs#L16) |
| `POST` | `/api/manager/reservations/{id:long}/cancel` | `ManagerOperationsController.CancelReservationAsync` | `Manager,Admin` | long id, [FromBody] CancelReservationModel model | `ApiResult<ReservationResponseModel>` | [ManagerOperationsController.cs#L21](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerOperationsController.cs#L21) |
| `POST` | `/api/manager/sessions/{id:long}/abnormal-close` | `ManagerOperationsController.CloseAbnormallyAsync` | `Manager,Admin` | long id, [FromBody] SessionDecisionModel model | `ApiResult<SessionResponseModel>` | [ManagerOperationsController.cs#L26](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerOperationsController.cs#L26) |
| `POST` | `/api/manager/sessions/{id:long}/lost-ticket/approve` | `ManagerOperationsController.ApproveLostTicketAsync` | `Manager,Admin` | long id, [FromBody] SessionDecisionModel model | `ApiResult<SessionResponseModel>` | [ManagerOperationsController.cs#L30](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerOperationsController.cs#L30) |
| `POST` | `/api/manager/sessions/{id:long}/lost-ticket/reject` | `ManagerOperationsController.RejectLostTicketAsync` | `Manager,Admin` | long id, [FromBody] SessionDecisionModel model | `ApiResult<SessionResponseModel>` | [ManagerOperationsController.cs#L34](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerOperationsController.cs#L34) |
| `GET` | `/api/manager/lots/{lotId:int}/plate-lists` | `ManagerOperationsController.GetPlateListsAsync` | `Manager,Admin` | int lotId, [FromQuery] string listType = null | `ApiResult<IReadOnlyList<PlateListEntryModel>>` | [ManagerOperationsController.cs#L38](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerOperationsController.cs#L38) |
| `POST` | `/api/manager/lots/{lotId:int}/plate-lists` | `ManagerOperationsController.AddPlateListEntryAsync` | `Manager,Admin` | int lotId, [FromBody] CreatePlateListEntryModel model | `ApiResult<PlateListEntryModel>` | [ManagerOperationsController.cs#L43](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerOperationsController.cs#L43) |
| `DELETE` | `/api/manager/lots/{lotId:int}/plate-lists/{id:int}` | `ManagerOperationsController.RemovePlateListEntryAsync` | `Manager,Admin` | int lotId, int id | `ApiResult<bool>` | [ManagerOperationsController.cs#L47](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerOperationsController.cs#L47) |
| `GET` | `/api/manager/lots/{lotId:int}/pricing-rules` | `ManagerPricingController.GetPricingRulesAsync` | `Manager,Admin` | int lotId, [FromQuery] string status = null | `ApiResult<IReadOnlyList<PricingRuleModel>>` | [ManagerPricingController.cs#L21](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerPricingController.cs#L21) |
| `GET` | `/api/manager/lots/{lotId:int}/pricing-rules/{id:int}` | `ManagerPricingController.GetPricingRuleByIdAsync` | `Manager,Admin` | int lotId, int id | `ApiResult<PricingRuleModel>` | [ManagerPricingController.cs#L26](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerPricingController.cs#L26) |
| `POST` | `/api/manager/lots/{lotId:int}/pricing-rules` | `ManagerPricingController.CreatePricingRuleAsync` | `Manager,Admin` | int lotId, [FromBody] CreatePricingRuleModel model | `ApiResult<PricingRuleModel>` | [ManagerPricingController.cs#L31](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerPricingController.cs#L31) |
| `PUT` | `/api/manager/lots/{lotId:int}/pricing-rules/{id:int}` | `ManagerPricingController.UpdatePricingRuleAsync` | `Manager,Admin` | int lotId, int id, [FromBody] UpdatePricingRuleModel model | `ApiResult<PricingRuleModel>` | [ManagerPricingController.cs#L36](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerPricingController.cs#L36) |
| `PATCH` | `/api/manager/lots/{lotId:int}/pricing-rules/{id:int}/status` | `ManagerPricingController.SetPricingRuleStatusAsync` | `Manager,Admin` | int lotId, int id, [FromBody] SetPricingRuleStatusModel model | `ApiResult<PricingRuleModel>` | [ManagerPricingController.cs#L41](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerPricingController.cs#L41) |
| `DELETE` | `/api/manager/lots/{lotId:int}/pricing-rules/{id:int}` | `ManagerPricingController.DeletePricingRuleAsync` | `Manager,Admin` | int lotId, int id | `ApiResult<bool>` | [ManagerPricingController.cs#L46](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerPricingController.cs#L46) |
| `POST` | `/api/manager/lots/{lotId:int}/pricing-rules/simulate` | `ManagerPricingController.SimulatePricingAsync` | `Manager,Admin` | int lotId, [FromBody] PricingSimulationRequestModel model | `ApiResult<PricingSimulationResultModel>` | [ManagerPricingController.cs#L54](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerPricingController.cs#L54) |

### Module 7: Vehicle Verification & Document Management (9 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/api/users/profile/rejected-vehicles` | `VehicleDocumentsController.GetRecentlyRejectedAsync` | `Authenticated` | None | `ApiResult<List<RejectedVehicleModel>>` | [VehicleDocumentsController.cs#L30](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/VehicleDocumentsController.cs#L30) |
| `GET` | `/api/users/profile/vehicles/{vehicleId:guid}/documents` | `VehicleDocumentsController.GetLatestAsync` | `Authenticated` | Guid vehicleId | `ApiResult<VehicleVerificationModel>` | [VehicleDocumentsController.cs#L38](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/VehicleDocumentsController.cs#L38) |
| `POST` | `/api/users/profile/vehicles/{vehicleId:guid}/documents/{documentType}/{side}` | `VehicleDocumentsController.UploadAsync` | `Authenticated` | Guid vehicleId, string documentType, string side, IFormFile file | `ApiResult<VehicleVerificationModel>` | [VehicleDocumentsController.cs#L49](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/VehicleDocumentsController.cs#L49) |
| `POST` | `/api/users/profile/vehicles/{vehicleId:guid}/documents/submit` | `VehicleDocumentsController.SubmitAsync` | `Authenticated` | Guid vehicleId | `ApiResult<VehicleVerificationModel>` | [VehicleDocumentsController.cs#L61](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/VehicleDocumentsController.cs#L61) |
| `GET` | `/api/users/profile/vehicles/{vehicleId:guid}/documents/files/{documentId:guid}` | `VehicleDocumentsController.GetFileAsync` | `Authenticated` | Guid vehicleId, Guid documentId | `IActionResult` | [VehicleDocumentsController.cs#L69](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/VehicleDocumentsController.cs#L69) |
| `GET` | `/api/vehicle-verifications` | `VehicleVerificationsController.GetQueueAsync` | `Staff,Manager,Admin` | [FromQuery] string status = null | `ApiResult<List<VehicleVerificationReviewModel>>` | [VehicleVerificationsController.cs#L24](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/VehicleVerificationsController.cs#L24) |
| `GET` | `/api/vehicle-verifications/{verificationId:guid}` | `VehicleVerificationsController.GetAsync` | `Staff,Manager,Admin` | Guid verificationId | `ApiResult<VehicleVerificationReviewModel>` | [VehicleVerificationsController.cs#L30](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/VehicleVerificationsController.cs#L30) |
| `GET` | `/api/vehicle-verifications/{verificationId:guid}/files/{documentId:guid}` | `VehicleVerificationsController.GetFileAsync` | `Staff,Manager,Admin` | Guid verificationId, Guid documentId | `IActionResult` | [VehicleVerificationsController.cs#L36](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/VehicleVerificationsController.cs#L36) |
| `POST` | `/api/vehicle-verifications/{verificationId:guid}/review` | `VehicleVerificationsController.ReviewAsync` | `Staff,Manager,Admin` | Guid verificationId, [FromBody] ReviewVehicleVerificationModel model | `ApiResult<VehicleVerificationReviewModel>` | [VehicleVerificationsController.cs#L45](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/VehicleVerificationsController.cs#L45) |

### Module 8: Payments & Customer Wallet (11 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/api/payments/{id:long}` | `PaymentsController.GetAsync` | `Authenticated` | long id | `ApiResult<PaymentResponseModel>` | [PaymentsController.cs#L30](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/PaymentsController.cs#L30) |
| `POST` | `/api/payments/{id:long}/check` | `PaymentsController.CheckAsync` | `Authenticated` | long id | `ApiResult<PaymentCallbackResultModel>` | [PaymentsController.cs#L38](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/PaymentsController.cs#L38) |
| `GET` | `/api/payments/simulation` | `PaymentsController.GetSimulation` | `[AllowAnonymous` | None | `ApiResult<PaymentSimulationModel>` | [PaymentsController.cs#L46](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/PaymentsController.cs#L46) |
| `POST` | `/api/payments/{id:long}/simulate-success` | `PaymentsController.SimulateSuccessAsync` | `Authenticated` | long id | `ApiResult<PaymentCallbackResultModel>` | [PaymentsController.cs#L57](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/PaymentsController.cs#L57) |
| `GET` | `/api/payments/vnpay/return` | `PaymentsController.VnPayReturnAsync` | `[AllowAnonymous` | None | `ApiResult<PaymentCallbackResultModel>` | [PaymentsController.cs#L65](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/PaymentsController.cs#L65) |
| `GET` | `/api/payments/vnpay/ipn` | `PaymentsController.VnPayIpnAsync` | `[AllowAnonymous` | None | `IActionResult` | [PaymentsController.cs#L74](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/PaymentsController.cs#L74) |
| `GET` | `/api/payments/momo/return` | `PaymentsController.MoMoReturnAsync` | `[AllowAnonymous` | None | `ApiResult<PaymentCallbackResultModel>` | [PaymentsController.cs#L103](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/PaymentsController.cs#L103) |
| `POST` | `/api/payments/momo/ipn` | `PaymentsController.MoMoIpnAsync` | `[AllowAnonymous` | [FromBody] Dictionary<string, JsonElement> body | `NoContent (204)` | [PaymentsController.cs#L112](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/PaymentsController.cs#L112) |
| `GET` | `/api/wallet` | `WalletController.GetAsync` | `Authenticated` | None | `ApiResult<WalletResponseModel>` | [WalletController.cs#L19](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/WalletController.cs#L19) |
| `POST` | `/api/wallet/topup` | `WalletController.TopUpAsync` | `Authenticated` | [FromBody] TopUpWalletModel model | `ApiResult<PaymentResponseModel>` | [WalletController.cs#L27](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/WalletController.cs#L27) |
| `POST` | `/api/wallet/withdraw` | `WalletController.WithdrawAsync` | `Authenticated` | [FromBody] WithdrawWalletModel model | `ApiResult<WalletResponseModel>` | [WalletController.cs#L38](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/WalletController.cs#L38) |

### Module 9: Revenue Reporting (2 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/api/admin/reports/revenue` | `AdminRevenueController.GetRevenueAsync` | `Admin` | [FromQuery] string startDate = null, [FromQuery] string endDate = null, [FromQuery] int? parkingLotId = null | `ApiResult<AdminRevenueReportResponse>` | [AdminRevenueController.cs#L19](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminRevenueController.cs#L19) |
| `GET` | `/api/manager/reports/revenue` | `ManagerRevenueController.GetRevenueAsync` | `Manager` | [FromQuery] string startDate = null, [FromQuery] string endDate = null, [FromQuery] int? parkingLotId = null | `ApiResult<ManagerRevenueReportResponse>` | [ManagerRevenueController.cs#L21](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/ManagerRevenueController.cs#L21) |

### Module 10: Admin User & System Management (8 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `POST` | `/api/admin/managers` | `AdminController.CreateManagerAsync` | `Admin` | [FromBody] CreateEmployeeAccountModel model | `ApiResult<CreateUserResponseModel>` | [AdminController.cs#L22](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminController.cs#L22) |
| `GET` | `/api/admin/managers` | `AdminController.GetManagersAsync` | `Admin` | [FromQuery] bool? isActive = null, [FromQuery] string searchTerm = null | `ApiResult<IEnumerable<EmployeeAccountResponseModel>>` | [AdminController.cs#L35](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminController.cs#L35) |
| `GET` | `/api/admin/managers/{id}` | `AdminController.GetManagerAsync` | `Admin` | string id | `ApiResult<EmployeeAccountResponseModel>` | [AdminController.cs#L42](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminController.cs#L42) |
| `PUT` | `/api/admin/managers/{id}` | `AdminController.UpdateManagerAsync` | `Admin` | string id, [FromBody] UpdateEmployeeAccountModel model | `ApiResult<EmployeeAccountResponseModel>` | [AdminController.cs#L48](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminController.cs#L48) |
| `PATCH` | `/api/admin/managers/{id}/status` | `AdminController.UpdateManagerStatusAsync` | `Admin` | string id, [FromBody] UpdateUserStatusModel model | `ApiResult<EmployeeAccountResponseModel>` | [AdminController.cs#L54](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminController.cs#L54) |
| `PUT` | `/api/admin/managers/{id}/lots` | `AdminController.AssignManagerLotsAsync` | `Admin` | string id, [FromBody] AssignParkingLotsModel model | `ApiResult<EmployeeAccountResponseModel>` | [AdminController.cs#L62](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminController.cs#L62) |
| `GET` | `/api/admin/users` | `AdminController.GetInternalUsersAsync` | `Admin` | [FromQuery] string role = null, [FromQuery] bool? isActive = null, [FromQuery] string searchTerm = null | `ApiResult<IEnumerable<UserResponseModel>>` | [AdminController.cs#L69](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminController.cs#L69) |
| `PATCH` | `/api/admin/users/{id}/status` | `AdminController.UpdateStatusAsync` | `Admin` | string id, [FromBody] UpdateUserStatusModel model | `ApiResult<bool>` | [AdminController.cs#L79](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminController.cs#L79) |

### Module 11: Admin Simulation & Background Jobs (2 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/api/admin/simulation` | `AdminSimulationController.GetAsync` | `Admin` | None | `ApiResult<ParkingSimulationResponseModel>` | [AdminSimulationController.cs#L19](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminSimulationController.cs#L19) |
| `PUT` | `/api/admin/simulation` | `AdminSimulationController.UpdateAsync` | `Admin` | [FromBody] UpdateParkingSimulationModel model | `ApiResult<ParkingSimulationResponseModel>` | [AdminSimulationController.cs#L23](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/AdminSimulationController.cs#L23) |

### Module 12: Notifications & SignalR Realtime Hubs (5 Endpoints)

| Verb | Route Path | Controller.Action | Auth Requirement | Parameters | Response Type | Source |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| `GET` | `/api/notifications` | `NotificationsController.GetMineAsync` | `User` | [FromQuery] bool unreadOnly = false, [FromQuery] int take = 30 | `ApiResult<NotificationListModel>` | [NotificationsController.cs#L20](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/NotificationsController.cs#L20) |
| `GET` | `/api/notifications/unread-count` | `NotificationsController.GetUnreadCountAsync` | `User` | None | `ApiResult<NotificationListModel>` | [NotificationsController.cs#L25](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/NotificationsController.cs#L25) |
| `POST` | `/api/notifications/{id:long}/read` | `NotificationsController.MarkReadAsync` | `User` | long id | `ApiResult<NotificationModel>` | [NotificationsController.cs#L30](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/NotificationsController.cs#L30) |
| `POST` | `/api/notifications/read-all` | `NotificationsController.MarkAllReadAsync` | `User` | None | `ApiResult<NotificationModel>` | [NotificationsController.cs#L35](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/NotificationsController.cs#L35) |
| `DELETE` | `/api/notifications/{id:long}` | `NotificationsController.DeleteAsync` | `User` | long id | `ApiResult<object>` | [NotificationsController.cs#L40](file:///C:/DEV/OJT-FSA-Team3/parking-management-system/src/PMS.API/Controllers/NotificationsController.cs#L40) |

### SignalR Realtime Hubs (3 Hubs)

| Hub Name | Route Endpoint | Authorization | Broadcast Events / Methods | Description | Source File |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **`LotHub`** | `/api/hubs/lots` | Anonymous / Public | `LotCapacityUpdated`, `SlotStatusChanged` | Realtime public slot availability and occupancy changes. | `src/PMS.API/Hubs/LotHub.cs` |
| **`ParkingOpsHub`** | `/api/hubs/parking-ops` | `Staff, Manager, Admin` | `GateRequestCreated`, `SessionUpdated`, `BarrierTriggered`, `PaymentReceived` | Lane gate entry/exit triggers, QR scan events, barrier commands. | `src/PMS.API/Hubs/ParkingOpsHub.cs` |
| **`VehicleVerificationHub`** | `/api/hubs/vehicle-verifications` | `Staff, Admin` | `VerificationSubmitted`, `VerificationReviewed` | Live document submission queue notifications for operators. | `src/PMS.API/Hubs/VehicleVerificationHub.cs` |

---

## 5. Authentication and Authorization Findings

### 5.1 Role-Based Access Control (RBAC) Matrix

| Functional Area | Anonymous | User (Customer) | Staff (Operator) | Manager | Admin |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Account Auth (Login, Register, OTP, 2FA)** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Profile & Vehicle Management** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Browse Public Lots & 2D Layouts** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Book & Cancel Reservations** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Customer Parking Sessions & Payments** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Customer Wallet (Topup, Withdraw, Balance)** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Gate Check-In & Check-Out Operations** | ❌ | ❌ | ✅ | ❌ | ❌ |
| **Cash Payment Collection at Gate** | ❌ | ❌ | ✅ | ❌ | ❌ |
| **Lost Ticket Reporting (Gate)** | ❌ | ❌ | ✅ | ❌ | ❌ |
| **Lost Ticket Approval / Rejection** | ❌ | ❌ | ❌ | ✅ | ❌ |
| **Abnormal Session Forced Close** | ❌ | ❌ | ❌ | ✅ | ❌ |
| **Lot Pricing Rule Configuration** | ❌ | ❌ | ❌ | ✅ (Assigned Lots) | ❌ |
| **Staff Account Management** | ❌ | ❌ | ❌ | ✅ (Assigned Lots) | ❌ |
| **Manager Revenue Reporting** | ❌ | ❌ | ❌ | ✅ (Assigned Lots) | ❌ |
| **Admin Revenue Reporting** | ❌ | ❌ | ❌ | ❌ | ✅ (All Lots) |
| **Manager Account CRUD** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Global User Status (Deactivate/Activate)** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Vehicle Document Review & Approval** | ❌ | ❌ | ✅ | ❌ | ✅ |
| **Simulation Mode Toggle** | ❌ | ❌ | ❌ | ❌ | ✅ |

### 5.2 Special Claim Enforcement (`2FA_Challenge`)

When a user enables Two-Factor Authentication (2FA), the login endpoint (`POST /api/Users/authenticate` or `POST /api/Users/login-otp/verify`) does not issue a standard session JWT. Instead, it generates a restricted intermediate challenge token:
```json
{
  "sub": "b2f6b4b4-...",
  "email": "user@example.com",
  "purpose": "2FA_Challenge",
  "exp": 1730248800
}
```
- This token is only permitted at `POST /api/Users/two-factor` (to submit the TOTP/SMS code) and `POST /api/Users/two-factor/resend`.
- All other endpoints validate whether `purpose == "2FA_Challenge"`. If true, the server aborts the request immediately with **HTTP 403 Forbidden**, preventing unverified session access.

### 5.3 SignalR WebSocket Handshake Authentication

SignalR connections to protected hubs (`ParkingOpsHub` and `VehicleVerificationHub`) cannot send standard HTTP headers during the browser WebSocket upgrade. The backend configures `JwtBearerOptions.Events.OnMessageReceived` in `Program.cs`:
- It extracts the token from the query string: `context.Request.Query["access_token"]`.
- If the request path begins with `/api/hubs/`, the query parameter is assigned to `context.Token`, allowing seamless authentication across WebSocket protocols.

---

## 6. Request/Response Contract Findings

### 6.1 Standard Pagination Model

List endpoints that support pagination conform to the following uniform request and response contracts:
- **Request Query Parameters:**
  - `pageNumber` (integer, 1-indexed, default: 1)
  - `pageSize` (integer, default: 10, max: 100)
  - `searchTerm` / `keyword` (string, optional)
- **Response Model (`PaginationResponseModel<T>`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "items": [ ... ],
      "pageNumber": 1,
      "pageSize": 10,
      "totalCount": 42,
      "totalPages": 5,
      "hasPreviousPage": false,
      "hasNextPage": true
    },
    "errors": []
  }
  ```

### 6.2 Date and Time Serialization Standards

1. **Timestamps:** Serialized as ISO-8601 UTC strings (`YYYY-MM-DDTHH:mm:ss.sssZ`).
2. **TimeOfDay Fields:** Serialized via custom JSON converters as 24-hour strings (`HH:mm:ss`) to support .NET `TimeOnly` (e.g., pricing rule Day/Night split boundaries).
3. **Date Filters:** Query parameters for date ranges accept `from` and `to` in ISO-8601 format (`YYYY-MM-DD` or `YYYY-MM-DDTHH:mm:ssZ`).

---

## 7. Frontend API Client and Service Structure

### 7.1 `axiosClient.ts` Architecture

The frontend communicates with the backend through a centralized Axios client (`src/api/axiosClient.ts`):
- **Base URL Resolution:**
  - Resolved via `DEFAULT_API_BASE_URL` (`https://localhost:44391/api`).
  - Supports developer override in `localStorage` via key `pms_custom_api_url`.
  - Normalizes Swagger URLs (e.g., stripping `/swagger/index.html` to ensure it ends with `/api`).
- **Request Interceptor:**
  - Generates and attaches `X-Correlation-Id` (UUID v4) for distributed tracing.
  - Retrieves the JWT from session storage and injects `Authorization: Bearer <token>`.
- **Response Interceptor:**
  - Automatically unpacks `ApiResult<T>`: returns `response.data.result` if `succeeded === true`.
  - On HTTP 401: Clears authentication tokens and redirects the user to the login screen.
  - Transforms backend error envelopes into structured `ApiError` objects using `toApiError(err)`.

### 7.2 Service Path Registry (`env.ts`)

The frontend abstracts route prefixes in `SERVICE_PATHS` (`src/config/env.ts`):
```typescript
export const SERVICE_PATHS: Record<ServiceKey, string> = {
  auth: '/Users',
  profile: '/Users/profile',
  staff: '/admin',
  team: '/manager',
  parking: '',
  capacity: '/v1/parking',
  reports: '/v1/reports',
  booking: '',
  gate: '/staff',
}
```

### 7.3 Mock Fallback Mechanism (`isMockService`)

- Controlled by the environment variable `VITE_MOCK_SERVICES`.
- Certain service methods check `if (isMockService(...))` before making HTTP calls.
- **Mocked Domains:**
  - `capacity`: Mocked via `capacityMock` because the backend does not expose `/v1/parking/lots/capacity`.
  - `reports`: Legacy timeseries and breakdown mock fallback.
  - `pricing`: Manager pricing demo mode.
  - `profile` / `vehicleVerification`: Verification review mock fallback.

---

## 8. Backend-to-Frontend API Mapping

### 8.1 Summary Statistics

- **Total Backend Endpoints:** 150
- **Total Frontend HTTP Calls:** 129
- **Matched Frontend Calls:** 121 (93.8%)
- **Unique Backend Endpoints Consumed by Frontend:** 114
- **Unconsumed Backend Endpoints:** 36
- **Unmatched Frontend Calls (Gaps / Mismatches):** 8

### 8.2 Inventory of Unconsumed Backend Endpoints (36 Endpoints)

The following 36 backend endpoints do not have direct consumer calls in the current frontend client. They are classified by technical reason:

| Backend Endpoint | Controller.Action | Reason for No Frontend Consumer |
| :--- | :--- | :--- |
| `GET /api/admin/managers` | `AdminController.GetManagersAsync` | Frontend uses `identityService.getUsers()` with role filtering instead. |
| `PATCH /api/admin/managers/{id}/status` | `AdminController.UpdateManagerStatusAsync` | Frontend uses unified `identityService.updateUserStatus`. |
| `GET /api/admin/reports/revenue` | `AdminRevenueController.GetRevenueAsync` | Frontend currently connects Manager revenue; Admin revenue screen pending integration. |
| `GET /api/lots/{id:int}` | `LotsController.GetByIdAsync` | Frontend loads all lots via `GET /api/lots` and filters in-memory. |
| `GET /api/manager/lots` | `ManagerController.GetLotsAsync` | Frontend reuses `GET /api/lots` with client-side manager lot filtering. |
| `GET /api/manager/staff/{id}` | `ManagerController.GetStaffByIdAsync` | Frontend retrieves staff in list form via `GET /api/manager/staff`. |
| `GET /api/manager/lots/{lotId:int}/reservations` | `ManagerOperationsController.GetReservationsAsync` | Frontend uses staff gate reservation lookup. |
| `POST /api/manager/reservations/{id:long}/cancel` | `ManagerOperationsController.CancelReservationAsync` | Frontend allows customers to cancel their own reservations via `/reservations/{id}/cancel`. |
| `POST /api/manager/sessions/{id:long}/abnormal-close` | `ManagerOperationsController.CloseAbnormallyAsync` | Advanced manager manual override endpoint; UI action not yet wired to button. |
| `GET /api/manager/lots/{lotId:int}/plate-lists` | `ManagerOperationsController.GetPlateListsAsync` | Blacklist/whitelist management UI pending implementation. |
| `POST /api/manager/lots/{lotId:int}/plate-lists` | `ManagerOperationsController.AddPlateListEntryAsync` | Blacklist/whitelist management UI pending implementation. |
| `DELETE /api/manager/lots/{lotId:int}/plate-lists/{id:int}` | `ManagerOperationsController.RemovePlateListEntryAsync` | Blacklist/whitelist management UI pending implementation. |
| `GET /api/manager/lots/{lotId:int}/pricing-rules` | `ManagerPricingController.GetPricingRulesAsync` | Frontend pricing rules service calls plural form. |
| `GET /api/manager/lots/{lotId:int}/pricing-rules/{id:int}` | `ManagerPricingController.GetPricingRuleByIdAsync` | Frontend loads full pricing list and selects locally. |
| `GET /api/parking-sessions/mine` | `ParkingSessionsController.GetMineAsync` | Customer session history accessed via `GET /api/parking-sessions/history`. |
| `GET /api/parking-sessions/{id:long}/quote` | `ParkingSessionsController.QuoteAsync` | Frontend uses gate staff quote endpoint `POST /staff/sessions/{id}/quote`. |
| `GET /api/payments/vnpay/return` | `PaymentsController.VnPayReturnAsync` | VNPay browser redirect return handler (handled by payment gateway return redirect). |
| `GET /api/payments/vnpay/ipn` | `PaymentsController.VnPayIpnAsync` | VNPay server-to-server asynchronous IPN webhook callback. |
| `GET /api/payments/momo/return` | `PaymentsController.MoMoReturnAsync` | MoMo browser redirect return handler. |
| `POST /api/payments/momo/ipn` | `PaymentsController.MoMoIpnAsync` | MoMo server-to-server asynchronous IPN webhook callback. |
| `GET /api/zones/{zoneId:int}/slots` | `SlotsController.GetByZoneIdAsync` | Frontend retrieves slots in bulk via visual layout API (`/public/lots/{id}/layout`). |
| `GET /api/slots/{id:int}` | `SlotsController.GetByIdAsync` | Slots are manipulated through visual layout designer. |
| `PUT /api/slots/{id:int}` | `SlotsController.UpdateAsync` | Direct slot editing bypassed in favor of template-based layout saving. |
| `PATCH /api/slots/{id:int}/status` | `SlotsController.UpdateStatusAsync` | Slot statuses are toggled via layout editor. |
| `GET /api/staff/lots/{lotId:int}/reservations` | `StaffGateController.GetReservationsAsync` | Staff looks up reservations via plate lookup endpoint. |
| `GET /api/staff/lots/{lotId:int}/plate-lists` | `StaffGateController.GetPlateListsAsync` | Gate plate blacklist query endpoint (automatic barrier gate check). |
| `GET /api/staff/sessions/{id:long}` | `StaffGateController.GetSessionAsync` | Staff loads sessions through list query `GET /api/staff/lots/{lotId}/sessions`. |
| `GET /api/Users/{id}` | `UsersController.GetByIdAsync` | Admin user management uses list retrieval. |
| `POST /api/Users/internal` | `UsersController.CreateInternalUserAsync` | Frontend uses role-specific `POST /admin/managers` and `POST /manager/staff`. |
| `PATCH /api/Users/{id}/status` | `UsersController.UpdateStatusAsync` | Frontend calls `PATCH /admin/users/{id}/status`. |
| `GET /api/Users/{id}/profile` | `UsersController.GetProfileByUserIdAsync` | User profile accessed via authenticated session `GET /Users/profile`. |
| `GET /api/users/profile/vehicles/.../files/{id}` | `VehicleDocumentsController.GetFileAsync` | Direct raw binary file stream (accessed via browser image/document viewer). |
| `GET /api/vehicle-verifications/.../files/{id}` | `VehicleVerificationsController.GetFileAsync` | Direct raw binary file stream for staff document verification. |
| `GET /api/lots/{lotId:int}/zones` | `ZonesController.GetByLotIdAsync` | Zones loaded as part of 2D layout model. |
| `GET /api/zones/{id:int}` | `ZonesController.GetByIdAsync` | Direct zone inspect bypassed in favor of layout model. |
| `PUT /api/zones/{id:int}` | `ZonesController.UpdateAsync` | Direct zone update bypassed in favor of layout model. |

---

## 9. Contract Mismatches and Unresolved Questions

Our audit identified 5 critical contract discrepancies between Frontend service expectations and Backend controller implementations:

### 1. Wallet Transaction History Endpoint Missing in Backend
- **Frontend Expectation:** In `src/api/wallet.service.ts`, `walletService.getTransactions()` makes an HTTP call:
  ```typescript
  getTransactions(): Promise<WalletTransactionDTO[]> {
    return http.get<WalletTransactionDTO[]>('/wallet/transactions')
  }
  ```
- **Backend Reality:** `WalletController.cs` does **NOT** expose `/api/wallet/transactions`. It only exposes:
  1. `GET /api/wallet`
  2. `POST /api/wallet/topup`
  3. `POST /api/wallet/withdraw`
- **Impact & Evidence:** In the backend, transaction history is returned inside `GET /api/wallet` under the property `WalletResponseModel.RecentTransactions`. Any direct invocation of `GET /api/wallet/transactions` in a live environment results in **HTTP 404 Not Found**.
- **Recommendation:** Either add `[HttpGet("transactions")]` to `WalletController.cs` or update `wallet.service.ts` to read from `GET /api/wallet`.

### 2. Parking Capacity Endpoint Missing in Backend
- **Frontend Expectation:** `src/api/parking.service.ts` calls `GET /v1/parking/lots/capacity` via `base = SERVICE_PATHS.capacity`.
- **Backend Reality:** The backend has no `/v1/parking/lots/capacity` route. Capacity updates are broadcast dynamically over SignalR `LotHub` (`/api/hubs/lots`) using event `LotCapacityUpdated`.
- **Impact & Evidence:** The frontend silently avoids a 404 only because `isMockService('capacity')` is active by default in development, routing to `capacityMock`.
- **Recommendation:** Document that live capacity is driven by SignalR `LotHub`, or implement a REST fallback endpoint in `LotsController.cs`.

### 3. Revenue Reporting Route Migration Discrepancy
- **Frontend Expectation:** `src/api/report.service.ts` contains 5 legacy methods calling:
  - `GET /v1/reports/revenue/kpi`
  - `GET /v1/reports/revenue/timeseries`
  - `GET /v1/reports/revenue/payment-methods`
  - `GET /v1/reports/revenue/by-lot`
  - `GET /v1/reports/revenue/breakdown`
- **Backend Reality:** The backend implemented role-separated revenue endpoints:
  - `GET /api/admin/reports/revenue` (`AdminRevenueController`)
  - `GET /api/manager/reports/revenue` (`ManagerRevenueController`)
- **Impact & Evidence:** The frontend has wired `getManagerRevenue()` to `GET /manager/reports/revenue`, but the remaining 5 legacy methods fall back to `revenueMock`.
- **Recommendation:** Phase 2 API documentation must explicitly document the approved role-separated endpoints and mark the legacy `/v1/reports/revenue/*` sub-paths as deprecated.

### 4. Payment Return Callback Discrepancy
- **Frontend Expectation:** `payment.service.ts` defines `getReturn(provider)` calling `GET /payments/{provider}/return`.
- **Backend Reality:** `PaymentsController.cs` implements provider-specific routes:
  - `GET /api/payments/vnpay/return`
  - `GET /api/payments/momo/return`
- **Impact & Evidence:** If `provider` is passed as `vnpay` or `momo`, this matches. However, the backend return endpoints return an HTML redirect or gateway response model, whereas the frontend expects a generic `PaymentDTO`.
- **Recommendation:** Clarify the exact flow for return handling vs. server-side IPN verification in the API guide.

### 5. Layout Public Routes vs Controller Base Path
- **Backend Implementation:** `LayoutsController.cs` is declared with `[Route("api/public/lots/{lotId:int}/layout")]`, but also contains `[HttpGet("/api/lot-templates")]`.
- **Frontend Implementation:** `layout.service.ts` calls `${SERVICE_PATHS.parking}/lot-templates` and `/public/lots/{lotId}/layout`.
- **Observation:** Route resolution succeeds because the frontend uses `DEFAULT_API_BASE_URL` ending in `/api`. This exact path structure must be documented clearly so third-party consumers understand the template route bypasses the lot ID.

---

## 10. Suggested API Documentation Outline (Phase 2 Blueprint)

When authorized to generate the full API documentation in Phase 2, the document should be structured into the following standard sections:

1. **Title, Scope, and System Architecture**
   - Base URLs (Development, Staging, Production).
   - Envelope pattern (`ApiResult<T>`) with concrete JSON examples.
   - HTTP status code conventions (200, 201, 204, 400, 401, 403, 404, 409, 422, 429, 500).
   - Rate limiting headers and throttle thresholds.

2. **Authentication, Authorization & Security Guide**
   - JWT acquisition (`POST /api/Users/authenticate`, `POST /api/Users/login-otp/verify`, `POST /api/Users/login-google`).
   - Token lifecycle, refresh, and expiration.
   - Two-Factor Authentication (2FA) verification sequence with `purpose=2FA_Challenge` token.
   - Role-Based Access Control matrix (Admin, Manager, Staff, User).

3. **Realtime Communication Guide (SignalR Hubs)**
   - WebSocket connection URLs (`/api/hubs/lots`, `/api/hubs/parking-ops`, `/api/hubs/vehicle-verifications`).
   - Query parameter authentication via `?access_token=...`.
   - Complete list of client-side event listeners and sample JSON payloads.

4. **Module-by-Module API Reference (12 Modules)**
   - For every endpoint:
     - Verb, Path, and Action Name.
     - Permission & Role requirements.
     - Request Headers (e.g. `Authorization`, `X-Correlation-Id`, `Content-Type`).
     - Path, Query, and Request Body schemas with data types, nullability, and validation rules.
     - Success response payload sample (`succeeded: true`).
     - Error response payload sample (`succeeded: false`) with realistic failure scenarios.

5. **End-to-End Business Workflows**
   - Workflow 1: Customer Reservation $\rightarrow$ Payment $\rightarrow$ Gate Arrival.
   - Workflow 2: Staff Gate Check-In $\rightarrow$ Occupancy Tracking $\rightarrow$ Check-Out $\rightarrow$ Payment Collection.
   - Workflow 3: Customer Vehicle Paper Upload $\rightarrow$ Operator Verification $\rightarrow$ Approval.
   - Workflow 4: Manager Lot Pricing Configuration $\rightarrow$ Hourly Calculation Simulation.
   - Workflow 5: Financial Revenue Reporting Reconciliation.

6. **OpenAPI / Swagger and Postman Artifacts**
   - Generation of an updated OpenAPI 3.0 JSON specification.
   - Export of a standardized Postman Collection v2.1 with pre-configured environment variables and test scripts.

---

## 11. Source File Paths and Line Numbers Supporting Key Findings

| Finding / Component | Repository | Source File Path | Line Range |
| :--- | :--- | :--- | :--- |
| `ApiController` Base Route | Backend | `src/PMS.API/Controllers/ApiController.cs` | L1-L15 |
| `ApiResult<T>` Envelope Model | Backend | `src/PMS.Application/Models/ApiResult.cs` | L1-L40 |
| Validation Model Filter | Backend | `src/PMS.API/Filters/ValidateModelAttribute.cs` | L1-L35 |
| Rate Limiting Middleware | Backend | `src/PMS.API/Middlewares/RateLimitingMiddleware.cs` | L1-L60 |
| Exception Handling Middleware | Backend | `src/PMS.API/Middlewares/ExceptionHandlingMiddleware.cs` | L1-L85 |
| JWT Token Generation & Claims | Backend | `src/PMS.Application/Helpers/JwtHelper.cs` | L1-L90 |
| SignalR Hub Handshake Auth | Backend | `src/PMS.API/Program.cs` | L80-L105 |
| Admin Revenue Controller | Backend | `src/PMS.API/Controllers/AdminRevenueController.cs` | L1-L45 |
| Manager Revenue Controller | Backend | `src/PMS.API/Controllers/ManagerRevenueController.cs` | L1-L50 |
| Staff Gate Operations | Backend | `src/PMS.API/Controllers/StaffGateController.cs` | L1-L240 |
| Vehicle Verification Operations | Backend | `src/PMS.API/Controllers/VehicleVerificationsController.cs` | L1-L80 |
| Centralized Axios Client | Frontend | `src/api/axiosClient.ts` | L1-L110 |
| Frontend Service Paths Registry | Frontend | `src/config/env.ts` | L1-L75 |
| Wallet Service (404 Discrepancy) | Frontend | `src/api/wallet.service.ts` | L1-L35 |
| Parking Capacity (Mock Discrepancy)| Frontend | `src/api/parking.service.ts` | L1-L45 |
| Revenue Reporting Service | Frontend | `src/api/report.service.ts` | L1-L90 |

---

## 12. Test Evidence and Limitations of Scan

### 12.1 Automated Test Evidence

A full discovery sweep was performed against the solution test assemblies:
- **Test Projects:**
  - `tests/PMS.UnitTests`
  - `tests/PMS.Api.IntegrationTests`
- **Discovered Test Count:** **922 individual tests**.
- **Coverage Areas:** User authentication, password rules, vehicle plate syntax, document upload constraints, parking zone slot math, session quote calculation, manager lot pricing versioning, and revenue reporting date windowing.

### 12.2 Scan Limitations

1. **Static Analysis Boundary:** The audit is grounded in source code AST inspection and build compilation. External runtime dependencies (e.g., live SMS OTP delivery, live VNPay/MoMo gateway callbacks in production) require sandbox/staging runtime verification.
2. **Third-Party Payment Gateways:** VNPay and MoMo IPN callbacks are server-to-server HTTP POST requests initiated by external banking infrastructure, which cannot be triggered solely from local browser UI tests without network tunnels (e.g. ngrok).

---

## 13. Git Status Before and After Scan

To ensure documentation integrity and respect the git boundary, repository status was recorded before and after all inspection actions:

### Git Status: Backend Repository (`develop`)
```text
On branch develop
Your branch is up to date with 'origin/develop'.

nothing to commit, working tree clean
```

### Git Status: Frontend Repository (`develop`)
```text
On branch develop
Your branch is up to date with 'origin/develop'.

nothing to commit, working tree clean
```

### Git Status: Report Repository (`main`)
```text
On branch main
Your branch is up to date with 'origin/main'.

Changes not staged for commit:
	deleted:    reports/parking-system/parking-system-backend-api-guide-pre-push.docx

Untracked files:
	reports/parking-system/api-documentation-project-scan.md
	reports/parking-system/api-guide-source-conformance-audit.md
	reports/parking-system/parking-system-backend-api-guide-pre-push v2.docx

no changes added to commit (use "git add" and/or "git commit -a")
```

---

## 14. Confirmation of Zero Code Changes and Zero Git Commit/Push

I hereby certify that:
1. **Zero Code Modifications:** Not a single line of application source code, configuration file, database migration, or test script was added, modified, renamed, or deleted in either `parking-management-system` or `parking-management-system-frontend`.
2. **Zero Git Mutations:** No `git commit`, `git push`, `git merge`, `git rebase`, `git checkout`, or branch modification commands were executed in any repository.
3. **Report Readiness:** This scan report stands ready for user and team lead review. Phase 2 (full API documentation generation) will not begin until explicit authorization is granted.
