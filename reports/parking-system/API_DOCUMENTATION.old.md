# Parking Management System -- Backend API Documentation

> **API Version:** 1.0.0  
> **Target Solution:** Parking Management System (PMS) Backend  
> **Protocol:** HTTPS (REST) & WSS (SignalR)  
> **Standard Format:** JSON (`application/json`)  
> **Document Classification:** Official Technical Integration Specification  

---

## Table of Contents

- [1. Document Overview](#1-document-overview)
  - [1.1 Purpose and Scope](#11-purpose-and-scope)
  - [1.2 Intended Audience](#12-intended-audience)
  - [1.3 API Surface & Coverage Summary](#13-api-surface-coverage-summary)
- [2. API Architecture & Technical Overview](#2-api-architecture-technical-overview)
  - [2.1 Architectural Overview](#21-architectural-overview)
  - [2.2 Base URLs and Environments](#22-base-urls-and-environments)
  - [2.3 API Versioning and URL Prefix](#23-api-versioning-and-url-prefix)
  - [2.4 Supported Content Types & Encodings](#24-supported-content-types-encodings)
  - [2.5 Route Aliasing Conventions](#25-route-aliasing-conventions)
- [3. Authentication and Authorization](#3-authentication-and-authorization)
  - [3.1 JWT Bearer Authentication Mechanism](#31-jwt-bearer-authentication-mechanism)
  - [3.2 Two-Factor Authentication (2FA) Challenge Flow](#32-two-factor-authentication-2fa-challenge-flow)
  - [3.3 Role-Based Access Control (RBAC) Matrix](#33-role-based-access-control-rbac-matrix)
  - [3.4 Facility Scope & Lot-Level Authorization](#34-facility-scope-lot-level-authorization)
- [4. Common API Conventions & Response Envelope](#4-common-api-conventions-response-envelope)
  - [4.1 HTTP Methods and Semantics](#41-http-methods-and-semantics)
  - [4.2 Standard Request Headers](#42-standard-request-headers)
  - [4.3 Standard Response Envelope (`ApiResult<T>`)](#43-standard-response-envelope-apiresultt)
  - [4.4 Pagination, Sorting, and Query Filtering](#44-pagination-sorting-and-query-filtering)
  - [4.5 Date, Time, and Monetary Formatting](#45-date-time-and-monetary-formatting)
- [5. API Reference](#5-api-reference)
  - [5.1 Module 01: Authentication & User Accounts](#51-module-01-authentication-user-accounts)
  - [5.2 Module 02: Parking Lots, Layouts, Zones & Slots](#52-module-02-parking-lots-layouts-zones-slots)
  - [5.3 Module 03: Customer Reservations](#53-module-03-customer-reservations)
  - [5.4 Module 04: Customer Parking Sessions](#54-module-04-customer-parking-sessions)
  - [5.5 Module 05: Staff Gate Operations & Gate Requests](#55-module-05-staff-gate-operations-gate-requests)
  - [5.6 Module 06: Manager Operations & Pricing](#56-module-06-manager-operations-pricing)
  - [5.7 Module 07: Vehicle Verification & Document Management](#57-module-07-vehicle-verification-document-management)
  - [5.8 Module 08: Payments & Customer Wallet](#58-module-08-payments-customer-wallet)
  - [5.9 Module 09: Revenue Reporting](#59-module-09-revenue-reporting)
  - [5.10 Module 10: Admin User & System Management](#510-module-10-admin-user-system-management)
  - [5.11 Module 11: Admin Simulation & Testing](#511-module-11-admin-simulation-testing)
  - [5.12 Module 12: Notifications](#512-module-12-notifications)
- [6. Real-Time API / SignalR Hubs](#6-real-time-api-signalr-hubs)
  - [6.1 SignalR Connection & Token Negotiation](#61-signalr-connection-token-negotiation)
  - [6.2 LotHub (`/api/hubs/lots`)](#62-lothub-apihubslots)
  - [6.3 ParkingOpsHub (`/api/hubs/parking-ops`)](#63-parkingopshub-apihubsparking-ops)
  - [6.4 VehicleVerificationHub (`/api/hubs/vehicle-verifications`)](#64-vehicleverificationhub-apihubsvehicle-verifications)
  - [6.5 Connection Resilience & Client Reconnect Policy](#65-connection-resilience-client-reconnect-policy)
- [7. Error Handling, Validation, and Rate Limiting](#7-error-handling-validation-and-rate-limiting)
  - [7.1 FluentValidation Request Pipeline](#71-fluentvalidation-request-pipeline)
  - [7.2 Global Exception Handling Middleware](#72-global-exception-handling-middleware)
  - [7.3 ASP.NET Core Rate Limiting Policies](#73-aspnet-core-rate-limiting-policies)
- [8. Shared Domain Enums & Integration Guidelines](#8-shared-domain-enums-integration-guidelines)
  - [8.1 Core Domain Enums](#81-core-domain-enums)
  - [8.2 Automated Gate Hardware & ANPR Camera Integration](#82-automated-gate-hardware-anpr-camera-integration)
  - [8.3 Payment Gateway Callback & Reconciliation Notes](#83-payment-gateway-callback-reconciliation-notes)
- [9. Glossary and Terminology](#9-glossary-and-terminology)

---

## 1. Document Overview

### 1.1 Purpose and Scope
This document provides the definitive, implementation-accurate API specification for the Parking Management System (PMS) backend. It details every exposed RESTful HTTP endpoint, real-time WebSocket communication hub, authentication and authorization policy, data transfer object (DTO), and system error contract.

The specification is compiled directly from the ASP.NET Core backend solution (`PMS.sln`) and serves as the single source of truth for frontend development, mobile integration, backend maintenance, and lead architect review.

### 1.2 Intended Audience
- **Frontend & Mobile Engineers:** Integrate UI workflows, form validations, data tables, and live telemetry against backend contracts.
- **Backend Engineers:** Maintain endpoint consistency, implement new features, and enforce architectural boundaries.
- **Quality Assurance (QA) Engineers:** Author automated integration tests, boundary tests, and verify system state machines.
- **Technical Leads & Architects:** Conduct security reviews, API governance, and operational auditing.

### 1.3 API Surface & Coverage Summary
The Parking Management System backend exposes:
- **150 Canonical RESTful Endpoints:** Spanning 12 functional modules, 22 controller classes, and 162 structured DTO models.
- **2 Method-Level Route Aliases:** Supporting alternate public lot and zone route paths for backward compatibility.
- **3 Dedicated SignalR Hubs:** Delivering push telemetry for 3D visual layouts, operational lane events, and vehicle paper verification.
- **12 Server-to-Client Real-Time Events:** Enabling immediate client UI updates without polling loops.

---

## 2. API Architecture & Technical Overview

### 2.1 Architectural Overview
The backend is architected using Clean Architecture patterns in .NET 8 / ASP.NET Core:
1. **API Layer (`PMS.API`):** Hosts ASP.NET Core Web API controllers, SignalR hubs, authentication middleware, exception handling filters, and rate limiting.
2. **Application Layer (`PMS.Application`):** Encapsulates business logic, DTOs, FluentValidation rules, Hangfire background jobs, and domain service interfaces.
3. **Infrastructure & Data Layer (`PMS.DataAccess`):** Implements Entity Framework Core persistence with PostgreSQL / SQL Server, payment gateway clients (VNPay, MoMo), and SMTP/Dev mail dispatchers.
4. **Domain Layer (`PMS.Core`):** Houses pure domain entities, enumerations, and aggregate roots.

### 2.2 Base URLs and Environments
| Environment | Base REST URL | SignalR Hubs Base URL | Description |
| :--- | :--- | :--- | :--- |
| **Local Development (HTTPS)** | `https://localhost:7001` | `https://localhost:7001/api/hubs` | Local developer workstation with self-signed SSL. |
| **Local Development (HTTP)** | `http://localhost:5000` | `http://localhost:5000/api/hubs` | Local plain HTTP dev port. |
| **Staging / Evaluation** | `https://staging-api.parkingsystem.vn` | `https://staging-api.parkingsystem.vn/api/hubs` | Staging environment for QA and client demo. |
| **Production** | `https://api.parkingsystem.vn` | `https://api.parkingsystem.vn/api/hubs` | High-availability production cluster. |

### 2.3 API Versioning and URL Prefix
All REST endpoints are uniformly prefixed with `/api/` (e.g. `/api/Users/login`, `/api/lots`, `/api/reservations`). Route parameters follow standard ASP.NET Core typed route constraints (e.g. `{id:int}`, `{id:long}`, `{vehicleId:guid}`). When calling endpoints, client applications omit the constraint type and supply the literal value (e.g. `/api/lots/1/zones`).

### 2.4 Supported Content Types & Encodings
- Standard Request Body: `application/json; charset=utf-8`
- Standard Response Body: `application/json; charset=utf-8`
- Binary File Uploads: `multipart/form-data` (avatar photos, KYC vehicle documents)
- Binary File Downloads: `image/jpeg`, `image/png`, or `application/pdf` via `FileResult`
- Real-Time Streams: WebSockets (SignalR duplex binary/JSON protocol)

### 2.5 Route Aliasing Conventions
To support clean plural REST naming alongside legacy client integrations, the backend defines method-level and class-level route aliases:

| Canonical Route | Route Alias | Controller Action | Description |
| :--- | :--- | :--- | :--- |
| `GET /api/public/lots` | `GET /api/public/parking-lots` | `LotsController.GetPublicLotsAsync` | Public active parking lot directory listing. |
| `GET /api/lots/{lotId}/zones` | `GET /api/parking-lots/{lotId}/zones` | `ZonesController.GetByLotIdAsync` | Zone listing for a specified parking lot. |
| `api/lots/*` | `api/parking-lots/*` | `LotsController` | Class-level route attribute accepts both `/api/lots` and `/api/parking-lots`. |

> [!NOTE]
> Client applications are recommended to consume canonical routes (`/api/lots`). Route aliases are fully supported by ASP.NET Core routing and execute identical controller logic.

---

## 3. Authentication and Authorization

### 3.1 JWT Bearer Authentication Mechanism
Protected endpoints require a JSON Web Token (JWT) transmitted via the standard HTTP `Authorization` header:
```http
Authorization: Bearer <jwt_token>
```
- **Signature Algorithm:** HMAC-SHA256 (`JwtHelper.cs`).
- **Token Lifetime:** 7 days (604,800 seconds).
- **Token Payload Claims:**
  - `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/nameidentifier`: Unique User GUID string.
  - `http://schemas.xmlsoap.org/ws/2005/05/identity/claims/email`: Registered account email address.
  - `http://schemas.microsoft.com/ws/2008/06/identity/claims/role`: Assigned role (`User`, `Staff`, `Manager`, or `Admin`).
  - `purpose`: Optional token purpose string (e.g. `2fa_auth` for intermediate two-factor challenges).

### 3.2 Two-Factor Authentication (2FA) Challenge Flow
When a customer or employee account has 2FA enabled:
1. `POST /api/Users/login` returns HTTP 200 with `requiresTwoFactor = true` and an intermediate `twoFactorToken` (valid for 5 minutes, restricted to 2FA verification).
2. The client application prompts the user for their 6-digit TOTP code and submits it to `POST /api/Users/2fa/login` with the intermediate token.
3. Upon successful validation, the backend issues the standard 7-day session JWT access token.

### 3.3 Role-Based Access Control (RBAC) Matrix
The parking platform defines four hierarchical roles:
- `User`: Customer driver booking reservations, managing vehicles, checking parking sessions, and funding XXIV wallet.
- `Staff`: Gate operator executing vehicle check-in, checkout, cash collection, and resolving barrier lane events.
- `Manager`: Facility supervisor configuring parking lot tariffs, inspecting revenue, and managing assigned staff.
- `Admin`: Global system administrator provisioning facilities, managing internal user accounts, and running simulation engines.

| Functional Area | Public / Anonymous | User (Customer) | Staff (Operator) | Manager | Admin |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Authentication (Login, Register, OTP)** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Customer Profile & Vehicle Management** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Browse Public Lots & 2D/3D Layouts** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Advance Parking Reservations** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Customer Active Sessions & History** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Gate Lane Operations (Check-in/Out)** | ❌ | ❌ | ✅ | ✅ | ✅ |
| **Staff Gate Cash Fee Collection** | ❌ | ❌ | ✅ | ✅ | ✅ |
| **Lot Management & 3D Template Apply** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Tariff & Pricing Rules Management** | ❌ | ❌ | ❌ | ✅ | ✅ |
| **Whitelist & Blacklist Plate Management** | ❌ | ❌ | ✅ (Read) | ✅ | ✅ |
| **Vehicle KYC Verification Review** | ❌ | ❌ | ✅ | ✅ | ✅ |
| **Customer XXIV Wallet (TopUp/Withdraw)** | ❌ | ❌ | ❌ | ❌ | ❌ |
| **Scoped Revenue Reports** | ❌ | ❌ | ❌ | ✅ (Assigned) | ❌ |
| **System-Wide Revenue Reports** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Internal Employee Account Provisioning** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Time Acceleration Simulation Mode** | ❌ | ❌ | ❌ | ❌ | ✅ |

### 3.4 Facility Scope & Lot-Level Authorization
In addition to ASP.NET Core role checks, facility operational endpoints enforce resource-level authorization via `LotScopeService.CanOperateAsync`:
- **Staff Operators:** Restricted strictly to the single parking lot facility assigned to their employee account (`Employee.ParkingLotId`).
- **Managers:** Restricted strictly to the portfolio of parking lots assigned to them via `AdminController.AssignManagerLotsAsync`.
- **Admins:** Unrestricted global access across all facilities.
- Violations result in an immediate **HTTP 403 Forbidden** rejection.

---

## 4. Common API Conventions & Response Envelope

### 4.1 HTTP Methods and Semantics
- `GET`: Idempotent read operations. Safe to retry. Returns `200 OK`.
- `POST`: Resource creation or state transition operations. Returns `200 OK` (or `201 Created` for reservations; `204 NoContent` for MoMo IPN).
- `PUT`: Complete resource replacement or idempotently updating an entire entity. Returns `200 OK`.
- `PATCH`: Partial state modification (e.g. status toggling, default vehicle selection). Returns `200 OK`.
- `DELETE`: Resource removal or soft deletion. Returns `200 OK`.

### 4.2 Standard Request Headers
```http
Content-Type: application/json
Accept: application/json
Authorization: Bearer <jwt_token>
```

### 4.3 Standard Response Envelope (`ApiResult<T>`)
All standard API responses wrap return payloads inside the generic `ApiResult<T>` envelope.

#### Success Envelope Format
```json
{
  "succeeded": true,
  "result": {},
  "errors": []
}
```

#### Error Envelope Format
```json
{
  "succeeded": false,
  "result": null,
  "errors": [
    "A clear, descriptive error message explaining why the operation failed."
  ]
}
```

### 4.4 Pagination, Sorting, and Query Filtering
Endpoints supporting pagination return `PaginationResponseModel<T>` within the `result` property:
```json
{
  "succeeded": true,
  "result": {
    "items": [],
    "pageNumber": 1,
    "pageSize": 10,
    "totalCount": 85,
    "totalPages": 9,
    "hasPreviousPage": false,
    "hasNextPage": true
  },
  "errors": []
}
```

### 4.5 Date, Time, and Monetary Formatting
- **Timestamps:** ISO 8601 extended format in UTC with trailing `Z` suffix (e.g. `2026-10-30T08:00:00Z`).
- **Monetary Amounts:** Exact numeric values expressed in Vietnamese Dong (VND) without thousands separators or currency symbols in JSON payloads (e.g. `50000` for 50,000 VND).

---

## 5. API Reference

### 5.1 Module 01: Authentication & User Accounts

#### 5.1.1 `GET` /api/Users

**Description:** Retrieves a filtered and searchable list of customer user accounts.

- **Controller / Action:** `UsersController.GetAllAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Query Parameters:**
  - `role` (string, optional, default: `null`): Filter parameter.
  - `isActive` (bool?, optional, default: `null`): Filter parameter.
  - `searchTerm` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IEnumerable<UserResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": "Id_value",
        "userName": "UserName_value",
        "fullName": "Nguyen Van A",
        "email": "user@example.com",
        "phoneNumber": "0987654321",
        "role": "Staff",
        "employeeCode": "RES-20261030-01",
        "isActive": true,
        "createdAt": "2026-10-30T08:00:00Z"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.

#### 5.1.2 `GET` /api/Users/{id}

**Description:** Retrieves full account details for a specific user by unique identifier.

- **Controller / Action:** `UsersController.GetByIdAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<UserResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "isActive": true,
      "createdAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.1.3 `POST` /api/Users/internal

**Description:** Provisions an internal system user account (Staff, Manager, or Admin) with assigned role.

- **Controller / Action:** `UsersController.CreateInternalUserAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Request Body (`CreateInternalUserModel`):**
  ```json
  {
    "userName": "UserName_value",
    "fullName": "Nguyen Van A",
    "email": "user@example.com",
    "password": "P@ssword123",
    "phoneNumber": "0987654321",
    "role": "Staff",
    "assignedParkingLot": "AssignedParkingLot_value"
  }
  ```
- **Success Response Contract:** `ApiResult<CreateUserResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "employeeCode": "RES-20261030-01"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.

#### 5.1.4 `PATCH` /api/Users/{id}/status

**Description:** Activates or deactivates an internal user account, toggling login capabilities.

- **Controller / Action:** `UsersController.UpdateStatusAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateUserStatusModel`):**
  ```json
  {
    "isActive": true
  }
  ```
- **Success Response Contract:** `ApiResult<bool>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": true,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.1.5 `POST` /api/Users/register

**Description:** Registers a new customer account using email, phone number, and password.

- **Controller / Action:** `UsersController.RegisterAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Success Response Contract:** `ApiResult<CreateUserResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "employeeCode": "RES-20261030-01"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.

#### 5.1.6 `POST` /api/Users/authenticate

**Description:** Authenticates customer/staff credentials and returns a JWT access token or a 2FA challenge token.

- **Controller / Action:** `UsersController.LoginAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Success Response Contract:** `ApiResult<LoginResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "username": "Username_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "requiresTwoFactor": false,
      "twoFactorToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "destination": "Destination_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.

#### 5.1.7 `POST` /api/Users/two-factor

**Description:** Verifies a 6-digit TOTP code against an active 2FA challenge token to complete authentication.

- **Controller / Action:** `UsersController.VerifyTwoFactorAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Request Body (`TwoFactorLoginModel`):**
  ```json
  {
    "twoFactorToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "code": "RES-20261030-01",
    "email": "user@example.com",
    "rememberMe": false
  }
  ```
- **Success Response Contract:** `ApiResult<LoginResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "username": "Username_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "requiresTwoFactor": false,
      "twoFactorToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "destination": "Destination_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.

#### 5.1.8 `POST` /api/Users/two-factor/resend

**Description:** Dispatches a new two-factor authentication OTP code to the user registered email or phone.

- **Controller / Action:** `UsersController.ResendTwoFactorOtpAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Request Body (`ResendTwoFactorModel`):**
  ```json
  {
    "twoFactorToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "email": "user@example.com"
  }
  ```
- **Success Response Contract:** `ApiResult<ResendTwoFactorResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "succeeded": true,
      "twoFactorToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "destination": "Destination_value",
      "message": "Message_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.

#### 5.1.9 `POST` /api/Users/login-otp/request

**Description:** Initiates passwordless login by emailing a single-use 6-digit OTP code to the user.

- **Controller / Action:** `UsersController.RequestLoginOtpAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Request Body (`RequestLoginOtpModel`):**
  ```json
  {
    "email": "user@example.com"
  }
  ```
- **Success Response Contract:** `ApiResult<string>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": "Operation executed successfully.",
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.

#### 5.1.10 `POST` /api/Users/login-otp/verify

**Description:** Authenticates a passwordless session by verifying the 6-digit email OTP code.

- **Controller / Action:** `UsersController.LoginWithOtpAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Request Body (`VerifyLoginOtpModel`):**
  ```json
  {
    "email": "user@example.com",
    "code": "RES-20261030-01"
  }
  ```
- **Success Response Contract:** `ApiResult<LoginResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "username": "Username_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "requiresTwoFactor": false,
      "twoFactorToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "destination": "Destination_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.

#### 5.1.11 `POST` /api/Users/forgot-password

**Description:** Requests an account password reset link/token delivered via email (rate-limited to 5 req/15m).

- **Controller / Action:** `UsersController.ForgotPasswordAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Request Body (`ForgotPasswordModel`):**
  ```json
  {
    "email": "user@example.com"
  }
  ```
- **Success Response Contract:** `ApiResult<string>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": "Operation executed successfully.",
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `429 Too Many Requests`: Request rate limit exceeded.

#### 5.1.12 `POST` /api/Users/reset-password

**Description:** Resets the account password using a validated password reset token and new password.

- **Controller / Action:** `UsersController.ResetPasswordAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Request Body (`ResetPasswordModel`):**
  ```json
  {
    "email": "user@example.com",
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "newPassword": "P@ssword123",
    "confirmPassword": "P@ssword123"
  }
  ```
- **Success Response Contract:** `ApiResult<string>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": "Operation executed successfully.",
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.

#### 5.1.13 `POST` /api/Users/login-google

**Description:** Authenticates a user via Google OAuth ID token exchange, provisioning an account if new.

- **Controller / Action:** `UsersController.LoginWithGoogleAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Request Body (`GoogleLoginModel`):**
  ```json
  {
    "idToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
  }
  ```
- **Success Response Contract:** `ApiResult<LoginResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "username": "Username_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "requiresTwoFactor": false,
      "twoFactorToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "destination": "Destination_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.

#### 5.1.14 `GET` /api/Users/profile

**Description:** Retrieves the complete profile and KYC identity information for the authenticated user.

- **Controller / Action:** `UsersController.GetProfileAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Success Response Contract:** `ApiResult<UserProfileResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "fullName": "Nguyen Van A",
      "phoneNumber": "0987654321",
      "email": "user@example.com",
      "avatarUrl": "https://example.com/payment/return",
      "citizenId": "079201012345",
      "isActive": true,
      "status": "Active",
      "hasPassword": false,
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "vehicles": [
        {
          "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "licensePlate": "59A-123.45",
          "vehicleTypeId": 1,
          "vehicleType": "VehicleType_value",
          "brand": "Brand_value",
          "model": "Model_value",
          "color": "Color_value",
          "citizenId": "079201012345",
          "isDefault": false,
          "isActive": true,
          "verificationStatus": "Active",
          "verificationNote": "VerificationNote_value",
          "createdAt": "2026-10-30T08:00:00Z",
          "updatedAt": "2026-10-30T08:00:00Z"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.1.15 `PUT` /api/Users/profile

**Description:** Updates personal contact information, full name, address, and citizen ID for the authenticated user.

- **Controller / Action:** `UsersController.UpdateProfileAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Request Body (`UpdateUserProfileModel`):**
  ```json
  {
    "fullName": "Nguyen Van A",
    "email": "user@example.com",
    "phoneNumber": "0987654321",
    "avatarUrl": "https://example.com/payment/return",
    "citizenId": "079201012345"
  }
  ```
- **Success Response Contract:** `ApiResult<UserProfileResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "fullName": "Nguyen Van A",
      "phoneNumber": "0987654321",
      "email": "user@example.com",
      "avatarUrl": "https://example.com/payment/return",
      "citizenId": "079201012345",
      "isActive": true,
      "status": "Active",
      "hasPassword": false,
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "vehicles": [
        {
          "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "licensePlate": "59A-123.45",
          "vehicleTypeId": 1,
          "vehicleType": "VehicleType_value",
          "brand": "Brand_value",
          "model": "Model_value",
          "color": "Color_value",
          "citizenId": "079201012345",
          "isDefault": false,
          "isActive": true,
          "verificationStatus": "Active",
          "verificationNote": "VerificationNote_value",
          "createdAt": "2026-10-30T08:00:00Z",
          "updatedAt": "2026-10-30T08:00:00Z"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.1.16 `GET` /api/Users/profile/check-unique

**Description:** Validates whether a phone number or citizen ID is already registered to another profile before updating.

- **Controller / Action:** `UsersController.CheckProfileUniquenessAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Query Parameters:**
  - `phone` (string?, optional, default: `null`): Filter parameter.
  - `citizenId` (string?, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<ProfileFieldAvailabilityModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "isPhoneAvailable": false,
      "phoneReason": "0987654321",
      "isCitizenIdAvailable": false,
      "citizenIdReason": "Operational requirement"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.1.17 `POST` /api/Users/profile/avatar

**Description:** Uploads and replaces the authenticated user avatar picture with a JPEG or PNG image.

- **Controller / Action:** `UsersController.UploadAvatarAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Request Body (`multipart/form-data`):**
  - `file` (binary, required): JPEG, PNG, or PDF document file stream.
- **Success Response Contract:** `ApiResult<UserProfileResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "fullName": "Nguyen Van A",
      "phoneNumber": "0987654321",
      "email": "user@example.com",
      "avatarUrl": "https://example.com/payment/return",
      "citizenId": "079201012345",
      "isActive": true,
      "status": "Active",
      "hasPassword": false,
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "vehicles": [
        {
          "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "licensePlate": "59A-123.45",
          "vehicleTypeId": 1,
          "vehicleType": "VehicleType_value",
          "brand": "Brand_value",
          "model": "Model_value",
          "color": "Color_value",
          "citizenId": "079201012345",
          "isDefault": false,
          "isActive": true,
          "verificationStatus": "Active",
          "verificationNote": "VerificationNote_value",
          "createdAt": "2026-10-30T08:00:00Z",
          "updatedAt": "2026-10-30T08:00:00Z"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.1.18 `GET` /api/Users/profile/vehicles

**Description:** Retrieves all registered vehicles, verification statuses, and license plates for the authenticated user.

- **Controller / Action:** `UsersController.GetVehiclesAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Success Response Contract:** `ApiResult<List<VehicleResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "licensePlate": "59A-123.45",
        "vehicleTypeId": 1,
        "vehicleType": "VehicleType_value",
        "brand": "Brand_value",
        "model": "Model_value",
        "color": "Color_value",
        "citizenId": "079201012345",
        "isDefault": false,
        "isActive": true,
        "verificationStatus": "Active",
        "verificationNote": "VerificationNote_value",
        "createdAt": "2026-10-30T08:00:00Z",
        "updatedAt": "2026-10-30T08:00:00Z"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.1.19 `GET` /api/Users/profile/vehicles/check-plate

**Description:** Checks whether a license plate number is available for registration or already bound to an active vehicle.

- **Controller / Action:** `UsersController.CheckPlateAvailabilityAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Query Parameters:**
  - `plate` (string, optional, default: `none`): Filter parameter.
  - `excludeVehicleId` (Guid?, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<PlateAvailabilityResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "licensePlate": "59A-123.45",
      "isAvailable": false,
      "reason": "Operational requirement"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.1.20 `POST` /api/Users/profile/vehicles

**Description:** Registers a new vehicle under the customer profile, initiating the KYC paper verification workflow.

- **Controller / Action:** `UsersController.AddVehicleAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Request Body (`CreateVehicleModel`):**
  ```json
  {
    "licensePlate": "59A-123.45",
    "vehicleTypeId": 1,
    "brand": "Brand_value",
    "model": "Model_value",
    "color": "Color_value",
    "citizenId": "079201012345",
    "isDefault": false
  }
  ```
- **Success Response Contract:** `ApiResult<VehicleResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "licensePlate": "59A-123.45",
      "vehicleTypeId": 1,
      "vehicleType": "VehicleType_value",
      "brand": "Brand_value",
      "model": "Model_value",
      "color": "Color_value",
      "citizenId": "079201012345",
      "isDefault": false,
      "isActive": true,
      "verificationStatus": "Active",
      "verificationNote": "VerificationNote_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.1.21 `PUT` /api/Users/profile/vehicles/{vehicleId:guid}

**Description:** Updates display label, make, and model information for an existing registered vehicle.

- **Controller / Action:** `UsersController.UpdateVehicleAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Path Parameters:**
  - `vehicleId` (required): Target vehicleId identifier.
- **Request Body (`UpdateVehicleModel`):**
  ```json
  {
    "licensePlate": "59A-123.45",
    "vehicleTypeId": 1,
    "brand": "Brand_value",
    "model": "Model_value",
    "color": "Color_value",
    "citizenId": "079201012345",
    "isDefault": false,
    "isActive": true
  }
  ```
- **Success Response Contract:** `ApiResult<VehicleResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "licensePlate": "59A-123.45",
      "vehicleTypeId": 1,
      "vehicleType": "VehicleType_value",
      "brand": "Brand_value",
      "model": "Model_value",
      "color": "Color_value",
      "citizenId": "079201012345",
      "isDefault": false,
      "isActive": true,
      "verificationStatus": "Active",
      "verificationNote": "VerificationNote_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.1.22 `DELETE` /api/Users/profile/vehicles/{vehicleId:guid}

**Description:** Soft-deletes or removes an unverified vehicle from the customer profile.

- **Controller / Action:** `UsersController.DeleteVehicleAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Path Parameters:**
  - `vehicleId` (required): Target vehicleId identifier.
- **Success Response Contract:** `ApiResult<bool>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": true,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.1.23 `PATCH` /api/Users/profile/vehicles/{vehicleId:guid}/default

**Description:** Sets a specified registered vehicle as the customer default vehicle for fast reservations.

- **Controller / Action:** `UsersController.SetDefaultVehicleAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Path Parameters:**
  - `vehicleId` (required): Target vehicleId identifier.
- **Success Response Contract:** `ApiResult<VehicleResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "licensePlate": "59A-123.45",
      "vehicleTypeId": 1,
      "vehicleType": "VehicleType_value",
      "brand": "Brand_value",
      "model": "Model_value",
      "color": "Color_value",
      "citizenId": "079201012345",
      "isDefault": false,
      "isActive": true,
      "verificationStatus": "Active",
      "verificationNote": "VerificationNote_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.1.24 `GET` /api/Users/profile/vehicle-types

**Description:** Retrieves all active system vehicle categories, capacity multipliers, and base price rules.

- **Controller / Action:** `UsersController.GetVehicleTypesAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Success Response Contract:** `ApiResult<List<VehicleTypeResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "code": "RES-20261030-01",
        "name": "Nguyen Van A",
        "description": "Description_value",
        "sortOrder": 1
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.1.25 `PUT` /api/Users/profile/change-password

**Description:** Changes the account password for the authenticated user after verifying current password.

- **Controller / Action:** `UsersController.ChangePasswordAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Request Body (`ChangePasswordModel`):**
  ```json
  {
    "currentPassword": "P@ssword123",
    "newPassword": "P@ssword123",
    "confirmPassword": "P@ssword123"
  }
  ```
- **Success Response Contract:** `ApiResult<bool>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": true,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.1.26 `GET` /api/Users/{id}/profile

**Description:** Allows an administrator to inspect profile and verification details for a specific user ID.

- **Controller / Action:** `UsersController.GetProfileByUserIdAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<UserProfileResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "fullName": "Nguyen Van A",
      "phoneNumber": "0987654321",
      "email": "user@example.com",
      "avatarUrl": "https://example.com/payment/return",
      "citizenId": "079201012345",
      "isActive": true,
      "status": "Active",
      "hasPassword": false,
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "vehicles": [
        {
          "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "licensePlate": "59A-123.45",
          "vehicleTypeId": 1,
          "vehicleType": "VehicleType_value",
          "brand": "Brand_value",
          "model": "Model_value",
          "color": "Color_value",
          "citizenId": "079201012345",
          "isDefault": false,
          "isActive": true,
          "verificationStatus": "Active",
          "verificationNote": "VerificationNote_value",
          "createdAt": "2026-10-30T08:00:00Z",
          "updatedAt": "2026-10-30T08:00:00Z"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

### 5.2 Module 02: Parking Lots, Layouts, Zones & Slots

> [!TIP]
> **Supported Route Aliases in Module 02:**
> - `GET /api/public/lots` may also be accessed via `GET /api/public/parking-lots`.
> - `GET /api/lots/{lotId}/zones` may also be accessed via `GET /api/parking-lots/{lotId}/zones`.
> - All `/api/lots/*` management routes support the `/api/parking-lots/*` prefix alias.

#### 5.2.1 `GET` /api/lot-templates

**Description:** Retrieves predefined 3D/2D parking layout templates used to construct standard parking lot configurations.

- **Controller / Action:** `LayoutsController.GetTemplates`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Success Response Contract:** `ApiResult<IReadOnlyList<LotTemplateResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": "Id_value",
        "name": "Nguyen Van A",
        "summary": "Summary_value",
        "slotsPerFloor": 1,
        "carSlots": 1,
        "motorbikeSlots": 1,
        "totalSlots": 100,
        "floors": [
          {
            "floorNo": 1,
            "name": "Nguyen Van A",
            "kind": "Kind_value",
            "rows": 1,
            "cols": 1
          }
        ]
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.

#### 5.2.2 `GET` /api/public/lots/{lotId:int}/layout

**Description:** Retrieves the published 3D visual layout metadata, coordinate boundaries, and zone geometry for visual navigation.

- **Controller / Action:** `LayoutsController.GetPublishedAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Success Response Contract:** `ApiResult<LayoutResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "parkingLotId": 1,
      "parkingLotName": "Central City Parking",
      "lotStatus": "Active",
      "version": 1,
      "templateId": "59A-123.45",
      "publishedAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "floors": [
        {
          "floorNo": 1,
          "name": "Nguyen Van A",
          "kind": "Kind_value",
          "rows": 1,
          "cols": 1
        }
      ],
      "zones": [
        {
          "zoneId": 1,
          "code": "RES-20261030-01",
          "name": "Nguyen Van A",
          "zoneType": "ZoneType_value",
          "floorNo": 1
        }
      ],
      "slots": [
        {
          "slotId": 1,
          "code": "RES-20261030-01",
          "zoneId": 1,
          "vehicleTypeCode": "RES-20261030-01",
          "floorNo": 1,
          "row": 1,
          "col": 1,
          "rotation": 1,
          "modelUrl": "https://example.com/payment/return"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `404 Not Found`: Target resource with specified identifier does not exist.
  - `429 Too Many Requests`: Request rate limit exceeded.

#### 5.2.3 `GET` /api/public/lots/{lotId:int}/layout/status

**Description:** Retrieves real-time slot occupancy statuses across the visual layout for 3D digital-twin rendering.

- **Controller / Action:** `LayoutsController.GetStatusAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Success Response Contract:** `ApiResult<LayoutStatusResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "parkingLotId": 1,
      "version": 1,
      "generatedAt": "2026-10-30T08:00:00Z",
      "slots": [
        {
          "slotId": 1,
          "status": "Active"
        }
      ],
      "zones": [
        {
          "zoneId": 1,
          "total": 100,
          "free": 1
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `404 Not Found`: Target resource with specified identifier does not exist.
  - `429 Too Many Requests`: Request rate limit exceeded.

#### 5.2.4 `GET` /api/public/lots/{lotId:int}/layout/2d

**Description:** Retrieves simplified 2D architectural coordinates and spatial boundaries of a parking lot for map views.

- **Controller / Action:** `LayoutsController.Get2DAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `detail` (string, optional, default: `LayoutDetail.Slots`): Filter parameter.
- **Success Response Contract:** `ApiResult<Layout2DResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "parkingLotId": 1,
      "parkingLotName": "Central City Parking",
      "lotStatus": "Active",
      "version": 1,
      "detail": "Detail_value",
      "floors": [
        {
          "floorNo": 1,
          "name": "Nguyen Van A",
          "kind": "Kind_value",
          "rows": 1,
          "cols": 1,
          "total": 100,
          "free": 1,
          "cells": [],
          "zoneBlocks": []
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `404 Not Found`: Target resource with specified identifier does not exist.
  - `429 Too Many Requests`: Request rate limit exceeded.

#### 5.2.5 `GET` /api/lots

**Description:** Retrieves a list of all parking lot facilities with operational statuses, capacity, and active pricing.

- **Controller / Action:** `LotsController.GetAllAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin,Manager)
- **Query Parameters:**
  - `searchTerm` (string, optional, default: `null`): Filter parameter.
  - `status` (string, optional, default: `null`): Filter parameter.
  - `includeDeleted` (bool, optional, default: `false`): Filter parameter.
- **Success Response Contract:** `ApiResult<IEnumerable<ParkingLotResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "code": "RES-20261030-01",
        "name": "Nguyen Van A",
        "address": "123 Le Loi Street, District 1, HCMC",
        "city": "City_value",
        "district": "District 1",
        "openTime": "06:00:00",
        "closeTime": "06:00:00",
        "is24Hours": false,
        "capacity": 100,
        "layoutTemplate": "59A-123.45",
        "imageUrl": "https://example.com/payment/return",
        "description": "Description_value",
        "phone": "0987654321",
        "status": "Active",
        "storedStatus": "Active",
        "isStaffed": false,
        "isBookable": false,
        "closedFrom": "2026-10-30T08:00:00Z",
        "closedUntil": "2026-10-30T08:00:00Z",
        "closureReason": "Operational requirement",
        "maintenanceNotice": "MaintenanceNotice_value",
        "partnerId": 1,
        "managerUserId": "ManagerUserId_value",
        "timeZoneId": "TimeZoneId_value",
        "isDeleted": false,
        "createdAt": "2026-10-30T08:00:00Z",
        "updatedAt": "2026-10-30T08:00:00Z",
        "createdBy": "CreatedBy_value",
        "totalZones": 100,
        "totalSlots": 100
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin,Manager) or facility assignment scope.

#### 5.2.6 `GET` /api/public/lots

**Description:** Retrieves public directory listing of operational parking lots with real-time available bays.

- **Controller / Action:** `LotsController.GetPublicLotsAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Query Parameters:**
  - `searchTerm` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IEnumerable<PublicParkingLotResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "code": "RES-20261030-01",
        "name": "Nguyen Van A",
        "address": "123 Le Loi Street, District 1, HCMC",
        "city": "City_value",
        "district": "District 1",
        "openTime": "06:00:00",
        "closeTime": "06:00:00",
        "is24Hours": false,
        "capacity": 100,
        "layoutTemplate": "59A-123.45",
        "imageUrl": "https://example.com/payment/return",
        "description": "Description_value",
        "phone": "0987654321",
        "status": "Active",
        "isBookable": false,
        "maintenanceNotice": "MaintenanceNotice_value",
        "totalBays": 100,
        "freeBays": 1,
        "baysByVehicle": [
          {
            "vehicleType": "VehicleType_value",
            "totalBays": 100,
            "freeBays": 1
          }
        ]
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.

#### 5.2.7 `GET` /api/public/lots/{id:int}/prices

**Description:** Retrieves public hourly and daily price tariffs for each vehicle category at a specific lot.

- **Controller / Action:** `LotsController.GetPublicPricesAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<IReadOnlyList<PublicLotPriceModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "vehicleTypeId": 1,
        "vehicleTypeCode": "RES-20261030-01",
        "vehicleTypeName": "Nguyen Van A",
        "chargeModel": "ChargeModel_value",
        "graceMinutes": 60,
        "firstBlockMinutes": 60,
        "firstBlockPrice": 50000,
        "blockMinutes": 60,
        "blockPrice": 50000,
        "dayStartTime": "06:00:00",
        "nightStartTime": "22:00:00",
        "dayPrice": 50000,
        "nightPrice": 50000,
        "maxDailyFee": 50000
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.8 `GET` /api/lots/{id:int}

**Description:** Retrieves comprehensive administrative configuration, zones, and capacity metrics for a parking lot.

- **Controller / Action:** `LotsController.GetByIdAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin,Manager)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<ParkingLotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Nguyen Van A",
      "address": "123 Le Loi Street, District 1, HCMC",
      "city": "City_value",
      "district": "District 1",
      "openTime": "06:00:00",
      "closeTime": "06:00:00",
      "is24Hours": false,
      "capacity": 100,
      "layoutTemplate": "59A-123.45",
      "imageUrl": "https://example.com/payment/return",
      "description": "Description_value",
      "phone": "0987654321",
      "status": "Active",
      "storedStatus": "Active",
      "isStaffed": false,
      "isBookable": false,
      "closedFrom": "2026-10-30T08:00:00Z",
      "closedUntil": "2026-10-30T08:00:00Z",
      "closureReason": "Operational requirement",
      "maintenanceNotice": "MaintenanceNotice_value",
      "partnerId": 1,
      "managerUserId": "ManagerUserId_value",
      "timeZoneId": "TimeZoneId_value",
      "isDeleted": false,
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "createdBy": "CreatedBy_value",
      "totalZones": 100,
      "totalSlots": 100
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin,Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.9 `POST` /api/lots

**Description:** Creates a new parking lot facility, defining geographical coordinates, address, and operating hours.

- **Controller / Action:** `LotsController.CreateAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Request Body (`CreateParkingLotModel`):**
  ```json
  {
    "code": "RES-20261030-01",
    "name": "Nguyen Van A",
    "address": "123 Le Loi Street, District 1, HCMC",
    "city": "City_value",
    "district": "District 1",
    "openTime": "06:00:00",
    "closeTime": "06:00:00",
    "is24Hours": false,
    "layoutTemplate": "59A-123.45",
    "imageUrl": "https://example.com/payment/return",
    "description": "Description_value",
    "phone": "0987654321",
    "managerUserId": "ManagerUserId_value",
    "timeZoneId": "TimeZoneId_value"
  }
  ```
- **Success Response Contract:** `ApiResult<ParkingLotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Nguyen Van A",
      "address": "123 Le Loi Street, District 1, HCMC",
      "city": "City_value",
      "district": "District 1",
      "openTime": "06:00:00",
      "closeTime": "06:00:00",
      "is24Hours": false,
      "capacity": 100,
      "layoutTemplate": "59A-123.45",
      "imageUrl": "https://example.com/payment/return",
      "description": "Description_value",
      "phone": "0987654321",
      "status": "Active",
      "storedStatus": "Active",
      "isStaffed": false,
      "isBookable": false,
      "closedFrom": "2026-10-30T08:00:00Z",
      "closedUntil": "2026-10-30T08:00:00Z",
      "closureReason": "Operational requirement",
      "maintenanceNotice": "MaintenanceNotice_value",
      "partnerId": 1,
      "managerUserId": "ManagerUserId_value",
      "timeZoneId": "TimeZoneId_value",
      "isDeleted": false,
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "createdBy": "CreatedBy_value",
      "totalZones": 100,
      "totalSlots": 100
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.

#### 5.2.10 `PUT` /api/lots/{id:int}

**Description:** Updates geographical, address, capacity, and operational parameters of an existing parking lot.

- **Controller / Action:** `LotsController.UpdateAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateParkingLotModel`):**
  ```json
  {
    "name": "Nguyen Van A",
    "address": "123 Le Loi Street, District 1, HCMC",
    "city": "City_value",
    "district": "District 1",
    "openTime": "06:00:00",
    "closeTime": "06:00:00",
    "is24Hours": false,
    "imageUrl": "https://example.com/payment/return",
    "description": "Description_value",
    "phone": "0987654321",
    "managerUserId": "ManagerUserId_value",
    "timeZoneId": "TimeZoneId_value"
  }
  ```
- **Success Response Contract:** `ApiResult<ParkingLotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Nguyen Van A",
      "address": "123 Le Loi Street, District 1, HCMC",
      "city": "City_value",
      "district": "District 1",
      "openTime": "06:00:00",
      "closeTime": "06:00:00",
      "is24Hours": false,
      "capacity": 100,
      "layoutTemplate": "59A-123.45",
      "imageUrl": "https://example.com/payment/return",
      "description": "Description_value",
      "phone": "0987654321",
      "status": "Active",
      "storedStatus": "Active",
      "isStaffed": false,
      "isBookable": false,
      "closedFrom": "2026-10-30T08:00:00Z",
      "closedUntil": "2026-10-30T08:00:00Z",
      "closureReason": "Operational requirement",
      "maintenanceNotice": "MaintenanceNotice_value",
      "partnerId": 1,
      "managerUserId": "ManagerUserId_value",
      "timeZoneId": "TimeZoneId_value",
      "isDeleted": false,
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "createdBy": "CreatedBy_value",
      "totalZones": 100,
      "totalSlots": 100
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.11 `PUT` /api/lots/{id:int}/template

**Description:** Applies a predefined grid template to automatically configure zones and parking bays for a lot.

- **Controller / Action:** `LotsController.ApplyTemplateAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`ApplyLotTemplateModel`):**
  ```json
  {
    "layoutTemplate": "59A-123.45"
  }
  ```
- **Success Response Contract:** `ApiResult<ParkingLotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Nguyen Van A",
      "address": "123 Le Loi Street, District 1, HCMC",
      "city": "City_value",
      "district": "District 1",
      "openTime": "06:00:00",
      "closeTime": "06:00:00",
      "is24Hours": false,
      "capacity": 100,
      "layoutTemplate": "59A-123.45",
      "imageUrl": "https://example.com/payment/return",
      "description": "Description_value",
      "phone": "0987654321",
      "status": "Active",
      "storedStatus": "Active",
      "isStaffed": false,
      "isBookable": false,
      "closedFrom": "2026-10-30T08:00:00Z",
      "closedUntil": "2026-10-30T08:00:00Z",
      "closureReason": "Operational requirement",
      "maintenanceNotice": "MaintenanceNotice_value",
      "partnerId": 1,
      "managerUserId": "ManagerUserId_value",
      "timeZoneId": "TimeZoneId_value",
      "isDeleted": false,
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "createdBy": "CreatedBy_value",
      "totalZones": 100,
      "totalSlots": 100
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.12 `PATCH` /api/lots/{id:int}/status

**Description:** Toggles the operational status of a parking lot (e.g. Open, Full, Closed, Maintenance).

- **Controller / Action:** `LotsController.UpdateStatusAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin,Manager)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateParkingLotStatusModel`):**
  ```json
  {
    "status": "Active",
    "closureReason": "Operational requirement",
    "maintenanceNotice": "MaintenanceNotice_value",
    "closedFrom": "2026-10-30T08:00:00Z",
    "closedUntil": "2026-10-30T08:00:00Z"
  }
  ```
- **Success Response Contract:** `ApiResult<bool>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": true,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin,Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.13 `DELETE` /api/lots/{id:int}

**Description:** Soft-deletes a parking lot facility and marks its infrastructure as retired.

- **Controller / Action:** `LotsController.DeleteAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<bool>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": true,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.14 `GET` /api/slots

**Description:** Retrieves individual parking slots, bay codes, and sensor bindings across lots and zones.

- **Controller / Action:** `SlotsController.GetAllAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin,Manager)
- **Query Parameters:**
  - `zoneId` (int?, optional, default: `null`): Filter parameter.
  - `vehicleTypeId` (int?, optional, default: `null`): Filter parameter.
  - `status` (string, optional, default: `null`): Filter parameter.
  - `isActive` (bool?, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IEnumerable<SlotResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "zoneId": 1,
        "zoneCode": "RES-20261030-01",
        "zoneName": "Zone A",
        "parkingLotId": 1,
        "parkingLotCode": "RES-20261030-01",
        "vehicleTypeId": 1,
        "vehicleTypeCode": "RES-20261030-01",
        "vehicleTypeName": "Nguyen Van A",
        "code": "RES-20261030-01",
        "status": "Active",
        "dedicatedVehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "dedicatedNote": "DedicatedNote_value",
        "isActive": true
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin,Manager) or facility assignment scope.

#### 5.2.15 `GET` /api/zones/{zoneId:int}/slots

**Description:** Retrieves all parking slots within a specific zone, including real-time sensor occupancy state.

- **Controller / Action:** `SlotsController.GetByZoneIdAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin,Manager)
- **Path Parameters:**
  - `zoneId` (required): Target zoneId identifier.
- **Success Response Contract:** `ApiResult<IEnumerable<SlotResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "zoneId": 1,
        "zoneCode": "RES-20261030-01",
        "zoneName": "Zone A",
        "parkingLotId": 1,
        "parkingLotCode": "RES-20261030-01",
        "vehicleTypeId": 1,
        "vehicleTypeCode": "RES-20261030-01",
        "vehicleTypeName": "Nguyen Van A",
        "code": "RES-20261030-01",
        "status": "Active",
        "dedicatedVehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "dedicatedNote": "DedicatedNote_value",
        "isActive": true
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin,Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.16 `GET` /api/slots/{id:int}

**Description:** Retrieves telemetry, hardware sensor bindings, and status details for a specific slot bay.

- **Controller / Action:** `SlotsController.GetByIdAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin,Manager)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<SlotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "zoneId": 1,
      "zoneCode": "RES-20261030-01",
      "zoneName": "Zone A",
      "parkingLotId": 1,
      "parkingLotCode": "RES-20261030-01",
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "Nguyen Van A",
      "code": "RES-20261030-01",
      "status": "Active",
      "dedicatedVehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "dedicatedNote": "DedicatedNote_value",
      "isActive": true
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin,Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.17 `PUT` /api/slots/{id:int}

**Description:** Updates slot code, vehicle type compatibility, and hardware sensor identifiers for a slot bay.

- **Controller / Action:** `SlotsController.UpdateAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateSlotModel`):**
  ```json
  {
    "vehicleTypeId": 1,
    "code": "RES-20261030-01",
    "dedicatedVehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "dedicatedNote": "DedicatedNote_value",
    "isActive": true
  }
  ```
- **Success Response Contract:** `ApiResult<SlotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "zoneId": 1,
      "zoneCode": "RES-20261030-01",
      "zoneName": "Zone A",
      "parkingLotId": 1,
      "parkingLotCode": "RES-20261030-01",
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "Nguyen Van A",
      "code": "RES-20261030-01",
      "status": "Active",
      "dedicatedVehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "dedicatedNote": "DedicatedNote_value",
      "isActive": true
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.18 `PATCH` /api/slots/{id:int}/status

**Description:** Manually overrides the operational status of a slot bay (e.g. Available, Maintenance, Blocked).

- **Controller / Action:** `SlotsController.UpdateStatusAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin,Manager)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateSlotStatusModel`):**
  ```json
  {
    "status": "Active"
  }
  ```
- **Success Response Contract:** `ApiResult<bool>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": true,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin,Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.19 `GET` /api/zones

**Description:** Retrieves zone configurations across lots, optionally filtered by parking lot ID.

- **Controller / Action:** `ZonesController.GetAllAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin,Manager)
- **Query Parameters:**
  - `lotId` (int?, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IEnumerable<ZoneResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "parkingLotId": 1,
        "parkingLotCode": "RES-20261030-01",
        "parkingLotName": "Central City Parking",
        "code": "RES-20261030-01",
        "name": "Nguyen Van A",
        "zoneType": "ZoneType_value",
        "floorNo": 1,
        "capacity": 100,
        "maxReservationSlots": 1,
        "layoutJson": "LayoutJson_value",
        "isActive": true,
        "createdAt": "2026-10-30T08:00:00Z",
        "updatedAt": "2026-10-30T08:00:00Z",
        "totalSlots": 100
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin,Manager) or facility assignment scope.

#### 5.2.20 `GET` /api/lots/{lotId:int}/zones

**Description:** Retrieves all operational zones, floor levels, and category allocations for a specific parking lot.

- **Controller / Action:** `ZonesController.GetByLotIdAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin,Manager)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Success Response Contract:** `ApiResult<IEnumerable<ZoneResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "parkingLotId": 1,
        "parkingLotCode": "RES-20261030-01",
        "parkingLotName": "Central City Parking",
        "code": "RES-20261030-01",
        "name": "Nguyen Van A",
        "zoneType": "ZoneType_value",
        "floorNo": 1,
        "capacity": 100,
        "maxReservationSlots": 1,
        "layoutJson": "LayoutJson_value",
        "isActive": true,
        "createdAt": "2026-10-30T08:00:00Z",
        "updatedAt": "2026-10-30T08:00:00Z",
        "totalSlots": 100
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin,Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.21 `GET` /api/zones/{id:int}

**Description:** Retrieves detailed configuration and capacity thresholds for a specific zone.

- **Controller / Action:** `ZonesController.GetByIdAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin,Manager)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<ZoneResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "parkingLotId": 1,
      "parkingLotCode": "RES-20261030-01",
      "parkingLotName": "Central City Parking",
      "code": "RES-20261030-01",
      "name": "Nguyen Van A",
      "zoneType": "ZoneType_value",
      "floorNo": 1,
      "capacity": 100,
      "maxReservationSlots": 1,
      "layoutJson": "LayoutJson_value",
      "isActive": true,
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "totalSlots": 100
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin,Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.2.22 `PUT` /api/zones/{id:int}

**Description:** Updates zone display name, reservation share percentage, and operational parameters.

- **Controller / Action:** `ZonesController.UpdateAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateZoneModel`):**
  ```json
  {
    "name": "Nguyen Van A",
    "zoneType": "ZoneType_value",
    "floorNo": 1,
    "capacity": 100,
    "maxReservationSlots": 1,
    "layoutJson": "LayoutJson_value",
    "isActive": true
  }
  ```
- **Success Response Contract:** `ApiResult<ZoneResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "parkingLotId": 1,
      "parkingLotCode": "RES-20261030-01",
      "parkingLotName": "Central City Parking",
      "code": "RES-20261030-01",
      "name": "Nguyen Van A",
      "zoneType": "ZoneType_value",
      "floorNo": 1,
      "capacity": 100,
      "maxReservationSlots": 1,
      "layoutJson": "LayoutJson_value",
      "isActive": true,
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "totalSlots": 100
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

### 5.3 Module 03: Customer Reservations

#### 5.3.1 `POST` /api/reservations

**Description:** Books an advance parking reservation at a specified lot and slot, creating an initial deposit hold.

- **Controller / Action:** `ReservationsController.CreateAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Request Body (`CreateReservationModel`):**
  ```json
  {
    "parkingLotId": 1,
    "slotId": 1,
    "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "arrivalTime": "2026-10-30T08:00:00Z",
    "bookingMode": "Scheduled",
    "departureTime": "2026-10-30T08:00:00Z",
    "payDeposit": false
  }
  ```
- **Success Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`201 Created`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "Central City Parking",
      "parkingLotAddress": "123 Le Loi Street, District 1, HCMC",
      "parkingLotDistrict": "District 1",
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "Toyota Camry 2024",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "Scheduled",
      "departureTime": "2026-10-30T08:00:00Z",
      "isSlotHeld": false,
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "depositAmount": 50000,
      "hasDeposit": false,
      "holdMinutes": 60,
      "holdSeconds": 1,
      "depositDueAt": "2026-10-30T08:00:00Z",
      "refundAmount": 50000,
      "cancelReason": "Operational requirement",
      "cancelledBy": "CancelledBy_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "confirmedAt": "2026-10-30T08:00:00Z",
      "canChange": true,
      "canCancel": true,
      "refundIfCancelledNow": 10000,
      "pendingPayment": {
        "id": 1001,
        "code": "RES-20261030-01",
        "provider": "VnPay",
        "method": "Wallet",
        "purpose": "ReservationDeposit",
        "channel": "MobileApp",
        "amount": 50000,
        "status": "Active",
        "payUrl": "https://example.com/payment/return",
        "expiresAt": "2026-10-30T08:00:00Z",
        "createdAt": "2026-10-30T08:00:00Z",
        "paidAt": "2026-10-30T08:00:00Z",
        "reservationId": 1001,
        "sessionId": 1001,
        "refundedAmount": 50000,
        "refundOfPaymentId": 1001,
        "message": "Message_value",
        "exitPass": "ExitPass_value",
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "callbackDeadline": "2026-10-30T08:00:00Z"
      }
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `201 Created`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `409 Conflict`: Slot bay collision or conflicting active session state.

#### 5.3.2 `GET` /api/reservations/mine

**Description:** Retrieves active and historical reservations created by the authenticated customer.

- **Controller / Action:** `ReservationsController.GetMineAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Query Parameters:**
  - `activeOnly` (bool, optional, default: `false`): Filter parameter.
- **Success Response Contract:** `ApiResult<IReadOnlyList<ReservationResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1001,
        "code": "RES-20261030-01",
        "status": "Active",
        "parkingLotId": 1,
        "parkingLotName": "Central City Parking",
        "parkingLotAddress": "123 Le Loi Street, District 1, HCMC",
        "parkingLotDistrict": "District 1",
        "zoneId": 1,
        "zoneName": "Zone A",
        "slotId": 1,
        "slotCode": "RES-20261030-01",
        "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "plateNumber": "59A-123.45",
        "vehicleLabel": "Toyota Camry 2024",
        "arrivalTime": "2026-10-30T08:00:00Z",
        "bookingMode": "Scheduled",
        "departureTime": "2026-10-30T08:00:00Z",
        "isSlotHeld": false,
        "checkInFrom": "2026-10-30T08:00:00Z",
        "checkInUntil": "2026-10-30T08:00:00Z",
        "depositAmount": 50000,
        "hasDeposit": false,
        "holdMinutes": 60,
        "holdSeconds": 1,
        "depositDueAt": "2026-10-30T08:00:00Z",
        "refundAmount": 50000,
        "cancelReason": "Operational requirement",
        "cancelledBy": "CancelledBy_value",
        "createdAt": "2026-10-30T08:00:00Z",
        "confirmedAt": "2026-10-30T08:00:00Z",
        "canChange": true,
        "canCancel": true,
        "refundIfCancelledNow": 10000,
        "pendingPayment": {
          "id": 1001,
          "code": "RES-20261030-01",
          "provider": "VnPay",
          "method": "Wallet",
          "purpose": "ReservationDeposit",
          "channel": "MobileApp",
          "amount": 50000,
          "status": "Active",
          "payUrl": "https://example.com/payment/return",
          "expiresAt": "2026-10-30T08:00:00Z",
          "createdAt": "2026-10-30T08:00:00Z",
          "paidAt": "2026-10-30T08:00:00Z",
          "reservationId": 1001,
          "sessionId": 1001,
          "refundedAmount": 50000,
          "refundOfPaymentId": 1001,
          "message": "Message_value",
          "exitPass": "ExitPass_value",
          "exitWindowEndsAt": "2026-10-30T08:00:00Z",
          "callbackDeadline": "2026-10-30T08:00:00Z"
        }
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.

#### 5.3.3 `GET` /api/reservations/upcoming

**Description:** Retrieves the customer next upcoming confirmed reservation requiring check-in.

- **Controller / Action:** `ReservationsController.GetUpcomingAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Success Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "Central City Parking",
      "parkingLotAddress": "123 Le Loi Street, District 1, HCMC",
      "parkingLotDistrict": "District 1",
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "Toyota Camry 2024",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "Scheduled",
      "departureTime": "2026-10-30T08:00:00Z",
      "isSlotHeld": false,
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "depositAmount": 50000,
      "hasDeposit": false,
      "holdMinutes": 60,
      "holdSeconds": 1,
      "depositDueAt": "2026-10-30T08:00:00Z",
      "refundAmount": 50000,
      "cancelReason": "Operational requirement",
      "cancelledBy": "CancelledBy_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "confirmedAt": "2026-10-30T08:00:00Z",
      "canChange": true,
      "canCancel": true,
      "refundIfCancelledNow": 10000,
      "pendingPayment": {
        "id": 1001,
        "code": "RES-20261030-01",
        "provider": "VnPay",
        "method": "Wallet",
        "purpose": "ReservationDeposit",
        "channel": "MobileApp",
        "amount": 50000,
        "status": "Active",
        "payUrl": "https://example.com/payment/return",
        "expiresAt": "2026-10-30T08:00:00Z",
        "createdAt": "2026-10-30T08:00:00Z",
        "paidAt": "2026-10-30T08:00:00Z",
        "reservationId": 1001,
        "sessionId": 1001,
        "refundedAmount": 50000,
        "refundOfPaymentId": 1001,
        "message": "Message_value",
        "exitPass": "ExitPass_value",
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "callbackDeadline": "2026-10-30T08:00:00Z"
      }
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.

#### 5.3.4 `GET` /api/reservations/{id:long}

**Description:** Retrieves detailed reservation data, parking lot location, booked slot code, and deposit receipt.

- **Controller / Action:** `ReservationsController.GetAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "Central City Parking",
      "parkingLotAddress": "123 Le Loi Street, District 1, HCMC",
      "parkingLotDistrict": "District 1",
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "Toyota Camry 2024",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "Scheduled",
      "departureTime": "2026-10-30T08:00:00Z",
      "isSlotHeld": false,
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "depositAmount": 50000,
      "hasDeposit": false,
      "holdMinutes": 60,
      "holdSeconds": 1,
      "depositDueAt": "2026-10-30T08:00:00Z",
      "refundAmount": 50000,
      "cancelReason": "Operational requirement",
      "cancelledBy": "CancelledBy_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "confirmedAt": "2026-10-30T08:00:00Z",
      "canChange": true,
      "canCancel": true,
      "refundIfCancelledNow": 10000,
      "pendingPayment": {
        "id": 1001,
        "code": "RES-20261030-01",
        "provider": "VnPay",
        "method": "Wallet",
        "purpose": "ReservationDeposit",
        "channel": "MobileApp",
        "amount": 50000,
        "status": "Active",
        "payUrl": "https://example.com/payment/return",
        "expiresAt": "2026-10-30T08:00:00Z",
        "createdAt": "2026-10-30T08:00:00Z",
        "paidAt": "2026-10-30T08:00:00Z",
        "reservationId": 1001,
        "sessionId": 1001,
        "refundedAmount": 50000,
        "refundOfPaymentId": 1001,
        "message": "Message_value",
        "exitPass": "ExitPass_value",
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "callbackDeadline": "2026-10-30T08:00:00Z"
      }
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.3.5 `PUT` /api/reservations/{id:long}

**Description:** Modifies scheduled arrival time or requested slot before the arrival window begins.

- **Controller / Action:** `ReservationsController.UpdateAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateReservationModel`):**
  ```json
  {
    "arrivalTime": "2026-10-30T08:00:00Z",
    "bookingMode": "Scheduled",
    "departureTime": "2026-10-30T08:00:00Z",
    "slotId": 1
  }
  ```
- **Success Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "Central City Parking",
      "parkingLotAddress": "123 Le Loi Street, District 1, HCMC",
      "parkingLotDistrict": "District 1",
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "Toyota Camry 2024",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "Scheduled",
      "departureTime": "2026-10-30T08:00:00Z",
      "isSlotHeld": false,
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "depositAmount": 50000,
      "hasDeposit": false,
      "holdMinutes": 60,
      "holdSeconds": 1,
      "depositDueAt": "2026-10-30T08:00:00Z",
      "refundAmount": 50000,
      "cancelReason": "Operational requirement",
      "cancelledBy": "CancelledBy_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "confirmedAt": "2026-10-30T08:00:00Z",
      "canChange": true,
      "canCancel": true,
      "refundIfCancelledNow": 10000,
      "pendingPayment": {
        "id": 1001,
        "code": "RES-20261030-01",
        "provider": "VnPay",
        "method": "Wallet",
        "purpose": "ReservationDeposit",
        "channel": "MobileApp",
        "amount": 50000,
        "status": "Active",
        "payUrl": "https://example.com/payment/return",
        "expiresAt": "2026-10-30T08:00:00Z",
        "createdAt": "2026-10-30T08:00:00Z",
        "paidAt": "2026-10-30T08:00:00Z",
        "reservationId": 1001,
        "sessionId": 1001,
        "refundedAmount": 50000,
        "refundOfPaymentId": 1001,
        "message": "Message_value",
        "exitPass": "ExitPass_value",
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "callbackDeadline": "2026-10-30T08:00:00Z"
      }
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.3.6 `POST` /api/reservations/{id:long}/cancel

**Description:** Cancels a pending or confirmed reservation and calculates applicable deposit refund policies.

- **Controller / Action:** `ReservationsController.CancelAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`CancelReservationModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Success Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "Central City Parking",
      "parkingLotAddress": "123 Le Loi Street, District 1, HCMC",
      "parkingLotDistrict": "District 1",
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "Toyota Camry 2024",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "Scheduled",
      "departureTime": "2026-10-30T08:00:00Z",
      "isSlotHeld": false,
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "depositAmount": 50000,
      "hasDeposit": false,
      "holdMinutes": 60,
      "holdSeconds": 1,
      "depositDueAt": "2026-10-30T08:00:00Z",
      "refundAmount": 50000,
      "cancelReason": "Operational requirement",
      "cancelledBy": "CancelledBy_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "confirmedAt": "2026-10-30T08:00:00Z",
      "canChange": true,
      "canCancel": true,
      "refundIfCancelledNow": 10000,
      "pendingPayment": {
        "id": 1001,
        "code": "RES-20261030-01",
        "provider": "VnPay",
        "method": "Wallet",
        "purpose": "ReservationDeposit",
        "channel": "MobileApp",
        "amount": 50000,
        "status": "Active",
        "payUrl": "https://example.com/payment/return",
        "expiresAt": "2026-10-30T08:00:00Z",
        "createdAt": "2026-10-30T08:00:00Z",
        "paidAt": "2026-10-30T08:00:00Z",
        "reservationId": 1001,
        "sessionId": 1001,
        "refundedAmount": 50000,
        "refundOfPaymentId": 1001,
        "message": "Message_value",
        "exitPass": "ExitPass_value",
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "callbackDeadline": "2026-10-30T08:00:00Z"
      }
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.3.7 `GET` /api/reservations/{id:long}/qr

**Description:** Generates the encrypted QR entry pass credential for access through the automated gate barrier.

- **Controller / Action:** `ReservationsController.GetQrAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<ReservationQrModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "reservationId": 1001,
      "code": "RES-20261030-01",
      "qrPayload": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "validFrom": "2026-10-30T08:00:00Z",
      "validUntil": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.3.8 `POST` /api/reservations/{id:long}/payments

**Description:** Initiates the reservation deposit payment settlement via customer wallet or payment gateway.

- **Controller / Action:** `ReservationsController.PayDepositAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`CreatePaymentModel`):**
  ```json
  {
    "provider": "VnPay",
    "returnUrl": "https://example.com/payment/return"
  }
  ```
- **Success Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "VnPay",
      "method": "Wallet",
      "purpose": "ReservationDeposit",
      "channel": "MobileApp",
      "amount": 50000,
      "status": "Active",
      "payUrl": "https://example.com/payment/return",
      "expiresAt": "2026-10-30T08:00:00Z",
      "createdAt": "2026-10-30T08:00:00Z",
      "paidAt": "2026-10-30T08:00:00Z",
      "reservationId": 1001,
      "sessionId": 1001,
      "refundedAmount": 50000,
      "refundOfPaymentId": 1001,
      "message": "Message_value",
      "exitPass": "ExitPass_value",
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "callbackDeadline": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

### 5.4 Module 04: Customer Parking Sessions

#### 5.4.1 `GET` /api/parking-sessions/mine

**Description:** Retrieves active and historical parking sessions belonging to the authenticated customer.

- **Controller / Action:** `ParkingSessionsController.GetMineAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Success Response Contract:** `ApiResult<IReadOnlyList<SessionResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1001,
        "code": "RES-20261030-01",
        "parkingLotId": 1,
        "status": "Active",
        "sessionType": "SessionType_value",
        "plateNumber": "59A-123.45",
        "hasPlate": false,
        "vehicleTypeId": 1,
        "vehicleTypeName": "Nguyen Van A",
        "entryTime": "2026-10-30T08:00:00Z",
        "entryConfirmedAt": "2026-10-30T08:00:00Z",
        "entryGateId": 1,
        "zoneId": 1,
        "zoneName": "Zone A",
        "slotId": 1,
        "slotCode": "RES-20261030-01",
        "reservationId": 1001,
        "reservationCode": "RES-20261030-01",
        "isCustomerAccount": false,
        "paidAmount": 50000,
        "finalFee": 50000,
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "exitTime": "2026-10-30T08:00:00Z",
        "isManualEntry": false,
        "noPlateReason": "59A-123.45",
        "isLostTicket": false,
        "lostTicketStatus": "Active",
        "lostTicketReportedBy": "LostTicketReportedBy_value",
        "lostTicketReportedAt": "2026-10-30T08:00:00Z",
        "lostTicketReason": "Operational requirement",
        "lostTicketDecisionNote": "LostTicketDecisionNote_value",
        "plateMismatch": false,
        "isOverdue": false,
        "zeroFeeReason": "Operational requirement",
        "overrideNote": "OverrideNote_value"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.

#### 5.4.2 `GET` /api/parking-sessions/history

**Description:** Retrieves paginated historical parking sessions with total spending metrics and date/vehicle filters.

- **Controller / Action:** `ParkingSessionsController.GetHistoryAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Query Parameters:**
  - `query` (ParkingHistoryQuery, optional, default: `none`): Filter parameter.
- **Success Response Contract:** `ApiResult<ParkingHistoryPageModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "items": [
        {
          "sessionId": 1001,
          "sessionCode": "SES-20261030-01",
          "status": "Completed",
          "sessionType": "Transient",
          "entryTime": "2026-10-30T08:00:00Z",
          "exitTime": "2026-10-30T10:30:00Z",
          "durationMinutes": 150,
          "parkingLotId": 1,
          "lotName": "Central City Parking",
          "lotDistrict": "District 1",
          "lotAddress": "123 Le Loi Street, District 1, HCMC",
          "zoneName": "Zone A",
          "slotCode": "A-101",
          "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "plateNumber": "59A-123.45",
          "vehicleTypeName": "Car (4-7 seats)",
          "reservationCode": "RES-20261030-01",
          "finalFee": 45000,
          "depositApplied": 20000,
          "paidAmount": 25000,
          "receiptNumber": "RCP-20261030-01"
        }
      ],
      "page": 1,
      "pageSize": 10,
      "totalCount": 42,
      "totalSpent": 1850000
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.

#### 5.4.3 `GET` /api/parking-sessions/{id:long}/receipt

**Description:** Retrieves electronic parking receipt, itemized billing breakdown, and proof of payment.

- **Controller / Action:** `ParkingSessionsController.GetReceiptAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<ParkingReceiptModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "customerName": "Nguyen Van A",
      "lotPhone": "0987654321",
      "originalFee": 50000,
      "discountAmount": 50000,
      "zeroFeeReason": "Operational requirement",
      "issuedAt": "2026-10-30T08:00:00Z",
      "payments": [
        {
          "paymentCode": "RES-20261030-01",
          "purpose": "ReservationDeposit",
          "method": "Wallet",
          "provider": "VnPay",
          "amount": 50000,
          "paidAt": "2026-10-30T08:00:00Z"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.4.4 `GET` /api/parking-sessions/current

**Description:** Retrieves currently active, in-progress vehicle parking sessions for the authenticated customer.

- **Controller / Action:** `ParkingSessionsController.GetCurrentAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Success Response Contract:** `ApiResult<IReadOnlyList<CurrentSessionModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "session": {
          "id": 1001,
          "sessionCode": "SES-20261030-01",
          "parkingLotId": 1,
          "slotId": 12,
          "plateNumber": "59A-123.45",
          "checkInTime": "2026-10-30T08:00:00Z",
          "status": "Active"
        },
        "lotName": "Central City Parking",
        "lotAddress": "123 Le Loi Street, District 1, HCMC",
        "lotDistrict": "District 1",
        "entryGateName": "Entry Lane 1",
        "entryMethod": "Plate",
        "vehicleLabel": "Toyota Camry 2024",
        "bookingMode": "Scheduled",
        "plannedDeparture": "2026-10-30T12:00:00Z",
        "elapsedMinutes": 90,
        "feeSoFar": 30000,
        "depositApplied": 20000,
        "alreadyPaid": 0,
        "amountDue": 10000,
        "tariff": {
          "graceMinutes": 15,
          "blockMinutes": 60,
          "blockPrice": 20000,
          "maxDailyFee": 150000
        },
        "pendingPayment": null,
        "exitPass": null
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.

#### 5.4.5 `GET` /api/parking-sessions/{id:long}/exit-pass

**Description:** Generates a digitally signed exit pass token allowing barrier lift after full settlement.

- **Controller / Action:** `ParkingSessionsController.GetExitPassAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<ExitPassModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "sessionId": 1001,
      "sessionCode": "SES-20261030-01",
      "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
      "prepaid": false,
      "validUntil": "2026-10-30T08:00:00Z",
      "amountDue": 50000
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.4.6 `GET` /api/parking-sessions/{id:long}/quote

**Description:** Calculates the real-time parking fee quote, elapsed time, and deposit credit for an active session.

- **Controller / Action:** `ParkingSessionsController.QuoteAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<CheckoutQuoteModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "session": {
        "id": 1001,
        "code": "RES-20261030-01",
        "parkingLotId": 1,
        "status": "Active",
        "sessionType": "SessionType_value",
        "plateNumber": "59A-123.45",
        "hasPlate": false,
        "vehicleTypeId": 1,
        "vehicleTypeName": "Nguyen Van A",
        "entryTime": "2026-10-30T08:00:00Z",
        "entryConfirmedAt": "2026-10-30T08:00:00Z",
        "entryGateId": 1,
        "zoneId": 1,
        "zoneName": "Zone A",
        "slotId": 1,
        "slotCode": "RES-20261030-01",
        "reservationId": 1001,
        "reservationCode": "RES-20261030-01",
        "isCustomerAccount": false,
        "paidAmount": 50000,
        "finalFee": 50000,
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "exitTime": "2026-10-30T08:00:00Z",
        "isManualEntry": false,
        "noPlateReason": "59A-123.45",
        "isLostTicket": false,
        "lostTicketStatus": "Active",
        "lostTicketReportedBy": "LostTicketReportedBy_value",
        "lostTicketReportedAt": "2026-10-30T08:00:00Z",
        "lostTicketReason": "Operational requirement",
        "lostTicketDecisionNote": "LostTicketDecisionNote_value",
        "plateMismatch": false,
        "isOverdue": false,
        "zeroFeeReason": "Operational requirement",
        "overrideNote": "OverrideNote_value"
      },
      "durationMinutes": 60,
      "fee": 50000,
      "lostTicketFee": 50000,
      "exitWindowLateFee": 50000,
      "depositApplied": 10000,
      "alreadyPaid": 50000,
      "amountDue": 50000,
      "depositRefund": 10000,
      "zeroFeeReason": "Operational requirement",
      "canOpenBarrier": true,
      "alerts": [
        {
          "code": "RES-20261030-01",
          "message": "Message_value",
          "blocking": false,
          "canOverride": true,
          "relatedSessionId": 1001
        }
      ],
      "days": [
        {
          "date": null,
          "minutes": 60,
          "blocks": 1,
          "amount": 50000,
          "capped": false
        }
      ],
      "pricingRuleId": 1,
      "pricingRuleVersion": 1
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.4.7 `POST` /api/parking-sessions/{id:long}/payments

**Description:** Settles parking checkout fees online using digital wallet balance or gateway redirect.

- **Controller / Action:** `ParkingSessionsController.PayAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`CreatePaymentModel`):**
  ```json
  {
    "provider": "VnPay",
    "returnUrl": "https://example.com/payment/return"
  }
  ```
- **Success Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "VnPay",
      "method": "Wallet",
      "purpose": "ReservationDeposit",
      "channel": "MobileApp",
      "amount": 50000,
      "status": "Active",
      "payUrl": "https://example.com/payment/return",
      "expiresAt": "2026-10-30T08:00:00Z",
      "createdAt": "2026-10-30T08:00:00Z",
      "paidAt": "2026-10-30T08:00:00Z",
      "reservationId": 1001,
      "sessionId": 1001,
      "refundedAmount": 50000,
      "refundOfPaymentId": 1001,
      "message": "Message_value",
      "exitPass": "ExitPass_value",
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "callbackDeadline": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

### 5.5 Module 05: Staff Gate Operations & Gate Requests

#### 5.5.1 `POST` /api/reservations/{reservationId:long}/check-in-request

**Description:** Submits a customer manual check-in request when ANPR camera fails to read the license plate at the gate.

- **Controller / Action:** `GateRequestsController.RequestCheckInAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `reservationId` (required): Target reservationId identifier.
- **Success Response Contract:** `ApiResult<GateRequestModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "kind": "Kind_value",
      "status": "Active",
      "parkingLotId": 1,
      "lotName": "Central City Parking",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "Toyota Camry 2024",
      "vehicleColor": "VehicleColor_value",
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "customerName": "Nguyen Van A",
      "customerPhone": "0987654321",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "reservationStatus": "Active",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "slotCode": "RES-20261030-01",
      "sessionId": 1001,
      "sessionCode": "SES-20261030-01",
      "sessionStatus": "Active",
      "entryTime": "2026-10-30T08:00:00Z",
      "paidAmount": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "problem": "Problem_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "handledAt": "2026-10-30T08:00:00Z",
      "note": "Note_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.
  - `409 Conflict`: Slot bay collision or conflicting active session state.

#### 5.5.2 `POST` /api/parking-sessions/{sessionId:long}/check-out-request

**Description:** Submits a customer manual check-out request when license plate or exit pass barcode fails automated recognition.

- **Controller / Action:** `GateRequestsController.RequestCheckOutAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `sessionId` (required): Target sessionId identifier.
- **Success Response Contract:** `ApiResult<GateRequestModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "kind": "Kind_value",
      "status": "Active",
      "parkingLotId": 1,
      "lotName": "Central City Parking",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "Toyota Camry 2024",
      "vehicleColor": "VehicleColor_value",
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "customerName": "Nguyen Van A",
      "customerPhone": "0987654321",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "reservationStatus": "Active",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "slotCode": "RES-20261030-01",
      "sessionId": 1001,
      "sessionCode": "SES-20261030-01",
      "sessionStatus": "Active",
      "entryTime": "2026-10-30T08:00:00Z",
      "paidAmount": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "problem": "Problem_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "handledAt": "2026-10-30T08:00:00Z",
      "note": "Note_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.3 `GET` /api/gate-requests/mine

**Description:** Retrieves active and historical manual gate opening assistance requests submitted by the authenticated user.

- **Controller / Action:** `GateRequestsController.GetMineAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Success Response Contract:** `ApiResult<GateRequestModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "kind": "Kind_value",
      "status": "Active",
      "parkingLotId": 1,
      "lotName": "Central City Parking",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "Toyota Camry 2024",
      "vehicleColor": "VehicleColor_value",
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "customerName": "Nguyen Van A",
      "customerPhone": "0987654321",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "reservationStatus": "Active",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "slotCode": "RES-20261030-01",
      "sessionId": 1001,
      "sessionCode": "SES-20261030-01",
      "sessionStatus": "Active",
      "entryTime": "2026-10-30T08:00:00Z",
      "paidAmount": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "problem": "Problem_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "handledAt": "2026-10-30T08:00:00Z",
      "note": "Note_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.

#### 5.5.4 `POST` /api/gate-requests/{requestId:long}/cancel

**Description:** Cancels a pending manual gate opening assistance request submitted by the customer.

- **Controller / Action:** `GateRequestsController.CancelAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `requestId` (required): Target requestId identifier.
- **Success Response Contract:** `ApiResult<GateRequestModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "kind": "Kind_value",
      "status": "Active",
      "parkingLotId": 1,
      "lotName": "Central City Parking",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "Toyota Camry 2024",
      "vehicleColor": "VehicleColor_value",
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "customerName": "Nguyen Van A",
      "customerPhone": "0987654321",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "reservationStatus": "Active",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "slotCode": "RES-20261030-01",
      "sessionId": 1001,
      "sessionCode": "SES-20261030-01",
      "sessionStatus": "Active",
      "entryTime": "2026-10-30T08:00:00Z",
      "paidAmount": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "problem": "Problem_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "handledAt": "2026-10-30T08:00:00Z",
      "note": "Note_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.5 `GET` /api/staff/lots/{lotId:int}/gate-requests

**Description:** Retrieves the queue of customer manual gate assistance requests pending operator decision for a lot.

- **Controller / Action:** `GateRequestsController.GetQueueAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `kind` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IReadOnlyList<GateRequestModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1001,
        "kind": "Kind_value",
        "status": "Active",
        "parkingLotId": 1,
        "lotName": "Central City Parking",
        "plateNumber": "59A-123.45",
        "vehicleLabel": "Toyota Camry 2024",
        "vehicleColor": "VehicleColor_value",
        "vehicleTypeId": 1,
        "vehicleTypeName": "Nguyen Van A",
        "customerName": "Nguyen Van A",
        "customerPhone": "0987654321",
        "reservationId": 1001,
        "reservationCode": "RES-20261030-01",
        "reservationStatus": "Active",
        "arrivalTime": "2026-10-30T08:00:00Z",
        "checkInFrom": "2026-10-30T08:00:00Z",
        "checkInUntil": "2026-10-30T08:00:00Z",
        "slotCode": "RES-20261030-01",
        "sessionId": 1001,
        "sessionCode": "SES-20261030-01",
        "sessionStatus": "Active",
        "entryTime": "2026-10-30T08:00:00Z",
        "paidAmount": 50000,
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "problem": "Problem_value",
        "createdAt": "2026-10-30T08:00:00Z",
        "handledAt": "2026-10-30T08:00:00Z",
        "note": "Note_value"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.6 `POST` /api/staff/gate-requests/{requestId:long}/approve

**Description:** Approves a customer manual gate assistance request, verifying plate identity and commanding barrier opening.

- **Controller / Action:** `GateRequestsController.ApproveAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `requestId` (required): Target requestId identifier.
- **Request Body (`ApproveGateRequestModel`):**
  ```json
  {
    "gateId": 1,
    "plateConfirmed": false,
    "allowWhenFull": false,
    "allowPlateMismatch": false,
    "overrideReason": "Operational requirement"
  }
  ```
- **Success Response Contract:** `ApiResult<GateRequestDecisionModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "request": {
        "id": 1001,
        "kind": "Kind_value",
        "status": "Active",
        "parkingLotId": 1,
        "lotName": "Central City Parking",
        "plateNumber": "59A-123.45",
        "vehicleLabel": "Toyota Camry 2024",
        "vehicleColor": "VehicleColor_value",
        "vehicleTypeId": 1,
        "vehicleTypeName": "Nguyen Van A",
        "customerName": "Nguyen Van A",
        "customerPhone": "0987654321",
        "reservationId": 1001,
        "reservationCode": "RES-20261030-01",
        "reservationStatus": "Active",
        "arrivalTime": "2026-10-30T08:00:00Z",
        "checkInFrom": "2026-10-30T08:00:00Z",
        "checkInUntil": "2026-10-30T08:00:00Z",
        "slotCode": "RES-20261030-01",
        "sessionId": 1001,
        "sessionCode": "SES-20261030-01",
        "sessionStatus": "Active",
        "entryTime": "2026-10-30T08:00:00Z",
        "paidAmount": 50000,
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "problem": "Problem_value",
        "createdAt": "2026-10-30T08:00:00Z",
        "handledAt": "2026-10-30T08:00:00Z",
        "note": "Note_value"
      },
      "barrierOpen": true,
      "checkIn": {
        "outcome": "Outcome_value",
        "barrierOpen": true,
        "alerts": [],
        "session": null,
        "ticket": "Ticket_value"
      },
      "checkOut": {
        "session": null,
        "barrierOpen": true,
        "depositRefund": 10000,
        "alerts": []
      },
      "alerts": [
        {
          "code": "RES-20261030-01",
          "message": "Message_value",
          "blocking": false,
          "canOverride": true,
          "relatedSessionId": 1001
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.7 `POST` /api/staff/gate-requests/{requestId:long}/refuse

**Description:** Rejects a customer manual gate assistance request with a recorded justification code.

- **Controller / Action:** `GateRequestsController.RefuseAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `requestId` (required): Target requestId identifier.
- **Request Body (`RefuseGateRequestModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Success Response Contract:** `ApiResult<GateRequestModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "kind": "Kind_value",
      "status": "Active",
      "parkingLotId": 1,
      "lotName": "Central City Parking",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "Toyota Camry 2024",
      "vehicleColor": "VehicleColor_value",
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "customerName": "Nguyen Van A",
      "customerPhone": "0987654321",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "reservationStatus": "Active",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "slotCode": "RES-20261030-01",
      "sessionId": 1001,
      "sessionCode": "SES-20261030-01",
      "sessionStatus": "Active",
      "entryTime": "2026-10-30T08:00:00Z",
      "paidAmount": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "problem": "Problem_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "handledAt": "2026-10-30T08:00:00Z",
      "note": "Note_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.8 `GET` /api/staff/lots

**Description:** Retrieves the parking lot assigned to the authenticated staff operator.

- **Controller / Action:** `StaffController.GetLotsAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff)
- **Success Response Contract:** `ApiResult<IEnumerable<AssignedParkingLotModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "code": "RES-20261030-01",
        "name": "Nguyen Van A"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff) or facility assignment scope.

#### 5.5.9 `GET` /api/staff/lots/{lotId:int}/gates

**Description:** Retrieves gate lanes, barrier controllers, and camera feeds configured at a parking lot.

- **Controller / Action:** `StaffGateController.GetGatesAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Success Response Contract:** `ApiResult<IReadOnlyList<GateResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "code": "RES-20261030-01",
        "name": "Nguyen Van A",
        "direction": "Direction_value"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.10 `GET` /api/staff/lots/{lotId:int}/zone-counters

**Description:** Retrieves real-time capacity and occupancy counters for all zones in a parking lot.

- **Controller / Action:** `StaffGateController.GetZoneCountersAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Success Response Contract:** `ApiResult<IReadOnlyList<ZoneCounterModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "zoneId": 1,
        "zoneCode": "RES-20261030-01",
        "zoneName": "Zone A",
        "vehicleTypeId": 1,
        "totalCapacity": 100,
        "occupiedSpaces": 100,
        "availableSpaces": 100
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.11 `GET` /api/staff/lots/{lotId:int}/sessions

**Description:** Retrieves open and active parking sessions at a lot, with optional license plate search.

- **Controller / Action:** `StaffGateController.GetSessionsAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `status` (string, optional, default: `null`): Filter parameter.
  - `plate` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IReadOnlyList<SessionResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1001,
        "code": "RES-20261030-01",
        "parkingLotId": 1,
        "status": "Active",
        "sessionType": "SessionType_value",
        "plateNumber": "59A-123.45",
        "hasPlate": false,
        "vehicleTypeId": 1,
        "vehicleTypeName": "Nguyen Van A",
        "entryTime": "2026-10-30T08:00:00Z",
        "entryConfirmedAt": "2026-10-30T08:00:00Z",
        "entryGateId": 1,
        "zoneId": 1,
        "zoneName": "Zone A",
        "slotId": 1,
        "slotCode": "RES-20261030-01",
        "reservationId": 1001,
        "reservationCode": "RES-20261030-01",
        "isCustomerAccount": false,
        "paidAmount": 50000,
        "finalFee": 50000,
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "exitTime": "2026-10-30T08:00:00Z",
        "isManualEntry": false,
        "noPlateReason": "59A-123.45",
        "isLostTicket": false,
        "lostTicketStatus": "Active",
        "lostTicketReportedBy": "LostTicketReportedBy_value",
        "lostTicketReportedAt": "2026-10-30T08:00:00Z",
        "lostTicketReason": "Operational requirement",
        "lostTicketDecisionNote": "LostTicketDecisionNote_value",
        "plateMismatch": false,
        "isOverdue": false,
        "zeroFeeReason": "Operational requirement",
        "overrideNote": "OverrideNote_value"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.12 `GET` /api/staff/lots/{lotId:int}/sessions/history

**Description:** Retrieves paginated historical sessions for audit and discrepancy review at the gate.

- **Controller / Action:** `StaffGateController.GetHistoryAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `plate` (string, optional, default: `null`): Filter parameter.
  - `page` (int, optional, default: `1`): Filter parameter.
  - `pageSize` (int, optional, default: `10`): Filter parameter.
- **Success Response Contract:** `ApiResult<StaffSessionHistoryPageModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "items": [
        {
          "id": 1001,
          "code": "RES-20261030-01",
          "parkingLotId": 1,
          "status": "Active",
          "sessionType": "SessionType_value",
          "plateNumber": "59A-123.45",
          "hasPlate": false,
          "vehicleTypeId": 1,
          "vehicleTypeName": "Nguyen Van A",
          "entryTime": "2026-10-30T08:00:00Z",
          "entryConfirmedAt": "2026-10-30T08:00:00Z",
          "entryGateId": 1,
          "zoneId": 1,
          "zoneName": "Zone A",
          "slotId": 1,
          "slotCode": "RES-20261030-01",
          "reservationId": 1001,
          "reservationCode": "RES-20261030-01",
          "isCustomerAccount": false,
          "paidAmount": 50000,
          "finalFee": 50000,
          "exitWindowEndsAt": "2026-10-30T08:00:00Z",
          "exitTime": "2026-10-30T08:00:00Z",
          "isManualEntry": false,
          "noPlateReason": "59A-123.45",
          "isLostTicket": false,
          "lostTicketStatus": "Active",
          "lostTicketReportedBy": "LostTicketReportedBy_value",
          "lostTicketReportedAt": "2026-10-30T08:00:00Z",
          "lostTicketReason": "Operational requirement",
          "lostTicketDecisionNote": "LostTicketDecisionNote_value",
          "plateMismatch": false,
          "isOverdue": false,
          "zeroFeeReason": "Operational requirement",
          "overrideNote": "OverrideNote_value"
        }
      ],
      "page": 1,
      "pageSize": 1,
      "totalCount": 100
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.13 `GET` /api/staff/lots/{lotId:int}/payments

**Description:** Retrieves settled fee payments collected at gate lanes, with cash/digital filters.

- **Controller / Action:** `StaffGateController.GetPaymentHistoryAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `method` (string, optional, default: `null`): Filter parameter.
  - `plate` (string, optional, default: `null`): Filter parameter.
  - `page` (int, optional, default: `1`): Filter parameter.
  - `pageSize` (int, optional, default: `10`): Filter parameter.
- **Success Response Contract:** `ApiResult<StaffPaymentHistoryPageModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "items": [
        {
          "id": 1001,
          "code": "RES-20261030-01",
          "paidAt": "2026-10-30T08:00:00Z",
          "sessionId": 1001,
          "sessionCode": "SES-20261030-01",
          "plateNumber": "59A-123.45",
          "method": "Wallet",
          "channel": "MobileApp",
          "amount": 50000,
          "status": "Active",
          "collectedByName": "Nguyen Van A"
        }
      ],
      "page": 1,
      "pageSize": 1,
      "totalCount": 100
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.14 `GET` /api/staff/lots/{lotId:int}/sessions/lookup

**Description:** Looks up an active parking session at the exit lane by vehicle plate number, ticket QR code, or session code.

- **Controller / Action:** `StaffGateController.LookupAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `plate` (string, optional, default: `null`): Filter parameter.
  - `ticket` (string, optional, default: `null`): Filter parameter.
  - `code` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<SessionResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "parkingLotId": 1,
      "status": "Active",
      "sessionType": "SessionType_value",
      "plateNumber": "59A-123.45",
      "hasPlate": false,
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "isCustomerAccount": false,
      "paidAmount": 50000,
      "finalFee": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "exitTime": "2026-10-30T08:00:00Z",
      "isManualEntry": false,
      "noPlateReason": "59A-123.45",
      "isLostTicket": false,
      "lostTicketStatus": "Active",
      "lostTicketReportedBy": "LostTicketReportedBy_value",
      "lostTicketReportedAt": "2026-10-30T08:00:00Z",
      "lostTicketReason": "Operational requirement",
      "lostTicketDecisionNote": "LostTicketDecisionNote_value",
      "plateMismatch": false,
      "isOverdue": false,
      "zeroFeeReason": "Operational requirement",
      "overrideNote": "OverrideNote_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.15 `GET` /api/staff/lots/{lotId:int}/reservations

**Description:** Retrieves today incoming and active advance reservations scheduled at a specified parking lot.

- **Controller / Action:** `StaffGateController.GetReservationsAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `status` (string, optional, default: `null`): Filter parameter.
  - `from` (DateTime?, optional, default: `null`): Filter parameter.
  - `to` (DateTime?, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IReadOnlyList<ReservationResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1001,
        "code": "RES-20261030-01",
        "status": "Active",
        "parkingLotId": 1,
        "parkingLotName": "Central City Parking",
        "parkingLotAddress": "123 Le Loi Street, District 1, HCMC",
        "parkingLotDistrict": "District 1",
        "zoneId": 1,
        "zoneName": "Zone A",
        "slotId": 1,
        "slotCode": "RES-20261030-01",
        "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "plateNumber": "59A-123.45",
        "vehicleLabel": "Toyota Camry 2024",
        "arrivalTime": "2026-10-30T08:00:00Z",
        "bookingMode": "Scheduled",
        "departureTime": "2026-10-30T08:00:00Z",
        "isSlotHeld": false,
        "checkInFrom": "2026-10-30T08:00:00Z",
        "checkInUntil": "2026-10-30T08:00:00Z",
        "depositAmount": 50000,
        "hasDeposit": false,
        "holdMinutes": 60,
        "holdSeconds": 1,
        "depositDueAt": "2026-10-30T08:00:00Z",
        "refundAmount": 50000,
        "cancelReason": "Operational requirement",
        "cancelledBy": "CancelledBy_value",
        "createdAt": "2026-10-30T08:00:00Z",
        "confirmedAt": "2026-10-30T08:00:00Z",
        "canChange": true,
        "canCancel": true,
        "refundIfCancelledNow": 10000,
        "pendingPayment": {
          "id": 1001,
          "code": "RES-20261030-01",
          "provider": "VnPay",
          "method": "Wallet",
          "purpose": "ReservationDeposit",
          "channel": "MobileApp",
          "amount": 50000,
          "status": "Active",
          "payUrl": "https://example.com/payment/return",
          "expiresAt": "2026-10-30T08:00:00Z",
          "createdAt": "2026-10-30T08:00:00Z",
          "paidAt": "2026-10-30T08:00:00Z",
          "reservationId": 1001,
          "sessionId": 1001,
          "refundedAmount": 50000,
          "refundOfPaymentId": 1001,
          "message": "Message_value",
          "exitPass": "ExitPass_value",
          "exitWindowEndsAt": "2026-10-30T08:00:00Z",
          "callbackDeadline": "2026-10-30T08:00:00Z"
        }
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.16 `GET` /api/staff/lots/{lotId:int}/reservations/lookup

**Description:** Looks up an advance reservation at the entry gate by reservation code, customer plate, or phone number.

- **Controller / Action:** `StaffGateController.LookupReservationAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `code` (string, optional, default: `null`): Filter parameter.
  - `qr` (string, optional, default: `null`): Filter parameter.
  - `plate` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<GateReservationLookupModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "reservation": {
        "id": 1001,
        "code": "RES-20261030-01",
        "status": "Active",
        "parkingLotId": 1,
        "parkingLotName": "Central City Parking",
        "parkingLotAddress": "123 Le Loi Street, District 1, HCMC",
        "parkingLotDistrict": "District 1",
        "zoneId": 1,
        "zoneName": "Zone A",
        "slotId": 1,
        "slotCode": "RES-20261030-01",
        "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "plateNumber": "59A-123.45",
        "vehicleLabel": "Toyota Camry 2024",
        "arrivalTime": "2026-10-30T08:00:00Z",
        "bookingMode": "Scheduled",
        "departureTime": "2026-10-30T08:00:00Z",
        "isSlotHeld": false,
        "checkInFrom": "2026-10-30T08:00:00Z",
        "checkInUntil": "2026-10-30T08:00:00Z",
        "depositAmount": 50000,
        "hasDeposit": false,
        "holdMinutes": 60,
        "holdSeconds": 1,
        "depositDueAt": "2026-10-30T08:00:00Z",
        "refundAmount": 50000,
        "cancelReason": "Operational requirement",
        "cancelledBy": "CancelledBy_value",
        "createdAt": "2026-10-30T08:00:00Z",
        "confirmedAt": "2026-10-30T08:00:00Z",
        "canChange": true,
        "canCancel": true,
        "refundIfCancelledNow": 10000,
        "pendingPayment": null
      },
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "vehicleBrand": "VehicleBrand_value",
      "vehicleModel": "VehicleModel_value",
      "vehicleColor": "VehicleColor_value",
      "customerName": "Nguyen Van A",
      "canCheckIn": true,
      "problem": "Problem_value",
      "openSessionCode": "SES-20261030-01"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.17 `GET` /api/staff/lots/{lotId:int}/plate-lists

**Description:** Retrieves active Whitelist and Blacklist license plate entries enforced at a parking lot.

- **Controller / Action:** `StaffGateController.GetPlateListsAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `listType` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IReadOnlyList<PlateListEntryModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "listType": "ListType_value",
        "plateNumber": "59A-123.45",
        "parkingLotId": 1,
        "reason": "Operational requirement",
        "expiresAt": "2026-10-30T08:00:00Z",
        "isActive": true,
        "createdAt": "2026-10-30T08:00:00Z"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.18 `POST` /api/staff/lots/{lotId:int}/check-in

**Description:** Executes vehicle check-in at a gate lane, validates reservations and blacklists, and pulses barrier open.

- **Controller / Action:** `StaffGateController.CheckInAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Request Body (`CheckInModel`):**
  ```json
  {
    "gateId": 1,
    "plateNumber": "59A-123.45",
    "noPlate": false,
    "noPlateReason": "59A-123.45",
    "vehicleTypeId": 1,
    "anprConfidence": 0.98,
    "plateConfirmed": false,
    "reservationQr": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "reservationCode": "RES-20261030-01",
    "ignoreReservation": false,
    "slotId": 1,
    "isManual": false,
    "manualReason": "Operational requirement",
    "allowBlacklisted": false,
    "allowWhenFull": false,
    "allowPlateMismatch": false,
    "overrideReason": "Operational requirement",
    "imageUrl": "https://example.com/payment/return"
  }
  ```
- **Success Response Contract:** `ApiResult<CheckInResultModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "outcome": "Outcome_value",
      "barrierOpen": true,
      "alerts": [
        {
          "code": "RES-20261030-01",
          "message": "Message_value",
          "blocking": false,
          "canOverride": true,
          "relatedSessionId": 1001
        }
      ],
      "session": {
        "id": 1001,
        "code": "RES-20261030-01",
        "parkingLotId": 1,
        "status": "Active",
        "sessionType": "SessionType_value",
        "plateNumber": "59A-123.45",
        "hasPlate": false,
        "vehicleTypeId": 1,
        "vehicleTypeName": "Nguyen Van A",
        "entryTime": "2026-10-30T08:00:00Z",
        "entryConfirmedAt": "2026-10-30T08:00:00Z",
        "entryGateId": 1,
        "zoneId": 1,
        "zoneName": "Zone A",
        "slotId": 1,
        "slotCode": "RES-20261030-01",
        "reservationId": 1001,
        "reservationCode": "RES-20261030-01",
        "isCustomerAccount": false,
        "paidAmount": 50000,
        "finalFee": 50000,
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "exitTime": "2026-10-30T08:00:00Z",
        "isManualEntry": false,
        "noPlateReason": "59A-123.45",
        "isLostTicket": false,
        "lostTicketStatus": "Active",
        "lostTicketReportedBy": "LostTicketReportedBy_value",
        "lostTicketReportedAt": "2026-10-30T08:00:00Z",
        "lostTicketReason": "Operational requirement",
        "lostTicketDecisionNote": "LostTicketDecisionNote_value",
        "plateMismatch": false,
        "isOverdue": false,
        "zeroFeeReason": "Operational requirement",
        "overrideNote": "OverrideNote_value"
      },
      "ticket": "Ticket_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.
  - `409 Conflict`: Slot bay collision or conflicting active session state.

#### 5.5.19 `GET` /api/staff/sessions/{id:long}

**Description:** Retrieves operational details and telemetry for an active or completed parking session at a lot.

- **Controller / Action:** `StaffGateController.GetSessionAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<SessionResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "parkingLotId": 1,
      "status": "Active",
      "sessionType": "SessionType_value",
      "plateNumber": "59A-123.45",
      "hasPlate": false,
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "isCustomerAccount": false,
      "paidAmount": 50000,
      "finalFee": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "exitTime": "2026-10-30T08:00:00Z",
      "isManualEntry": false,
      "noPlateReason": "59A-123.45",
      "isLostTicket": false,
      "lostTicketStatus": "Active",
      "lostTicketReportedBy": "LostTicketReportedBy_value",
      "lostTicketReportedAt": "2026-10-30T08:00:00Z",
      "lostTicketReason": "Operational requirement",
      "lostTicketDecisionNote": "LostTicketDecisionNote_value",
      "plateMismatch": false,
      "isOverdue": false,
      "zeroFeeReason": "Operational requirement",
      "overrideNote": "OverrideNote_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.20 `POST` /api/staff/sessions/{id:long}/entry-confirm

**Description:** Confirms that a vehicle has physically crossed the lane barrier into the facility.

- **Controller / Action:** `StaffGateController.ConfirmEntryAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<SessionResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "parkingLotId": 1,
      "status": "Active",
      "sessionType": "SessionType_value",
      "plateNumber": "59A-123.45",
      "hasPlate": false,
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "isCustomerAccount": false,
      "paidAmount": 50000,
      "finalFee": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "exitTime": "2026-10-30T08:00:00Z",
      "isManualEntry": false,
      "noPlateReason": "59A-123.45",
      "isLostTicket": false,
      "lostTicketStatus": "Active",
      "lostTicketReportedBy": "LostTicketReportedBy_value",
      "lostTicketReportedAt": "2026-10-30T08:00:00Z",
      "lostTicketReason": "Operational requirement",
      "lostTicketDecisionNote": "LostTicketDecisionNote_value",
      "plateMismatch": false,
      "isOverdue": false,
      "zeroFeeReason": "Operational requirement",
      "overrideNote": "OverrideNote_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.21 `POST` /api/staff/sessions/{id:long}/entry-cancel

**Description:** Cancels a pending entry check-in when a vehicle reverses or is rejected at the gate lane.

- **Controller / Action:** `StaffGateController.CancelEntryAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<SessionResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "parkingLotId": 1,
      "status": "Active",
      "sessionType": "SessionType_value",
      "plateNumber": "59A-123.45",
      "hasPlate": false,
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "isCustomerAccount": false,
      "paidAmount": 50000,
      "finalFee": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "exitTime": "2026-10-30T08:00:00Z",
      "isManualEntry": false,
      "noPlateReason": "59A-123.45",
      "isLostTicket": false,
      "lostTicketStatus": "Active",
      "lostTicketReportedBy": "LostTicketReportedBy_value",
      "lostTicketReportedAt": "2026-10-30T08:00:00Z",
      "lostTicketReason": "Operational requirement",
      "lostTicketDecisionNote": "LostTicketDecisionNote_value",
      "plateMismatch": false,
      "isOverdue": false,
      "zeroFeeReason": "Operational requirement",
      "overrideNote": "OverrideNote_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.22 `POST` /api/staff/sessions/{id:long}/quote

**Description:** Calculates parking fees, lost-ticket fines, and overtime charges at the exit lane.

- **Controller / Action:** `StaffGateController.QuoteAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`CheckoutQuoteRequestModel`):**
  ```json
  {
    "gateId": 1,
    "plateNumber": "59A-123.45",
    "anprConfidence": 0.98
  }
  ```
- **Success Response Contract:** `ApiResult<CheckoutQuoteModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "session": {
        "id": 1001,
        "code": "RES-20261030-01",
        "parkingLotId": 1,
        "status": "Active",
        "sessionType": "SessionType_value",
        "plateNumber": "59A-123.45",
        "hasPlate": false,
        "vehicleTypeId": 1,
        "vehicleTypeName": "Nguyen Van A",
        "entryTime": "2026-10-30T08:00:00Z",
        "entryConfirmedAt": "2026-10-30T08:00:00Z",
        "entryGateId": 1,
        "zoneId": 1,
        "zoneName": "Zone A",
        "slotId": 1,
        "slotCode": "RES-20261030-01",
        "reservationId": 1001,
        "reservationCode": "RES-20261030-01",
        "isCustomerAccount": false,
        "paidAmount": 50000,
        "finalFee": 50000,
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "exitTime": "2026-10-30T08:00:00Z",
        "isManualEntry": false,
        "noPlateReason": "59A-123.45",
        "isLostTicket": false,
        "lostTicketStatus": "Active",
        "lostTicketReportedBy": "LostTicketReportedBy_value",
        "lostTicketReportedAt": "2026-10-30T08:00:00Z",
        "lostTicketReason": "Operational requirement",
        "lostTicketDecisionNote": "LostTicketDecisionNote_value",
        "plateMismatch": false,
        "isOverdue": false,
        "zeroFeeReason": "Operational requirement",
        "overrideNote": "OverrideNote_value"
      },
      "durationMinutes": 60,
      "fee": 50000,
      "lostTicketFee": 50000,
      "exitWindowLateFee": 50000,
      "depositApplied": 10000,
      "alreadyPaid": 50000,
      "amountDue": 50000,
      "depositRefund": 10000,
      "zeroFeeReason": "Operational requirement",
      "canOpenBarrier": true,
      "alerts": [
        {
          "code": "RES-20261030-01",
          "message": "Message_value",
          "blocking": false,
          "canOverride": true,
          "relatedSessionId": 1001
        }
      ],
      "days": [
        {
          "date": null,
          "minutes": 60,
          "blocks": 1,
          "amount": 50000,
          "capped": false
        }
      ],
      "pricingRuleId": 1,
      "pricingRuleVersion": 1
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.23 `POST` /api/staff/sessions/{id:long}/cancel-payment

**Description:** Cancels an in-progress payment attempt at the gate, resetting lane terminal state.

- **Controller / Action:** `StaffGateController.CancelPaymentAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<SessionResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "parkingLotId": 1,
      "status": "Active",
      "sessionType": "SessionType_value",
      "plateNumber": "59A-123.45",
      "hasPlate": false,
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "isCustomerAccount": false,
      "paidAmount": 50000,
      "finalFee": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "exitTime": "2026-10-30T08:00:00Z",
      "isManualEntry": false,
      "noPlateReason": "59A-123.45",
      "isLostTicket": false,
      "lostTicketStatus": "Active",
      "lostTicketReportedBy": "LostTicketReportedBy_value",
      "lostTicketReportedAt": "2026-10-30T08:00:00Z",
      "lostTicketReason": "Operational requirement",
      "lostTicketDecisionNote": "LostTicketDecisionNote_value",
      "plateMismatch": false,
      "isOverdue": false,
      "zeroFeeReason": "Operational requirement",
      "overrideNote": "OverrideNote_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.24 `POST` /api/staff/sessions/{id:long}/payments/cash

**Description:** Records physical cash fee collection by gate staff and authorizes exit barrier opening.

- **Controller / Action:** `StaffGateController.PayCashAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`CashPaymentModel`):**
  ```json
  {
    "amount": 50000,
    "prepaid": false
  }
  ```
- **Success Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "VnPay",
      "method": "Wallet",
      "purpose": "ReservationDeposit",
      "channel": "MobileApp",
      "amount": 50000,
      "status": "Active",
      "payUrl": "https://example.com/payment/return",
      "expiresAt": "2026-10-30T08:00:00Z",
      "createdAt": "2026-10-30T08:00:00Z",
      "paidAt": "2026-10-30T08:00:00Z",
      "reservationId": 1001,
      "sessionId": 1001,
      "refundedAmount": 50000,
      "refundOfPaymentId": 1001,
      "message": "Message_value",
      "exitPass": "ExitPass_value",
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "callbackDeadline": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.25 `POST` /api/staff/sessions/{id:long}/payments/online

**Description:** Initiates an in-person QR payment at the gate lane via payment gateway.

- **Controller / Action:** `StaffGateController.PayOnlineAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`LanePaymentModel`):**
  ```json
  {
    "provider": "VnPay",
    "prepaid": false
  }
  ```
- **Success Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "VnPay",
      "method": "Wallet",
      "purpose": "ReservationDeposit",
      "channel": "MobileApp",
      "amount": 50000,
      "status": "Active",
      "payUrl": "https://example.com/payment/return",
      "expiresAt": "2026-10-30T08:00:00Z",
      "createdAt": "2026-10-30T08:00:00Z",
      "paidAt": "2026-10-30T08:00:00Z",
      "reservationId": 1001,
      "sessionId": 1001,
      "refundedAmount": 50000,
      "refundOfPaymentId": 1001,
      "message": "Message_value",
      "exitPass": "ExitPass_value",
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "callbackDeadline": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.26 `POST` /api/staff/sessions/{id:long}/check-out

**Description:** Finalizes parking session checkout at the gate, validating payment clearance and commanding exit barrier opening.

- **Controller / Action:** `StaffGateController.CheckoutAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`CheckoutModel`):**
  ```json
  {
    "gateId": 1,
    "plateNumber": "59A-123.45",
    "photoChecked": false,
    "allowPlateMismatch": false,
    "overrideReason": "Operational requirement",
    "isManual": false,
    "manualReason": "Operational requirement"
  }
  ```
- **Success Response Contract:** `ApiResult<CheckoutResultModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "session": {
        "id": 1001,
        "code": "RES-20261030-01",
        "parkingLotId": 1,
        "status": "Active",
        "sessionType": "SessionType_value",
        "plateNumber": "59A-123.45",
        "hasPlate": false,
        "vehicleTypeId": 1,
        "vehicleTypeName": "Nguyen Van A",
        "entryTime": "2026-10-30T08:00:00Z",
        "entryConfirmedAt": "2026-10-30T08:00:00Z",
        "entryGateId": 1,
        "zoneId": 1,
        "zoneName": "Zone A",
        "slotId": 1,
        "slotCode": "RES-20261030-01",
        "reservationId": 1001,
        "reservationCode": "RES-20261030-01",
        "isCustomerAccount": false,
        "paidAmount": 50000,
        "finalFee": 50000,
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "exitTime": "2026-10-30T08:00:00Z",
        "isManualEntry": false,
        "noPlateReason": "59A-123.45",
        "isLostTicket": false,
        "lostTicketStatus": "Active",
        "lostTicketReportedBy": "LostTicketReportedBy_value",
        "lostTicketReportedAt": "2026-10-30T08:00:00Z",
        "lostTicketReason": "Operational requirement",
        "lostTicketDecisionNote": "LostTicketDecisionNote_value",
        "plateMismatch": false,
        "isOverdue": false,
        "zeroFeeReason": "Operational requirement",
        "overrideNote": "OverrideNote_value"
      },
      "barrierOpen": true,
      "depositRefund": 10000,
      "alerts": [
        {
          "code": "RES-20261030-01",
          "message": "Message_value",
          "blocking": false,
          "canOverride": true,
          "relatedSessionId": 1001
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.5.27 `POST` /api/staff/sessions/{id:long}/lost-ticket

**Description:** Initiates a lost ticket resolution workflow at the exit gate, applying penalty tariffs per lot policy.

- **Controller / Action:** `StaffGateController.LostTicketAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`LostTicketModel`):**
  ```json
  {
    "reason": "Operational requirement",
    "conditionsMet": false
  }
  ```
- **Success Response Contract:** `ApiResult<SessionResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "parkingLotId": 1,
      "status": "Active",
      "sessionType": "SessionType_value",
      "plateNumber": "59A-123.45",
      "hasPlate": false,
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "isCustomerAccount": false,
      "paidAmount": 50000,
      "finalFee": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "exitTime": "2026-10-30T08:00:00Z",
      "isManualEntry": false,
      "noPlateReason": "59A-123.45",
      "isLostTicket": false,
      "lostTicketStatus": "Active",
      "lostTicketReportedBy": "LostTicketReportedBy_value",
      "lostTicketReportedAt": "2026-10-30T08:00:00Z",
      "lostTicketReason": "Operational requirement",
      "lostTicketDecisionNote": "LostTicketDecisionNote_value",
      "plateMismatch": false,
      "isOverdue": false,
      "zeroFeeReason": "Operational requirement",
      "overrideNote": "OverrideNote_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

### 5.6 Module 06: Manager Operations & Pricing

#### 5.6.1 `GET` /api/manager/lots

**Description:** Retrieves the list of parking lots assigned to and managed by the authenticated manager.

- **Controller / Action:** `ManagerController.GetLotsAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager)
- **Success Response Contract:** `ApiResult<IEnumerable<AssignedParkingLotModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "code": "RES-20261030-01",
        "name": "Nguyen Van A"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager) or facility assignment scope.

#### 5.6.2 `GET` /api/manager/staff

**Description:** Retrieves staff operators assigned to lots under the authenticated manager jurisdiction.

- **Controller / Action:** `ManagerController.GetStaffAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager)
- **Query Parameters:**
  - `parkingLotId` (int?, optional, default: `null`): Filter parameter.
  - `isActive` (bool?, optional, default: `null`): Filter parameter.
  - `searchTerm` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IEnumerable<EmployeeAccountResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": "Id_value",
        "userName": "UserName_value",
        "fullName": "Nguyen Van A",
        "email": "user@example.com",
        "phoneNumber": "0987654321",
        "role": "Staff",
        "employeeCode": "RES-20261030-01",
        "isActive": true,
        "status": "Active",
        "createdAt": "2026-10-30T08:00:00Z",
        "parkingLots": [
          {
            "id": 1,
            "code": "RES-20261030-01",
            "name": "Nguyen Van A"
          }
        ]
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager) or facility assignment scope.

#### 5.6.3 `GET` /api/manager/staff/{id}

**Description:** Retrieves account details and assigned parking lot for a specific staff operator managed by the current user.

- **Controller / Action:** `ManagerController.GetStaffByIdAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "isActive": true,
      "status": "Active",
      "createdAt": "2026-10-30T08:00:00Z",
      "parkingLots": [
        {
          "id": 1,
          "code": "RES-20261030-01",
          "name": "Nguyen Van A"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.4 `POST` /api/manager/staff

**Description:** Provisions a new staff gate operator account assigned to a managed parking lot.

- **Controller / Action:** `ManagerController.CreateStaffAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager)
- **Request Body (`CreateStaffAccountModel`):**
  ```json
  {
    "userName": "UserName_value",
    "fullName": "Nguyen Van A",
    "email": "user@example.com",
    "phoneNumber": "0987654321",
    "password": "P@ssword123",
    "confirmPassword": "P@ssword123",
    "parkingLotId": 1
  }
  ```
- **Success Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "isActive": true,
      "status": "Active",
      "createdAt": "2026-10-30T08:00:00Z",
      "parkingLots": [
        {
          "id": 1,
          "code": "RES-20261030-01",
          "name": "Nguyen Van A"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager) or facility assignment scope.

#### 5.6.5 `PUT` /api/manager/staff/{id}

**Description:** Updates profile information and parking lot assignment for a staff operator.

- **Controller / Action:** `ManagerController.UpdateStaffAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateEmployeeAccountModel`):**
  ```json
  {
    "fullName": "Nguyen Van A",
    "email": "user@example.com",
    "phoneNumber": "0987654321"
  }
  ```
- **Success Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "isActive": true,
      "status": "Active",
      "createdAt": "2026-10-30T08:00:00Z",
      "parkingLots": [
        {
          "id": 1,
          "code": "RES-20261030-01",
          "name": "Nguyen Van A"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.6 `PUT` /api/manager/staff/{id}/lot

**Description:** Reassigns a staff operator to a different parking lot facility managed by the current user.

- **Controller / Action:** `ManagerController.AssignStaffLotAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`AssignStaffParkingLotModel`):**
  ```json
  {
    "parkingLotId": 1
  }
  ```
- **Success Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "isActive": true,
      "status": "Active",
      "createdAt": "2026-10-30T08:00:00Z",
      "parkingLots": [
        {
          "id": 1,
          "code": "RES-20261030-01",
          "name": "Nguyen Van A"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.7 `PATCH` /api/manager/staff/{id}/status

**Description:** Toggles active employment status for a staff operator account, enabling or disabling gate operations.

- **Controller / Action:** `ManagerController.UpdateStaffStatusAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateUserStatusModel`):**
  ```json
  {
    "isActive": true
  }
  ```
- **Success Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "isActive": true,
      "status": "Active",
      "createdAt": "2026-10-30T08:00:00Z",
      "parkingLots": [
        {
          "id": 1,
          "code": "RES-20261030-01",
          "name": "Nguyen Van A"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.8 `DELETE` /api/manager/staff/{id}

**Description:** Soft-deletes or archives a staff operator account managed by the current user.

- **Controller / Action:** `ManagerController.DeleteStaffAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<bool>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": true,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.9 `GET` /api/manager/lots/{lotId:int}/reservations

**Description:** Retrieves reservation records across a managed lot with date range and status filtering for manager oversight.

- **Controller / Action:** `ManagerOperationsController.GetReservationsAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `status` (string, optional, default: `null`): Filter parameter.
  - `from` (DateTime?, optional, default: `null`): Filter parameter.
  - `to` (DateTime?, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IReadOnlyList<ReservationResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1001,
        "code": "RES-20261030-01",
        "status": "Active",
        "parkingLotId": 1,
        "parkingLotName": "Central City Parking",
        "parkingLotAddress": "123 Le Loi Street, District 1, HCMC",
        "parkingLotDistrict": "District 1",
        "zoneId": 1,
        "zoneName": "Zone A",
        "slotId": 1,
        "slotCode": "RES-20261030-01",
        "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "plateNumber": "59A-123.45",
        "vehicleLabel": "Toyota Camry 2024",
        "arrivalTime": "2026-10-30T08:00:00Z",
        "bookingMode": "Scheduled",
        "departureTime": "2026-10-30T08:00:00Z",
        "isSlotHeld": false,
        "checkInFrom": "2026-10-30T08:00:00Z",
        "checkInUntil": "2026-10-30T08:00:00Z",
        "depositAmount": 50000,
        "hasDeposit": false,
        "holdMinutes": 60,
        "holdSeconds": 1,
        "depositDueAt": "2026-10-30T08:00:00Z",
        "refundAmount": 50000,
        "cancelReason": "Operational requirement",
        "cancelledBy": "CancelledBy_value",
        "createdAt": "2026-10-30T08:00:00Z",
        "confirmedAt": "2026-10-30T08:00:00Z",
        "canChange": true,
        "canCancel": true,
        "refundIfCancelledNow": 10000,
        "pendingPayment": {
          "id": 1001,
          "code": "RES-20261030-01",
          "provider": "VnPay",
          "method": "Wallet",
          "purpose": "ReservationDeposit",
          "channel": "MobileApp",
          "amount": 50000,
          "status": "Active",
          "payUrl": "https://example.com/payment/return",
          "expiresAt": "2026-10-30T08:00:00Z",
          "createdAt": "2026-10-30T08:00:00Z",
          "paidAt": "2026-10-30T08:00:00Z",
          "reservationId": 1001,
          "sessionId": 1001,
          "refundedAmount": 50000,
          "refundOfPaymentId": 1001,
          "message": "Message_value",
          "exitPass": "ExitPass_value",
          "exitWindowEndsAt": "2026-10-30T08:00:00Z",
          "callbackDeadline": "2026-10-30T08:00:00Z"
        }
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.10 `POST` /api/manager/reservations/{id:long}/cancel

**Description:** Allows a manager to cancel an advance reservation from the facility side with a 100% deposit refund.

- **Controller / Action:** `ManagerOperationsController.CancelReservationAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`CancelReservationModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Success Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "Central City Parking",
      "parkingLotAddress": "123 Le Loi Street, District 1, HCMC",
      "parkingLotDistrict": "District 1",
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "Toyota Camry 2024",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "Scheduled",
      "departureTime": "2026-10-30T08:00:00Z",
      "isSlotHeld": false,
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "depositAmount": 50000,
      "hasDeposit": false,
      "holdMinutes": 60,
      "holdSeconds": 1,
      "depositDueAt": "2026-10-30T08:00:00Z",
      "refundAmount": 50000,
      "cancelReason": "Operational requirement",
      "cancelledBy": "CancelledBy_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "confirmedAt": "2026-10-30T08:00:00Z",
      "canChange": true,
      "canCancel": true,
      "refundIfCancelledNow": 10000,
      "pendingPayment": {
        "id": 1001,
        "code": "RES-20261030-01",
        "provider": "VnPay",
        "method": "Wallet",
        "purpose": "ReservationDeposit",
        "channel": "MobileApp",
        "amount": 50000,
        "status": "Active",
        "payUrl": "https://example.com/payment/return",
        "expiresAt": "2026-10-30T08:00:00Z",
        "createdAt": "2026-10-30T08:00:00Z",
        "paidAt": "2026-10-30T08:00:00Z",
        "reservationId": 1001,
        "sessionId": 1001,
        "refundedAmount": 50000,
        "refundOfPaymentId": 1001,
        "message": "Message_value",
        "exitPass": "ExitPass_value",
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "callbackDeadline": "2026-10-30T08:00:00Z"
      }
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.11 `POST` /api/manager/sessions/{id:long}/abnormal-close

**Description:** Forces abnormal closure of an orphaned or ghost parking session, waiving uncollectible fees.

- **Controller / Action:** `ManagerOperationsController.CloseAbnormallyAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`SessionDecisionModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Success Response Contract:** `ApiResult<SessionResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "parkingLotId": 1,
      "status": "Active",
      "sessionType": "SessionType_value",
      "plateNumber": "59A-123.45",
      "hasPlate": false,
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "isCustomerAccount": false,
      "paidAmount": 50000,
      "finalFee": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "exitTime": "2026-10-30T08:00:00Z",
      "isManualEntry": false,
      "noPlateReason": "59A-123.45",
      "isLostTicket": false,
      "lostTicketStatus": "Active",
      "lostTicketReportedBy": "LostTicketReportedBy_value",
      "lostTicketReportedAt": "2026-10-30T08:00:00Z",
      "lostTicketReason": "Operational requirement",
      "lostTicketDecisionNote": "LostTicketDecisionNote_value",
      "plateMismatch": false,
      "isOverdue": false,
      "zeroFeeReason": "Operational requirement",
      "overrideNote": "OverrideNote_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.12 `POST` /api/manager/sessions/{id:long}/lost-ticket/approve

**Description:** Authorizes a customer lost-ticket fee waiver or adjustment, approving vehicle release.

- **Controller / Action:** `ManagerOperationsController.ApproveLostTicketAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`SessionDecisionModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Success Response Contract:** `ApiResult<SessionResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "parkingLotId": 1,
      "status": "Active",
      "sessionType": "SessionType_value",
      "plateNumber": "59A-123.45",
      "hasPlate": false,
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "isCustomerAccount": false,
      "paidAmount": 50000,
      "finalFee": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "exitTime": "2026-10-30T08:00:00Z",
      "isManualEntry": false,
      "noPlateReason": "59A-123.45",
      "isLostTicket": false,
      "lostTicketStatus": "Active",
      "lostTicketReportedBy": "LostTicketReportedBy_value",
      "lostTicketReportedAt": "2026-10-30T08:00:00Z",
      "lostTicketReason": "Operational requirement",
      "lostTicketDecisionNote": "LostTicketDecisionNote_value",
      "plateMismatch": false,
      "isOverdue": false,
      "zeroFeeReason": "Operational requirement",
      "overrideNote": "OverrideNote_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.13 `POST` /api/manager/sessions/{id:long}/lost-ticket/reject

**Description:** Rejects a lost-ticket dispute, requiring full standard penalty settlement before vehicle exit.

- **Controller / Action:** `ManagerOperationsController.RejectLostTicketAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`SessionDecisionModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Success Response Contract:** `ApiResult<SessionResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "parkingLotId": 1,
      "status": "Active",
      "sessionType": "SessionType_value",
      "plateNumber": "59A-123.45",
      "hasPlate": false,
      "vehicleTypeId": 1,
      "vehicleTypeName": "Nguyen Van A",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "Zone A",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "isCustomerAccount": false,
      "paidAmount": 50000,
      "finalFee": 50000,
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "exitTime": "2026-10-30T08:00:00Z",
      "isManualEntry": false,
      "noPlateReason": "59A-123.45",
      "isLostTicket": false,
      "lostTicketStatus": "Active",
      "lostTicketReportedBy": "LostTicketReportedBy_value",
      "lostTicketReportedAt": "2026-10-30T08:00:00Z",
      "lostTicketReason": "Operational requirement",
      "lostTicketDecisionNote": "LostTicketDecisionNote_value",
      "plateMismatch": false,
      "isOverdue": false,
      "zeroFeeReason": "Operational requirement",
      "overrideNote": "OverrideNote_value"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.14 `GET` /api/manager/lots/{lotId:int}/plate-lists

**Description:** Retrieves Whitelist and Blacklist license plate entries configured for a managed parking lot.

- **Controller / Action:** `ManagerOperationsController.GetPlateListsAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `listType` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IReadOnlyList<PlateListEntryModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "listType": "ListType_value",
        "plateNumber": "59A-123.45",
        "parkingLotId": 1,
        "reason": "Operational requirement",
        "expiresAt": "2026-10-30T08:00:00Z",
        "isActive": true,
        "createdAt": "2026-10-30T08:00:00Z"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.15 `POST` /api/manager/lots/{lotId:int}/plate-lists

**Description:** Adds a vehicle plate to the lot Whitelist (VIP/exempt) or Blacklist (banned/flagged) with mandatory reason.

- **Controller / Action:** `ManagerOperationsController.AddPlateListEntryAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Request Body (`CreatePlateListEntryModel`):**
  ```json
  {
    "listType": "ListType_value",
    "plateNumber": "59A-123.45",
    "reason": "Operational requirement",
    "expiresAt": "2026-10-30T08:00:00Z"
  }
  ```
- **Success Response Contract:** `ApiResult<PlateListEntryModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "listType": "ListType_value",
      "plateNumber": "59A-123.45",
      "parkingLotId": 1,
      "reason": "Operational requirement",
      "expiresAt": "2026-10-30T08:00:00Z",
      "isActive": true,
      "createdAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.16 `DELETE` /api/manager/lots/{lotId:int}/plate-lists/{id:int}

**Description:** Removes or deactivates a license plate entry from the lot Whitelist or Blacklist.

- **Controller / Action:** `ManagerOperationsController.RemovePlateListEntryAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<bool>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": true,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.17 `GET` /api/manager/lots/{lotId:int}/pricing-rules

**Description:** Retrieves all pricing rules and tariff structures configured for a managed parking lot.

- **Controller / Action:** `ManagerPricingController.GetPricingRulesAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Query Parameters:**
  - `status` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IReadOnlyList<PricingRuleModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": 1,
        "code": "RES-20261030-01",
        "name": "Nguyen Van A",
        "parkingLotId": 1,
        "zoneId": 1,
        "vehicleTypeId": 1,
        "vehicleTypeCode": "RES-20261030-01",
        "vehicleTypeName": "Nguyen Van A",
        "chargeModel": "ChargeModel_value",
        "entryFee": 50000,
        "firstBlockMinutes": 60,
        "firstBlockPrice": 50000,
        "blockMinutes": 60,
        "blockPrice": 50000,
        "dayStartTime": "06:00:00",
        "nightStartTime": "22:00:00",
        "dayPrice": 50000,
        "nightPrice": 50000,
        "maxDailyFee": 50000,
        "graceMinutes": 60,
        "overnightSurcharge": 10000,
        "overstayAfterHours": 1,
        "overstaySurchargePerDay": 10000,
        "lostTicketFee": 50000,
        "noShowFee": 50000,
        "currency": "VND",
        "priority": 1,
        "effectiveFrom": "2026-10-30T08:00:00Z",
        "effectiveTo": "2026-10-30T08:00:00Z",
        "status": "Active",
        "versionNo": 1,
        "approvedBy": "ApprovedBy_value",
        "approvedAt": "2026-10-30T08:00:00Z",
        "createdBy": "CreatedBy_value",
        "createdAt": "2026-10-30T08:00:00Z",
        "updatedAt": "2026-10-30T08:00:00Z"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.18 `GET` /api/manager/lots/{lotId:int}/pricing-rules/{id:int}

**Description:** Retrieves detailed tariff blocks, grace periods, and vehicle rates for a specific pricing rule.

- **Controller / Action:** `ManagerPricingController.GetPricingRuleByIdAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<PricingRuleModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Nguyen Van A",
      "parkingLotId": 1,
      "zoneId": 1,
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "Nguyen Van A",
      "chargeModel": "ChargeModel_value",
      "entryFee": 50000,
      "firstBlockMinutes": 60,
      "firstBlockPrice": 50000,
      "blockMinutes": 60,
      "blockPrice": 50000,
      "dayStartTime": "06:00:00",
      "nightStartTime": "22:00:00",
      "dayPrice": 50000,
      "nightPrice": 50000,
      "maxDailyFee": 50000,
      "graceMinutes": 60,
      "overnightSurcharge": 10000,
      "overstayAfterHours": 1,
      "overstaySurchargePerDay": 10000,
      "lostTicketFee": 50000,
      "noShowFee": 50000,
      "currency": "VND",
      "priority": 1,
      "effectiveFrom": "2026-10-30T08:00:00Z",
      "effectiveTo": "2026-10-30T08:00:00Z",
      "status": "Active",
      "versionNo": 1,
      "approvedBy": "ApprovedBy_value",
      "approvedAt": "2026-10-30T08:00:00Z",
      "createdBy": "CreatedBy_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.19 `POST` /api/manager/lots/{lotId:int}/pricing-rules

**Description:** Creates a new draft pricing rule with step-rate blocks, day/night tariffs, and grace periods.

- **Controller / Action:** `ManagerPricingController.CreatePricingRuleAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Request Body (`CreatePricingRuleModel`):**
  ```json
  {
    "code": "RES-20261030-01",
    "name": "Nguyen Van A",
    "zoneId": 1,
    "vehicleTypeId": 1,
    "chargeModel": "ChargeModel_value",
    "entryFee": 50000,
    "firstBlockMinutes": 60,
    "firstBlockPrice": 50000,
    "blockMinutes": 60,
    "blockPrice": 50000,
    "dayStartTime": "06:00:00",
    "nightStartTime": "22:00:00",
    "dayPrice": 50000,
    "nightPrice": 50000,
    "maxDailyFee": 50000,
    "graceMinutes": 60,
    "overnightSurcharge": 10000,
    "overstayAfterHours": 1,
    "overstaySurchargePerDay": 10000,
    "lostTicketFee": 50000,
    "noShowFee": 50000,
    "currency": "VND",
    "priority": 1,
    "effectiveFrom": "2026-10-30T08:00:00Z",
    "effectiveTo": "2026-10-30T08:00:00Z",
    "status": "Active"
  }
  ```
- **Success Response Contract:** `ApiResult<PricingRuleModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Nguyen Van A",
      "parkingLotId": 1,
      "zoneId": 1,
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "Nguyen Van A",
      "chargeModel": "ChargeModel_value",
      "entryFee": 50000,
      "firstBlockMinutes": 60,
      "firstBlockPrice": 50000,
      "blockMinutes": 60,
      "blockPrice": 50000,
      "dayStartTime": "06:00:00",
      "nightStartTime": "22:00:00",
      "dayPrice": 50000,
      "nightPrice": 50000,
      "maxDailyFee": 50000,
      "graceMinutes": 60,
      "overnightSurcharge": 10000,
      "overstayAfterHours": 1,
      "overstaySurchargePerDay": 10000,
      "lostTicketFee": 50000,
      "noShowFee": 50000,
      "currency": "VND",
      "priority": 1,
      "effectiveFrom": "2026-10-30T08:00:00Z",
      "effectiveTo": "2026-10-30T08:00:00Z",
      "status": "Active",
      "versionNo": 1,
      "approvedBy": "ApprovedBy_value",
      "approvedAt": "2026-10-30T08:00:00Z",
      "createdBy": "CreatedBy_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.20 `PUT` /api/manager/lots/{lotId:int}/pricing-rules/{id:int}

**Description:** Modifies parameters and vehicle rate tiers of an existing draft pricing rule.

- **Controller / Action:** `ManagerPricingController.UpdatePricingRuleAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
  - `id` (required): Target id identifier.
- **Request Body (`UpdatePricingRuleModel`):**
  ```json
  {
    "name": "Nguyen Van A",
    "zoneId": 1,
    "vehicleTypeId": 1,
    "chargeModel": "ChargeModel_value",
    "entryFee": 50000,
    "firstBlockMinutes": 60,
    "firstBlockPrice": 50000,
    "blockMinutes": 60,
    "blockPrice": 50000,
    "dayStartTime": "06:00:00",
    "nightStartTime": "22:00:00",
    "dayPrice": 50000,
    "nightPrice": 50000,
    "maxDailyFee": 50000,
    "graceMinutes": 60,
    "overnightSurcharge": 10000,
    "overstayAfterHours": 1,
    "overstaySurchargePerDay": 10000,
    "lostTicketFee": 50000,
    "noShowFee": 50000,
    "priority": 1,
    "effectiveFrom": "2026-10-30T08:00:00Z",
    "effectiveTo": "2026-10-30T08:00:00Z",
    "status": "Active",
    "reason": "Operational requirement"
  }
  ```
- **Success Response Contract:** `ApiResult<PricingRuleModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Nguyen Van A",
      "parkingLotId": 1,
      "zoneId": 1,
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "Nguyen Van A",
      "chargeModel": "ChargeModel_value",
      "entryFee": 50000,
      "firstBlockMinutes": 60,
      "firstBlockPrice": 50000,
      "blockMinutes": 60,
      "blockPrice": 50000,
      "dayStartTime": "06:00:00",
      "nightStartTime": "22:00:00",
      "dayPrice": 50000,
      "nightPrice": 50000,
      "maxDailyFee": 50000,
      "graceMinutes": 60,
      "overnightSurcharge": 10000,
      "overstayAfterHours": 1,
      "overstaySurchargePerDay": 10000,
      "lostTicketFee": 50000,
      "noShowFee": 50000,
      "currency": "VND",
      "priority": 1,
      "effectiveFrom": "2026-10-30T08:00:00Z",
      "effectiveTo": "2026-10-30T08:00:00Z",
      "status": "Active",
      "versionNo": 1,
      "approvedBy": "ApprovedBy_value",
      "approvedAt": "2026-10-30T08:00:00Z",
      "createdBy": "CreatedBy_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.21 `PATCH` /api/manager/lots/{lotId:int}/pricing-rules/{id:int}/status

**Description:** Transitions pricing rule lifecycle state (e.g. Draft -> Active, Active -> Inactive, Archived).

- **Controller / Action:** `ManagerPricingController.SetPricingRuleStatusAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
  - `id` (required): Target id identifier.
- **Request Body (`SetPricingRuleStatusModel`):**
  ```json
  {
    "status": "Active",
    "reason": "Operational requirement"
  }
  ```
- **Success Response Contract:** `ApiResult<PricingRuleModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Nguyen Van A",
      "parkingLotId": 1,
      "zoneId": 1,
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "Nguyen Van A",
      "chargeModel": "ChargeModel_value",
      "entryFee": 50000,
      "firstBlockMinutes": 60,
      "firstBlockPrice": 50000,
      "blockMinutes": 60,
      "blockPrice": 50000,
      "dayStartTime": "06:00:00",
      "nightStartTime": "22:00:00",
      "dayPrice": 50000,
      "nightPrice": 50000,
      "maxDailyFee": 50000,
      "graceMinutes": 60,
      "overnightSurcharge": 10000,
      "overstayAfterHours": 1,
      "overstaySurchargePerDay": 10000,
      "lostTicketFee": 50000,
      "noShowFee": 50000,
      "currency": "VND",
      "priority": 1,
      "effectiveFrom": "2026-10-30T08:00:00Z",
      "effectiveTo": "2026-10-30T08:00:00Z",
      "status": "Active",
      "versionNo": 1,
      "approvedBy": "ApprovedBy_value",
      "approvedAt": "2026-10-30T08:00:00Z",
      "createdBy": "CreatedBy_value",
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.22 `DELETE` /api/manager/lots/{lotId:int}/pricing-rules/{id:int}

**Description:** Archives or soft-deletes an inactive pricing rule from a managed parking lot.

- **Controller / Action:** `ManagerPricingController.DeletePricingRuleAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<bool>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": true,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.6.23 `POST` /api/manager/lots/{lotId:int}/pricing-rules/simulate

**Description:** Simulates tariff calculations for a hypothetical parking duration under draft or active pricing rules.

- **Controller / Action:** `ManagerPricingController.SimulatePricingAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager,Admin)
- **Path Parameters:**
  - `lotId` (required): Target lotId identifier.
- **Request Body (`PricingSimulationRequestModel`):**
  ```json
  {
    "vehicleTypeId": 1,
    "vehicleTypeCode": "RES-20261030-01",
    "ruleId": 1,
    "entryTime": "2026-10-30T08:00:00Z",
    "exitTime": "2026-10-30T08:00:00Z",
    "depositAmount": 50000,
    "isLostTicket": false
  }
  ```
- **Success Response Contract:** `ApiResult<PricingSimulationResultModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "durationMinutes": 60,
      "withinGrace": false,
      "grossFee": 50000,
      "lostTicketFee": 50000,
      "exitWindowLateFee": 50000,
      "depositApplied": 10000,
      "depositRefund": 10000,
      "amountDue": 50000,
      "days": [
        {
          "date": null,
          "minutes": 60,
          "blocks": 1,
          "amount": 50000,
          "capped": false
        }
      ],
      "ruleId": 1,
      "ruleVersion": 1,
      "ruleName": "Nguyen Van A"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

### 5.7 Module 07: Vehicle Verification & Document Management

#### 5.7.1 `GET` /api/users/profile/rejected-vehicles

**Description:** Retrieves customer vehicle registrations rejected during KYC review within the last 30 days.

- **Controller / Action:** `VehicleDocumentsController.GetRecentlyRejectedAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Success Response Contract:** `ApiResult<List<RejectedVehicleModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "verificationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "licensePlate": "59A-123.45",
        "brand": "Brand_value",
        "model": "Model_value",
        "reviewNote": "ReviewNote_value",
        "decidedAt": "2026-10-30T08:00:00Z"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.7.2 `GET` /api/users/profile/vehicles/{vehicleId:guid}/documents

**Description:** Retrieves the latest KYC document package, registration papers, and verification status for a customer vehicle.

- **Controller / Action:** `VehicleDocumentsController.GetLatestAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Path Parameters:**
  - `vehicleId` (required): Target vehicleId identifier.
- **Success Response Contract:** `ApiResult<VehicleVerificationModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "verificationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "licensePlate": "59A-123.45",
      "status": "Active",
      "statusReason": "Active",
      "documents": [
        {
          "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "documentType": "DocumentType_value",
          "side": "Side_value",
          "fileName": "Nguyen Van A",
          "contentType": "ContentType_value",
          "sizeBytes": 1001,
          "uploadedAt": "2026-10-30T08:00:00Z",
          "isAvailable": false
        }
      ],
      "missingDocuments": [
        "Sample item"
      ],
      "createdAt": "2026-10-30T08:00:00Z",
      "submittedAt": "2026-10-30T08:00:00Z",
      "decidedAt": "2026-10-30T08:00:00Z",
      "reviewNote": "ReviewNote_value",
      "filesDeleteAt": "2026-10-30T08:00:00Z",
      "filesDeletedAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.7.3 `POST` /api/users/profile/vehicles/{vehicleId:guid}/documents/{documentType}/{side}

**Description:** Uploads a front or back document photo (registration paper, insurance certificate) for vehicle verification.

- **Controller / Action:** `VehicleDocumentsController.UploadAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Path Parameters:**
  - `vehicleId` (required): Target vehicleId identifier.
  - `documentType` (required): Target documentType identifier.
  - `side` (required): Target side identifier.
- **Request Body (`multipart/form-data`):**
  - `file` (binary, required): JPEG, PNG, or PDF document file stream.
- **Success Response Contract:** `ApiResult<VehicleVerificationModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "verificationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "licensePlate": "59A-123.45",
      "status": "Active",
      "statusReason": "Active",
      "documents": [
        {
          "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "documentType": "DocumentType_value",
          "side": "Side_value",
          "fileName": "Nguyen Van A",
          "contentType": "ContentType_value",
          "sizeBytes": 1001,
          "uploadedAt": "2026-10-30T08:00:00Z",
          "isAvailable": false
        }
      ],
      "missingDocuments": [
        "Sample item"
      ],
      "createdAt": "2026-10-30T08:00:00Z",
      "submittedAt": "2026-10-30T08:00:00Z",
      "decidedAt": "2026-10-30T08:00:00Z",
      "reviewNote": "ReviewNote_value",
      "filesDeleteAt": "2026-10-30T08:00:00Z",
      "filesDeletedAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.7.4 `POST` /api/users/profile/vehicles/{vehicleId:guid}/documents/submit

**Description:** Submits an uploaded vehicle document package to the operator verification queue for review and approval.

- **Controller / Action:** `VehicleDocumentsController.SubmitAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Path Parameters:**
  - `vehicleId` (required): Target vehicleId identifier.
- **Success Response Contract:** `ApiResult<VehicleVerificationModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "verificationId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "licensePlate": "59A-123.45",
      "status": "Active",
      "statusReason": "Active",
      "documents": [
        {
          "documentId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
          "documentType": "DocumentType_value",
          "side": "Side_value",
          "fileName": "Nguyen Van A",
          "contentType": "ContentType_value",
          "sizeBytes": 1001,
          "uploadedAt": "2026-10-30T08:00:00Z",
          "isAvailable": false
        }
      ],
      "missingDocuments": [
        "Sample item"
      ],
      "createdAt": "2026-10-30T08:00:00Z",
      "submittedAt": "2026-10-30T08:00:00Z",
      "decidedAt": "2026-10-30T08:00:00Z",
      "reviewNote": "ReviewNote_value",
      "filesDeleteAt": "2026-10-30T08:00:00Z",
      "filesDeletedAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.
  - `422 Unprocessable`: Mandatory registration paper sides are missing.

#### 5.7.5 `GET` /api/users/profile/vehicles/{vehicleId:guid}/documents/files/{documentId:guid}

**Description:** Streams the binary photo or PDF scan of an uploaded vehicle registration document.

- **Controller / Action:** `VehicleDocumentsController.GetFileAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Path Parameters:**
  - `vehicleId` (required): Target vehicleId identifier.
  - `documentId` (required): Target documentId identifier.
- **Success Response Contract:** `IActionResult`
- **Success Response Example (`200 OK`):** Binary octet stream with `Content-Type: image/jpeg`, `image/png`, or `application/pdf`.
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.7.6 `GET` /api/vehicle-verifications

**Description:** Retrieves the operator review queue of pending customer vehicle KYC submissions.

- **Controller / Action:** `VehicleVerificationsController.GetQueueAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Query Parameters:**
  - `status` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<List<VehicleVerificationReviewModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "customerName": "Nguyen Van A",
        "customerCitizenId": "079201012345",
        "customerDateOfBirth": null,
        "vehicleType": "VehicleType_value",
        "brand": "Brand_value",
        "model": "Model_value",
        "color": "Color_value",
        "ownerCitizenId": "079201012345"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.

#### 5.7.7 `GET` /api/vehicle-verifications/{verificationId:guid}

**Description:** Retrieves detailed inspection information and submitted document images for a specific vehicle verification request.

- **Controller / Action:** `VehicleVerificationsController.GetAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `verificationId` (required): Target verificationId identifier.
- **Success Response Contract:** `ApiResult<VehicleVerificationReviewModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "customerName": "Nguyen Van A",
      "customerCitizenId": "079201012345",
      "customerDateOfBirth": null,
      "vehicleType": "VehicleType_value",
      "brand": "Brand_value",
      "model": "Model_value",
      "color": "Color_value",
      "ownerCitizenId": "079201012345"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.7.8 `GET` /api/vehicle-verifications/{verificationId:guid}/files/{documentId:guid}

**Description:** Streams high-resolution binary document files for operator verification inspection.

- **Controller / Action:** `VehicleVerificationsController.GetFileAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `verificationId` (required): Target verificationId identifier.
  - `documentId` (required): Target documentId identifier.
- **Success Response Contract:** `IActionResult`
- **Success Response Example (`200 OK`):** Binary octet stream with `Content-Type: image/jpeg`, `image/png`, or `application/pdf`.
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.7.9 `POST` /api/vehicle-verifications/{verificationId:guid}/review

**Description:** Records operator approval or rejection decision for a vehicle registration with optional rejection reason.

- **Controller / Action:** `VehicleVerificationsController.ReviewAsync`
- **Authentication / Authorization:** Bearer JWT Required (Staff,Manager,Admin)
- **Path Parameters:**
  - `verificationId` (required): Target verificationId identifier.
- **Request Body (`ReviewVehicleVerificationModel`):**
  ```json
  {
    "approve": false,
    "note": "Note_value"
  }
  ```
- **Success Response Contract:** `ApiResult<VehicleVerificationReviewModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "customerName": "Nguyen Van A",
      "customerCitizenId": "079201012345",
      "customerDateOfBirth": null,
      "vehicleType": "VehicleType_value",
      "brand": "Brand_value",
      "model": "Model_value",
      "color": "Color_value",
      "ownerCitizenId": "079201012345"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Staff,Manager,Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

### 5.8 Module 08: Payments & Customer Wallet

#### 5.8.1 `GET` /api/payments/{id:long}

**Description:** Retrieves payment status, transaction codes, gateway URLs, and financial settlement details for a payment record.

- **Controller / Action:** `PaymentsController.GetAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "VnPay",
      "method": "Wallet",
      "purpose": "ReservationDeposit",
      "channel": "MobileApp",
      "amount": 50000,
      "status": "Active",
      "payUrl": "https://example.com/payment/return",
      "expiresAt": "2026-10-30T08:00:00Z",
      "createdAt": "2026-10-30T08:00:00Z",
      "paidAt": "2026-10-30T08:00:00Z",
      "reservationId": 1001,
      "sessionId": 1001,
      "refundedAmount": 50000,
      "refundOfPaymentId": 1001,
      "message": "Message_value",
      "exitPass": "ExitPass_value",
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "callbackDeadline": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.8.2 `POST` /api/payments/{id:long}/check

**Description:** Polls the external payment gateway (VNPay or MoMo) to reconcile and confirm the settlement of a pending transaction.

- **Controller / Action:** `PaymentsController.CheckAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<PaymentCallbackResultModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "signatureValid": false,
      "outcome": "Outcome_value",
      "message": "Message_value",
      "payment": {
        "id": 1001,
        "code": "RES-20261030-01",
        "provider": "VnPay",
        "method": "Wallet",
        "purpose": "ReservationDeposit",
        "channel": "MobileApp",
        "amount": 50000,
        "status": "Active",
        "payUrl": "https://example.com/payment/return",
        "expiresAt": "2026-10-30T08:00:00Z",
        "createdAt": "2026-10-30T08:00:00Z",
        "paidAt": "2026-10-30T08:00:00Z",
        "reservationId": 1001,
        "sessionId": 1001,
        "refundedAmount": 50000,
        "refundOfPaymentId": 1001,
        "message": "Message_value",
        "exitPass": "ExitPass_value",
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "callbackDeadline": "2026-10-30T08:00:00Z"
      }
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.8.3 `GET` /api/payments/simulation

**Description:** Checks whether sandbox payment simulation is enabled on the backend server for testing and evaluation purposes.

- **Controller / Action:** `PaymentsController.GetSimulation`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Success Response Contract:** `ApiResult<PaymentSimulationModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "enabled": false
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.

#### 5.8.4 `POST` /api/payments/{id:long}/simulate-success

**Description:** Simulates successful gateway payment callback in development/testing environments to confirm deposits or wallet credits.

- **Controller / Action:** `PaymentsController.SimulateSuccessAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<PaymentCallbackResultModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "signatureValid": false,
      "outcome": "Outcome_value",
      "message": "Message_value",
      "payment": {
        "id": 1001,
        "code": "RES-20261030-01",
        "provider": "VnPay",
        "method": "Wallet",
        "purpose": "ReservationDeposit",
        "channel": "MobileApp",
        "amount": 50000,
        "status": "Active",
        "payUrl": "https://example.com/payment/return",
        "expiresAt": "2026-10-30T08:00:00Z",
        "createdAt": "2026-10-30T08:00:00Z",
        "paidAt": "2026-10-30T08:00:00Z",
        "reservationId": 1001,
        "sessionId": 1001,
        "refundedAmount": 50000,
        "refundOfPaymentId": 1001,
        "message": "Message_value",
        "exitPass": "ExitPass_value",
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "callbackDeadline": "2026-10-30T08:00:00Z"
      }
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.8.5 `GET` /api/payments/vnpay/return

**Description:** Processes browser redirect return from VNPay payment gateway and displays checkout outcome.

- **Controller / Action:** `PaymentsController.VnPayReturnAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Success Response Contract:** `ApiResult<PaymentCallbackResultModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "signatureValid": false,
      "outcome": "Outcome_value",
      "message": "Message_value",
      "payment": {
        "id": 1001,
        "code": "RES-20261030-01",
        "provider": "VnPay",
        "method": "Wallet",
        "purpose": "ReservationDeposit",
        "channel": "MobileApp",
        "amount": 50000,
        "status": "Active",
        "payUrl": "https://example.com/payment/return",
        "expiresAt": "2026-10-30T08:00:00Z",
        "createdAt": "2026-10-30T08:00:00Z",
        "paidAt": "2026-10-30T08:00:00Z",
        "reservationId": 1001,
        "sessionId": 1001,
        "refundedAmount": 50000,
        "refundOfPaymentId": 1001,
        "message": "Message_value",
        "exitPass": "ExitPass_value",
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "callbackDeadline": "2026-10-30T08:00:00Z"
      }
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.

#### 5.8.6 `GET` /api/payments/vnpay/ipn

**Description:** Receives asynchronous server-to-server IPN webhook from VNPay to finalize payment settlement.

- **Controller / Action:** `PaymentsController.VnPayIpnAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Success Response Contract:** `IActionResult`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "RspCode": "00",
    "Message": "Confirm Success"
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.

#### 5.8.7 `GET` /api/payments/momo/return

**Description:** Processes browser redirect return from MoMo payment gateway and displays checkout outcome.

- **Controller / Action:** `PaymentsController.MoMoReturnAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Success Response Contract:** `ApiResult<PaymentCallbackResultModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "signatureValid": false,
      "outcome": "Outcome_value",
      "message": "Message_value",
      "payment": {
        "id": 1001,
        "code": "RES-20261030-01",
        "provider": "VnPay",
        "method": "Wallet",
        "purpose": "ReservationDeposit",
        "channel": "MobileApp",
        "amount": 50000,
        "status": "Active",
        "payUrl": "https://example.com/payment/return",
        "expiresAt": "2026-10-30T08:00:00Z",
        "createdAt": "2026-10-30T08:00:00Z",
        "paidAt": "2026-10-30T08:00:00Z",
        "reservationId": 1001,
        "sessionId": 1001,
        "refundedAmount": 50000,
        "refundOfPaymentId": 1001,
        "message": "Message_value",
        "exitPass": "ExitPass_value",
        "exitWindowEndsAt": "2026-10-30T08:00:00Z",
        "callbackDeadline": "2026-10-30T08:00:00Z"
      }
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.

#### 5.8.8 `POST` /api/payments/momo/ipn

**Description:** Receives asynchronous server-to-server IPN webhook from MoMo to finalize payment settlement.

- **Controller / Action:** `PaymentsController.MoMoIpnAsync`
- **Authentication / Authorization:** None (Public / Anonymous)
- **Success Response Contract:** `NoContent (204)`
- **Success Response Example (`204 NoContent`):** Empty response body.
- **Applicable HTTP Status Codes:**
  - `204 NoContent`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.

#### 5.8.9 `GET` /api/wallet

**Description:** Retrieves the customer XXIV wallet balance, hold amounts, daily limits, and recent transaction history.

- **Controller / Action:** `WalletController.GetAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Success Response Contract:** `ApiResult<WalletResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "balance": 50000,
      "holdAmount": 50000,
      "currency": "VND",
      "status": "Active",
      "dailyTopUpLimit": 10000,
      "topUpRemainingToday": 10000,
      "minTopUp": 10000,
      "recentTransactions": [
        {
          "id": "Id_value",
          "title": "Title_value",
          "subtitle": "Subtitle_value",
          "amount": 50000,
          "status": "Active",
          "type": "Type_value",
          "method": "Wallet",
          "createdAt": "2026-10-30T08:00:00Z"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.8.10 `POST` /api/wallet/topup

**Description:** Initiates an online wallet top-up transaction via VNPay or MoMo gateway to credit customer balance.

- **Controller / Action:** `WalletController.TopUpAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Request Body (`TopUpWalletModel`):**
  ```json
  {
    "amount": 50000,
    "provider": "VnPay",
    "returnUrl": "https://example.com/payment/return"
  }
  ```
- **Success Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "VnPay",
      "method": "Wallet",
      "purpose": "ReservationDeposit",
      "channel": "MobileApp",
      "amount": 50000,
      "status": "Active",
      "payUrl": "https://example.com/payment/return",
      "expiresAt": "2026-10-30T08:00:00Z",
      "createdAt": "2026-10-30T08:00:00Z",
      "paidAt": "2026-10-30T08:00:00Z",
      "reservationId": 1001,
      "sessionId": 1001,
      "refundedAmount": 50000,
      "refundOfPaymentId": 1001,
      "message": "Message_value",
      "exitPass": "ExitPass_value",
      "exitWindowEndsAt": "2026-10-30T08:00:00Z",
      "callbackDeadline": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

#### 5.8.11 `POST` /api/wallet/withdraw

**Description:** Requests a wallet balance withdrawal to the customer registered bank account or MoMo phone number.

- **Controller / Action:** `WalletController.WithdrawAsync`
- **Authentication / Authorization:** Bearer JWT Required (Authenticated)
- **Request Body (`WithdrawWalletModel`):**
  ```json
  {
    "amount": 50000,
    "method": "Wallet",
    "bankName": "Nguyen Van A",
    "bankAccount": "BankAccount_value"
  }
  ```
- **Success Response Contract:** `ApiResult<WalletResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "balance": 50000,
      "holdAmount": 50000,
      "currency": "VND",
      "status": "Active",
      "dailyTopUpLimit": 10000,
      "topUpRemainingToday": 10000,
      "minTopUp": 10000,
      "recentTransactions": [
        {
          "id": "Id_value",
          "title": "Title_value",
          "subtitle": "Subtitle_value",
          "amount": 50000,
          "status": "Active",
          "type": "Type_value",
          "method": "Wallet",
          "createdAt": "2026-10-30T08:00:00Z"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Authenticated) or facility assignment scope.

### 5.9 Module 09: Revenue Reporting

#### 5.9.1 `GET` /api/admin/reports/revenue

**Description:** Retrieves system-wide aggregated revenue metrics, date range telemetry, and payment method breakdowns.

- **Controller / Action:** `AdminRevenueController.GetRevenueAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Query Parameters:**
  - `startDate` (string, optional, default: `null`): Filter parameter.
  - `endDate` (string, optional, default: `null`): Filter parameter.
  - `parkingLotId` (int?, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<AdminRevenueReportResponse>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "startDate": "2026-10-30",
      "endDate": "2026-10-30",
      "systemTotalRevenue": 50000,
      "systemTotalSessions": 100,
      "activeLotsCount": 100,
      "byLot": [
        {
          "parkingLotId": 1,
          "parkingLotName": "Central City Parking",
          "totalRevenue": 50000,
          "totalSessions": 100,
          "managerUserId": "ManagerUserId_value"
        }
      ],
      "paymentMethods": [
        {
          "method": "Wallet",
          "amount": 50000,
          "percentage": 10000
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.

#### 5.9.2 `GET` /api/manager/reports/revenue

**Description:** Retrieves aggregated revenue reports and income analytics strictly scoped to manager-assigned parking lots.

- **Controller / Action:** `ManagerRevenueController.GetRevenueAsync`
- **Authentication / Authorization:** Bearer JWT Required (Manager)
- **Query Parameters:**
  - `startDate` (string, optional, default: `null`): Filter parameter.
  - `endDate` (string, optional, default: `null`): Filter parameter.
  - `parkingLotId` (int?, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<ManagerRevenueReportResponse>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "managerUserId": "ManagerUserId_value",
      "startDate": "2026-10-30",
      "endDate": "2026-10-30",
      "totalRevenue": 50000,
      "totalSessions": 100,
      "byLot": [
        {
          "parkingLotId": 1,
          "parkingLotName": "Central City Parking",
          "totalRevenue": 50000,
          "totalSessions": 100,
          "managerUserId": "ManagerUserId_value"
        }
      ],
      "paymentMethods": [
        {
          "method": "Wallet",
          "amount": 50000,
          "percentage": 10000
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Manager) or facility assignment scope.

### 5.10 Module 10: Admin User & System Management

#### 5.10.1 `POST` /api/admin/managers

**Description:** Provisions a new facility Manager account and binds initial parking lot operational assignments.

- **Controller / Action:** `AdminController.CreateManagerAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Request Body (`CreateEmployeeAccountModel`):**
  ```json
  {
    "userName": "UserName_value",
    "fullName": "Nguyen Van A",
    "email": "user@example.com",
    "password": "P@ssword123",
    "phoneNumber": "0987654321",
    "assignedParkingLot": "AssignedParkingLot_value",
    "parkingLotIds": [
      1,
      2,
      3
    ]
  }
  ```
- **Success Response Contract:** `ApiResult<CreateUserResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "employeeCode": "RES-20261030-01"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.

#### 5.10.2 `GET` /api/admin/managers

**Description:** Retrieves all Manager accounts with their active status and assigned parking lot portfolios.

- **Controller / Action:** `AdminController.GetManagersAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Query Parameters:**
  - `isActive` (bool?, optional, default: `null`): Filter parameter.
  - `searchTerm` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IEnumerable<EmployeeAccountResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": "Id_value",
        "userName": "UserName_value",
        "fullName": "Nguyen Van A",
        "email": "user@example.com",
        "phoneNumber": "0987654321",
        "role": "Staff",
        "employeeCode": "RES-20261030-01",
        "isActive": true,
        "status": "Active",
        "createdAt": "2026-10-30T08:00:00Z",
        "parkingLots": [
          {
            "id": 1,
            "code": "RES-20261030-01",
            "name": "Nguyen Van A"
          }
        ]
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.

#### 5.10.3 `GET` /api/admin/managers/{id}

**Description:** Retrieves full account details, contact information, and assigned lots for a specific Manager.

- **Controller / Action:** `AdminController.GetManagerAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "isActive": true,
      "status": "Active",
      "createdAt": "2026-10-30T08:00:00Z",
      "parkingLots": [
        {
          "id": 1,
          "code": "RES-20261030-01",
          "name": "Nguyen Van A"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.10.4 `PUT` /api/admin/managers/{id}

**Description:** Updates account credentials and personal information for an existing Manager.

- **Controller / Action:** `AdminController.UpdateManagerAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateEmployeeAccountModel`):**
  ```json
  {
    "fullName": "Nguyen Van A",
    "email": "user@example.com",
    "phoneNumber": "0987654321"
  }
  ```
- **Success Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "isActive": true,
      "status": "Active",
      "createdAt": "2026-10-30T08:00:00Z",
      "parkingLots": [
        {
          "id": 1,
          "code": "RES-20261030-01",
          "name": "Nguyen Van A"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.10.5 `PATCH` /api/admin/managers/{id}/status

**Description:** Toggles active status for a Manager account, immediately activating or suspending access.

- **Controller / Action:** `AdminController.UpdateManagerStatusAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateUserStatusModel`):**
  ```json
  {
    "isActive": true
  }
  ```
- **Success Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "isActive": true,
      "status": "Active",
      "createdAt": "2026-10-30T08:00:00Z",
      "parkingLots": [
        {
          "id": 1,
          "code": "RES-20261030-01",
          "name": "Nguyen Van A"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.10.6 `PUT` /api/admin/managers/{id}/lots

**Description:** Replaces or modifies the full portfolio of parking lots assigned to a specific Manager.

- **Controller / Action:** `AdminController.AssignManagerLotsAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`AssignParkingLotsModel`):**
  ```json
  {
    "parkingLotIds": [
      1,
      2,
      3
    ]
  }
  ```
- **Success Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "Nguyen Van A",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "isActive": true,
      "status": "Active",
      "createdAt": "2026-10-30T08:00:00Z",
      "parkingLots": [
        {
          "id": 1,
          "code": "RES-20261030-01",
          "name": "Nguyen Van A"
        }
      ]
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.10.7 `GET` /api/admin/users

**Description:** Retrieves system employee accounts (Staff, Managers, Admins) with role, active status, and search filters.

- **Controller / Action:** `AdminController.GetInternalUsersAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Query Parameters:**
  - `role` (string, optional, default: `null`): Filter parameter.
  - `isActive` (bool?, optional, default: `null`): Filter parameter.
  - `searchTerm` (string, optional, default: `null`): Filter parameter.
- **Success Response Contract:** `ApiResult<IEnumerable<UserResponseModel>>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": [
      {
        "id": "Id_value",
        "userName": "UserName_value",
        "fullName": "Nguyen Van A",
        "email": "user@example.com",
        "phoneNumber": "0987654321",
        "role": "Staff",
        "employeeCode": "RES-20261030-01",
        "isActive": true,
        "createdAt": "2026-10-30T08:00:00Z"
      }
    ],
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.

#### 5.10.8 `PATCH` /api/admin/users/{id}/status

**Description:** Activates or deactivates an internal system account, immediately toggling login access.

- **Controller / Action:** `AdminController.UpdateStatusAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Request Body (`UpdateUserStatusModel`):**
  ```json
  {
    "isActive": true
  }
  ```
- **Success Response Contract:** `ApiResult<bool>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": true,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

### 5.11 Module 11: Admin Simulation & Testing

#### 5.11.1 `GET` /api/admin/simulation

**Description:** Retrieves global simulation parameters, accelerated time scale factor, and artificial clock offsets.

- **Controller / Action:** `AdminSimulationController.GetAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Success Response Contract:** `ApiResult<ParkingSimulationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "enabled": false,
      "blockSeconds": 1,
      "depositHoldSeconds": 1,
      "holdSeconds": 1,
      "depositPaymentSeconds": 1,
      "earlyArrivalSeconds": 1,
      "cancelRefundSeconds": 1,
      "exitWindowSeconds": 1,
      "normal": {
        "depositHoldMinutes": 60,
        "holdMinutes": 60,
        "depositPaymentMinutes": 60,
        "earlyArrivalMinutes": 60,
        "cancelRefundMinutes": 60,
        "exitWindowMinutes": 60
      },
      "updatedAt": "2026-10-30T08:00:00Z",
      "updatedBy": "2026-10-30"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.

#### 5.11.2 `PUT` /api/admin/simulation

**Description:** Configures sandbox simulation engine parameters to accelerate fee calculation and reservation hold testing.

- **Controller / Action:** `AdminSimulationController.UpdateAsync`
- **Authentication / Authorization:** Bearer JWT Required (Admin)
- **Request Body (`UpdateParkingSimulationModel`):**
  ```json
  {
    "enabled": false,
    "blockSeconds": 1,
    "depositHoldSeconds": 1,
    "holdSeconds": 1,
    "depositPaymentSeconds": 1,
    "earlyArrivalSeconds": 1,
    "cancelRefundSeconds": 1,
    "exitWindowSeconds": 1
  }
  ```
- **Success Response Contract:** `ApiResult<ParkingSimulationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "enabled": false,
      "blockSeconds": 1,
      "depositHoldSeconds": 1,
      "holdSeconds": 1,
      "depositPaymentSeconds": 1,
      "earlyArrivalSeconds": 1,
      "cancelRefundSeconds": 1,
      "exitWindowSeconds": 1,
      "normal": {
        "depositHoldMinutes": 60,
        "holdMinutes": 60,
        "depositPaymentMinutes": 60,
        "earlyArrivalMinutes": 60,
        "cancelRefundMinutes": 60,
        "exitWindowMinutes": 60
      },
      "updatedAt": "2026-10-30T08:00:00Z",
      "updatedBy": "2026-10-30"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (Admin) or facility assignment scope.

### 5.12 Module 12: Notifications

#### 5.12.1 `GET` /api/notifications

**Description:** Retrieves in-app notifications for the authenticated user with unread-only filtering and pagination.

- **Controller / Action:** `NotificationsController.GetMineAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Query Parameters:**
  - `unreadOnly` (bool, optional, default: `false`): Filter parameter.
  - `take` (int, optional, default: `30`): Filter parameter.
- **Success Response Contract:** `ApiResult<NotificationListModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "items": [
        {
          "id": 1001,
          "type": "Type_value",
          "title": "Title_value",
          "message": "Message_value",
          "refType": "RefType_value",
          "refId": "RefId_value",
          "eventAt": "2026-10-30T08:00:00Z",
          "data": null,
          "createdAt": "2026-10-30T08:00:00Z",
          "readAt": "2026-10-30T08:00:00Z"
        }
      ],
      "unreadCount": 100
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.

#### 5.12.2 `GET` /api/notifications/unread-count

**Description:** Retrieves the integer count of unread notifications for display in the application navigation badge.

- **Controller / Action:** `NotificationsController.GetUnreadCountAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Success Response Contract:** `ApiResult<int>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": 5,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.

#### 5.12.3 `POST` /api/notifications/{id:long}/read

**Description:** Marks a specific notification as read, updating its read timestamp.

- **Controller / Action:** `NotificationsController.MarkReadAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<NotificationModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "type": "Type_value",
      "title": "Title_value",
      "message": "Message_value",
      "refType": "RefType_value",
      "refId": "RefId_value",
      "eventAt": "2026-10-30T08:00:00Z",
      "data": null,
      "createdAt": "2026-10-30T08:00:00Z",
      "readAt": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

#### 5.12.4 `POST` /api/notifications/read-all

**Description:** Marks all unread notifications as read for the user, returning the integer count of updated items.

- **Controller / Action:** `NotificationsController.MarkAllReadAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Success Response Contract:** `ApiResult<int>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": 12,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `400 Bad Request`: Request validation failure or business constraint violation.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.

#### 5.12.5 `DELETE` /api/notifications/{id:long}

**Description:** Permanently deletes a notification item from the customer notification feed.

- **Controller / Action:** `NotificationsController.DeleteAsync`
- **Authentication / Authorization:** Bearer JWT Required (User)
- **Path Parameters:**
  - `id` (required): Target id identifier.
- **Success Response Contract:** `ApiResult<object>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": null,
    "errors": []
  }
  ```
- **Applicable HTTP Status Codes:**
  - `200 OK`: Request completed successfully.
  - `401 Unauthorized`: Missing, expired, or invalid Bearer authentication token.
  - `403 Forbidden`: Caller lacks required role (User) or facility assignment scope.
  - `404 Not Found`: Target resource with specified identifier does not exist.

---

## 6. Real-Time API / SignalR Hubs

The Parking Management System provides three dedicated ASP.NET Core SignalR hubs mapped in `src/PMS.API/Program.cs`.

### 6.1 SignalR Connection & Token Negotiation
SignalR clients establish WebSocket connections using the standard query string token negotiation protocol:
```javascript
const connection = new signalR.HubConnectionBuilder()
  .withUrl("https://localhost:7001/api/hubs/parking-ops", {
    accessTokenFactory: () => authService.getJwtToken()
  })
  .withAutomaticReconnect([0, 2000, 5000, 10000, 30000])
  .build();
```

### 6.2 LotHub (`/api/hubs/lots`)
- **Hub Endpoint:** `/api/hubs/lots`
- **Authentication:** Public / Anonymous
- **Target Group:** `lot-{lotId}`
- **Client-to-Server Methods:**
  - `JoinLot(int lotId)`: Subscribes client connection to live telemetry for specified lot.
  - `LeaveLot(int lotId)`: Unsubscribes client connection from specified lot.
- **Server-to-Client Events:**

| Event Name | Payload Schema | Description |
| :--- | :--- | :--- |
| `SlotStatusChanged` | `{ "lotId": int, "slotId": int, "status": string }` | Broadcast when a bay changes status (`Available`, `Occupied`, `Reserved`, `Maintenance`). |
| `LotStatusChanged` | `{ "lotId": int, "status": string }` | Broadcast when lot operational status updates (`Open`, `Full`, `Closed`). |
| `LayoutChanged` | `{ "lotId": int }` | Alerts 3D clients that lot layout geometry was modified. |
| `ZoneCounterChanged`| `{ "lotId": int, "zoneId": int, "availableSpaces": int, "occupiedSpaces": int, "totalCapacity": int }` | Broadcast when vehicle movement updates zone occupancy counts. |

### 6.3 ParkingOpsHub (`/api/hubs/parking-ops`)
- **Hub Endpoint:** `/api/hubs/parking-ops`
- **Authentication:** Bearer JWT Required (passed via query parameter `?access_token=<token>`).
- **Groups:** `ops-lot-{lotId}` (Staff and Managers are validated against facility assignment scope before joining).
- **Client-to-Server Methods:**
  - `JoinLot(int lotId)`: Subscribes operator connection to operational stream for assigned lot.
  - `LeaveLot(int lotId)`: Unsubscribes operator from operational stream.
- **Server-to-Client Events:**

| Event Name | Target Audience | Payload Schema | Description |
| :--- | :--- | :--- | :--- |
| `SessionChanged` | Ops Group (`ops-lot-{lotId}`) | `{ "lotId": int, "sessionId": long, "status": string, "plateNumber": string }` | Live vehicle check-in, checkout, or lost-ticket updates. |
| `PaymentChanged` | Customer & Ops Group | `{ "paymentId": long, "status": string, "purpose": string, "reservationId": long?, "sessionId": long? }` | Real-time payment settlement notifications. |
| `ZoneCounterChanged` | Ops Group | `{ "lotId": int, "zoneId": int, "availableSpaces": int, "occupiedSpaces": int, "totalCapacity": int }` | Live count updates for gate operators. |
| `OpsAlert` | Ops Group | `{ "lotId": int, "code": string, "message": string, "sessionId": long? }` | Operational gate alarms (e.g. barrier timeout, plate mismatch). |
| `GateRequestChanged`| Customer & Ops Group | `{ "lotId": int, "requestId": long, "kind": string, "status": string }` | Remote barrier opening requests and decisions. |
| `ReservationChanged`| Customer (`Clients.User`) | `{ "reservationId": long, "lotId": int, "status": string, "slotId": int? }` | Customer reservation confirmation, hold, or cancellation alerts. |
| `ReservationReminder`| Customer (`Clients.User`)| `{ "reservationId": long, "kind": "Arrival" \| "HoldEnding", "at": string }` | Upcoming arrival and grace period expiration warnings. |
| `NotificationCreated`| Customer (`Clients.User`)| `{ "notification": NotificationModel, "unreadCount": int }` | In-app user notification push with updated badge counter. |

### 6.4 VehicleVerificationHub (`/api/hubs/vehicle-verifications`)
- **Hub Endpoint:** `/api/hubs/vehicle-verifications`
- **Authentication:** Bearer JWT Required (passed via query parameter `?access_token=<token>`).
- **Target Group:** `vehicle-verification-reviewers` (Staff, Managers, and Admins automatically join upon connection).
- **Server-to-Client Events:**

| Event Name | Target Audience | Payload Schema | Description |
| :--- | :--- | :--- | :--- |
| `VerificationChanged` | Document Uploader & Reviewers | `{ "verificationId": guid, "vehicleId": guid, "status": string }` | Broadcast when vehicle papers are uploaded, submitted, approved, or rejected. |

### 6.5 Connection Resilience & Client Reconnect Policy
Frontend applications should handle network hiccups using exponential backoff retry:
1. Initial reconnect attempt after 0s, 2s, 5s, 10s, up to 30s intervals.
2. Upon reconnection, re-execute `JoinLot(lotId)` to rejoin group subscriptions.
3. Periodically query REST fallback endpoints (`GET /api/notifications/unread-count`, `GET /api/parking-sessions/current`) if connection is dropped.

---

## 7. Error Handling, Validation, and Rate Limiting

### 7.1 FluentValidation Request Pipeline
Request payloads are validated automatically using **FluentValidation** (`src/PMS.Application/Models/Validators/*.cs`). `ValidateModelAttribute` intercepts invalid requests prior to controller execution and returns **HTTP 400 Bad Request** using the standard envelope:
```json
{
  "succeeded": false,
  "result": null,
  "errors": [
    "Plate number must conform to Vietnamese syntax (e.g. 59A-123.45).",
    "VehicleTypeId must correspond to an active system vehicle category."
  ]
}
```

### 7.2 Global Exception Handling Middleware
Unhandled exceptions are captured centrally by `ExceptionHandlingMiddleware` (`src/PMS.API/Middlewares/ExceptionHandlingMiddleware.cs`):

| Exception Class | HTTP Status Code | Description |
| :--- | :---: | :--- |
| `NotFoundException` / `ResourceNotFoundException` | **HTTP 404 Not Found** | Target resource does not exist (e.g. unknown parking lot ID or session ID). |
| `BadRequestException` | **HTTP 400 Bad Request** | Request violates business rules (e.g. vehicle already checked in). |
| `ForbiddenException` | **HTTP 403 Forbidden** | Caller lacks role permissions or is not assigned to the managed lot. |
| `UnprocessableRequestException` | **HTTP 422 Unprocessable** | Request syntactically valid but semantically unprocessable (e.g. missing mandatory paper sides). |
| `ConflictException` | **HTTP 409 Conflict** | State collision (e.g. slot already booked by concurrent reservation). |
| Unhandled Exception | **HTTP 500 Internal Server Error** | Unexpected crash. In production environments, error details are sanitized. |

### 7.3 ASP.NET Core Rate Limiting Policies
Enforced by `RateLimitingMiddleware.cs` via ASP.NET Core rate limiting:
1. **`PasswordResetRateLimit`:** Applied to `POST /api/Users/forgot-password`. Allows a maximum of **5 requests per 15 minutes** per IP address. Exceeding requests return **HTTP 429 Too Many Requests**.
2. **`PublicLayoutRateLimit`:** Applied to `GET /api/public/lots/{lotId}/layout`. Allows a maximum of **120 requests per minute** per IP address to protect visual layout geometry from scraping. Exceeding requests return **HTTP 429 Too Many Requests**.

---

## 8. Shared Domain Enums & Integration Guidelines

### 8.1 Core Domain Enums

#### VehicleType
- `Motorbike`: Two-wheeled vehicles, standard bay dimensions.
- `Car`: Standard passenger cars (4-7 seats).
- `Truck`: Commercial vehicles or oversized passenger vans.
- `Bicycle`: Non-motorized two-wheeled cycles.

#### ReservationStatus
- `Pending`: Initial hold created, awaiting deposit settlement.
- `Confirmed`: Deposit confirmed, slot reserved for scheduled arrival window.
- `CheckedIn`: Customer arrived and checked in at the entry gate lane.
- `Cancelled`: Reservation cancelled by customer or manager; deposit refund policy applied.
- `Expired`: Customer failed to arrive before arrival window elapsed; deposit forfeited.

#### ParkingSessionStatus
- `Active`: Vehicle is physically present inside the parking facility.
- `Completed`: Vehicle has settled all fees and checked out through exit lane.
- `Overstay`: Vehicle exceeded planned departure window; overtime tariffs accumulating.
- `LostTicket`: Customer reported lost entry credential; penalty fee workflow active.
- `ClosedAbnormally`: Ghost session closed by facility manager with fee waiver.

#### PaymentMethod & PaymentProvider
- `PaymentProvider`: `VnPay`, `MoMo`, `Wallet`, `Cash`.
- `PaymentPurpose`: `ReservationDeposit`, `SessionPayment`, `WalletTopUp`, `LostTicketFine`, `OverstayFee`.

### 8.2 Automated Gate Hardware & ANPR Camera Integration
The automated entry/exit gate lane workflow integrates with physical barriers and cameras via the following sequence:
1. **Vehicle Loop Detection:** Ground inductive loop detects incoming vehicle at lane.
2. **ANPR Trigger:** IP camera captures high-resolution frame and recognizes license plate string.
3. **Lane Evaluation:** Staff terminal or automated gateway executes `POST /api/staff/sessions/check-in`:
   - Checks blacklist entries (`PlateListService`).
   - Validates active reservations for the recognized plate.
4. **Barrier Command:** Backend returns `ApiResult<SessionResponseModel>`. Lane controller pulses barrier relay.
5. **Crossing Confirmation:** Exit loop triggers `POST /api/staff/sessions/{id}/confirm-entry`. Live count broadcasts over `LotHub`.

### 8.3 Payment Gateway Callback & Reconciliation Notes
- **Idempotent Settlement:** Both browser redirect return (`/return`) and server-to-server IPN (`/ipn`) verify HMAC signatures and apply payments idempotently.
- **Localhost Development Limitation:** Gateway webhooks cannot reach `localhost` directly. Developers should use the `POST /api/payments/{id}/simulate-success` endpoint or tools like `ngrok` during integration testing.

---

## 9. Glossary and Terminology

| Term | Definition |
| :--- | :--- |
| **ANPR** | Automatic Number Plate Recognition (camera optical character recognition reading license plates). |
| **ApiResult<T>** | Standard ASP.NET Core response envelope wrapping payloads in `succeeded`, `result`, and `errors`. |
| **Exit Pass** | Cryptographically signed single-use token authorizing barrier lift after payment settlement. |
| **Ghost Session** | An open parking session caused by vehicle tailgating, sensor misreads, or hardware fault. |
| **Grace Period** | Initial time window (e.g. 15 minutes) during which vehicle exit is free or deposit holds remain valid. |
| **Hold Amount** | Temporary balance reservation in customer XXIV wallet securing parking bay access. |
| **IPN** | Instant Payment Notification (asynchronous server-to-server webhook from VNPay or MoMo). |
| **Lot Scope** | Resource-level security policy restricting staff and managers strictly to assigned parking lots. |
| **TOTP** | Time-based One-Time Password used during 2FA authentication challenges. |
| **XXIV Wallet** | In-app prepaid digital wallet balance used for instant one-touch gate payments. |
