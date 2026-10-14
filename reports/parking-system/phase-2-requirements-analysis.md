# Phase 2.1: Validated and Corrected Parking System Requirements Analysis Report

## 1. Executive Summary

This report delivers a rigorous, validated requirements and architectural analysis of the **Parking System** project. It updates and corrects the preliminary Phase 2 analysis by establishing auditable traceability across:
1. The official Software Requirements Specification (**SRS v1.3**, October 11, 2026, by `@f`).
2. The official **Implementation Plan** (October 11, 2026, by `@FrankyMilky`).
3. The empirical source-code baseline documented in the Phase 1 full-stack reconnaissance report (`phase-1-fullstack-codebase-scan.md`).

### Key Validated Findings

- **Verified Functional Requirement Inventory**: Exactly **61 active Functional Requirements (FRs)** are specified across 10 functional modules. The apparent numbering gap between `FR-OPS-05` and `FR-OPS-07` is conclusively verified: `FR-OPS-06` (*Quản lý ca / Shift management*) was intentionally removed in v1.3 (along with the former `UC-06`, `Shift` entities, and cash-handover rules) without renumbering subsequent IDs to preserve historical requirement stability.
- **Verified Use Cases and NFR Counts**: Exactly **10 Use Cases (UC-01 to UC-10)** and **18 Non-Functional Requirements (NFR-01 to NFR-14, plus NFR-3D-01 to NFR-3D-04)** are defined in the SRS and accounted for in the Implementation Plan.
- **Reconciliation of the `dbo.Users.Id` Question**: The pending question regarding the identifier of `dbo.Users` is classified as **Partially resolved in source code; runtime/deployed-schema confirmation remains necessary**. In the codebase, ASP.NET Identity's `ApplicationUser` maps to table `"Users"`, with primary key column `"UserId"` typed as `nvarchar(450)`. However, the live deployed SQL Server schema cannot be proven from source files alone without connecting to a deployed database.
- **Implementation Presence vs. Requirement Compliance**: Source code presence does not equate to requirement satisfaction. While user registration, login, profiles, and vehicle CRUD exist in `UsersController.cs` and `UserScreen.jsx`, they lack key SRS v1.3 requirements (such as phone/OTP login and internal 2FA). Core parking operations (lots, zones, slots, manual check-in/out, ANPR, 3D WebGL rendering, reservations, online payments, and reports) have **no API implementation** in the codebase.
- **Plan Phasing vs. Codebase Inversion**: The Implementation Plan schedules Customer registration and Vehicle management in **Phase 2** (`B2.7`, `F2.3`), whereas the team implemented them first. Meanwhile, the foundational Phase 1 operational core (Lots, Zones, Slots, `ZoneCounter`, Pricing Engine, Manual POS sessions) remains completely unbuilt.
- **Scope Boundary Discipline**: SRS Section 2.5 explicitly excludes Subscriptions/passes, E-invoices, Wallets/Loyalty points, Tenant multi-tenancy, Partner revenue sharing, and Shift management. The 63 scaffolded tables in `PmsDbContext.cs` include legacy entities related to these excluded areas. However, **no files or entities should be deleted unilaterally** without a formal team dependency review and migration impact assessment.

---

## 2. Source Documents and Validation Method

### 2.1 Primary Document Artifacts
The analysis was performed directly on the original DOCX source files archived in the reporting repository:

| Document Identity | Physical File Name | File Size | SHA-256 Hash | Author & Date |
| :--- | :--- | :--- | :--- | :--- |
| **SRS Document** | `SRS v1.3 – Hệ thống Quản lý Nhiều Bãi Đỗ Xe.docx` | 137,445 bytes | `F72686FB2CED8ED23070850F6F2FFF1661E33F054E9869928685C9F87BA5834A` | `@f` · Oct 11, 2026 |
| **Implementation Plan** | `Implementation Plan - Hệ thống Quản lý Nhiều Bãi Đỗ Xe.docx` | 118,372 bytes | `333630CB21CD5D69EFD01A74D099A356A045B58B0632A69D5244CBA96AADCFBA` | `@FrankyMilky` · Oct 11, 2026 |

*Location*: Both files are preserved byte-for-byte in `reports/parking-system/source-documents/`.

### 2.2 Validation Methodology
1. **Direct XML Extraction**: Both documents were unpacked and parsed at the XML node level (`word/document.xml`) to extract every table cell, heading, text element, footnote, and embedded image reference without translation loss or summary distortion.
2. **Embedded Diagram Verification**:
   - `SRS v1.3`: Extracted and verified `word/media/image1.png` (finite state machine diagram for Slot, Session, and Reservation).
   - `Implementation Plan`: Extracted and verified `word/media/image1.png` (3-phase delivery roadmap diagram).
3. **Evidence Classification Standards**:
   - `[SOURCE FACT]`: Explicitly stated in the SRS or Implementation Plan text.
   - `[CODE OBSERVATION]`: Directly verified in repository source files.
   - `[ANALYTICAL FINDING]`: Deduction based on comparing sources and code.
   - `[UNVERIFIED / UNKNOWN]`: Requires runtime execution or stakeholder decision.
4. **Git Safety Compliance**: Zero Git commands were run against the team repositories (`C:\DEV\OJT-FSA-Team3` and subdirectories). All Git interactions are restricted exclusively to `C:\DEV\OJT-report`.

---

## 3. Verified FR Inventory and Traceability Matrix

### 3.1 Auditable Functional Requirement Inventory
The SRS contains **61 active Functional Requirements** across 10 modules:
- `FR-AUTH` (8 requirements): `FR-AUTH-01` through `FR-AUTH-08`.
- `FR-LOT` (6 requirements): `FR-LOT-01` through `FR-LOT-06`.
- `FR-3D` (8 requirements): `FR-3D-01` through `FR-3D-08`.
- `FR-OPS` (8 requirements): `FR-OPS-01` through `FR-OPS-05`, and `FR-OPS-07` through `FR-OPS-09` (*Note: `FR-OPS-06` was removed in v1.3*).
- `FR-PRC` (5 requirements): `FR-PRC-01` through `FR-PRC-05`.
- `FR-RSV` (6 requirements): `FR-RSV-01` through `FR-RSV-06`.
- `FR-PAY` (7 requirements): `FR-PAY-01` through `FR-PAY-07`.
- `FR-CUS` (3 requirements): `FR-CUS-01` through `FR-CUS-03`.
- `FR-RPT` (4 requirements): `FR-RPT-01` through `FR-RPT-04`.
- `FR-SYS` (6 requirements): `FR-SYS-01` through `FR-SYS-06`.
- **Total Active FRs**: $8 + 6 + 8 + 8 + 5 + 6 + 7 + 3 + 4 + 6 = \mathbf{61}$.

### 3.2 The `FR-OPS-06` Gap Investigation
- **Inquiry**: Does `FR-OPS-06` exist, or is it a documentation omission?
- **Finding**: `FR-OPS-06` does **not** exist as an active requirement in SRS v1.3.
- **Evidence**:
  1. SRS Section 2.4 (Change Log from v1.2): *"Bỏ FR-OPS-06, UC-06, 'phân lịch ca' trong FR-SYS-01, thực thể Shift, 'chênh lệch tiền ca' trong FR-RPT-03 và dòng ca trong ma trận."*
  2. SRS Section 3 (Preamble): *"Có 61 yêu cầu chức năng chia theo 10 module... Mã FR giữ ổn định so với v1.2; FR-OPS-06 (quản lý ca) bị bỏ và không đánh số lại."*
- **Conclusion**: The gap between `FR-OPS-05` and `FR-OPS-07` is deliberate. The total denominator of 61 active requirements is verified.

### 3.3 Complete Traceability Matrix
The table below maps every active Functional Requirement to its source section, stated priority, corresponding Implementation Plan task, planned phase, plan coverage assessment, and source-code status.

*Coverage Assessment Labels*:
- `COVERED`: Explicitly addressed in the plan with tasks and effort.
- `PARTIALLY COVERED`: Addressed in part; specific sub-rules or edge-cases omitted.
- `PLANNING SEQUENCING DIFFERENCE`: Scheduled in a phase different from current codebase progress.
- `NOT MENTIONED IN PLAN`: Absent from the plan work-breakdown structure.

| Req ID | Requirement Summary | SRS Ref | Priority | Plan Task(s) | Plan Phase | Plan Coverage | Current Codebase Status |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **FR-AUTH-01** | User registration/login (Phone/Email + OTP/Pwd) | 3.1 | Must | `B2.7`, `F2.3` | Phase 2 | `PLANNING SEQUENCING DIFFERENCE` | Partially aligned in source (Pwd auth built; OTP absent) |
| **FR-AUTH-02** | Internal staff login + 2FA | 3.1 | Must | `B1.3`, `F1.2` | Phase 1 | `COVERED` | Not found in inspected source (FE mocks 2FA) |
| **FR-AUTH-03** | RBAC across 4 roles with lot-level scoping | 3.1 | Must | `B1.3` | Phase 1 | `COVERED` | Not found in inspected source (`UserParkingLot` unbuilt) |
| **FR-AUTH-04** | Admin management of internal accounts | 3.1 | Must | `B1.3`, `F1.3` | Phase 1 | `COVERED` | Not found in inspected source (only Admin get profile exists) |
| **FR-AUTH-05** | Audit log for administrative & operational actions | 3.1 | Must | `B1.3`, `F1.3` | Phase 1 | `COVERED` | Not found in inspected source (`AuditLog` entity unused) |
| **FR-AUTH-06** | Guest public access without login | 3.1 | Must | `B1.5`, `F1.11`| Phase 1 | `COVERED` | Partially aligned in source (FE has guest view; lacks data) |
| **FR-AUTH-07** | Manager creates/manages Staff in lot | 3.1 | Must | `B1.3`, `F1.3` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-AUTH-08** | Internal account activation via link & forced 2FA | 3.1 | Must | `B1.3`, `F1.2` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-LOT-01** | CRUD parking lots (Admin) | 3.2 | Must | `B1.4`, `F1.4` | Phase 1 | `COVERED` | Present in entity only; API not found |
| **FR-LOT-02** | Configure Zone/Floor/Row and Slots by vehicle type| 3.2 | Must | `B1.4`, `F1.4` | Phase 1 | `COVERED` | Present in entities only; API not found |
| **FR-LOT-03** | 4 slot states + ZoneCounter tracking for transient| 3.2 | Must | `B1.4` | Phase 1 | `COVERED` | Not found in inspected source (`ZoneCounter` absent) |
| **FR-LOT-04** | Lot status toggling & maintenance notice | 3.2 | Should | `B1.4`, `F1.4` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-LOT-05** | Public lot listing with live occupancy & prices | 3.2 | Must | `B1.5`, `F1.11`| Phase 1 | `COVERED` | Not found in inspected source |
| **FR-LOT-06** | Configure reservation slot quotas per zone | 3.2 | Should | `B1.4`, `F1.4` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-3D-01** | 3D lot rendering from published layout | 3.3 | Must | `B1.5`, `F1.9` | Phase 1 | `COVERED` | Not found in inspected source (Three.js/GLB absent) |
| **FR-3D-02** | Real-time color updates via WebSocket | 3.3 | Must | `B1.5`, `F1.9` | Phase 1 | `COVERED` | Not found in inspected source (SignalR absent) |
| **FR-3D-03** | 3D interactions (rotate, pan, zoom, click slot) | 3.3 | Must | `F1.9` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-3D-04** | Select empty slot on 3D to initiate booking | 3.3 | Must | `F2.4` | Phase 2 | `COVERED` | Partially aligned in source (FE SVG click mocked) |
| **FR-3D-05** | 3D Layout Editor (Draft/Publish lifecycle) | 3.3 | Should | `B3.1`, `F3.1` | Phase 3 | `COVERED` | Not found in inspected source |
| **FR-3D-06** | 2D canvas/SVG fallback + Level of Detail | 3.3 | Should | `F1.10` | Phase 1 | `COVERED` | Partially aligned in source (FE uses 2D SVG grid) |
| **FR-3D-07** | Parked car marker & pathfinding guidance | 3.3 | Could | `B3.4`, `F3.3` | Phase 3 | `COVERED` | Not found in inspected source |
| **FR-3D-08** | Public 3D data privacy (no license plates) | 3.3 | Must | `B1.5` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-OPS-01** | Check-in flow (ANPR / QR ticket / Barrier open) | 3.4 | Must | `B1.7`, `B2.2`, `F1.7`| Phase 1 / 2 | `COVERED` | Not found in inspected source |
| **FR-OPS-02** | Check-out flow (ANPR / Exit Window / Payment) | 3.4 | Must | `B1.7`, `B2.2`, `F1.7`| Phase 1 / 2 | `COVERED` | Not found in inspected source |
| **FR-OPS-03** | Manual check-in/out with mandatory reason | 3.4 | Must | `B1.7`, `F1.7` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-OPS-04** | Plate discrepancy verification at gate | 3.4 | Must | `B1.7`, `B2.2`, `F2.1`| Phase 1 / 2 | `COVERED` | Not found in inspected source |
| **FR-OPS-05** | Lost ticket handling with Maker-Checker approval | 3.4 | Must | `B1.8`, `F1.8` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-OPS-07** | Blacklist/Whitelist plate management & alerts | 3.4 | Should | `B2.2`, `F2.7` | Phase 2 | `COVERED` | Present in entity only; API not found |
| **FR-OPS-08** | Incident recording with photo attachment | 3.4 | Should | `B2.9`, `F2.7` | Phase 2 | `COVERED` | Present in entity only; API not found |
| **FR-OPS-09** | Edge Service offline gate autonomy & sync | 3.4 | Must | `B2.1`, `F2.1` | Phase 2 | `COVERED` | Not found in inspected source |
| **FR-PRC-01** | Configure pricing rules by lot/zone/vehicle | 3.5 | Must | `B1.6`, `F1.5` | Phase 1 | `COVERED` | Present in entity only; calculation engine absent |
| **FR-PRC-02** | Fee models (blocks, hourly, day/night, ceiling) | 3.5 | Must | `B1.6` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-PRC-03** | Grace period, overnight fee, Exit Window overdue | 3.5 | Must | `B1.6`, `B2.5` | Phase 1 / 2 | `COVERED` | Not found in inspected source |
| **FR-PRC-04** | Historical pricing rule versioning | 3.5 | Should | `B1.6`, `F1.5` | Phase 1 | `COVERED` | Present in entity only |
| **FR-PRC-05** | Dynamic pricing (peak/holiday) & vouchers | 3.5 | Could | `B3.3`, `F3.4` | Phase 3 | `COVERED` | Not found in inspected source |
| **FR-RSV-01** | Search lots, compare prices, 3D preview | 3.6 | Must | `F2.4` | Phase 2 | `COVERED` | Partially aligned in source (FE has mock search) |
| **FR-RSV-02** | Slot booking with hold duration & deposit | 3.6 | Must | `B2.3`, `F2.4` | Phase 2 | `COVERED` | Not found in inspected source (deposit unbuilt) |
| **FR-RSV-03** | Change/Cancel reservation & refund policy | 3.6 | Should | `B2.3`, `F2.4` | Phase 2 | `COVERED` | Not found in inspected source |
| **FR-RSV-04** | Generate QR reservation check-in code | 3.6 | Should | `B2.3`, `F2.2` | Phase 2 | `COVERED` | Partially aligned in source (FE displays mock QR) |
| **FR-RSV-05** | Auto-expire hold & charge no-show fee | 3.6 | Should | `B2.3` | Phase 2 | `COVERED` | Not found in inspected source (worker absent) |
| **FR-RSV-06** | Configure reservation policy (hold, deposit) | 3.6 | Must | `B2.3`, `F2.7` | Phase 2 | `COVERED` | Not found in inspected source |
| **FR-PAY-01** | Cash collection at POS | 3.7 | Must | `B1.8`, `F1.7` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-PAY-02** | Online payment (VNPay, MoMo, VietQR) | 3.7 | Must | `B2.4`, `F2.5` | Phase 2 | `COVERED` | Partially aligned in source (FE mock modal only) |
| **FR-PAY-03** | Dynamic QR payment at exit for Guest | 3.7 | Must | `B2.4`, `F2.2` | Phase 2 | `COVERED` | Not found in inspected source |
| **FR-PAY-04** | Refund/adjustment via Maker-Checker | 3.7 | Must | `B1.8`, `F1.8` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-PAY-05** | Daily reconciliation with Payment Gateway | 3.7 | Must | `B2.6`, `F2.7` | Phase 2 | `COVERED` | Not found in inspected source |
| **FR-PAY-06** | Issue digital receipt via Push/SMS/Email | 3.7 | Should | `B2.8`, `F2.5` | Phase 2 | `COVERED` | Not found in inspected source |
| **FR-PAY-07** | Exit Window countdown & overtime fees | 3.7 | Must | `B2.5`, `F2.5` | Phase 2 | `COVERED` | Partially aligned in source (FE has simulated timer) |
| **FR-CUS-01** | Manage customer profile and vehicles | 3.8 | Must | `B2.7`, `F2.3` | Phase 2 | `PLANNING SEQUENCING DIFFERENCE` | Static evidence suggests alignment (Built in BE/FE) |
| **FR-CUS-02** | View parking session history & entry photos | 3.8 | Must | `B2.7`, `F2.6` | Phase 2 | `COVERED` | Not found in inspected source |
| **FR-CUS-03** | Track live parked vehicle & pre-pay | 3.8 | Should | `B2.7`, `F2.5` | Phase 2 | `COVERED` | Partially aligned in source (FE simulated card) |
| **FR-RPT-01** | Multi-lot overview dashboard | 3.9 | Must | `B1.10`, `F1.12`| Phase 1 | `COVERED` | Not found in inspected source |
| **FR-RPT-02** | Revenue reports with multiple filters | 3.9 | Must | `B1.10`, `F1.12`| Phase 1 | `COVERED` | Not found in inspected source |
| **FR-RPT-03** | Exception reports (overtime, manual barrier) | 3.9 | Should | `B2.9`, `F2.8` | Phase 2 | `COVERED` | Not found in inspected source |
| **FR-RPT-04** | Export reports to Excel/PDF | 3.9 | Should | `B2.9`, `F2.8` | Phase 2 | `COVERED` | Not found in inspected source |
| **FR-SYS-01** | Employee management & gate assignment | 3.10 | Must | `B1.9`, `F1.3` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-SYS-02** | Global parameters (currency, timezone, exit) | 3.10 | Must | `B1.9`, `F1.6` | Phase 1 | `COVERED` | Not found in inspected source |
| **FR-SYS-03** | Device catalog & Online/Offline heartbeat | 3.10 | Must | `B1.9`, `B2.9`, `F1.6`| Phase 1 / 2 | `COVERED` | Not found in inspected source |
| **FR-SYS-04** | Payment gateway keys & webhook configuration | 3.10 | Must | `B2.4`, `F2.7` | Phase 2 | `COVERED` | Not found in inspected source |
| **FR-SYS-05** | Backup and Disaster Recovery | 3.10 | Must | `B2.10` | Phase 2 | `COVERED` | Not found in inspected source |
| **FR-SYS-06** | System notifications (Push/SMS/Email) | 3.10 | Should | `B2.8`, `F2.9` | Phase 2 | `COVERED` | Partially aligned in source (MailKit built; Push/SMS absent) |

*Summary*: Out of 61 active Functional Requirements, **59 are COVERED** by planned tasks in their respective phases, and **2 (`FR-AUTH-01`, `FR-CUS-01`) have a PLANNING SEQUENCING DIFFERENCE** because they are scheduled for Phase 2 in the plan but already have implementation presence in the codebase. Zero requirements are completely omitted from the plan.

---

## 4. Verified Use Case Inventory and Traceability

### 4.1 Use Case Count and Enumeration
SRS Section 5 specifies **exactly 10 Use Cases (UC-01 to UC-10)**. Former `UC-06` (*Kết ca / Shift checkout*) from v1.2 was dropped, and use cases were renumbered.

| UC ID | Use Case Name | Primary Actor | Main Execution Flow Summary | Plan Tasks | Plan Phase |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **UC-01** | Xe vào bãi (Vehicle Entry) | Staff, Guest, User, E3 | Sensor $\rightarrow$ ANPR reads plate $\rightarrow$ Blacklist & reservation check $\rightarrow$ Session created $\rightarrow$ Gate opens | `B1.7`, `B2.2`, `F1.7` | Phase 1 / 2 |
| **UC-02** | Xe ra bãi (Vehicle Exit) | Staff, Guest, User, E3 | Plate read $\rightarrow$ Photo match $\rightarrow$ Exit Window check $\rightarrow$ Payment verified $\rightarrow$ Gate opens $\rightarrow$ Session closed | `B1.7`, `B2.2`, `F1.7` | Phase 1 / 2 |
| **UC-03** | Xem sơ đồ bãi 3D (View 3D Layout) | Guest, User | Select lot $\rightarrow$ Load published GLB/JSON $\rightarrow$ Dựng 3D $\rightarrow$ WebSocket delta updates $\rightarrow$ Click slot | `B1.5`, `F1.9` | Phase 1 |
| **UC-04** | Đặt chỗ trước qua 3D (3D Pre-booking)| User, E1 | Find lot $\rightarrow$ Select slot/time $\rightarrow$ Slot status `Giữ chỗ` $\rightarrow$ Pay deposit $\rightarrow$ Issue QR check-in | `B2.3`, `F2.4` | Phase 2 |
| **UC-05** | Xử lý mất vé (Lost Ticket Handling) | Staff, Manager | Staff identifies vehicle/driver $\rightarrow$ Computes penalty $\rightarrow$ Manager approves $\rightarrow$ Collect fee $\rightarrow$ Open gate | `B1.8`, `F1.8` | Phase 1 |
| **UC-06** | Thiết lập bảng giá (Pricing Setup) | Manager | Select lot/zone $\rightarrow$ Configure fee rules $\rightarrow$ Set effective date $\rightarrow$ Save (old price archived) | `B1.6`, `F1.5` | Phase 1 |
| **UC-07** | Thiết kế Layout 3D (3D Layout Design)| Manager, Admin | Open Layout Editor $\rightarrow$ Place slots/zones $\rightarrow$ Save Draft $\rightarrow$ Preview $\rightarrow$ Publish | `B3.1`, `F3.1` | Phase 3 |
| **UC-08** | Xem báo cáo chuỗi (Chain Reports) | Admin, Manager | Filter by date/lot $\rightarrow$ Load dashboard $\rightarrow$ Export Excel/PDF | `B1.10`, `F1.12` | Phase 1 |
| **UC-09** | Tạo tài khoản nhân sự (Staff Creation)| Admin, Manager | Manager enters staff info for lot $\rightarrow$ BE verifies scope $\rightarrow$ Activation link sent $\rightarrow$ Staff sets pwd/2FA | `B1.3`, `F1.3` | Phase 1 |
| **UC-10** | Hoàn tiền / miễn phí (Refund / Free Exit)| Staff, Manager | Staff/Manager proposes adjustment $\rightarrow$ Different Manager approves $\rightarrow$ New adjusting transaction logged | `B1.8`, `F1.8` | Phase 1 |

*Conclusion*: All 10 Use Cases are fully represented and mapped across the 3 phases of the Implementation Plan.

---

## 5. Verified NFR Inventory and Traceability

### 5.1 NFR Count and Breakdown
SRS Section 6 specifies **exactly 18 Non-Functional Requirements**:
- 14 General System NFRs (`NFR-01` to `NFR-14`).
- 4 3D Visualization NFRs (`NFR-3D-01` to `NFR-3D-04`).
- Total: $14 + 4 = \mathbf{18}$.

| NFR ID | Category | Specific Constraint / Target | Plan Task(s) | Verification Method |
| :--- | :--- | :--- | :--- | :--- |
| **NFR-01** | Performance | Ingress/egress transit $\le 3$ seconds (including online sync) | `B2.5`, `B2.11` | Load test with simulated gate barrier |
| **NFR-02** | Performance | 12-month dashboard query for 50 lots loads $\le 3$ seconds | `B3.5`, `F3.5` | Database aggregation query benchmarking |
| **NFR-03** | Scalability | Support $\ge 100$ lots, 10,000 transits/day/lot (horizontal scale) | `B2.11`, `B3.6` | Distributed stress test on API cluster |
| **NFR-04** | Availability | Cloud uptime $\ge 99.5\%$; gates function offline during WAN outage | `B2.1`, `B2.11` | Physical network disconnection simulation |
| **NFR-05** | Security | TLS encryption, password hashing, RBAC, 2FA, OWASP Top 10 | `B1.3`, `B2.11` | Security audit & static vulnerability scan |
| **NFR-06** | Security | Zero credit card data stored; PCI-DSS compliance via gateway | `B2.4` | Gateway tokenization architecture audit |
| **NFR-07** | Privacy | Image retention limits; compliance with Decree 13/2023/ND-CP | `B2.10` | Automated daily purge job verification |
| **NFR-08** | Integrity | Financial records immutable; adjustments via new transactions | `B1.8` | Database constraint & transaction audit |
| **NFR-09** | Reliability | RPO $\le 15$ minutes, RTO $\le 2$ hours | `B2.10` | Database backup & restore drill |
| **NFR-10** | Usability | POS Staff interface $\le 3$ taps per standard transit | `F1.7`, `F1.13` | Usability stopwatch click-stream test |
| **NFR-11** | Compatibility | Chrome, Edge, Safari; Android 9+, iOS 15+ | `F2.10` | Cross-browser & device matrix testing |
| **NFR-12** | Integration | IoT hardware protocols: REST, MQTT, WebSockets, vendor SDKs | `B2.1` | Edge Service protocol integration test |
| **NFR-13** | Operations | Centralized logging, real-time metrics, CI/CD, OpenAPI contract | `B1.1`, `F1.1` | Pipeline inspection & OpenAPI validation |
| **NFR-14** | Consistency | Edge sync idempotency key; 1 plate cannot have 2 open sessions| `B1.7`, `B2.1` | Concurrency test with dual-gate entry |
| **NFR-3D-01** | 3D Performance| $\ge 30$ FPS on mid-range devices with 500 slots (LOD/instancing)| `F1.9`, `F3.2` | Frame rate profiling on target mobile device |
| **NFR-3D-02** | 3D Load Time | Initial model load $\le 5$ seconds over 4G (GLB $\le 10$ MB, Draco)| `B3.2`, `F3.2` | Network-throttled asset load benchmark |
| **NFR-3D-03** | 3D Real-time | Slot state & ZoneCounter update on 3D $\le 2$ seconds | `B1.5`, `F1.9` | End-to-end WebSocket latency benchmark |
| **NFR-3D-04** | 3D Fallback | WebGL 2 requirement with automatic 2D fallback | `F1.10` | Canvas fallback test with WebGL disabled |

---

## 6. Corrected `dbo.Users.Id` Investigation

### 6.1 Statement of the Problem
- **SRS Section 11, Open Question 1**: *"Cột Id của dbo.Users là kiểu gì (int, uniqueidentifier hay nvarchar(450))? Cần biết để tạo khóa ngoại."*
- **Implementation Plan Section 5 & Table 6**: Declares Open Question 1 as a blocking dependency that blocks item `B1.2` and all subsequent database scripting.

### 6.2 Codebase Evidence Analysis
A thorough inspection of the backend source files reveals the exact configuration:

1. **Entity Definition (`src/PMS.DataAccess/Identity/ApplicationUser.cs`)**:
   ```csharp
   public class ApplicationUser : IdentityUser
   {
       public string Role { get; set; } = "User";

       [NotMapped]
       public string UserId
       {
           get => Id;
           set => Id = value;
       }
   }
   ```
   `ApplicationUser` derives from `IdentityUser`. The base `IdentityUser.Id` property is of type C# `string`.
2. **EF Core Configuration (`src/PMS.DataAccess/Persistence/Configurations/ApplicationUserConfiguration.cs`)**:
   ```csharp
   builder.ToTable("Users");

   builder.Property(u => u.Id)
       .HasColumnName("UserId");
   ```
   The entity maps to table `"Users"`, and the primary key property `Id` is explicitly mapped to database column `"UserId"`.
3. **Migration Specification (`src/PMS.DataAccess/Persistence/Migrations/20261001025913_InitialCreate.cs:28-52`)**:
   ```csharp
   migrationBuilder.CreateTable(
       name: "Users",
       columns: table => new
       {
           UserId = table.Column<string>(type: "nvarchar(450)", nullable: false),
           ...
       },
       constraints: table =>
       {
           table.PrimaryKey("PK_Users", x => x.UserId);
       });
   ```
   The column is generated as `nvarchar(450)` and defined as the primary key.
4. **Foreign Key Usage (`src/PMS.DataAccess/Persistence/AutomatedMigration.cs:72`)**:
   In the startup raw SQL DDL script for `Customers`, the column is defined as:
   ```sql
   [UserId] [nvarchar](450) NULL
   ```
   with a non-clustered index `[IX_Customers_UserId]`.

### 6.3 Reconciled Conclusion and Formal Status
- **Physical Column vs. Property Name**: `dbo.Users.Id` and `dbo.Users.UserId` refer to the **same single physical primary key column**, which in SQL Server is named `UserId`. The name `Id` is the C# property on `IdentityUser`.
- **Classification**: **Partially resolved; runtime/deployed-schema confirmation remains necessary**.
- **Rationale**: While the source code (EF Core configurations, migrations, and DDL scripts in `AutomatedMigration.cs`) uniformly specifies the table as `dbo.Users` with PK column `UserId` of type `nvarchar(450)`, the actual *deployed* database schema in any external or production SQL Server instance cannot be definitively verified without runtime database inspection (which is prohibited in this read-only phase). Furthermore, there is a naming discrepancy where the SRS questions reference `dbo.Users.Id` while the physical column generated by EF Core is named `dbo.Users.UserId`.

---

## 7. Implementation Evidence and Compliance Status

To ensure engineering rigor, the table below maps each major functional capability to six explicit levels of evidence:
1. `Not found in inspected source`
2. `Present in source`
3. `Partially aligned with requirements`
4. `Static evidence suggests alignment`
5. `Test-verified`
6. `Runtime-verified`

| Capability / API Surface | Inspected Source Locations | Evidence Level | Alignment & Gaps vs. SRS v1.3 |
| :--- | :--- | :--- | :--- |
| **User Registration** | `POST /api/users/register`<br>`RegisterScreen.jsx` | `Partially aligned with requirements` | Creates user and customer, but uses password-only (no OTP per `FR-AUTH-01`). Auto-generates username from email prefix. |
| **User Authentication** | `POST /api/users/authenticate`<br>`LoginScreen.jsx` | `Partially aligned with requirements` | Authenticates and returns JWT. Searches strictly by `UserName`; rejects email/phone lookup. 2FA is mocked on FE. |
| **Customer Profile** | `GET/PUT /api/users/profile`<br>`UserScreen.jsx` | `Static evidence suggests alignment` | Profile fields match `UserProfileResponseModel`. Covered by unit tests in `UserProfileServiceTests.cs`. |
| **Vehicle CRUD** | `GET/POST/PUT/DELETE/PATCH /api/users/profile/vehicles`<br>`UserScreen.jsx` | `Static evidence suggests alignment` | Vehicle creation, plate normalization, default selection, and soft deletion implemented. Unit-tested. Integration tests absent. |
| **Vehicle Types Catalog** | `GET /api/users/profile/vehicle-types` | `Static evidence suggests alignment` | Returns active vehicle types. Seeded in `AutomatedMigration.cs`. |
| **Change Password** | `PUT /api/users/profile/change-password` | `Static evidence suggests alignment` | Password change via `UserManager`. Unit-tested in `UserProfileServiceTests.cs`. |
| **Parking Lots / Zones / Slots** | None (Entities in `PMS.Core` only) | `Not found in inspected source` | Zero controllers or API endpoints exist for lot or slot management (`FR-LOT-01` to `06`). |
| **ZoneCounter** | None | `Not found in inspected source` | `ZoneCounter` does not exist as an entity or database table in the codebase. |
| **3D WebGL Rendering** | `UserScreen.jsx` (2D SVG only) | `Not found in inspected source` | Three.js, Babylon.js, WebGL 2, GLB asset loaders, and SignalR streaming hubs are absent. |
| **Gate Operations / ANPR** | None (Entities in `PMS.Core` only) | `Not found in inspected source` | Zero session ingress/egress endpoints or Edge hardware communication handlers exist. |
| **Pricing Engine** | None (Entities in `PMS.Core` only) | `Not found in inspected source` | Zero fee calculation algorithms or tariff evaluation services exist (`FR-PRC-01` to `05`). |
| **Reservations & Booking** | `UserScreen.jsx` (Mock form & SVG) | `Not found in inspected source` | In-memory mock in frontend; zero backend reservation endpoints or deposit policies exist. |
| **Payments & Check-out** | `UserScreen.jsx` (Mock alert) | `Not found in inspected source` | Zero gateway integrations (VNPay, MoMo, VietQR) exist in backend. |
| **Reporting & Dashboards** | None | `Not found in inspected source` | Zero reporting controllers, aggregation queries, or export services exist. |

*Crucial Finding*: While User Profile and Vehicle Management have reached `Static evidence suggests alignment`, **core parking operations remain at `Not found in inspected source`**.

---

## 8. In-Scope and Out-of-Scope Findings

### 8.1 Explicitly Out-of-Scope Domains (SRS Section 2.5)
The SRS explicitly designates the following 7 domains as excluded from the current version:
1. **Periodic passes**: Weekly, monthly, quarterly, annual, and corporate subscription packages.
2. **Electronic Invoicing**: Government e-invoice integration; system issues basic payment receipts only.
3. **Internal Accounts**: Stored-value wallets, Auto-Pay balance deduction, customer loyalty points, reviews, and complaints.
4. **Multi-Tenancy & Open APIs**: Multi-owner tenant separation, public developer APIs, and partner revenue sharing.
5. **Shift Operations**: Staff shift assignment, physical cash handover, and shift cash reconciliation.
6. **Per-Slot Sensors**: Individual ultrasonic or video sensors for transient vehicles.
7. **AI & External Maps**: Machine learning predictive models and third-party map navigation (Google Maps, Mapbox).

### 8.2 Codebase Residuals vs. Scope Boundaries
The backend solution contains 63 scaffolded entity files in `src/PMS.Core/Entities/` inherited from legacy `pmsv2.sql`. Several entity clusters directly represent out-of-scope functionality:

| Out-of-Scope Category | Related Entity Classes in `PMS.Core/Entities` | Assessment & Recommended Handling |
| :--- | :--- | :--- |
| **Periodic Passes** | `Subscription.cs`, `SubscriptionPlan.cs`, `SubscriptionRenewal.cs` | **Unused scaffolded entities**. Retain in code repository pending formal team decision; do not reference in new code. |
| **Electronic Invoices** | `Einvoice.cs` | **Unused scaffolded entity**. Retain without active migration. |
| **Internal Wallets & Loyalty** | `Wallet.cs`, `WalletTransaction.cs`, `LinkedPaymentAccount.cs`, `LoyaltyTransaction.cs` | **Unused scaffolded entities**. Retain without active migration. |
| **Customer Feedback** | `LotReview.cs`, `CustomerFeedback.cs` | **Unused scaffolded entities**. Retain without active migration. |
| **Shift Management** | `Shift.cs`, `ShiftSchedule.cs`, `ShiftTemplate.cs`, `ShiftCashCount.cs` | **Deprecated scaffolded entities**. Explicitly discarded in SRS v1.3 Section 2.4. Retain in codebase; do not generate migrations. |
| **Partner Revenue** | `Partner.cs` | **Unused scaffolded entity**. Retain without active migration. |

### 8.3 Preservation Principle
**Mandatory Directive**: **Do not delete, modify, or refactor any out-of-scope entity classes or source files.** Deletion would alter the team's shared repository history and could introduce build-break cascades in `PmsDbContext.cs`. Any entity cleanup must remain a deferred architectural proposal requiring team consensus and lead architect approval.

---

## 9. Open Business Questions and Additional Ambiguities

SRS Section 11 records 8 open questions. Below is their validated operational impact and current status:

| # | Question Summary | Document Context | Unresolved Status & Operational Impact |
| :- | :--- | :--- | :--- |
| **Q1** | Data type of `dbo.Users.Id` | SRS 11, Plan 6 | **Partially resolved in source code** (`nvarchar(450)`). Live deployed schema requires verification when database access is granted. |
| **Q2** | Confirmation of 15m Exit Window & 15m Hold time | SRS 2.3, 3.7 | **Unresolved**. If unconfirmed, default timers in `B2.3` and `B2.5` remain open to dispute. |
| **Q3** | Deposit & No-Show calculation formula | SRS 11, Plan 6 | **Unresolved**. Blocks `ReservationPolicy` schema, reservation API payload, and payment gateway charge intent calculation. |
| **Q4** | Zone overflow behavior when transient space is full | SRS 11, Plan 6 | **Unresolved / Critical**. Blocks gate entry logic (`FR-OPS-01`). If barrier blocks entry, a physical turnaround lane is needed; if it admits, capacity is breached. |
| **Q5** | ANPR confidence threshold for manual staff check | SRS 11, Plan 6 | **Unresolved** (suggested 85%). Governs automatic barrier opening vs. mandatory staff verification on POS. |
| **Q6** | Maximum gate offline duration & accepted payments | SRS 11, Plan 6 | **Unresolved**. Governs offline cache limits, queue size, and cash fraud liability. |
| **Q7** | Manager multi-lot scope & single-Manager approvals | SRS 11, Plan 6 | **Unresolved / Critical**. Governs Maker-Checker rules (`FR-PAY-04`, `FR-OPS-05`). If a lot has only 1 Manager, approval requests could deadlock without an escalation path. |
| **Q8** | Image retention duration under Decree 13/2023/ND-CP | SRS 11, Plan 6 | **Unresolved** (suggested 90 days). Governs automated storage cleanup job (`B2.10`). |

### Additional Ambiguities Discovered During Review
- **A1. Email vs. Username Login Conflict**: `LoginScreen.jsx` labels the field "Username", but users commonly attempt to log in using their email address. `UserService.cs:64` evaluates only `u.UserName == loginUserModel.Username`. A formal decision is needed on whether `LoginAsync` should query `u.UserName == input || u.Email == input`.
- **A2. Missing `ZoneCounter` Concurrency Pattern**: SRS Section 2.3 mandates `ZoneCounter` updates inside the same transaction as `Session` creation. In high-traffic multi-gate environments, updating a single integer row in SQL Server will cause row-lock contention. The Implementation Plan proposes Redis caching, but the cache-invalidation and write-behind reconciliation strategy is not specified.

---

## 10. Implementation Plan Phase and Effort Review

### 10.1 Effort Number Verification
The Implementation Plan estimates total effort as:
- **Backend (BE)**: 288 person-days (`B1.1` to `B3.6`).
- **Frontend (FE)**: 293 person-days (`F1.1` to `F3.5`).
- **Combined Total**: $\mathbf{581\text{ person-days}}$ ($\approx 288 + 293$).

### 10.2 Effort Representation and Limitations
- **Effort vs. Elapsed Time**: The 581 person-days represent raw individual engineering effort. It does **not** represent calendar duration, an approved milestone commitment, or a project schedule.
- **Sprint Assumptions**: The plan assumes 2-week sprints, but explicitly notes in Section 1 that calendar delivery dates cannot be set until team size and allocation are confirmed.
- **Excluded Work**: The 581 person-days explicitly **exclude** real-hardware integration (cameras, barriers, ticket printers), automated hardware driver testing, security penetration testing, and user acceptance testing (UAT).

### 10.3 Phase Sequencing Analysis
- **Plan Assumption**:
  - Phase 1 (232 person-days): Core MVP operations (Lots, Zones, Slots, Manual Sessions, Pricing, Cash POS, 3D viewer).
  - Phase 2 (242 person-days): Automation, Edge, Online Payments, and Customer/Vehicle APIs (`B2.7`, `F2.3`).
  - Phase 3 (107 person-days): Layout Editor 3D, Dynamic Pricing, Optimization.
- **Codebase Reality**: The development team has completed Phase 2's User/Vehicle profile features while Phase 1's foundational parking operations are unbuilt.
- **Impact**: This represents an **implementation sequencing divergence**. It does not invalidate Phase 1 or Phase 2, but requires updating project tracking to reflect that customer profile work has been completed ahead of schedule, while core parking operations remain the critical path for Phase 1 MVP acceptance.

---

## 11. Corrections Made to the Previous Report

| Item / Finding | Previous Phase 2 Statement | Validated and Corrected Finding | Rationale / Source Evidence |
| :--- | :--- | :--- | :--- |
| **`FR-OPS-06` Numbering Gap** | Stated as an unrenumbered gap without full context. | **Conclusively verified as intentionally removed**. | SRS 2.4 and 3 explicitly state `FR-OPS-06` (Shift management) was removed in v1.3 and IDs were not renumbered to preserve stability. |
| **Active FR Count** | Claimed 61 FRs without individual inventory. | **Verified as exactly 61 active FRs** with complete 61-row auditable traceability matrix. | Extracted directly from XML tables across all 10 modules. |
| **`dbo.Users.Id` Resolution** | Stated as fully resolved and unblocked. | **Reclassified as partially resolved in code; runtime confirmation necessary**. | EF Core models `UserId` as `nvarchar(450)`, but physical deployed database schema cannot be proven without runtime inspection. Discrepancy between SRS name `Id` and column name `UserId` documented. |
| **Implementation Compliance** | Implied features were "Complete" based on code presence. | **Separated into 6 evidence levels**. User/Vehicle features classified as `Static evidence suggests alignment` or `Partially aligned`. | Code presence does not prove runtime correctness or complete SRS compliance. |
| **Out-of-Scope Entity Handling**| Suggested purging or deleting scaffolded out-of-scope entities. | **Corrected to mandatory preservation**. | Project guidelines forbid deleting team files. Entity cleanup is deferred as a team recommendation only. |
| **Effort Number Interpretation**| Cited 581 person-days as a project schedule. | **Clarified as raw estimated effort (person-days)**, not calendar duration or an approved delivery schedule. | Plan Section 1 notes calendar dates depend on staffing headcount. |

---

## 12. Remaining Limitations and Required Follow-up Actions

### 12.1 Analytical Limitations
1. **No Live Database Verification**: Schema conclusions are derived from EF Core configurations and migrations. Live database tables, constraints, and triggers remain uninspected.
2. **No Hardware Test Environment**: Edge Service, ANPR accuracy, and barrier opening latency ($\le 3$s) cannot be verified until physical or simulated IoT hardware is connected.
3. **Team Git Metadata Uninspected**: Git branches, remotes, and commit histories of the team repositories remain uninspected in compliance with the absolute Git prohibition.

### 12.2 Required Follow-up Actions (Awaiting Stakeholder Approval)
1. **Business Clarifications**: Present Open Questions Q2, Q3, Q4, and Q7 to the Project Manager and Business Analyst for formal decisions.
2. **Backlog Re-alignment**: Re-align project sprint tracking to account for completed User/Vehicle APIs and prioritize unbuilt Phase 1 core parking modules (Lots, Zones, Slots, `ZoneCounter`, Manual Sessions, and Pricing Engine).
3. **Phase 3 Authorization**: Maintain a strict read-only posture. Do not modify team source code or begin implementation until formal stakeholder approval is granted.
