# XXIV Parking Management System — Backend API Open Questions & Architectural Backlog

> **Document Target:** Backend Engineering Team, Tech Lead, Product Owner  
> **Source Baseline:** `c:\DEV\OJT-FSA-Team3\parking-management-system` (.NET 9 Web API)  
> **Output Date:** 2026-10-30  
> **Status:** Open for Architecture Review  
> **Classification:** Technical Discovery & Specification Clarification  

---

## 1. Executive Summary

During the comprehensive, implementation-level reconstruction of the Parking Management System (PMS) Backend API documentation, the source code was treated as the single source of truth. Wherever implementation details were ambiguous, intentionally decoupled, managed outside version control, or represented architectural discrepancies between code and business expectations, they were marked as `TBD - needs verification` in the API specification.

This document compiles all **10 high-priority open questions**, **domain inconsistencies**, and **recommended API improvements** requiring decisions from the backend owner and stakeholders.

---

## 2. Open Questions & Items Requiring Backend Owner Decision

### Q-01: Deployment Environments, Base URLs, and External Swagger Access
- **Source Finding:** In `launchSettings.json` and `appsettings.json`, only local development hosts (`https://localhost:5001`, `http://localhost:5000`, `https://localhost:44391`, `http://localhost:54180`) are committed. Staging and production URLs are not defined in codebase configuration.
- **Current Documentation Impact:** Section 1.5 lists Staging and Production hostnames as `TBD - needs verification`.
- **Questions for Backend Lead:**
  1. What are the official staging and production domain names (e.g., `https://staging-api.xxivparking.com` / `https://api.xxivparking.com`)?
  2. Will Swagger UI (`/swagger`) be publicly accessible, restricted via basic authentication, or completely disabled in production environments (`app.Environment.IsProduction()`)?

---

### Q-02: Self-Service Two-Factor Authentication (2FA) Lifecycle
- **Source Finding:** The API provides endpoints to verify Email OTP challenges (`POST /api/Users/two-factor`) and resend challenge emails (`POST /api/Users/two-factor/resend`). However, there is **no REST endpoint** allowing a user or administrator to enable, disable, or verify the initial activation of 2FA. The `TwoFactorEnabled` property on `AspNetUsers` is modified only through seeders or direct database access.
- **Current Documentation Impact:** Documented as an administrative/database pre-configuration in Section 3.1.
- **Questions for Backend Lead:**
  1. Should a self-service toggle endpoint (e.g., `POST /api/users/profile/2fa/toggle` or `POST /api/users/profile/2fa/enable` requiring password confirmation) be added to the roadmap?
  2. Is 2FA intended to be mandatory for all internal roles (`Staff`, `Manager`, `Admin`) while optional for `User`?

---

### Q-03: `Tariff.GraceMinutes` Database Column vs. Active Team Policy
- **Source Finding:** The domain model and migration history include a `GraceMinutes` property (defaulting to 15 in database entities). However, `FeeCalculator.cs:17-21` contains an explicit runtime override:
  ```csharp
  // Team amendment: no free grace period. The first block is owed immediately upon entry.
  ```
  The runtime calculator hard-pins grace time to zero and assesses the first pricing block immediately upon vehicle entry.
  Furthermore, `Tariff.GraceMinutes` is preserved in DTOs and API responses.
- **Current Documentation Impact:** Documented in Section 6.1 and Section 7.4 as preserved in schema but overridden at runtime.
- **Questions for Product Owner:**
  1. Is the "no free grace period" rule permanent across all facility tiers, or will certain lots/zones offer a complimentary drop-off window in the future?
  2. If permanent, should `GraceMinutes` be formally deprecated and scheduled for schema removal, or kept dormant for contract flexibility?

---

### Q-04: Stateless JWT Token Revocation & Password Reset Invalidation
- **Source Finding:** JWT tokens are issued with a fixed **7-day lifetime** (`DateTime.UtcNow.AddDays(7)`, `JwtHelper.cs:40`). The backend is completely stateless: it does not validate a `SecurityStamp` on incoming bearer tokens, and there is no token blacklist. If a user resets their password (`POST /api/Users/reset-password`), changes their password (`POST /api/Users/change-password`), or is deactivated by an administrator (`PATCH /api/admin/users/{id}/status`), all existing tokens remain cryptographically valid until their natural 7-day expiration.
- **Current Documentation Impact:** Explicitly highlighted in Section 3.1 as a security trade-off.
- **Questions for Backend Lead:**
  1. Is a 7-day stateless lifetime acceptable for mobile customer convenience, or should token lifetime be shortened (e.g., 15–60 minutes) combined with a secure refresh token mechanism?
  2. Should an in-memory or Redis-backed token revocation / `SecurityStamp` validation filter be implemented to immediately disconnect compromised or deactivated accounts?

---

### Q-05: Safety Controls for Simulated Payment Gateway
- **Source Finding:** `POST /api/payments/{id}/simulate-success` enables bypassing real banking rails and immediately transitions a payment to `Succeeded`. It is guarded by `Payments:AllowSimulatedPayments == true` in `appsettings.json`.
- **Current Documentation Impact:** Documented in Section 8.7 with explicit warnings regarding its testing-only nature.
- **Questions for Backend Lead:**
  1. Is there an automated CI/CD pipeline rule or startup assertion in `Program.cs` ensuring `AllowSimulatedPayments` can never evaluate to `true` when `ASPNETCORE_ENVIRONMENT == "Production"`?
  2. Should this endpoint be restricted to `[Authorize(Roles = "Admin")]` rather than callable by any customer owning the transaction?

---

### Q-06: File Upload Limits & Media Type Consistency
- **Source Finding:**
  - Avatar uploads (`POST /api/users/profile/avatar`) enforce a strict **2 MB** limit and accept only images (`image/jpeg`, `image/png`, `image/gif`, `image/webp`).
  - Vehicle KYC document uploads (`POST /api/users/profile/vehicles/{id}/documents`) enforce a **10 MB** limit and accept images and PDFs (`image/jpeg`, `image/png`, `application/pdf`).
  - Global Kestrel request body limit in `Program.cs` defaults to ASP.NET Core standard (30 MB).
- **Current Documentation Impact:** Documented accurately in Section 2.8 and Section 8.1.
- **Questions for Backend Lead:**
  1. Should mobile clients be advised to perform client-side image compression prior to uploading vehicle registration documents to minimize S3/MinIO/blob storage costs?
  2. Are thumbnail generation background jobs planned for uploaded vehicle documents to accelerate staff verification queue loading?

---

### Q-07: Google OAuth Multi-Client Audience Support
- **Source Finding:** `POST /api/Users/google-login` verifies incoming Google ID tokens via `GoogleJsonWebSignature.ValidateAsync` using a single configured client ID (`Authentication:Google:ClientId`).
- **Current Documentation Impact:** Documented in Section 3.2.
- **Questions for Mobile & Backend Leads:**
  1. Are the Android and iOS Flutter apps configured to exchange authorization codes with the Web Client ID as audience (`server_client_id`), or will mobile-specific Google Client IDs be added to the backend verification list?

---

### Q-08: SignalR Horizontal Scaling & Backplane Strategy
- **Source Finding:** All three SignalR hubs (`LotHub`, `ParkingOpsHub`, `VehicleVerificationHub`) are registered via standard in-memory SignalR (`builder.Services.AddSignalR()`). Hub groups (`lot-{id}`, `ops-lot-{id}`, `vehicle-verification-reviewers`) are maintained strictly within single-process memory.
- **Current Documentation Impact:** Documented in Section 9.
- **Questions for DevOps & Backend Lead:**
  1. When scaling out the backend behind an AWS ALB or Azure Load Balancer across multiple container replicas, is Redis backplane (`Microsoft.AspNetCore.SignalR.StackExchangeRedis`) or Azure SignalR Service planned?
  2. Until a backplane is deployed, will load balancers be configured with sticky sessions (cookie affinity)?

---

### Q-09: Production CORS Policy
- **Source Finding:** `Program.cs` defines a global CORS policy:
  ```csharp
  builder.Services.AddCors(options =>
  {
      options.AddDefaultPolicy(policy =>
      {
          policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader();
      });
  });
  ```
- **Current Documentation Impact:** Documented in Section 2.9 as wildcard origin policy.
- **Questions for Security Lead:**
  1. Prior to production release, what exact origins must be whitelisted (e.g., `https://portal.xxivparking.com`, `https://admin.xxivparking.com`) to prevent cross-site request forgery and unwanted embedding?

---

### Q-10: Hangfire Dashboard Security in Deployed Environments
- **Source Finding:** `app.MapHangfireDashboard("/hangfire")` is mapped in `Program.cs` without an explicit custom `IDashboardAuthorizationFilter`. By default, Hangfire restricts dashboard access strictly to local requests (`127.0.0.1` / `::1`).
- **Current Documentation Impact:** Documented in Section 1.3.
- **Questions for Backend Lead:**
  1. Will Hangfire dashboard access in staging/production be protected using JWT role claims (`Admin` role), basic authentication, or reverse proxy network IP filtering?

---

## 3. Discrepancies and Inconsistencies Discovered in Source Code

| Area | Code Reference | Issue Description | Recommended Resolution |
|:---|:---|:---|:---|
| **Class Prefix Redundancy** | `LotsController.cs:17-18` | Class is decorated with both `[Route("api/lots")]` and `[Route("api/parking-lots")]`. | Formally standardize on `api/lots` across documentation; document `api/parking-lots` as legacy alias. |
| **Method Route Alias** | `LotsController.cs:33-34` | Method `GetPublicLots` has routes `""` and `"/api/public/parking-lots"`. | Standardize on canonical route `/api/lots`; mark `/api/public/parking-lots` as secondary compatibility alias. |
| **Zone Route Alias** | `LotsController.cs:64-65` | Method `GetLotZones` has route `"{id:int}/zones"` and `"/api/parking-lots/{lotId:int}/zones"`. | Standardize on `/api/lots/{id}/zones`. |
| **Duplicate User Endpoints** | `AdminController.cs:70` vs `UsersController.cs:30` | `/api/admin/users` queries internal employee accounts, while `/api/Users` queries all accounts. | Keep clear distinction: `/api/admin/users` is the internal employee directory; `/api/Users` is global user directory. |
| **Duplicate Status Endpoints** | `AdminController.cs:80` vs `UsersController.cs:53` | `PATCH /api/admin/users/{id}/status` and `PATCH /api/Users/{id}/status` execute the exact same service method. | Deprecate `PATCH /api/Users/{id}/status` in favor of scoped `PATCH /api/admin/users/{id}/status`. |
| **Payment Return Type** | `PaymentsController.cs:63-70` | Gateway returns (`/vnpay/return`, `/momo/return`) return JSON envelopes rather than 302 redirects to client app. | Mobile and Web clients must handle the JSON payload and execute client-side navigation. |

---

## 4. Architectural Recommendations for Next Sprint

1. **Idempotency Keys for Financial and Gate Endpoints:**
   - Introduce an `X-Idempotency-Key` header filter for `POST /api/payments/checkout`, `POST /api/wallet/topup`, and `POST /api/staff/lots/{id}/check-in` to prevent double-charging or duplicate entry creation in poor network connectivity conditions.
2. **Correlation / Trace ID Middleware:**
   - Implement an ambient `TraceIdMiddleware` that reads `X-Correlation-ID` from incoming requests (or generates a new GUID) and attaches it to ASP.NET Core `HttpContext.TraceIdentifier` and all outgoing HTTP response headers.
3. **Comprehensive Rate-Limiting Coverage:**
   - Currently, rate limiting only covers public layout endpoints (120 req/min) and password reset (5 req/15 min).
   - Recommend adding:
     - Auth rate limit: 10 failed login attempts per IP per 5 minutes.
     - Top-up / payment rate limit: 30 requests per minute per user ID.
     - Gate OCR scanning rate limit: 60 requests per minute per camera client.
4. **Refresh Token Flow Implementation:**
   - Replace the static 7-day JWT with short-lived access tokens (15–30 minutes) and cryptographically random, revocable refresh tokens stored in the database with device fingerprinting.

---

*This concludes the Open Questions and Architectural Audit report.*  
*Deliverable maintained under `C:\DEV\OJT-report\reports\parking-system\open-questions.md`.*
