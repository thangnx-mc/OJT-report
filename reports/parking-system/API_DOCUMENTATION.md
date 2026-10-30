# Parking Management System -- Backend API Documentation

> **API Version:** 1.0.0  
> **Protocol:** HTTPS (REST) & WSS (SignalR)  
> **Data Format:** JSON (`application/json`)  

---

## Table of Contents

- [1. API Overview](#1-api-overview)
  - [1.1 Purpose and Scope](#11-purpose-and-scope)
  - [1.2 Base URLs and Environments](#12-base-urls-and-environments)
  - [1.3 REST API vs. SignalR Real-Time Communication](#13-rest-api-vs-signalr-real-time-communication)
- [2. API Conventions](#2-api-conventions)
  - [2.1 HTTP Methods and Semantics](#21-http-methods-and-semantics)
  - [2.2 Content Types](#22-content-types)
  - [2.3 Response Envelope Contract](#23-response-envelope-contract)
  - [2.4 Pagination and Query Filtering](#24-pagination-and-query-filtering)
  - [2.5 Data Types and Formatting Standards](#25-data-types-and-formatting-standards)
  - [2.6 Non-Standard Response Endpoints](#26-non-standard-response-endpoints)
- [3. Authentication and Authorization](#3-authentication-and-authorization)
  - [3.1 JWT Bearer Authentication](#31-jwt-bearer-authentication)
  - [3.2 Roles and Access Control Matrix](#32-roles-and-access-control-matrix)
  - [3.3 Two-Factor Authentication (2FA) Challenge Flow](#33-two-factor-authentication-2fa-challenge-flow)
  - [3.4 Passwordless OTP and OAuth Sign-In](#34-passwordless-otp-and-oauth-sign-in)
  - [3.5 Real-Time Hub Authentication](#35-real-time-hub-authentication)
- [4. API Reference](#4-api-reference)
  - [4.1 Module 01: Authentication & User Accounts](#41-module-01-authentication-user-accounts)
  - [4.2 Module 02: Parking Lots, Layouts, Zones & Slots](#42-module-02-parking-lots-layouts-zones-slots)
  - [4.3 Module 03: Customer Reservations](#43-module-03-customer-reservations)
  - [4.4 Module 04: Customer Parking Sessions](#44-module-04-customer-parking-sessions)
  - [4.5 Module 05: Staff Gate Operations & Gate Requests](#45-module-05-staff-gate-operations-gate-requests)
  - [4.6 Module 06: Manager Operations & Pricing](#46-module-06-manager-operations-pricing)
  - [4.7 Module 07: Vehicle Verification & Document Management](#47-module-07-vehicle-verification-document-management)
  - [4.8 Module 08: Payments & Customer Wallet](#48-module-08-payments-customer-wallet)
  - [4.9 Module 09: Revenue Reporting](#49-module-09-revenue-reporting)
  - [4.10 Module 10: Admin User & System Management](#410-module-10-admin-user-system-management)
  - [4.11 Module 11: Admin Simulation & Testing](#411-module-11-admin-simulation-testing)
  - [4.12 Module 12: Notifications](#412-module-12-notifications)
- [5. SignalR Real-Time Communication](#5-signalr-real-time-communication)
  - [5.1 LotHub (/api/hubs/lots)](#51-lothub-apihubslots)
  - [5.2 ParkingOpsHub (/api/hubs/parking-ops)](#52-parkingopshub-apihubsparking-ops)
  - [5.3 VehicleVerificationHub (/api/hubs/vehicle-verifications)](#53-vehicleverificationhub-apihubsvehicle-verifications)
- [6. Validation and Error Handling](#6-validation-and-error-handling)
  - [6.1 Model Validation Pipeline](#61-model-validation-pipeline)
  - [6.2 HTTP Status Code Error Mapping](#62-http-status-code-error-mapping)
  - [6.3 Rate Limiting Policies](#63-rate-limiting-policies)

---

## 1. API Overview

### 1.1 Purpose and Scope
The **Parking Management System (PMS)** backend API provides a comprehensive programmatic interface for managing physical parking facilities, automated barrier gates, vehicle recognition, online reservations, digital customer wallets, payment gateway integrations, vehicle paper verifications, dynamic pricing, and financial revenue reporting.

### 1.2 Base URLs and Environments
The API runs over HTTPS and uses the base path `/api`. The default development configurations defined in `src/PMS.API/Properties/launchSettings.json` and `src/config/env.ts` are:

| Environment Profile | Base URL | Transport | Notes |
| :--- | :--- | :---: | :--- |
| **IIS Express (Default Local Dev)** | `https://localhost:44391/api` | HTTPS | Default endpoint configured in frontend development client (`DEFAULT_API_BASE_URL`). |
| **Kestrel HTTPS Profile** | `https://localhost:5001/api` | HTTPS | Direct Kestrel runtime endpoint. |
| **Kestrel HTTP Profile** | `http://localhost:5000/api` | HTTP | Direct unencrypted fallback. |

In browser environments, the API base URL can also be dynamically customized via `localStorage` setting `pms_custom_api_url`.

### 1.3 REST API vs. SignalR Real-Time Communication
- **REST APIs (HTTP/HTTPS):** Stateless, client-initiated request/response interactions. All mutations (check-in, checkout, reservations, payments, and CRUD) must be executed over REST.
- **SignalR Hubs (WebSockets / SSE):** Persistent duplex connections. Never used for state mutations; used exclusively by the server to broadcast asynchronous notifications, sensor updates, gate triggers, and queue changes.

---

## 2. API Conventions

### 2.1 HTTP Methods and Semantics
- `GET`: Idempotent data retrieval. Query strings are used for filtering, pagination, and sorting.
- `POST`: Resource creation, state mutations, and non-idempotent business commands.
- `PUT`: Complete idempotent resource updates.
- `PATCH`: Partial state updates and status transitions.
- `DELETE`: Resource archival or removal.

### 2.2 Content Types
- Standard Request and Response: `application/json; charset=utf-8`.
- Binary File Uploads: `multipart/form-data` (avatar pictures, vehicle document photos).
- File Streams: `image/jpeg`, `image/png`, or `application/pdf` via `FileResult`.

### 2.3 Response Envelope Contract
All standard endpoints wrap responses in the generic `ApiResult<T>` envelope.

#### Success Response (`200 OK` / `201 Created`)
```json
{
  "succeeded": true,
  "result": { ... },
  "errors": []
}
```

#### Failure Response (`400`, `401`, `403`, `404`, `409`, `422`, `500`)
```json
{
  "succeeded": false,
  "result": null,
  "errors": [
    "A descriptive error message explaining why the request failed."
  ]
}
```

### 2.4 Pagination and Query Filtering
Endpoints supporting pagination accept standard query parameters:
- `pageNumber` (integer, default: `1`): 1-indexed page index.
- `pageSize` (integer, default: `10`, maximum: `100`): Items per page.
- `searchTerm` (string, optional): Keyword search string.

Paginated responses return `PaginationResponseModel<T>` within `result`:
```json
{
  "succeeded": true,
  "result": {
    "items": [ ... ],
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

### 2.5 Data Types and Formatting Standards
- **Timestamps:** ISO-8601 UTC strings (`YYYY-MM-DDTHH:mm:ss.sssZ`).
- **TimeOfDay:** 24-hour time strings formatted as `HH:mm:ss` (e.g. `06:00:00`).
- **Entity Identifiers:**
  - `Guid` (UUID v4): Users, Vehicles, Verifications, Uploaded Documents.
  - `int` (32-bit Integer): Lots, Zones, Slots, Pricing Rules, Notifications, Blacklist entries.
  - `long` (64-bit Integer): Reservations, Parking Sessions, Gate Requests, Payments.

### 2.6 Non-Standard Response Endpoints
1. **Binary File Streams:** `GET /api/users/profile/vehicles/{id}/documents/files/{docId}` and `GET /api/vehicle-verifications/{id}/files/{docId}` stream binary data.
2. **Payment Gateway Callbacks:** `GET /api/payments/vnpay/return` and `GET /api/payments/momo/return` issue HTTP 302 redirects.
3. **SignalR Connections:** `/api/hubs/*` negotiate WebSocket handshake protocols.

---

## 3. Authentication and Authorization

### 3.1 JWT Bearer Authentication
Protected endpoints require an `Authorization` header containing a valid Bearer token:
```http
Authorization: Bearer <jwt_token>
```
- **Algorithm:** HMAC-SHA256 (`JwtHelper.cs`).
- **Token Lifetime:** 7 days (604,800 seconds).
- **Token Claims:** `nameidentifier` (User GUID), `email`, `role`, and optional `purpose`.

### 3.2 Roles and Access Control Matrix
The system enforces four distinct roles:
- `User`: End-customer driver.
- `Staff`: Gate operator executing physical lane operations.
- `Manager`: Facility supervisor administering assigned parking lots and tariffs.
- `Admin`: Global administrator.

| Functional Area | Public / Anonymous | User (Customer) | Staff (Operator) | Manager | Admin |
| :--- | :---: | :---: | :---: | :---: | :---: |
| **Authentication (Login, Register, OTP)** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **User Profile & Vehicle CRUD** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Browse Public Lots & 2D Layouts** | ✅ | ✅ | ✅ | ✅ | ✅ |
| **Customer Reservations** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Customer Parking Sessions & Payments** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Customer Digital Wallet Topup** | ❌ | ✅ | ❌ | ❌ | ❌ |
| **Gate Check-In & Check-Out Execution** | ❌ | ❌ | ✅ | ❌ | ❌ |
| **Gate Cash Collection** | ❌ | ❌ | ✅ | ❌ | ❌ |
| **Lost Ticket Reporting at Gate** | ❌ | ❌ | ✅ | ❌ | ❌ |
| **Lost Ticket Approval / Rejection** | ❌ | ❌ | ❌ | ✅ (Assigned Lots) | ❌ |
| **Abnormal Session Forced Closure** | ❌ | ❌ | ❌ | ✅ (Assigned Lots) | ❌ |
| **Pricing Rule Administration** | ❌ | ❌ | ❌ | ✅ (Assigned Lots) | ❌ |
| **Staff Account Management** | ❌ | ❌ | ❌ | ✅ (Assigned Lots) | ❌ |
| **Manager Revenue Reporting** | ❌ | ❌ | ❌ | ✅ (Assigned Lots) | ❌ |
| **Admin Revenue Reporting (System-wide)**| ❌ | ❌ | ❌ | ❌ | ✅ |
| **Manager Account CRUD** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **User Account Status Toggle** | ❌ | ❌ | ❌ | ❌ | ✅ |
| **Vehicle Paper Review & Approval** | ❌ | ❌ | ✅ | ❌ | ✅ |
| **Simulation Sandbox Controls** | ❌ | ❌ | ❌ | ❌ | ✅ |

### 3.3 Two-Factor Authentication (2FA) Challenge Flow
When 2FA is enabled:
1. Initial login produces an intermediate token containing claim `purpose = "2FA_Challenge"`.
2. All standard endpoints reject this intermediate token with **HTTP 403 Forbidden**.
3. The intermediate token is authorized only at:
   - `POST /api/Users/two-factor` (submits 6-digit TOTP code).
   - `POST /api/Users/two-factor/resend` (resends code).
4. Successful code submission returns the final session JWT containing the assigned role.

### 3.4 Passwordless OTP and OAuth Sign-In
- **Passwordless Login:** Client calls `POST /api/Users/login-otp/request` to receive a 6-digit code via email, then submits it to `POST /api/Users/login-otp/verify` to receive a session JWT.
- **Google Sign-In:** Client passes a Google ID token to `POST /api/Users/login-google` with `{ "idToken": "..." }`. The backend validates the token, provisions the account if absent, and returns a session JWT.
- **Password Reset:** Client calls `POST /api/Users/forgot-password` (rate-limited to 5 requests per 15 minutes), receives an email token, and submits `POST /api/Users/reset-password`.

### 3.5 Real-Time Hub Authentication
Because WebSocket browser connections cannot supply custom HTTP headers during connection establishment, SignalR clients pass the JWT in the query string:
```text
wss://<host>/api/hubs/parking-ops?access_token=<jwt_token>
```
`src/PMS.API/Program.cs` intercepts this query parameter in `OnMessageReceived` and attaches the authenticated principal to the hub context.

---

## 4. API Reference

This section documents the 150 Backend HTTP endpoints, organized across 12 functional modules.

### 4.1 Module 01: Authentication & User Accounts

#### 4.1.1 `GET` /api/Users

**Description:** Retrieves all registered customer accounts.

- **Action Method:** `UsersController.GetAllAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Query Parameters:**
  - `role` (string, default: `null`)
  - `isActive` (bool?, default: `null`)
  - `searchTerm` (string, default: `null`)
- **Response Contract:** `ApiResult<IEnumerable<UserResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.1.2 `GET` /api/Users/{id}

**Description:** Retrieves a single user account by its unique identifier.

- **Action Method:** `UsersController.GetByIdAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (string, required)
- **Response Contract:** `ApiResult<UserResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "FullName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.1.3 `POST` /api/Users/internal

**Description:** Provisions an internal staff or manager user account.

- **Action Method:** `UsersController.CreateInternalUserAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Request Body (`CreateInternalUserModel`):**
  ```json
  {
    "userName": "UserName_value",
    "fullName": "FullName_value",
    "email": "user@example.com",
    "password": "P@ssword123",
    "phoneNumber": "0987654321",
    "role": "Staff",
    "assignedParkingLot": "AssignedParkingLot_value"
  }
  ```
- **Response Contract:** `ApiResult<CreateUserResponseModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.1.4 `PATCH` /api/Users/{id}/status

**Description:** Activates or deactivates an internal user account.

- **Action Method:** `UsersController.UpdateStatusAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (string, required)
- **Request Body (`UpdateUserStatusModel`):**
  ```json
  {
    "isActive": true
  }
  ```
- **Response Contract:** `ApiResult<bool>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.1.5 `POST` /api/Users/register

**Description:** Registers a new customer account with email and password.

- **Action Method:** `UsersController.RegisterAsync`
- **Authentication:** None (Public)
- **Response Contract:** `ApiResult<CreateUserResponseModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.1.6 `POST` /api/Users/authenticate

**Description:** Authenticates user credentials and issues a JWT session token (or a 2FA challenge token if 2FA is active).

- **Action Method:** `UsersController.LoginAsync`
- **Authentication:** None (Public)
- **Response Contract:** `ApiResult<LoginResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "username": "Username_value",
      "fullName": "FullName_value",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "token": "Token_value",
      "requiresTwoFactor": false,
      "twoFactorToken": "TwoFactorToken_value",
      "destination": "Destination_value"
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.1.7 `POST` /api/Users/two-factor

**Description:** Verifies a 6-digit TOTP code against a 2FA challenge token to complete authentication.

- **Action Method:** `UsersController.VerifyTwoFactorAsync`
- **Authentication:** None (Public)
- **Request Body (`TwoFactorLoginModel`):**
  ```json
  {
    "twoFactorToken": "TwoFactorToken_value",
    "code": "RES-20261030-01",
    "email": "user@example.com",
    "rememberMe": false
  }
  ```
- **Response Contract:** `ApiResult<LoginResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "username": "Username_value",
      "fullName": "FullName_value",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "token": "Token_value",
      "requiresTwoFactor": false,
      "twoFactorToken": "TwoFactorToken_value",
      "destination": "Destination_value"
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.1.8 `POST` /api/Users/two-factor/resend

**Description:** Resends a two-factor verification OTP code to the user.

- **Action Method:** `UsersController.ResendTwoFactorOtpAsync`
- **Authentication:** None (Public)
- **Request Body (`ResendTwoFactorModel`):**
  ```json
  {
    "twoFactorToken": "TwoFactorToken_value",
    "email": "user@example.com"
  }
  ```
- **Response Contract:** `ApiResult<ResendTwoFactorResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "succeeded": true,
      "twoFactorToken": "TwoFactorToken_value",
      "destination": "Destination_value",
      "message": "Message_value"
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.1.9 `POST` /api/Users/login-otp/request

**Description:** Initiates passwordless login by dispatching a 6-digit OTP code to the registered email address.

- **Action Method:** `UsersController.RequestLoginOtpAsync`
- **Authentication:** None (Public)
- **Response Contract:** `ApiResult<string>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.1.10 `POST` /api/Users/login-otp/verify

**Description:** Authenticates a passwordless login request by verifying the received 6-digit OTP code.

- **Action Method:** `UsersController.LoginWithOtpAsync`
- **Authentication:** None (Public)
- **Response Contract:** `ApiResult<LoginResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "username": "Username_value",
      "fullName": "FullName_value",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "token": "Token_value",
      "requiresTwoFactor": false,
      "twoFactorToken": "TwoFactorToken_value",
      "destination": "Destination_value"
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.1.11 `POST` /api/Users/forgot-password

**Description:** Generates and emails a password reset token to the requested account (rate-limited).

- **Action Method:** `UsersController.ForgotPasswordAsync`
- **Authentication:** None (Public)
- **Request Body (`ForgotPasswordModel`):**
  ```json
  {
    "email": "user@example.com"
  }
  ```
- **Response Contract:** `ApiResult<string>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.1.12 `POST` /api/Users/reset-password

**Description:** Resets an account password using a validated password reset token.

- **Action Method:** `UsersController.ResetPasswordAsync`
- **Authentication:** None (Public)
- **Request Body (`ResetPasswordModel`):**
  ```json
  {
    "email": "user@example.com",
    "token": "Token_value",
    "newPassword": "P@ssword123",
    "confirmPassword": "P@ssword123"
  }
  ```
- **Response Contract:** `ApiResult<string>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.1.13 `POST` /api/Users/login-google

**Description:** Authenticates an account via Google OAuth token exchange.

- **Action Method:** `UsersController.LoginWithGoogleAsync`
- **Authentication:** None (Public)
- **Response Contract:** `ApiResult<LoginResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "username": "Username_value",
      "fullName": "FullName_value",
      "email": "user@example.com",
      "phoneNumber": "0987654321",
      "role": "Staff",
      "employeeCode": "RES-20261030-01",
      "token": "Token_value",
      "requiresTwoFactor": false,
      "twoFactorToken": "TwoFactorToken_value",
      "destination": "Destination_value"
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.1.14 `GET` /api/Users/profile

**Description:** Retrieves the authenticated user profile information.

- **Action Method:** `UsersController.GetProfileAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Response Contract:** `ApiResult<UserProfileResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "fullName": "FullName_value",
      "phoneNumber": "0987654321",
      "email": "user@example.com",
      "avatarUrl": "https://example.com/payment/return",
      "citizenId": "CitizenId_value",
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
          "citizenId": "CitizenId_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.1.15 `PUT` /api/Users/profile

**Description:** Updates the authenticated user personal profile information.

- **Action Method:** `UsersController.UpdateProfileAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Request Body (`UpdateUserProfileModel`):**
  ```json
  {
    "fullName": "FullName_value",
    "email": "user@example.com",
    "phoneNumber": "0987654321",
    "avatarUrl": "https://example.com/payment/return",
    "citizenId": "CitizenId_value"
  }
  ```
- **Response Contract:** `ApiResult<UserProfileResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "fullName": "FullName_value",
      "phoneNumber": "0987654321",
      "email": "user@example.com",
      "avatarUrl": "https://example.com/payment/return",
      "citizenId": "CitizenId_value",
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
          "citizenId": "CitizenId_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.1.16 `GET` /api/Users/profile/check-unique

**Description:** Executes CheckProfileUniquenessAsync operation.

- **Action Method:** `UsersController.CheckProfileUniquenessAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Query Parameters:**
  - `phone` (string?, default: `null`)
  - `citizenId` (string?, default: `null`)
- **Response Contract:** `ApiResult<ProfileFieldAvailabilityModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.1.17 `POST` /api/Users/profile/avatar

**Description:** Uploads a new profile picture for the authenticated user.

- **Action Method:** `UsersController.UploadAvatarAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Response Contract:** `ApiResult<UserProfileResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "fullName": "FullName_value",
      "phoneNumber": "0987654321",
      "email": "user@example.com",
      "avatarUrl": "https://example.com/payment/return",
      "citizenId": "CitizenId_value",
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
          "citizenId": "CitizenId_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.1.18 `GET` /api/Users/profile/vehicles

**Description:** Retrieves all registered vehicles associated with the authenticated customer.

- **Action Method:** `UsersController.GetVehiclesAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Response Contract:** `ApiResult<List<VehicleResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.1.19 `GET` /api/Users/profile/vehicles/check-plate

**Description:** Executes CheckPlateAvailabilityAsync operation.

- **Action Method:** `UsersController.CheckPlateAvailabilityAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Query Parameters:**
  - `plate` (string, default: `none`)
  - `excludeVehicleId` (Guid?, default: `null`)
- **Response Contract:** `ApiResult<PlateAvailabilityResponseModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.1.20 `POST` /api/Users/profile/vehicles

**Description:** Registers a new vehicle under the authenticated customer profile.

- **Action Method:** `UsersController.AddVehicleAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Request Body (`CreateVehicleModel`):**
  ```json
  {
    "licensePlate": "59A-123.45",
    "vehicleTypeId": 1,
    "brand": "Brand_value",
    "model": "Model_value",
    "color": "Color_value",
    "citizenId": "CitizenId_value",
    "isDefault": false
  }
  ```
- **Response Contract:** `ApiResult<VehicleResponseModel>`
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
      "citizenId": "CitizenId_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.1.21 `PUT` /api/Users/profile/vehicles/{vehicleId:guid}

**Description:** Updates details of an existing registered vehicle.

- **Action Method:** `UsersController.UpdateVehicleAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Path Parameters:**
  - `vehicleId` (guid, required)
- **Request Body (`UpdateVehicleModel`):**
  ```json
  {
    "licensePlate": "59A-123.45",
    "vehicleTypeId": 1,
    "brand": "Brand_value",
    "model": "Model_value",
    "color": "Color_value",
    "citizenId": "CitizenId_value",
    "isDefault": false,
    "isActive": true
  }
  ```
- **Response Contract:** `ApiResult<VehicleResponseModel>`
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
      "citizenId": "CitizenId_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.1.22 `DELETE` /api/Users/profile/vehicles/{vehicleId:guid}

**Description:** Removes or deactivates a registered vehicle from the customer profile.

- **Action Method:** `UsersController.DeleteVehicleAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Path Parameters:**
  - `vehicleId` (guid, required)
- **Response Contract:** `ApiResult<bool>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.1.23 `PATCH` /api/Users/profile/vehicles/{vehicleId:guid}/default

**Description:** Sets a registered vehicle as the customer default vehicle.

- **Action Method:** `UsersController.SetDefaultVehicleAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Path Parameters:**
  - `vehicleId` (guid, required)
- **Response Contract:** `ApiResult<VehicleResponseModel>`
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
      "citizenId": "CitizenId_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.1.24 `GET` /api/Users/profile/vehicle-types

**Description:** Retrieves active vehicle categories supported by the parking system.

- **Action Method:** `UsersController.GetVehicleTypesAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Response Contract:** `ApiResult<List<VehicleTypeResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.1.25 `PUT` /api/Users/profile/change-password

**Description:** Changes the account password for the authenticated user.

- **Action Method:** `UsersController.ChangePasswordAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Request Body (`ChangePasswordModel`):**
  ```json
  {
    "currentPassword": "P@ssword123",
    "newPassword": "P@ssword123",
    "confirmPassword": "P@ssword123"
  }
  ```
- **Response Contract:** `ApiResult<bool>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.1.26 `GET` /api/Users/{id}/profile

**Description:** Executes GetProfileByUserIdAsync operation.

- **Action Method:** `UsersController.GetProfileByUserIdAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (string, required)
- **Response Contract:** `ApiResult<UserProfileResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "userId": "UserId_value",
      "fullName": "FullName_value",
      "phoneNumber": "0987654321",
      "email": "user@example.com",
      "avatarUrl": "https://example.com/payment/return",
      "citizenId": "CitizenId_value",
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
          "citizenId": "CitizenId_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

---

### 4.2 Module 02: Parking Lots, Layouts, Zones & Slots

#### 4.2.1 `GET` /api/lot-templates

**Description:** Executes GetTemplates operation.

- **Action Method:** `LayoutsController.GetTemplates`
- **Authentication:** None (Public)
- **Response Contract:** `ApiResult<IReadOnlyList<LotTemplateResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.2.2 `GET` /api/public/lots/{lotId:int}/layout

**Description:** Executes GetPublishedAsync operation.

- **Action Method:** `LayoutsController.GetPublishedAsync`
- **Authentication:** None (Public)
- **Path Parameters:**
  - `lotId` (int, required)
- **Response Contract:** `ApiResult<LayoutResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "parkingLotId": 1,
      "parkingLotName": "ParkingLotName_value",
      "lotStatus": "Active",
      "version": 1,
      "templateId": "59A-123.45",
      "publishedAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "floors": [
        {
          "floorNo": 1,
          "name": "Name_value",
          "kind": "Kind_value",
          "rows": 1,
          "cols": 1
        }
      ],
      "zones": [
        {
          "zoneId": 1,
          "code": "RES-20261030-01",
          "name": "Name_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `404 Not Found`: Target entity not found.

#### 4.2.3 `GET` /api/public/lots/{lotId:int}/layout/status

**Description:** Retrieves real-time occupancy and slot statuses across the visual layout.

- **Action Method:** `LayoutsController.GetStatusAsync`
- **Authentication:** None (Public)
- **Path Parameters:**
  - `lotId` (int, required)
- **Response Contract:** `ApiResult<LayoutStatusResponseModel>`
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
          "total": 1,
          "free": 1
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `404 Not Found`: Target entity not found.

#### 4.2.4 `GET` /api/public/lots/{lotId:int}/layout/2d

**Description:** Executes Get2DAsync operation.

- **Action Method:** `LayoutsController.Get2DAsync`
- **Authentication:** None (Public)
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `detail` (string, default: `LayoutDetail.Slots`)
- **Response Contract:** `ApiResult<Layout2DResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "parkingLotId": 1,
      "parkingLotName": "ParkingLotName_value",
      "lotStatus": "Active",
      "version": 1,
      "detail": "Detail_value",
      "floors": [
        {
          "floorNo": 1,
          "name": "Name_value",
          "kind": "Kind_value",
          "rows": 1,
          "cols": 1,
          "total": 1,
          "free": 1,
          "cells": [],
          "zoneBlocks": []
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `404 Not Found`: Target entity not found.

#### 4.2.5 `GET` /api/lots

**Description:** Retrieves all registered customer accounts.

- **Action Method:** `LotsController.GetAllAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin,Manager`
- **Query Parameters:**
  - `searchTerm` (string, default: `null`)
  - `status` (string, default: `null`)
  - `includeDeleted` (bool, default: `false`)
- **Response Contract:** `ApiResult<IEnumerable<ParkingLotResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.2.6 `GET` /api/public/lots

**Description:** Retrieves public information for all active parking lots.

- **Action Method:** `LotsController.GetPublicLotsAsync`
- **Authentication:** None (Public)
- **Query Parameters:**
  - `searchTerm` (string, default: `null`)
- **Response Contract:** `ApiResult<IEnumerable<PublicParkingLotResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.2.7 `GET` /api/public/lots/{id:int}/prices

**Description:** Retrieves public pricing tariff schedules for customer viewing.

- **Action Method:** `LotsController.GetPublicPricesAsync`
- **Authentication:** None (Public)
- **Path Parameters:**
  - `id` (int, required)
- **Response Contract:** `ApiResult<IReadOnlyList<PublicLotPriceModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `404 Not Found`: Target entity not found.

#### 4.2.8 `GET` /api/lots/{id:int}

**Description:** Retrieves a single user account by its unique identifier.

- **Action Method:** `LotsController.GetByIdAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin,Manager`
- **Path Parameters:**
  - `id` (int, required)
- **Response Contract:** `ApiResult<ParkingLotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Name_value",
      "address": "Address_value",
      "city": "City_value",
      "district": "District_value",
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
      "totalZones": 1,
      "totalSlots": 1
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.2.9 `POST` /api/lots

**Description:** Creates a new parking lot facility.

- **Action Method:** `LotsController.CreateAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Request Body (`CreateParkingLotModel`):**
  ```json
  {
    "code": "RES-20261030-01",
    "name": "Name_value",
    "address": "Address_value",
    "city": "City_value",
    "district": "District_value",
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
- **Response Contract:** `ApiResult<ParkingLotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Name_value",
      "address": "Address_value",
      "city": "City_value",
      "district": "District_value",
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
      "totalZones": 1,
      "totalSlots": 1
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.2.10 `PUT` /api/lots/{id:int}

**Description:** Updates configuration settings of an existing parking lot.

- **Action Method:** `LotsController.UpdateAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (int, required)
- **Request Body (`UpdateParkingLotModel`):**
  ```json
  {
    "name": "Name_value",
    "address": "Address_value",
    "city": "City_value",
    "district": "District_value",
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
- **Response Contract:** `ApiResult<ParkingLotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Name_value",
      "address": "Address_value",
      "city": "City_value",
      "district": "District_value",
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
      "totalZones": 1,
      "totalSlots": 1
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.2.11 `PUT` /api/lots/{id:int}/template

**Description:** Applies a predefined grid template to generate zones and slots for a lot.

- **Action Method:** `LotsController.ApplyTemplateAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (int, required)
- **Request Body (`ApplyLotTemplateModel`):**
  ```json
  {
    "layoutTemplate": "59A-123.45"
  }
  ```
- **Response Contract:** `ApiResult<ParkingLotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Name_value",
      "address": "Address_value",
      "city": "City_value",
      "district": "District_value",
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
      "totalZones": 1,
      "totalSlots": 1
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.2.12 `PATCH` /api/lots/{id:int}/status

**Description:** Activates or deactivates an internal user account.

- **Action Method:** `LotsController.UpdateStatusAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin,Manager`
- **Path Parameters:**
  - `id` (int, required)
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
- **Response Contract:** `ApiResult<bool>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.2.13 `DELETE` /api/lots/{id:int}

**Description:** Deactivates or removes a parking lot.

- **Action Method:** `LotsController.DeleteAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (int, required)
- **Response Contract:** `ApiResult<bool>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.2.14 `GET` /api/slots

**Description:** Retrieves all registered customer accounts.

- **Action Method:** `SlotsController.GetAllAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin,Manager`
- **Query Parameters:**
  - `zoneId` (int?, default: `null`)
  - `vehicleTypeId` (int?, default: `null`)
  - `status` (string, default: `null`)
  - `isActive` (bool?, default: `null`)
- **Response Contract:** `ApiResult<IEnumerable<SlotResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.2.15 `GET` /api/zones/{zoneId:int}/slots

**Description:** Manages individual parking bay slots, status overrides, and sensor bindings.

- **Action Method:** `SlotsController.GetByZoneIdAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin,Manager`
- **Path Parameters:**
  - `zoneId` (int, required)
- **Response Contract:** `ApiResult<IEnumerable<SlotResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.2.16 `GET` /api/slots/{id:int}

**Description:** Retrieves a single user account by its unique identifier.

- **Action Method:** `SlotsController.GetByIdAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin,Manager`
- **Path Parameters:**
  - `id` (int, required)
- **Response Contract:** `ApiResult<SlotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "zoneId": 1,
      "zoneCode": "RES-20261030-01",
      "zoneName": "ZoneName_value",
      "parkingLotId": 1,
      "parkingLotCode": "RES-20261030-01",
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "VehicleTypeName_value",
      "code": "RES-20261030-01",
      "status": "Active",
      "dedicatedVehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "dedicatedNote": "DedicatedNote_value",
      "isActive": true
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.2.17 `PUT` /api/slots/{id:int}

**Description:** Manages individual parking bay slots, status overrides, and sensor bindings.

- **Action Method:** `SlotsController.UpdateAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (int, required)
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
- **Response Contract:** `ApiResult<SlotResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "zoneId": 1,
      "zoneCode": "RES-20261030-01",
      "zoneName": "ZoneName_value",
      "parkingLotId": 1,
      "parkingLotCode": "RES-20261030-01",
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "VehicleTypeName_value",
      "code": "RES-20261030-01",
      "status": "Active",
      "dedicatedVehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "dedicatedNote": "DedicatedNote_value",
      "isActive": true
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.2.18 `PATCH` /api/slots/{id:int}/status

**Description:** Activates or deactivates an internal user account.

- **Action Method:** `SlotsController.UpdateStatusAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin,Manager`
- **Path Parameters:**
  - `id` (int, required)
- **Request Body (`UpdateSlotStatusModel`):**
  ```json
  {
    "status": "Active"
  }
  ```
- **Response Contract:** `ApiResult<bool>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.2.19 `GET` /api/zones

**Description:** Retrieves all registered customer accounts.

- **Action Method:** `ZonesController.GetAllAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin,Manager`
- **Query Parameters:**
  - `lotId` (int?, default: `null`)
- **Response Contract:** `ApiResult<IEnumerable<ZoneResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.2.20 `GET` /api/lots/{lotId:int}/zones

**Description:** Manages zone configurations within parking lot levels.

- **Action Method:** `ZonesController.GetByLotIdAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin,Manager`
- **Path Parameters:**
  - `lotId` (int, required)
- **Response Contract:** `ApiResult<IEnumerable<ZoneResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.2.21 `GET` /api/zones/{id:int}

**Description:** Retrieves a single user account by its unique identifier.

- **Action Method:** `ZonesController.GetByIdAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin,Manager`
- **Path Parameters:**
  - `id` (int, required)
- **Response Contract:** `ApiResult<ZoneResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "parkingLotId": 1,
      "parkingLotCode": "RES-20261030-01",
      "parkingLotName": "ParkingLotName_value",
      "code": "RES-20261030-01",
      "name": "Name_value",
      "zoneType": "ZoneType_value",
      "floorNo": 1,
      "capacity": 100,
      "maxReservationSlots": 1,
      "layoutJson": "LayoutJson_value",
      "isActive": true,
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "totalSlots": 1
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.2.22 `PUT` /api/zones/{id:int}

**Description:** Manages zone configurations within parking lot levels.

- **Action Method:** `ZonesController.UpdateAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (int, required)
- **Request Body (`UpdateZoneModel`):**
  ```json
  {
    "name": "Name_value",
    "zoneType": "ZoneType_value",
    "floorNo": 1,
    "capacity": 100,
    "maxReservationSlots": 1,
    "layoutJson": "LayoutJson_value",
    "isActive": true
  }
  ```
- **Response Contract:** `ApiResult<ZoneResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "parkingLotId": 1,
      "parkingLotCode": "RES-20261030-01",
      "parkingLotName": "ParkingLotName_value",
      "code": "RES-20261030-01",
      "name": "Name_value",
      "zoneType": "ZoneType_value",
      "floorNo": 1,
      "capacity": 100,
      "maxReservationSlots": 1,
      "layoutJson": "LayoutJson_value",
      "isActive": true,
      "createdAt": "2026-10-30T08:00:00Z",
      "updatedAt": "2026-10-30T08:00:00Z",
      "totalSlots": 1
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

---

### 4.3 Module 03: Customer Reservations

#### 4.3.1 `POST` /api/reservations

**Description:** Executes CreateAsync operation.

- **Action Method:** `ReservationsController.CreateAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Request Body (`CreateReservationModel`):**
  ```json
  {
    "parkingLotId": 1,
    "slotId": 1,
    "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "arrivalTime": "2026-10-30T08:00:00Z",
    "bookingMode": "BookingMode_value",
    "departureTime": "2026-10-30T08:00:00Z",
    "payDeposit": false
  }
  ```
- **Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "ParkingLotName_value",
      "parkingLotAddress": "ParkingLotAddress_value",
      "parkingLotDistrict": "ParkingLotDistrict_value",
      "zoneId": 1,
      "zoneName": "ZoneName_value",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "VehicleLabel_value",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "BookingMode_value",
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
        "provider": "Provider_value",
        "method": "Method_value",
        "purpose": "Purpose_value",
        "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.3.2 `GET` /api/reservations/mine

**Description:** Retrieves parking session history for the authenticated customer.

- **Action Method:** `ReservationsController.GetMineAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Query Parameters:**
  - `activeOnly` (bool, default: `false`)
- **Response Contract:** `ApiResult<IReadOnlyList<ReservationResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.3.3 `GET` /api/reservations/upcoming

**Description:** Retrieves the next upcoming active reservation for the authenticated customer.

- **Action Method:** `ReservationsController.GetUpcomingAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "ParkingLotName_value",
      "parkingLotAddress": "ParkingLotAddress_value",
      "parkingLotDistrict": "ParkingLotDistrict_value",
      "zoneId": 1,
      "zoneName": "ZoneName_value",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "VehicleLabel_value",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "BookingMode_value",
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
        "provider": "Provider_value",
        "method": "Method_value",
        "purpose": "Purpose_value",
        "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.3.4 `GET` /api/reservations/{id:long}

**Description:** Executes GetAsync operation.

- **Action Method:** `ReservationsController.GetAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "ParkingLotName_value",
      "parkingLotAddress": "ParkingLotAddress_value",
      "parkingLotDistrict": "ParkingLotDistrict_value",
      "zoneId": 1,
      "zoneName": "ZoneName_value",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "VehicleLabel_value",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "BookingMode_value",
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
        "provider": "Provider_value",
        "method": "Method_value",
        "purpose": "Purpose_value",
        "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.3.5 `PUT` /api/reservations/{id:long}

**Description:** Executes UpdateAsync operation.

- **Action Method:** `ReservationsController.UpdateAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`UpdateReservationModel`):**
  ```json
  {
    "arrivalTime": "2026-10-30T08:00:00Z",
    "bookingMode": "BookingMode_value",
    "departureTime": "2026-10-30T08:00:00Z",
    "slotId": 1
  }
  ```
- **Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "ParkingLotName_value",
      "parkingLotAddress": "ParkingLotAddress_value",
      "parkingLotDistrict": "ParkingLotDistrict_value",
      "zoneId": 1,
      "zoneName": "ZoneName_value",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "VehicleLabel_value",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "BookingMode_value",
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
        "provider": "Provider_value",
        "method": "Method_value",
        "purpose": "Purpose_value",
        "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.3.6 `POST` /api/reservations/{id:long}/cancel

**Description:** Executes CancelAsync operation.

- **Action Method:** `ReservationsController.CancelAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`CancelReservationModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "ParkingLotName_value",
      "parkingLotAddress": "ParkingLotAddress_value",
      "parkingLotDistrict": "ParkingLotDistrict_value",
      "zoneId": 1,
      "zoneName": "ZoneName_value",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "VehicleLabel_value",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "BookingMode_value",
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
        "provider": "Provider_value",
        "method": "Method_value",
        "purpose": "Purpose_value",
        "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.3.7 `GET` /api/reservations/{id:long}/qr

**Description:** Generates the entry QR code credential for a confirmed reservation.

- **Action Method:** `ReservationsController.GetQrAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<ReservationQrModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "reservationId": 1001,
      "code": "RES-20261030-01",
      "qrPayload": "QrPayload_value",
      "validFrom": "2026-10-30T08:00:00Z",
      "validUntil": "2026-10-30T08:00:00Z"
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.3.8 `POST` /api/reservations/{id:long}/payments

**Description:** Executes PayDepositAsync operation.

- **Action Method:** `ReservationsController.PayDepositAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`CreatePaymentModel`):**
  ```json
  {
    "provider": "Provider_value",
    "returnUrl": "https://example.com/payment/return"
  }
  ```
- **Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "Provider_value",
      "method": "Method_value",
      "purpose": "Purpose_value",
      "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

---

### 4.4 Module 04: Customer Parking Sessions

#### 4.4.1 `GET` /api/parking-sessions/mine

**Description:** Retrieves parking session history for the authenticated customer.

- **Action Method:** `ParkingSessionsController.GetMineAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Response Contract:** `ApiResult<IReadOnlyList<SessionResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.4.2 `GET` /api/parking-sessions/history

**Description:** Retrieves paginated historical parking sessions for the authenticated customer.

- **Action Method:** `ParkingSessionsController.GetHistoryAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Query Parameters:**
  - `query` (ParkingHistoryQuery, default: `none`)
- **Response Contract:** `ApiResult<IReadOnlyList<SessionResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.4.3 `GET` /api/parking-sessions/{id:long}/receipt

**Description:** Retrieves payment receipt and session billing breakdown.

- **Action Method:** `ParkingSessionsController.GetReceiptAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<ParkingReceiptModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "customerName": "CustomerName_value",
      "lotPhone": "0987654321",
      "originalFee": 50000,
      "discountAmount": 50000,
      "zeroFeeReason": "Operational requirement",
      "issuedAt": "2026-10-30T08:00:00Z",
      "payments": [
        {
          "paymentCode": "RES-20261030-01",
          "purpose": "Purpose_value",
          "method": "Method_value",
          "provider": "Provider_value",
          "amount": 50000,
          "paidAt": "2026-10-30T08:00:00Z"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.4.4 `GET` /api/parking-sessions/current

**Description:** Retrieves active in-progress parking session for the authenticated customer.

- **Action Method:** `ParkingSessionsController.GetCurrentAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Response Contract:** `ApiResult<ParkingReceiptModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "customerName": "CustomerName_value",
      "lotPhone": "0987654321",
      "originalFee": 50000,
      "discountAmount": 50000,
      "zeroFeeReason": "Operational requirement",
      "issuedAt": "2026-10-30T08:00:00Z",
      "payments": [
        {
          "paymentCode": "RES-20261030-01",
          "purpose": "Purpose_value",
          "method": "Method_value",
          "provider": "Provider_value",
          "amount": 50000,
          "paidAt": "2026-10-30T08:00:00Z"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.4.5 `GET` /api/parking-sessions/{id:long}/exit-pass

**Description:** Generates an exit pass QR code for a settled parking session.

- **Action Method:** `ParkingSessionsController.GetExitPassAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<ExitPassModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "sessionId": 1001,
      "sessionCode": "RES-20261030-01",
      "token": "Token_value",
      "prepaid": false,
      "validUntil": "2026-10-30T08:00:00Z",
      "amountDue": 50000
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.4.6 `GET` /api/parking-sessions/{id:long}/quote

**Description:** Calculates the real-time parking fee quote for an active customer session.

- **Action Method:** `ParkingSessionsController.QuoteAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<CheckoutQuoteModel>`
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
        "vehicleTypeName": "VehicleTypeName_value",
        "entryTime": "2026-10-30T08:00:00Z",
        "entryConfirmedAt": "2026-10-30T08:00:00Z",
        "entryGateId": 1,
        "zoneId": 1,
        "zoneName": "ZoneName_value",
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
      "alreadyPaid": 10000,
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.4.7 `POST` /api/parking-sessions/{id:long}/payments

**Description:** Executes PayAsync operation.

- **Action Method:** `ParkingSessionsController.PayAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`CreatePaymentModel`):**
  ```json
  {
    "provider": "Provider_value",
    "returnUrl": "https://example.com/payment/return"
  }
  ```
- **Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "Provider_value",
      "method": "Method_value",
      "purpose": "Purpose_value",
      "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

---

### 4.5 Module 05: Staff Gate Operations & Gate Requests

#### 4.5.1 `POST` /api/reservations/{reservationId:long}/check-in-request

**Description:** Handles manual gate-open assistance requests from drivers and operator approvals.

- **Action Method:** `GateRequestsController.RequestCheckInAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `reservationId` (long, required)
- **Response Contract:** `ApiResult<GateRequestModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "kind": "Kind_value",
      "status": "Active",
      "parkingLotId": 1,
      "lotName": "LotName_value",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "VehicleLabel_value",
      "vehicleColor": "VehicleColor_value",
      "vehicleTypeId": 1,
      "vehicleTypeName": "VehicleTypeName_value",
      "customerName": "CustomerName_value",
      "customerPhone": "0987654321",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "reservationStatus": "Active",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "slotCode": "RES-20261030-01",
      "sessionId": 1001,
      "sessionCode": "RES-20261030-01",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.2 `POST` /api/parking-sessions/{sessionId:long}/check-out-request

**Description:** Handles manual gate-open assistance requests from drivers and operator approvals.

- **Action Method:** `GateRequestsController.RequestCheckOutAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `sessionId` (long, required)
- **Response Contract:** `ApiResult<GateRequestModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "kind": "Kind_value",
      "status": "Active",
      "parkingLotId": 1,
      "lotName": "LotName_value",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "VehicleLabel_value",
      "vehicleColor": "VehicleColor_value",
      "vehicleTypeId": 1,
      "vehicleTypeName": "VehicleTypeName_value",
      "customerName": "CustomerName_value",
      "customerPhone": "0987654321",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "reservationStatus": "Active",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "slotCode": "RES-20261030-01",
      "sessionId": 1001,
      "sessionCode": "RES-20261030-01",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.3 `GET` /api/gate-requests/mine

**Description:** Retrieves parking session history for the authenticated customer.

- **Action Method:** `GateRequestsController.GetMineAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Response Contract:** `ApiResult<GateRequestModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "kind": "Kind_value",
      "status": "Active",
      "parkingLotId": 1,
      "lotName": "LotName_value",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "VehicleLabel_value",
      "vehicleColor": "VehicleColor_value",
      "vehicleTypeId": 1,
      "vehicleTypeName": "VehicleTypeName_value",
      "customerName": "CustomerName_value",
      "customerPhone": "0987654321",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "reservationStatus": "Active",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "slotCode": "RES-20261030-01",
      "sessionId": 1001,
      "sessionCode": "RES-20261030-01",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.5.4 `POST` /api/gate-requests/{requestId:long}/cancel

**Description:** Handles manual gate-open assistance requests from drivers and operator approvals.

- **Action Method:** `GateRequestsController.CancelAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `requestId` (long, required)
- **Response Contract:** `ApiResult<GateRequestModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "kind": "Kind_value",
      "status": "Active",
      "parkingLotId": 1,
      "lotName": "LotName_value",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "VehicleLabel_value",
      "vehicleColor": "VehicleColor_value",
      "vehicleTypeId": 1,
      "vehicleTypeName": "VehicleTypeName_value",
      "customerName": "CustomerName_value",
      "customerPhone": "0987654321",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "reservationStatus": "Active",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "slotCode": "RES-20261030-01",
      "sessionId": 1001,
      "sessionCode": "RES-20261030-01",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.5 `GET` /api/staff/lots/{lotId:int}/gate-requests

**Description:** Handles manual gate-open assistance requests from drivers and operator approvals.

- **Action Method:** `GateRequestsController.GetQueueAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `kind` (string, default: `null`)
- **Response Contract:** `ApiResult<IReadOnlyList<GateRequestModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.6 `POST` /api/staff/gate-requests/{requestId:long}/approve

**Description:** Handles manual gate-open assistance requests from drivers and operator approvals.

- **Action Method:** `GateRequestsController.ApproveAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `requestId` (long, required)
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
- **Response Contract:** `ApiResult<GateRequestDecisionModel>`
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
        "lotName": "LotName_value",
        "plateNumber": "59A-123.45",
        "vehicleLabel": "VehicleLabel_value",
        "vehicleColor": "VehicleColor_value",
        "vehicleTypeId": 1,
        "vehicleTypeName": "VehicleTypeName_value",
        "customerName": "CustomerName_value",
        "customerPhone": "0987654321",
        "reservationId": 1001,
        "reservationCode": "RES-20261030-01",
        "reservationStatus": "Active",
        "arrivalTime": "2026-10-30T08:00:00Z",
        "checkInFrom": "2026-10-30T08:00:00Z",
        "checkInUntil": "2026-10-30T08:00:00Z",
        "slotCode": "RES-20261030-01",
        "sessionId": 1001,
        "sessionCode": "RES-20261030-01",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.7 `POST` /api/staff/gate-requests/{requestId:long}/refuse

**Description:** Handles manual gate-open assistance requests from drivers and operator approvals.

- **Action Method:** `GateRequestsController.RefuseAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `requestId` (long, required)
- **Request Body (`RefuseGateRequestModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Response Contract:** `ApiResult<GateRequestModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "kind": "Kind_value",
      "status": "Active",
      "parkingLotId": 1,
      "lotName": "LotName_value",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "VehicleLabel_value",
      "vehicleColor": "VehicleColor_value",
      "vehicleTypeId": 1,
      "vehicleTypeName": "VehicleTypeName_value",
      "customerName": "CustomerName_value",
      "customerPhone": "0987654321",
      "reservationId": 1001,
      "reservationCode": "RES-20261030-01",
      "reservationStatus": "Active",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "checkInFrom": "2026-10-30T08:00:00Z",
      "checkInUntil": "2026-10-30T08:00:00Z",
      "slotCode": "RES-20261030-01",
      "sessionId": 1001,
      "sessionCode": "RES-20261030-01",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.8 `GET` /api/staff/lots

**Description:** Executes GetLotsAsync operation.

- **Action Method:** `StaffController.GetLotsAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff`
- **Response Contract:** `ApiResult<IEnumerable<AssignedParkingLotModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.5.9 `GET` /api/staff/lots/{lotId:int}/gates

**Description:** Retrieves physical entry and exit gate configurations for a parking lot.

- **Action Method:** `StaffGateController.GetGatesAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Response Contract:** `ApiResult<IReadOnlyList<GateResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.10 `GET` /api/staff/lots/{lotId:int}/zone-counters

**Description:** Retrieves live vehicle counts and capacity balances for lot zones.

- **Action Method:** `StaffGateController.GetZoneCountersAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Response Contract:** `ApiResult<IReadOnlyList<ZoneCounterModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.11 `GET` /api/staff/lots/{lotId:int}/sessions

**Description:** Retrieves active and pending parking sessions for the staff-assigned lot.

- **Action Method:** `StaffGateController.GetSessionsAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `status` (string, default: `null`)
  - `plate` (string, default: `null`)
- **Response Contract:** `ApiResult<IReadOnlyList<SessionResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.12 `GET` /api/staff/lots/{lotId:int}/sessions/history

**Description:** Retrieves paginated historical parking sessions for the authenticated customer.

- **Action Method:** `StaffGateController.GetHistoryAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `plate` (string, default: `null`)
  - `page` (int, default: `1`)
  - `pageSize` (int, default: `10`)
- **Response Contract:** `ApiResult<StaffSessionHistoryPageModel>`
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
          "vehicleTypeName": "VehicleTypeName_value",
          "entryTime": "2026-10-30T08:00:00Z",
          "entryConfirmedAt": "2026-10-30T08:00:00Z",
          "entryGateId": 1,
          "zoneId": 1,
          "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.13 `GET` /api/staff/lots/{lotId:int}/payments

**Description:** Executes GetPaymentHistoryAsync operation.

- **Action Method:** `StaffGateController.GetPaymentHistoryAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `method` (string, default: `null`)
  - `plate` (string, default: `null`)
  - `page` (int, default: `1`)
  - `pageSize` (int, default: `10`)
- **Response Contract:** `ApiResult<StaffPaymentHistoryPageModel>`
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
          "sessionCode": "RES-20261030-01",
          "plateNumber": "59A-123.45",
          "method": "Method_value",
          "channel": "Channel_value",
          "amount": 50000,
          "status": "Active",
          "collectedByName": "CollectedByName_value"
        }
      ],
      "page": 1,
      "pageSize": 1,
      "totalCount": 100
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.14 `GET` /api/staff/lots/{lotId:int}/sessions/lookup

**Description:** Executes LookupAsync operation.

- **Action Method:** `StaffGateController.LookupAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `plate` (string, default: `null`)
  - `ticket` (string, default: `null`)
  - `code` (string, default: `null`)
- **Response Contract:** `ApiResult<SessionResponseModel>`
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
      "vehicleTypeName": "VehicleTypeName_value",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.15 `GET` /api/staff/lots/{lotId:int}/reservations

**Description:** Executes GetReservationsAsync operation.

- **Action Method:** `StaffGateController.GetReservationsAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `status` (string, default: `null`)
  - `from` (DateTime?, default: `null`)
  - `to` (DateTime?, default: `null`)
- **Response Contract:** `ApiResult<IReadOnlyList<ReservationResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.16 `GET` /api/staff/lots/{lotId:int}/reservations/lookup

**Description:** Looks up advance reservations matching an incoming vehicle plate at the gate.

- **Action Method:** `StaffGateController.LookupReservationAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `code` (string, default: `null`)
  - `qr` (string, default: `null`)
  - `plate` (string, default: `null`)
- **Response Contract:** `ApiResult<GateReservationLookupModel>`
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
        "parkingLotName": "ParkingLotName_value",
        "parkingLotAddress": "ParkingLotAddress_value",
        "parkingLotDistrict": "ParkingLotDistrict_value",
        "zoneId": 1,
        "zoneName": "ZoneName_value",
        "slotId": 1,
        "slotCode": "RES-20261030-01",
        "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
        "plateNumber": "59A-123.45",
        "vehicleLabel": "VehicleLabel_value",
        "arrivalTime": "2026-10-30T08:00:00Z",
        "bookingMode": "BookingMode_value",
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
      "vehicleTypeName": "VehicleTypeName_value",
      "vehicleBrand": "VehicleBrand_value",
      "vehicleModel": "VehicleModel_value",
      "vehicleColor": "VehicleColor_value",
      "customerName": "CustomerName_value",
      "canCheckIn": true,
      "problem": "Problem_value",
      "openSessionCode": "RES-20261030-01"
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.17 `GET` /api/staff/lots/{lotId:int}/plate-lists

**Description:** Retrieves blacklisted and whitelisted plates active for the lot.

- **Action Method:** `StaffGateController.GetPlateListsAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `listType` (string, default: `null`)
- **Response Contract:** `ApiResult<IReadOnlyList<PlateListEntryModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.18 `POST` /api/staff/lots/{lotId:int}/check-in

**Description:** Executes vehicle check-in at a gate lane, validates reservations/blacklists, and triggers barrier opening.

- **Action Method:** `StaffGateController.CheckInAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
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
    "reservationQr": "ReservationQr_value",
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
- **Response Contract:** `ApiResult<CheckInResultModel>`
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
        "vehicleTypeName": "VehicleTypeName_value",
        "entryTime": "2026-10-30T08:00:00Z",
        "entryConfirmedAt": "2026-10-30T08:00:00Z",
        "entryGateId": 1,
        "zoneId": 1,
        "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.19 `GET` /api/staff/sessions/{id:long}

**Description:** Executes GetSessionAsync operation.

- **Action Method:** `StaffGateController.GetSessionAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<SessionResponseModel>`
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
      "vehicleTypeName": "VehicleTypeName_value",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.20 `POST` /api/staff/sessions/{id:long}/entry-confirm

**Description:** Confirms that the vehicle has successfully crossed the physical barrier into the lot.

- **Action Method:** `StaffGateController.ConfirmEntryAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<SessionResponseModel>`
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
      "vehicleTypeName": "VehicleTypeName_value",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.21 `POST` /api/staff/sessions/{id:long}/entry-cancel

**Description:** Cancels a pending entry check-in when a vehicle reverses away from the lane.

- **Action Method:** `StaffGateController.CancelEntryAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<SessionResponseModel>`
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
      "vehicleTypeName": "VehicleTypeName_value",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.22 `POST` /api/staff/sessions/{id:long}/quote

**Description:** Calculates parking fees, lost-ticket fines, and overtime charges at the exit lane.

- **Action Method:** `StaffGateController.QuoteAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`CheckoutQuoteRequestModel`):**
  ```json
  {
    "gateId": 1,
    "plateNumber": "59A-123.45",
    "anprConfidence": 0.98
  }
  ```
- **Response Contract:** `ApiResult<CheckoutQuoteModel>`
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
        "vehicleTypeName": "VehicleTypeName_value",
        "entryTime": "2026-10-30T08:00:00Z",
        "entryConfirmedAt": "2026-10-30T08:00:00Z",
        "entryGateId": 1,
        "zoneId": 1,
        "zoneName": "ZoneName_value",
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
      "alreadyPaid": 10000,
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.23 `POST` /api/staff/sessions/{id:long}/cancel-payment

**Description:** Cancels an in-progress payment attempt at the gate.

- **Action Method:** `StaffGateController.CancelPaymentAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<SessionResponseModel>`
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
      "vehicleTypeName": "VehicleTypeName_value",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.24 `POST` /api/staff/sessions/{id:long}/payments/cash

**Description:** Records cash fee collection by gate staff and authorizes exit barrier opening.

- **Action Method:** `StaffGateController.PayCashAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`CashPaymentModel`):**
  ```json
  {
    "amount": 50000,
    "prepaid": false
  }
  ```
- **Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "Provider_value",
      "method": "Method_value",
      "purpose": "Purpose_value",
      "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.25 `POST` /api/staff/sessions/{id:long}/payments/online

**Description:** Initiates an in-person QR payment at the gate lane via payment gateway.

- **Action Method:** `StaffGateController.PayOnlineAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`LanePaymentModel`):**
  ```json
  {
    "provider": "Provider_value",
    "prepaid": false
  }
  ```
- **Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "Provider_value",
      "method": "Method_value",
      "purpose": "Purpose_value",
      "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.26 `POST` /api/staff/sessions/{id:long}/check-out

**Description:** Executes CheckoutAsync operation.

- **Action Method:** `StaffGateController.CheckoutAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
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
- **Response Contract:** `ApiResult<CheckoutResultModel>`
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
        "vehicleTypeName": "VehicleTypeName_value",
        "entryTime": "2026-10-30T08:00:00Z",
        "entryConfirmedAt": "2026-10-30T08:00:00Z",
        "entryGateId": 1,
        "zoneId": 1,
        "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.5.27 `POST` /api/staff/sessions/{id:long}/lost-ticket

**Description:** Executes LostTicketAsync operation.

- **Action Method:** `StaffGateController.LostTicketAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`LostTicketModel`):**
  ```json
  {
    "reason": "Operational requirement",
    "conditionsMet": false
  }
  ```
- **Response Contract:** `ApiResult<SessionResponseModel>`
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
      "vehicleTypeName": "VehicleTypeName_value",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

---

### 4.6 Module 06: Manager Operations & Pricing

#### 4.6.1 `GET` /api/manager/lots

**Description:** Retrieves parking lots assigned to the authenticated Manager.

- **Action Method:** `ManagerController.GetLotsAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager`
- **Response Contract:** `ApiResult<IEnumerable<AssignedParkingLotModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.6.2 `GET` /api/manager/staff

**Description:** Retrieves staff accounts assigned to the manager lots.

- **Action Method:** `ManagerController.GetStaffAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager`
- **Query Parameters:**
  - `parkingLotId` (int?, default: `null`)
  - `isActive` (bool?, default: `null`)
  - `searchTerm` (string, default: `null`)
- **Response Contract:** `ApiResult<IEnumerable<EmployeeAccountResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.6.3 `GET` /api/manager/staff/{id}

**Description:** Executes GetStaffByIdAsync operation.

- **Action Method:** `ManagerController.GetStaffByIdAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager`
- **Path Parameters:**
  - `id` (string, required)
- **Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "FullName_value",
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
          "name": "Name_value"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.4 `POST` /api/manager/staff

**Description:** Provisions a new staff operator account assigned to the manager lots.

- **Action Method:** `ManagerController.CreateStaffAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager`
- **Request Body (`CreateStaffAccountModel`):**
  ```json
  {
    "userName": "UserName_value",
    "fullName": "FullName_value",
    "email": "user@example.com",
    "phoneNumber": "0987654321",
    "password": "P@ssword123",
    "confirmPassword": "P@ssword123",
    "parkingLotId": 1
  }
  ```
- **Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "FullName_value",
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
          "name": "Name_value"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.6.5 `PUT` /api/manager/staff/{id}

**Description:** Updates staff operator account details.

- **Action Method:** `ManagerController.UpdateStaffAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager`
- **Path Parameters:**
  - `id` (string, required)
- **Request Body (`UpdateEmployeeAccountModel`):**
  ```json
  {
    "fullName": "FullName_value",
    "email": "user@example.com",
    "phoneNumber": "0987654321"
  }
  ```
- **Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "FullName_value",
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
          "name": "Name_value"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.6 `PUT` /api/manager/staff/{id}/lot

**Description:** Assigns or reassigns a staff operator to a specific parking lot.

- **Action Method:** `ManagerController.AssignStaffLotAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager`
- **Path Parameters:**
  - `id` (string, required)
- **Request Body (`AssignStaffParkingLotModel`):**
  ```json
  {
    "parkingLotId": 1
  }
  ```
- **Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "FullName_value",
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
          "name": "Name_value"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.7 `PATCH` /api/manager/staff/{id}/status

**Description:** Activates or deactivates a staff operator account.

- **Action Method:** `ManagerController.UpdateStaffStatusAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager`
- **Path Parameters:**
  - `id` (string, required)
- **Request Body (`UpdateUserStatusModel`):**
  ```json
  {
    "isActive": true
  }
  ```
- **Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "FullName_value",
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
          "name": "Name_value"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.8 `DELETE` /api/manager/staff/{id}

**Description:** Removes a staff operator account.

- **Action Method:** `ManagerController.DeleteStaffAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager`
- **Path Parameters:**
  - `id` (string, required)
- **Response Contract:** `ApiResult<bool>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.9 `GET` /api/manager/lots/{lotId:int}/reservations

**Description:** Executes GetReservationsAsync operation.

- **Action Method:** `ManagerOperationsController.GetReservationsAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `status` (string, default: `null`)
  - `from` (DateTime?, default: `null`)
  - `to` (DateTime?, default: `null`)
- **Response Contract:** `ApiResult<IReadOnlyList<ReservationResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.10 `POST` /api/manager/reservations/{id:long}/cancel

**Description:** Executes CancelReservationAsync operation.

- **Action Method:** `ManagerOperationsController.CancelReservationAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`CancelReservationModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Response Contract:** `ApiResult<ReservationResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "status": "Active",
      "parkingLotId": 1,
      "parkingLotName": "ParkingLotName_value",
      "parkingLotAddress": "ParkingLotAddress_value",
      "parkingLotDistrict": "ParkingLotDistrict_value",
      "zoneId": 1,
      "zoneName": "ZoneName_value",
      "slotId": 1,
      "slotCode": "RES-20261030-01",
      "vehicleId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
      "plateNumber": "59A-123.45",
      "vehicleLabel": "VehicleLabel_value",
      "arrivalTime": "2026-10-30T08:00:00Z",
      "bookingMode": "BookingMode_value",
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
        "provider": "Provider_value",
        "method": "Method_value",
        "purpose": "Purpose_value",
        "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.11 `POST` /api/manager/sessions/{id:long}/abnormal-close

**Description:** Forces abnormal session closure in exceptional conditions (e.g. vehicle towed).

- **Action Method:** `ManagerOperationsController.CloseAbnormallyAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`SessionDecisionModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Response Contract:** `ApiResult<SessionResponseModel>`
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
      "vehicleTypeName": "VehicleTypeName_value",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.12 `POST` /api/manager/sessions/{id:long}/lost-ticket/approve

**Description:** Approves a lost-ticket report, allowing fee payment with standard penalty.

- **Action Method:** `ManagerOperationsController.ApproveLostTicketAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`SessionDecisionModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Response Contract:** `ApiResult<SessionResponseModel>`
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
      "vehicleTypeName": "VehicleTypeName_value",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.13 `POST` /api/manager/sessions/{id:long}/lost-ticket/reject

**Description:** Rejects a lost-ticket report, requiring customer to present full vehicle documentation.

- **Action Method:** `ManagerOperationsController.RejectLostTicketAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `id` (long, required)
- **Request Body (`SessionDecisionModel`):**
  ```json
  {
    "reason": "Operational requirement"
  }
  ```
- **Response Contract:** `ApiResult<SessionResponseModel>`
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
      "vehicleTypeName": "VehicleTypeName_value",
      "entryTime": "2026-10-30T08:00:00Z",
      "entryConfirmedAt": "2026-10-30T08:00:00Z",
      "entryGateId": 1,
      "zoneId": 1,
      "zoneName": "ZoneName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.14 `GET` /api/manager/lots/{lotId:int}/plate-lists

**Description:** Retrieves blacklist and whitelist plate entries for manager-assigned lots.

- **Action Method:** `ManagerOperationsController.GetPlateListsAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `listType` (string, default: `null`)
- **Response Contract:** `ApiResult<IReadOnlyList<PlateListEntryModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.15 `POST` /api/manager/lots/{lotId:int}/plate-lists

**Description:** Adds a license plate to the lot blacklist or whitelist.

- **Action Method:** `ManagerOperationsController.AddPlateListEntryAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Request Body (`CreatePlateListEntryModel`):**
  ```json
  {
    "listType": "ListType_value",
    "plateNumber": "59A-123.45",
    "reason": "Operational requirement",
    "expiresAt": "2026-10-30T08:00:00Z"
  }
  ```
- **Response Contract:** `ApiResult<PlateListEntryModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.16 `DELETE` /api/manager/lots/{lotId:int}/plate-lists/{id:int}

**Description:** Removes a license plate entry from the lot blacklist or whitelist.

- **Action Method:** `ManagerOperationsController.RemovePlateListEntryAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
  - `id` (int, required)
- **Response Contract:** `ApiResult<bool>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.17 `GET` /api/manager/lots/{lotId:int}/pricing-rules

**Description:** Retrieves active and historical pricing tariff rules for a parking lot.

- **Action Method:** `ManagerPricingController.GetPricingRulesAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Query Parameters:**
  - `status` (string, default: `null`)
- **Response Contract:** `ApiResult<IReadOnlyList<PricingRuleModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.18 `GET` /api/manager/lots/{lotId:int}/pricing-rules/{id:int}

**Description:** Retrieves details of a specific pricing rule.

- **Action Method:** `ManagerPricingController.GetPricingRuleByIdAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
  - `id` (int, required)
- **Response Contract:** `ApiResult<PricingRuleModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Name_value",
      "parkingLotId": 1,
      "zoneId": 1,
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "VehicleTypeName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.19 `POST` /api/manager/lots/{lotId:int}/pricing-rules

**Description:** Creates a new pricing rule for a parking lot.

- **Action Method:** `ManagerPricingController.CreatePricingRuleAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
- **Request Body (`CreatePricingRuleModel`):**
  ```json
  {
    "code": "RES-20261030-01",
    "name": "Name_value",
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
- **Response Contract:** `ApiResult<PricingRuleModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Name_value",
      "parkingLotId": 1,
      "zoneId": 1,
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "VehicleTypeName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.20 `PUT` /api/manager/lots/{lotId:int}/pricing-rules/{id:int}

**Description:** Updates an existing pricing rule, creating a new version in pricing history.

- **Action Method:** `ManagerPricingController.UpdatePricingRuleAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
  - `id` (int, required)
- **Request Body (`UpdatePricingRuleModel`):**
  ```json
  {
    "name": "Name_value",
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
- **Response Contract:** `ApiResult<PricingRuleModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Name_value",
      "parkingLotId": 1,
      "zoneId": 1,
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "VehicleTypeName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.21 `PATCH` /api/manager/lots/{lotId:int}/pricing-rules/{id:int}/status

**Description:** Activates, deactivates, or archives a pricing rule.

- **Action Method:** `ManagerPricingController.SetPricingRuleStatusAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
  - `id` (int, required)
- **Request Body (`SetPricingRuleStatusModel`):**
  ```json
  {
    "status": "Active",
    "reason": "Operational requirement"
  }
  ```
- **Response Contract:** `ApiResult<PricingRuleModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1,
      "code": "RES-20261030-01",
      "name": "Name_value",
      "parkingLotId": 1,
      "zoneId": 1,
      "vehicleTypeId": 1,
      "vehicleTypeCode": "RES-20261030-01",
      "vehicleTypeName": "VehicleTypeName_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.22 `DELETE` /api/manager/lots/{lotId:int}/pricing-rules/{id:int}

**Description:** Archives a pricing rule.

- **Action Method:** `ManagerPricingController.DeletePricingRuleAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
  - `id` (int, required)
- **Response Contract:** `ApiResult<bool>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.6.23 `POST` /api/manager/lots/{lotId:int}/pricing-rules/simulate

**Description:** Simulates fee calculations across hypothetical durations using active pricing rules.

- **Action Method:** `ManagerPricingController.SimulatePricingAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager,Admin`
- **Path Parameters:**
  - `lotId` (int, required)
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
- **Response Contract:** `ApiResult<PricingSimulationResultModel>`
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
      "ruleName": "RuleName_value"
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

---

### 4.7 Module 07: Vehicle Verification & Document Management

#### 4.7.1 `GET` /api/users/profile/rejected-vehicles

**Description:** Retrieves list of rejected vehicle submissions within the last 30 days.

- **Action Method:** `VehicleDocumentsController.GetRecentlyRejectedAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Response Contract:** `ApiResult<List<RejectedVehicleModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.7.2 `GET` /api/users/profile/vehicles/{vehicleId:guid}/documents

**Description:** Executes GetLatestAsync operation.

- **Action Method:** `VehicleDocumentsController.GetLatestAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Path Parameters:**
  - `vehicleId` (guid, required)
- **Response Contract:** `ApiResult<VehicleVerificationModel>`
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
          "fileName": "FileName_value",
          "contentType": "ContentType_value",
          "sizeBytes": 1001,
          "uploadedAt": "2026-10-30T08:00:00Z",
          "isAvailable": false
        }
      ],
      "missingDocuments": [
        "string_item"
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.7.3 `POST` /api/users/profile/vehicles/{vehicleId:guid}/documents/{documentType}/{side}

**Description:** Executes UploadAsync operation.

- **Action Method:** `VehicleDocumentsController.UploadAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Path Parameters:**
  - `vehicleId` (guid, required)
  - `documentType` (string, required)
  - `side` (string, required)
- **Response Contract:** `ApiResult<VehicleVerificationModel>`
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
          "fileName": "FileName_value",
          "contentType": "ContentType_value",
          "sizeBytes": 1001,
          "uploadedAt": "2026-10-30T08:00:00Z",
          "isAvailable": false
        }
      ],
      "missingDocuments": [
        "string_item"
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.7.4 `POST` /api/users/profile/vehicles/{vehicleId:guid}/documents/submit

**Description:** Executes SubmitAsync operation.

- **Action Method:** `VehicleDocumentsController.SubmitAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Path Parameters:**
  - `vehicleId` (guid, required)
- **Response Contract:** `ApiResult<VehicleVerificationModel>`
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
          "fileName": "FileName_value",
          "contentType": "ContentType_value",
          "sizeBytes": 1001,
          "uploadedAt": "2026-10-30T08:00:00Z",
          "isAvailable": false
        }
      ],
      "missingDocuments": [
        "string_item"
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.7.5 `GET` /api/users/profile/vehicles/{vehicleId:guid}/documents/files/{documentId:guid}

**Description:** Streams the binary image/PDF of an uploaded vehicle document.

- **Action Method:** `VehicleDocumentsController.GetFileAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Path Parameters:**
  - `vehicleId` (guid, required)
  - `documentId` (guid, required)
- **Response Contract:** `IActionResult`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.7.6 `GET` /api/vehicle-verifications

**Description:** Retrieves the operator queue of vehicle paper registrations pending review.

- **Action Method:** `VehicleVerificationsController.GetQueueAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Query Parameters:**
  - `status` (string, default: `null`)
- **Response Contract:** `ApiResult<List<VehicleVerificationReviewModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.7.7 `GET` /api/vehicle-verifications/{verificationId:guid}

**Description:** Executes GetAsync operation.

- **Action Method:** `VehicleVerificationsController.GetAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `verificationId` (guid, required)
- **Response Contract:** `ApiResult<VehicleVerificationReviewModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "customerName": "CustomerName_value",
      "customerCitizenId": "CustomerCitizenId_value",
      "customerDateOfBirth": null,
      "vehicleType": "VehicleType_value",
      "brand": "Brand_value",
      "model": "Model_value",
      "color": "Color_value",
      "ownerCitizenId": "OwnerCitizenId_value"
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.7.8 `GET` /api/vehicle-verifications/{verificationId:guid}/files/{documentId:guid}

**Description:** Streams the binary image/PDF of an uploaded vehicle document.

- **Action Method:** `VehicleVerificationsController.GetFileAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `verificationId` (guid, required)
  - `documentId` (guid, required)
- **Response Contract:** `IActionResult`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.7.9 `POST` /api/vehicle-verifications/{verificationId:guid}/review

**Description:** Records operator approval or rejection for submitted vehicle documents.

- **Action Method:** `VehicleVerificationsController.ReviewAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Staff,Manager,Admin`
- **Path Parameters:**
  - `verificationId` (guid, required)
- **Request Body (`ReviewVehicleVerificationModel`):**
  ```json
  {
    "approve": false,
    "note": "Note_value"
  }
  ```
- **Response Contract:** `ApiResult<VehicleVerificationReviewModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "customerName": "CustomerName_value",
      "customerCitizenId": "CustomerCitizenId_value",
      "customerDateOfBirth": null,
      "vehicleType": "VehicleType_value",
      "brand": "Brand_value",
      "model": "Model_value",
      "color": "Color_value",
      "ownerCitizenId": "OwnerCitizenId_value"
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

---

### 4.8 Module 08: Payments & Customer Wallet

#### 4.8.1 `GET` /api/payments/{id:long}

**Description:** Executes GetAsync operation.

- **Action Method:** `PaymentsController.GetAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "Provider_value",
      "method": "Method_value",
      "purpose": "Purpose_value",
      "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.8.2 `POST` /api/payments/{id:long}/check

**Description:** Executes CheckAsync operation.

- **Action Method:** `PaymentsController.CheckAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<PaymentCallbackResultModel>`
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
        "provider": "Provider_value",
        "method": "Method_value",
        "purpose": "Purpose_value",
        "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.8.3 `GET` /api/payments/simulation

**Description:** Executes GetSimulation operation.

- **Action Method:** `PaymentsController.GetSimulation`
- **Authentication:** None (Public)
- **Response Contract:** `ApiResult<PaymentSimulationModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.8.4 `POST` /api/payments/{id:long}/simulate-success

**Description:** Executes SimulateSuccessAsync operation.

- **Action Method:** `PaymentsController.SimulateSuccessAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<PaymentCallbackResultModel>`
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
        "provider": "Provider_value",
        "method": "Method_value",
        "purpose": "Purpose_value",
        "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.8.5 `GET` /api/payments/vnpay/return

**Description:** Handles browser redirect return from VNPay gateway.

- **Action Method:** `PaymentsController.VnPayReturnAsync`
- **Authentication:** None (Public)
- **Response Contract:** `ApiResult<PaymentCallbackResultModel>`
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
        "provider": "Provider_value",
        "method": "Method_value",
        "purpose": "Purpose_value",
        "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.8.6 `GET` /api/payments/vnpay/ipn

**Description:** Receives asynchronous server-to-server IPN webhook from VNPay.

- **Action Method:** `PaymentsController.VnPayIpnAsync`
- **Authentication:** None (Public)
- **Response Contract:** `IActionResult`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.8.7 `GET` /api/payments/momo/return

**Description:** Handles browser redirect return from MoMo gateway.

- **Action Method:** `PaymentsController.MoMoReturnAsync`
- **Authentication:** None (Public)
- **Response Contract:** `ApiResult<PaymentCallbackResultModel>`
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
        "provider": "Provider_value",
        "method": "Method_value",
        "purpose": "Purpose_value",
        "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.8.8 `POST` /api/payments/momo/ipn

**Description:** Receives asynchronous server-to-server IPN webhook from MoMo.

- **Action Method:** `PaymentsController.MoMoIpnAsync`
- **Authentication:** None (Public)
- **Response Contract:** `NoContent (204)`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.

#### 4.8.9 `GET` /api/wallet

**Description:** Executes GetAsync operation.

- **Action Method:** `WalletController.GetAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Response Contract:** `ApiResult<WalletResponseModel>`
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
          "method": "Method_value",
          "createdAt": "2026-10-30T08:00:00Z"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.8.10 `POST` /api/wallet/topup

**Description:** Executes TopUpAsync operation.

- **Action Method:** `WalletController.TopUpAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Request Body (`TopUpWalletModel`):**
  ```json
  {
    "amount": 50000,
    "provider": "Provider_value",
    "returnUrl": "https://example.com/payment/return"
  }
  ```
- **Response Contract:** `ApiResult<PaymentResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": 1001,
      "code": "RES-20261030-01",
      "provider": "Provider_value",
      "method": "Method_value",
      "purpose": "Purpose_value",
      "channel": "Channel_value",
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.8.11 `POST` /api/wallet/withdraw

**Description:** Executes WithdrawAsync operation.

- **Action Method:** `WalletController.WithdrawAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Authenticated`
- **Request Body (`WithdrawWalletModel`):**
  ```json
  {
    "amount": 50000,
    "method": "Method_value",
    "bankName": "BankName_value",
    "bankAccount": "BankAccount_value"
  }
  ```
- **Response Contract:** `ApiResult<WalletResponseModel>`
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
          "method": "Method_value",
          "createdAt": "2026-10-30T08:00:00Z"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

---

### 4.9 Module 09: Revenue Reporting

#### 4.9.1 `GET` /api/admin/reports/revenue

**Description:** Retrieves system-wide aggregated revenue metrics across all parking lots.

- **Action Method:** `AdminRevenueController.GetRevenueAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Query Parameters:**
  - `startDate` (string, default: `null`)
  - `endDate` (string, default: `null`)
  - `parkingLotId` (int?, default: `null`)
- **Response Contract:** `ApiResult<AdminRevenueReportResponse>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "startDate": "2026-10-30",
      "endDate": "2026-10-30",
      "systemTotalRevenue": 50000,
      "systemTotalSessions": 1,
      "activeLotsCount": 100,
      "byLot": [
        {
          "parkingLotId": 1,
          "parkingLotName": "ParkingLotName_value",
          "totalRevenue": 50000,
          "totalSessions": 1,
          "managerUserId": "ManagerUserId_value"
        }
      ],
      "paymentMethods": [
        {
          "method": "Method_value",
          "amount": 50000,
          "percentage": 10000
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.9.2 `GET` /api/manager/reports/revenue

**Description:** Retrieves aggregated revenue metrics and payment-method breakdown scoped to manager-assigned lots.

- **Action Method:** `ManagerRevenueController.GetRevenueAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Manager`
- **Query Parameters:**
  - `startDate` (string, default: `null`)
  - `endDate` (string, default: `null`)
  - `parkingLotId` (int?, default: `null`)
- **Response Contract:** `ApiResult<ManagerRevenueReportResponse>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "managerUserId": "ManagerUserId_value",
      "startDate": "2026-10-30",
      "endDate": "2026-10-30",
      "totalRevenue": 50000,
      "totalSessions": 1,
      "byLot": [
        {
          "parkingLotId": 1,
          "parkingLotName": "ParkingLotName_value",
          "totalRevenue": 50000,
          "totalSessions": 1,
          "managerUserId": "ManagerUserId_value"
        }
      ],
      "paymentMethods": [
        {
          "method": "Method_value",
          "amount": 50000,
          "percentage": 10000
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

---

### 4.10 Module 10: Admin User & System Management

#### 4.10.1 `POST` /api/admin/managers

**Description:** Creates a new Manager account and assigns managed parking lots.

- **Action Method:** `AdminController.CreateManagerAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Request Body (`CreateEmployeeAccountModel`):**
  ```json
  {
    "userName": "UserName_value",
    "fullName": "FullName_value",
    "email": "user@example.com",
    "password": "P@ssword123",
    "phoneNumber": "0987654321",
    "assignedParkingLot": "AssignedParkingLot_value",
    "parkingLotIds": [
      1,
      2
    ]
  }
  ```
- **Response Contract:** `ApiResult<CreateUserResponseModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.10.2 `GET` /api/admin/managers

**Description:** Retrieves all Manager accounts with their lot assignments.

- **Action Method:** `AdminController.GetManagersAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Query Parameters:**
  - `isActive` (bool?, default: `null`)
  - `searchTerm` (string, default: `null`)
- **Response Contract:** `ApiResult<IEnumerable<EmployeeAccountResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.10.3 `GET` /api/admin/managers/{id}

**Description:** Retrieves details of a specific Manager account.

- **Action Method:** `AdminController.GetManagerAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (string, required)
- **Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "FullName_value",
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
          "name": "Name_value"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.10.4 `PUT` /api/admin/managers/{id}

**Description:** Updates Manager account information.

- **Action Method:** `AdminController.UpdateManagerAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (string, required)
- **Request Body (`UpdateEmployeeAccountModel`):**
  ```json
  {
    "fullName": "FullName_value",
    "email": "user@example.com",
    "phoneNumber": "0987654321"
  }
  ```
- **Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "FullName_value",
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
          "name": "Name_value"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.10.5 `PATCH` /api/admin/managers/{id}/status

**Description:** Activates or deactivates a Manager account.

- **Action Method:** `AdminController.UpdateManagerStatusAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (string, required)
- **Request Body (`UpdateUserStatusModel`):**
  ```json
  {
    "isActive": true
  }
  ```
- **Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "FullName_value",
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
          "name": "Name_value"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.10.6 `PUT` /api/admin/managers/{id}/lots

**Description:** Updates parking lot assignments for a Manager.

- **Action Method:** `AdminController.AssignManagerLotsAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (string, required)
- **Request Body (`AssignParkingLotsModel`):**
  ```json
  {
    "parkingLotIds": [
      1,
      2
    ]
  }
  ```
- **Response Contract:** `ApiResult<EmployeeAccountResponseModel>`
- **Success Response Example (`200 OK`):**
  ```json
  {
    "succeeded": true,
    "result": {
      "id": "Id_value",
      "userName": "UserName_value",
      "fullName": "FullName_value",
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
          "name": "Name_value"
        }
      ]
    },
    "errors": []
  }
  ```
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.10.7 `GET` /api/admin/users

**Description:** Executes GetInternalUsersAsync operation.

- **Action Method:** `AdminController.GetInternalUsersAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Query Parameters:**
  - `role` (string, default: `null`)
  - `isActive` (bool?, default: `null`)
  - `searchTerm` (string, default: `null`)
- **Response Contract:** `ApiResult<IEnumerable<UserResponseModel>>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.10.8 `PATCH` /api/admin/users/{id}/status

**Description:** Activates or deactivates an internal user account.

- **Action Method:** `AdminController.UpdateStatusAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Path Parameters:**
  - `id` (string, required)
- **Request Body (`UpdateUserStatusModel`):**
  ```json
  {
    "isActive": true
  }
  ```
- **Response Contract:** `ApiResult<bool>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

---

### 4.11 Module 11: Admin Simulation & Testing

#### 4.11.1 `GET` /api/admin/simulation

**Description:** Retrieves current sandbox simulation mode settings and simulated system clock.

- **Action Method:** `AdminSimulationController.GetAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
- **Response Contract:** `ApiResult<ParkingSimulationResponseModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.11.2 `PUT` /api/admin/simulation

**Description:** Updates simulation mode parameters (e.g. speed factor, artificial time offset).

- **Action Method:** `AdminSimulationController.UpdateAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: Admin`
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
- **Response Contract:** `ApiResult<ParkingSimulationResponseModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

---

### 4.12 Module 12: Notifications

#### 4.12.1 `GET` /api/notifications

**Description:** Retrieves parking session history for the authenticated customer.

- **Action Method:** `NotificationsController.GetMineAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Query Parameters:**
  - `unreadOnly` (bool, default: `false`)
  - `take` (int, default: `30`)
- **Response Contract:** `ApiResult<NotificationListModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.12.2 `GET` /api/notifications/unread-count

**Description:** Retrieves count of unread notifications for the authenticated user.

- **Action Method:** `NotificationsController.GetUnreadCountAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Response Contract:** `ApiResult<NotificationListModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.12.3 `POST` /api/notifications/{id:long}/read

**Description:** Marks a specific notification as read.

- **Action Method:** `NotificationsController.MarkReadAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<NotificationModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

#### 4.12.4 `POST` /api/notifications/read-all

**Description:** Marks all notifications for the authenticated user as read.

- **Action Method:** `NotificationsController.MarkAllReadAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Response Contract:** `ApiResult<NotificationModel>`
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
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.

#### 4.12.5 `DELETE` /api/notifications/{id:long}

**Description:** Deletes an in-app notification.

- **Action Method:** `NotificationsController.DeleteAsync`
- **Authentication:** Bearer JWT Required
- **Authorization / Roles:** `Role: User`
- **Path Parameters:**
  - `id` (long, required)
- **Response Contract:** `ApiResult<object>`
- **Relevant Status Codes:**
  - `200 OK` / `201 Created`: Request succeeded.
  - `400 Bad Request`: Validation failure or business constraint violation.
  - `401 Unauthorized`: Missing or invalid Bearer token.
  - `403 Forbidden`: Insufficient role permissions or lot assignment scope.
  - `404 Not Found`: Target entity not found.

---

## 5. SignalR Real-Time Communication

The Parking Management System provides three real-time SignalR hubs mapped in `src/PMS.API/Program.cs`.

### 5.1 LotHub (`/api/hubs/lots`)
- **Endpoint:** `/api/hubs/lots`
- **Authentication:** Public / Anonymous
- **Groups:** `lot-{lotId}`
- **Client-to-Server Methods:**
  - `JoinLot(int lotId)`: Joins group for real-time notifications for specified parking lot.
  - `LeaveLot(int lotId)`: Leaves group for specified parking lot.
- **Server-to-Client Events:**

| Event Name | Payload Schema | Description |
| :--- | :--- | :--- |
| `SlotStatusChanged` | `{ "lotId": int, "slotId": int, "status": string }` | Broadcast when a bay changes status (`Available`, `Occupied`, `Reserved`, `Maintenance`). |
| `LotStatusChanged` | `{ "lotId": int, "status": string }` | Broadcast when lot operational status updates (`Open`, `Full`, `Closed`). |
| `LayoutChanged` | `{ "lotId": int }` | Alerts clients that lot layout geometry was modified. |
| `ZoneCounterChanged`| `{ "lotId": int, "zoneId": int, "availableSpaces": int, "occupiedSpaces": int, "totalCapacity": int }` | Broadcast when vehicle movement updates zone occupancy counts. |

### 5.2 ParkingOpsHub (`/api/hubs/parking-ops`)
- **Endpoint:** `/api/hubs/parking-ops`
- **Authentication:** Bearer JWT required (passed via query parameter `?access_token=<token>`).
- **Groups:** `ops-lot-{lotId}` (Staff and Managers are verified against lot assignment scope before joining).
- **Client-to-Server Methods:**
  - `JoinLot(int lotId)`: Staff/Manager joins operational event stream for assigned lot.
  - `LeaveLot(int lotId)`: Leaves operational event stream.
- **Server-to-Client Events:**

| Event Name | Target Audience | Payload Schema | Description |
| :--- | :--- | :--- | :--- |
| `SessionChanged` | Ops Group (`ops-lot-{lotId}`) | `{ "lotId": int, "sessionId": long, "status": string, "plateNumber": string }` | Live vehicle check-in, checkout, or lost-ticket updates. |
| `PaymentChanged` | Customer & Ops Group | `{ "paymentId": long, "status": string, "amount": decimal, "purpose": string }` | Real-time payment settlement notifications. |
| `ZoneCounterChanged` | Ops Group | Zone occupancy counter payload | Live count updates for gate operators. |
| `OpsAlert` | Ops Group | `{ "lotId": int, "code": string, "message": string, "sessionId": long? }` | Operational gate alarms (e.g. barrier timeout, plate mismatch). |
| `GateRequestChanged`| Customer & Ops Group | Gate request status payload | Remote barrier opening requests and decisions. |
| `ReservationChanged`| Customer (`Clients.User`) | `{ "reservationId": long, "lotId": int, "status": string, "slotId": int? }` | Customer reservation confirmation, hold, or cancellation alerts. |
| `ReservationReminder`| Customer (`Clients.User`)| `{ "reservationId": long, "kind": "Arrival" \| "HoldEnding", "at": string }` | Upcoming arrival and grace period expiration warnings. |
| `NotificationCreated`| Customer (`Clients.User`)| `{ "notification": NotificationModel, "unreadCount": int }` | In-app user notification push with updated badge counter. |

### 5.3 VehicleVerificationHub (`/api/hubs/vehicle-verifications`)
- **Endpoint:** `/api/hubs/vehicle-verifications`
- **Authentication:** Bearer JWT required (passed via query parameter `?access_token=<token>`).
- **Groups:** `vehicle-verification-reviewers` (Staff, Managers, and Admins automatically join upon connection).
- **Server-to-Client Events:**

| Event Name | Target Audience | Payload Schema | Description |
| :--- | :--- | :--- | :--- |
| `VerificationChanged` | Document Uploader & Reviewers Group | `{ "verificationId": guid, "vehicleId": guid, "status": string }` | Broadcast when vehicle papers are uploaded, submitted, approved, or rejected. |

---

## 6. Validation and Error Handling

### 6.1 Model Validation Pipeline
Request payloads are validated automatically using **FluentValidation** (`src/PMS.Application/Models/Validators/*.cs`). `ValidateModelAttribute` (`src/PMS.API/Filters/ValidateModelAttribute.cs`) intercepts invalid requests prior to controller execution and returns **HTTP 400 Bad Request** using the standard envelope:
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

### 6.2 HTTP Status Code Error Mapping
Global exceptions are captured by `ExceptionHandlingMiddleware` (`src/PMS.API/Middlewares/ExceptionHandlingMiddleware.cs`):

| Exception Class | HTTP Status Code | Description |
| :--- | :---: | :--- |
| `NotFoundException` / `ResourceNotFoundException` | **HTTP 404 Not Found** | Target resource does not exist (e.g. unknown parking lot ID or session ID). |
| `BadRequestException` | **HTTP 400 Bad Request** | Request violates business rules (e.g. vehicle already checked in). |
| `ForbiddenException` | **HTTP 403 Forbidden** | Caller lacks role permissions or is not assigned to the managed lot. |
| `UnprocessableRequestException` | **HTTP 422 Unprocessable** | Request syntactically valid but semantically unprocessable (e.g. missing mandatory paper sides). |
| `ConflictException` | **HTTP 409 Conflict** | State collision (e.g. slot already booked by concurrent reservation). |
| Unhandled Exception | **HTTP 500 Internal Server Error** | Unexpected crash. In production environments, error details are sanitized. |

### 6.3 Rate Limiting Policies
Enforced by `RateLimitingMiddleware.cs` via ASP.NET Core rate limiting:
1. **`PasswordResetRateLimit`:** Applied to `POST /api/Users/forgot-password`. Allows a maximum of **5 requests per 15 minutes** per IP address. Exceeding requests return **HTTP 429 Too Many Requests**.
2. **`PublicLayoutRateLimit`:** Applied to `GET /api/public/lots/{lotId}/layout`. Allows a maximum of **120 requests per minute** per IP address to protect visual layout geometry from scraping. Exceeding requests return **HTTP 429 Too Many Requests**.

