# UML Design Models
# Unified Patient Access & Clinical Intelligence Platform

<!-- Schema: .propel/templates/design-model-template.md -->
<!-- Artifact: model | Workflow: /design-model | Version: 1.0.0 -->

---

## Document Control

| Attribute  | Value                                                     |
| ---------- | --------------------------------------------------------- |
| Status     | Draft                                                     |
| Version    | 1.0.0                                                     |
| Created    | 2026-05-23                                                |
| Updated    | 2026-05-23                                                |
| Author     | GitHub Copilot — generated from spec.md v1.0.0            |
| Source     | `.propel/context/docs/spec.md`                            |

---

## 1. Introduction

This document provides the comprehensive UML visual model set for the **Unified Patient Access & Clinical Intelligence Platform**. All diagrams are derived from the Functional Requirements Specification (`spec.md`) and serve as the authoritative visual reference for architecture, data design, and system behaviour.

### Diagram Inventory

| ID    | Diagram Type          | File                              | Description                                              |
| ----- | --------------------- | --------------------------------- | -------------------------------------------------------- |
| DM-01 | Conceptual Model      | `conceptual-model.puml`           | Domain entity class diagram with all aggregate roots     |
| DM-02 | Component Diagram     | `component-diagram.puml`          | System components, services, and integration boundaries  |
| DM-03 | Deployment Diagram    | `deployment-diagram.puml`         | Infrastructure topology and runtime environment          |
| DM-04 | DFD Level-0 (Context) | `dfd-level0.puml`                 | System context — external entities and data flows        |
| DM-05 | DFD Level-1           | `dfd-level1.puml`                 | Internal processes, data stores, and inter-process flows |
| DM-06 | ERD                   | `erd.puml`                        | Logical entity-relationship diagram (PostgreSQL schema)  |
| DM-07 | Sequence — Auth       | `seq-authentication.puml`         | Login, session management, and logout flows              |
| DM-08 | Sequence — Booking    | `seq-appointment-booking.puml`    | Patient appointment booking end-to-end                   |
| DM-09 | Sequence — Slot Swap  | `seq-slot-swap.puml`              | Background dynamic preferred-slot swap                   |
| DM-10 | Sequence — Extraction | `seq-document-extraction.puml`    | Clinical document upload, NLP extraction, staff review   |
| DM-11 | Sequence — Coding     | `seq-medical-coding.puml`         | AI code suggestion, staff review, and finalization       |
| DM-12 | Sequence — Registration  | `seq-registration.puml`           | Patient self-registration + email verification + password reset flow |
| DM-13 | Sequence — Intake        | `seq-patient-intake.puml`         | AI and manual intake modes, mode switch, auto-save, post-submit edit  |
| DM-14 | Sequence — Walk-In       | `seq-walk-in-booking.puml`        | Staff walk-in booking, same-day queue, arrived marking, no-show auto-mark |
| DM-15 | Sequence — Admin Ops     | `seq-admin-operations.puml`       | User management (UC-009), audit log query/export (UC-010), success metrics (FR-118–FR-120) |
| DM-UC1| Use Case — Overview   | `uc-system-overview.puml`         | System-level use case diagram (from spec.md §9.1)        |
| DM-UC2| Use Case — Booking    | `uc-booking-flow.puml`            | Booking and slot-swap use case detail (from spec.md §9.2)|
| DM-UC3| Use Case — Clinical   | `uc-clinical-intelligence.puml`   | Clinical intelligence use case detail (from spec.md §9.3)|

All `.puml` sources are in `.propel/context/docs/uml-models/`.

---

## 2. Conceptual Domain Model (DM-01)

**File:** `.propel/context/docs/uml-models/conceptual-model.puml`

The conceptual model identifies all domain entities and their relationships across six bounded contexts.

### Bounded Contexts

| Context               | Key Entities                                                                 |
| --------------------- | ---------------------------------------------------------------------------- |
| Identity & Access     | `User`, `Patient`, `Staff`, `Admin`, `Role`                                  |
| Scheduling            | `AppointmentSlot`, `Appointment`, `WaitlistEntry`, `NoShowRisk`, `Reminder`  |
| Patient Intake        | `IntakeForm`, `IntakeMode`                                                   |
| Insurance             | `InsuranceEntry`, `DummyInsuranceRecord`, `ValidationStatus`                 |
| Clinical Intelligence | `ClinicalDocument`, `ExtractedField`, `ConflictAlert`, `PatientRecord360`    |
| Medical Coding        | `MedicalCodingSession`, `CodeSuggestion`, `CodeType`, `CodeStatus`           |

### Key Design Decisions

- `User` is the abstract base entity; `Patient`, `Staff`, and `Admin` extend it via table-per-type inheritance.
- `Appointment` holds a foreign key to `AppointmentSlot` (not embedded), enabling slot state management independently of booking lifecycle.
- `WaitlistEntry` links to both the patient's **current** `Appointment` and the **preferred** `AppointmentSlot`, supporting atomic slot-swap without data loss.
- `PatientRecord360` is a single aggregated view per patient, updated by the `Patient 360 Aggregator` service after each extraction review.
- `AuditLogEntry` is write-once by design — no entity (including `Admin`) may issue UPDATE or DELETE against this table.

---

## 3. Component Diagram (DM-02)

**File:** `.propel/context/docs/uml-models/component-diagram.puml`

### Layer Summary

| Layer                         | Components                                                                                           |
| ----------------------------- | ---------------------------------------------------------------------------------------------------- |
| Presentation                  | Patient Portal SPA, Staff Dashboard SPA, Admin Console SPA                                           |
| API Gateway                   | ASP.NET Core BFF — JWT validation, RBAC enforcement, TLS 1.2+ termination, rate limiting             |
| Core Application Services     | Auth, Appointment, Intake, Insurance Validation, Admin, Audit                                        |
| Background / Async Services   | Slot-Swap Engine, Reminder Scheduler, NLP Extraction Worker (all Windows Services)                   |
| Clinical Intelligence Services| Document Processing, NLP/AI Engine, Patient 360 Aggregator, Medical Coding, No-Show Risk Engine      |
| Data Layer                    | PostgreSQL (primary), Upstash Redis (sessions + slot lock), File Storage (PDFs)                      |
| External Services             | Email Provider (SMTP/SendGrid), SMS Provider (Twilio), Google Calendar API, Outlook ICS              |

### Critical Integration Points

- All API calls from SPAs traverse the **API Gateway** — no direct service-to-service calls from the frontend.
- The **Slot-Swap Engine** uses Redis `SETNX` for atomic locking to prevent double-booking (FR-025).
- The **NLP Extraction Worker** operates asynchronously; the Document Processing Service enqueues jobs and responds immediately with `202 Accepted`.
- **Audit Service** receives events from all services as fire-and-forget; failures are retried but never block the originating service.

---

## 4. Deployment Diagram (DM-03)

**File:** `.propel/context/docs/uml-models/deployment-diagram.puml`

### Infrastructure Topology

| Tier                  | Technology                                      | Constraint (AC-1, AC-7)             |
| --------------------- | ----------------------------------------------- | ------------------------------------ |
| Frontend Hosting      | Vercel / Netlify (free tier)                    | CDN-delivered, HTTPS enforced        |
| Backend Runtime       | Windows Server + IIS (TLS 1.2+ binding :443)    | IIS hosted ASP.NET Core              |
| Background Workers    | Windows Services (3 services)                   | Same Windows Server node             |
| Primary Database      | PostgreSQL 17 (self-hosted / free)              | Encrypted at rest via pgcrypto; TLS connections enforced |
| Session Cache & Lock  | Upstash Redis (free tier, cloud-managed)        | TLS port 6380; SETNX for slot lock   |
| File Storage          | PostgreSQL BYTEA column in `clinical_documents` | PDFs stored in DB (ADR-006 in design.md); max 10 MB per file; covered by DB backup and pgcrypto encryption |
| External              | Twilio (SMS), SMTP/SendGrid (email), GCal API   | Free-tier integrations               |

### Security Notes

- TLS 1.2+ is mandatory for all inbound and outbound network traffic (NFR-001, AC-9).
- PostgreSQL data is encrypted at rest (HIPAA compliance).
- PDF documents are stored encrypted on the server filesystem.
- Redis connections use TLS (port 6380).

---

## 5. Data Flow Diagrams

### 5.1 Level-0 Context Diagram (DM-04)

**File:** `.propel/context/docs/uml-models/dfd-level0.puml`

The Level-0 DFD establishes the system boundary and identifies all external entities and their data exchanges with the platform.

| External Entity     | Data INTO System                                   | Data FROM System                                     |
| ------------------- | -------------------------------------------------- | ---------------------------------------------------- |
| Patient             | Registration, credentials, bookings, intake, docs  | Confirmations, reminders, calendar links, 360° view  |
| Staff               | Walk-in bookings, arrivals, clinical notes, reviews| Queue, risk dashboard, extraction panel, coding panel|
| Admin               | User CRUD, role assignments, audit queries         | User lists, audit log (CSV export)                   |
| Email Provider      | —                                                  | Confirmation PDFs, reminders, swap notifications     |
| SMS Provider        | —                                                  | 2h reminders, swap SMS alerts                        |
| Calendar Services   | —                                                  | Google Calendar events, Outlook .ics files           |
| Dummy Insurance DB  | Match results                                      | Validation requests                                  |

### 5.2 Level-1 Process Diagram (DM-05)

**File:** `.propel/context/docs/uml-models/dfd-level1.puml`

The Level-1 DFD decomposes the platform into 12 discrete processes with their associated data stores.

| Process                     | Key Data Stores                         | Downstream Triggers                          |
| --------------------------- | --------------------------------------- | --------------------------------------------- |
| P1 — Authentication         | DS1 (Users), Redis (sessions)           | P12 (audit)                                   |
| P2 — Appointment Booking    | DS2 (Appointments), DS3 (Waitlist)      | P3 (slot-swap), P4 (reminders), P6 (insurance)|
| P3 — Slot-Swap Engine       | DS2, DS3                                | P4 (swap notification), P12 (audit)           |
| P4 — Reminders & Calendar   | DS10 (Reminders), DS2                   | Email, SMS, Calendar external                 |
| P5 — Patient Intake         | DS4 (Intake Forms)                      | P12 (audit)                                   |
| P6 — Insurance Validation   | DS5 (Insurance)                         | P10 (risk score input)                        |
| P7 — Document Extraction    | DS6 (Documents)                         | P8 (360° aggregation), P12 (audit)            |
| P8 — Patient 360° Aggregator| DS6, DS7 (Patient Record)               | P9 (coding), P12 (audit)                      |
| P9 — Medical Coding         | DS8 (Coding Sessions), DS7              | P12 (audit)                                   |
| P10 — No-Show Risk Engine   | DS2, DS5, DS11 (Risk Scores)            | Staff risk dashboard (FR-035, FR-036); also feeds P13 |
| P11 — Admin & User Mgmt     | DS1 (Users)                             | P12 (audit)                                   |
| P12 — Audit Logging         | DS9 (Audit Log, append-only)            | Admin export                                  |
| P13 — No-Show Auto-Mark     | DS2 (Appointments)                      | Sets appointment status to `No-Show` when scheduled end time passes without `Arrived` (FR-058); writes audit log entry (FR-112) |
| P14 — Admin Success Metrics | DS2, DS9, DS6                           | Aggregates no-show rate (FR-118), critical conflicts count (FR-119), platform adoption stats (FR-120) for Admin dashboard |

---

## 6. Entity Relationship Diagram (DM-06)

**File:** `.propel/context/docs/uml-models/erd.puml`

### Entity Summary

| Entity                      | Purpose                                                           | Key Constraints                                  |
| --------------------------- | ----------------------------------------------------------------- | ------------------------------------------------ |
| `users`                     | Base account for all roles                                        | `email` UNIQUE; `role` ENUM                      |
| `patients`                  | Patient demographics; extends `users`                             | PK = FK → `users`                                |
| `staff_profiles`            | Staff identity; extends `users`                                   | PK = FK → `users`                                |
| `appointment_slots`         | Available time slots per provider/specialty                       | `is_available` managed atomically via Redis lock |
| `appointments`              | Confirmed bookings                                                | `reference_number` UNIQUE; `status` ENUM (Confirmed, Cancelled, Rescheduled, No-Show — FR-058) |
| `waitlist_entries`          | Patient preference queue for unavailable slots                    | FIFO by `created_at` (BR-2)                      |
| `no_show_risk_scores`       | Rule-based risk assessment per appointment                        | 1-to-1 with `appointments`                       |
| `reminders`                 | Scheduled email/SMS notification records                          | `reminder_type` ENUM; `status` ENUM              |
| `intake_forms`              | Patient intake data (AI or manual mode)                           | `mode` ENUM; auto-save via `last_saved_at`       |
| `insurance_entries`         | Soft-check insurance validation per booking                       | `validation_status` ENUM; non-blocking (BR-3)    |
| `dummy_insurance_records`   | Internal dummy insurer dataset (seed data)                        | Read-only reference table                        |
| `password_reset_tokens`     | Time-limited single-use tokens for patient password reset         | `token_hash` bcrypt; `expires_at` max 1 h; `is_used` set on first use; nightly purge (FR-006, DR-015) |
| `clinical_documents`        | Uploaded PDF files stored as BYTEA in DB (ADR-006)                | `status` ENUM; no filesystem reference; max 10 MB |
| `extracted_fields`          | NLP-extracted data fields with confidence                         | `confidence_level` ENUM; `is_accepted` flag       |
| `conflict_alerts`           | Detected data conflicts across documents                          | `is_acknowledged` required before commit (BR-9)  |
| `patient_records`           | Aggregated 360° patient view (JSONB columns)                      | 1-to-1 with `patients`                           |
| `medical_coding_sessions`   | Staff-reviewed ICD-10/CPT coding session                          | `agreement_rate` tracked per session (FR-096)    |
| `code_suggestions`          | AI-suggested or manually added codes                              | `status` ENUM: ACCEPTED, REJECTED, OVERRIDDEN, MANUAL |
| `audit_logs`                | Immutable event log for all platform actions                      | Append-only; no UPDATE/DELETE permitted (BR-6)   |

### Referential Integrity Notes

- `conflict_alerts.document_a_id` and `document_b_id` both reference `clinical_documents` — this is a self-referential multi-FK pattern requiring explicit constraint naming.
- `audit_logs.actor_role` includes `SYSTEM` to accommodate background service events (slot-swap, reminder delivery) where no user actor exists.
- `appointments.staff_id` is nullable — populated only for walk-in bookings (FR-051).
- `password_reset_tokens.user_id` references `users`; only Patient-role users may request a reset (FR-006); tokens are invalidated immediately on first use and hard-deleted by a nightly cleanup job.

---

## 7. Sequence Diagrams

### 7.1 Authentication & Session Management (DM-07)

**File:** `.propel/context/docs/uml-models/seq-authentication.puml`
**References:** UC-002, FR-002–FR-008

Covers three flows:

1. **Login** — credential validation, bcrypt verify, Redis session creation (15 min TTL).
2. **Authenticated Request** — session validation and TTL slide on each request; 401 on expiry.
3. **Logout** — explicit Redis session deletion and audit log entry.
4. **Password Reset** — patient requests reset; system generates a time-limited single-use token, stores `token_hash` in `password_reset_tokens`, and emails a reset link; on link click the token is validated (expiry + `is_used` check), new password is hashed and saved, token is marked used, and an audit log entry is written (FR-006).

Key security behaviours modelled:

- Generic "Invalid credentials" response (no field-level hint) — prevents user enumeration (OWASP A7).
- Role claim is never returned beyond the consuming role's scope (FR-008).
- Session timeout triggers audit log entry (FR-115).

### 7.2 Patient Appointment Booking (DM-08)

**File:** `.propel/context/docs/uml-models/seq-appointment-booking.puml`
**References:** UC-003, FR-011–FR-017, FR-061–FR-065

End-to-end flow covering:

1. Slot search and display.
2. Duplicate booking guard (same provider, same day — FR-017).
3. Insurance soft-check (non-blocking — FR-065).
4. Redis `SETNX` atomic slot reservation (prevents race-condition double-booking — FR-025).
5. Waitlist entry creation for preferred slot (FR-013).
6. No-show risk computation trigger (FR-035).
7. Confirmation email with PDF + reminder scheduling (FR-014, FR-031, FR-032).

### 7.3 Dynamic Preferred-Slot Swap (DM-09)

**File:** `.propel/context/docs/uml-models/seq-slot-swap.puml`
**References:** UC-004, FR-021–FR-025

Background Windows Service flow:

1. Slot-free event published on cancellation or reschedule.
2. FIFO waitlist query — earliest entry timestamp wins (BR-2).
3. Redis `SETNX` atomic lock prevents concurrent double-swap (FR-025).
4. Database transaction atomically updates booking, releases old slot, removes waitlist entry.
5. Email + SMS notification sent to patient (FR-023).
6. Full audit trail logged (FR-112).
7. Entire process completes within 60 seconds (FR-024).

### 7.4 Clinical Document Extraction (DM-10)

**File:** `.propel/context/docs/uml-models/seq-document-extraction.puml`
**References:** UC-007, FR-071–FR-080

Five-phase flow:

1. **Upload** — file format/size validation; encrypted storage; `202 Accepted` response (async).
2. **Extraction** — NLP engine parses PDF, assigns HIGH/LOW confidence per field.
3. **Aggregation** — Patient 360 Aggregator de-duplicates and detects critical conflicts.
4. **Review** — Staff views extraction panel; colour-coded by confidence; conflict alerts displayed.
5. **Commit** — Accepted/corrected fields written to `patient_records`; full audit trail.

Conflict acknowledgement is mandatory before commit (BR-9).

### 7.5 Medical Coding Review & Approval (DM-11)

**File:** `.propel/context/docs/uml-models/seq-medical-coding.puml`
**References:** UC-008, FR-091–FR-096

Flow:

1. Auto-triggered after patient record update — AI generates ICD-10 and CPT suggestions.
2. Each suggestion includes confidence %, source snippet, and extracted text (FR-093).
3. Staff can Accept, Override (with replacement code + reason), or Reject (with mandatory justification).
4. Manual code addition is supported (logged as `MANUAL` status).
5. Finalization computes AI-Human Agreement Rate (FR-096) and writes immutable audit entry (FR-095).

---

### 7.6 Patient Registration & Password Reset (DM-12)

**File:** `.propel/context/docs/uml-models/seq-registration.puml`
**References:** UC-001, FR-001, FR-006, FR-007

Covers two flows:

1. **Registration** — patient submits email, password, name, DOB, phone; system validates uniqueness; hashes password (Argon2id); creates account with role = Patient; sends verification email; patient activates account via link.
   - A1: duplicate email — generic error, no information about which field failed (OWASP A7).
   - A2: weak password — policy requirements displayed; submission blocked.
2. **Password Reset** — patient submits email; system looks up account (no hint if not found); generates a cryptographically random token; stores `token_hash` + `expires_at` (1 h) + `is_used = false` in `password_reset_tokens`; sends reset link via email. On link click: token hash is validated and expiry checked; new password is hashed; token `is_used` set to `true`; audit log entry written (FR-115).

### 7.7 Patient Intake (DM-13)

**File:** `.propel/context/docs/uml-models/seq-patient-intake.puml`
**References:** UC-005, FR-041–FR-046

Flow:

1. Patient selects AI Conversational or Manual Form mode (FR-041, FR-042).
2. *(AI mode)* State machine advances through fixed intake questions; each answer is auto-saved to `intake_forms` (FR-046).
3. *(Manual mode)* Patient fills structured form; debounced auto-save on each field change (FR-046).
4. **Mode switch** — all data collected in the current mode is transferred to the new mode's interface without loss (FR-043).
5. Patient reviews summary and submits; intake status set to `complete`; linked to appointment record.
6. **Post-submit edit** — patient modifies a field; change is saved and audit log entry written (FR-044).

### 7.8 Walk-In Booking & No-Show Auto-Mark (DM-14)

**File:** `.propel/context/docs/uml-models/seq-walk-in-booking.puml`
**References:** UC-006, FR-051–FR-058

Flow:

1. Staff searches same-day available slots (FR-051).
2. Staff enters patient details; system checks for existing account by email/phone.
3. *(Existing patient)* booking linked to existing account. *(New patient)* staff optionally creates account (FR-052).
4. Walk-in booking persisted; patient added to same-day queue (FR-053); audit log entry written.
5. Staff marks patient as "Arrived" (FR-054); queue updated; audit log entry written.
6. **No-Show Auto-Mark** — scheduled background job (P13 in DFD Level-1) checks all appointments past their scheduled end time with no `Arrived` status; sets `appointment.status = No-Show`; writes audit log entry (FR-058, FR-112).

### 7.9 Admin Operations & Success Metrics (DM-15)

**File:** `.propel/context/docs/uml-models/seq-admin-operations.puml`
**References:** UC-009, UC-010, FR-101–FR-105, FR-111–FR-117, FR-118–FR-120

Covers three flows:

1. **User Management** — Admin creates Staff account with temp password (FR-101); assigns/modifies roles (FR-102); deactivates/reactivates accounts (FR-103, FR-104); system rejects Admin role assignment via UI (FR-105); all changes written to audit log.
2. **Audit Log Query & Export** — Admin filters log by date range, actor, action type (FR-117); system returns paginated results; Admin exports as CSV.
3. **Success Metrics Dashboard** — Admin opens metrics panel; system queries: no-show rate % over selected date range (FR-118); count of critical conflicts detected (FR-119); total patient accounts and appointments booked (FR-120). All queries read-only against aggregated data.

---

## 8. Cross-Cutting Design Decisions

### 8.1 Security Architecture

| Concern              | Design Decision                                                                         | FR / NFR       |
| -------------------- | --------------------------------------------------------------------------------------- | -------------- |
| Authentication       | JWT tokens; server-side session in Redis with 15 min TTL + sliding expiry               | FR-005, NFR-003|
| Password Storage     | bcrypt/Argon2 hash+salt; never stored in plaintext                                     | FR-007, NFR-001|
| RBAC                 | Enforced at API Gateway layer; role not returned beyond consuming role (FR-008)         | FR-004, NFR-002|
| Transport Security   | TLS 1.2+ mandatory on all service-to-service and client-to-server channels             | NFR-001, AC-9  |
| Audit Immutability   | `audit_logs` table enforced append-only via DB trigger / row-level security            | FR-116, NFR-006|
| Admin Provisioning   | Admin accounts seeded via CLI only; no self-registration path in API                   | FR-105, BR-7   |

### 8.2 Concurrency & Atomicity

| Scenario                        | Mechanism                                           | FR     |
| ------------------------------- | --------------------------------------------------- | ------ |
| Slot reservation during booking | Redis `SETNX` with 10 s TTL                         | FR-025 |
| Slot-swap atomic execution      | Redis `SETNX` with 30 s TTL + PostgreSQL transaction| FR-025 |
| Duplicate booking prevention    | DB-level unique constraint + application check      | FR-017 |

### 8.3 Async Processing Patterns

| Process                  | Pattern                              | Timeout / SLA   |
| ------------------------ | ------------------------------------ | --------------- |
| NLP Document Extraction  | Async job queue (Windows Service)    | Best-effort     |
| Slot-Swap Engine         | Event-driven (slot-free event)       | ≤ 60 s (FR-024) |
| Reminder Delivery        | Cron-scheduled Windows Service       | 24h / 2h windows|
| Code Suggestion          | Triggered post-record-update         | Best-effort     |

---

## 9. Diagram File Index

All PlantUML source files (`.puml`) are located in:

```
.propel/context/docs/uml-models/
├── conceptual-model.puml          ← DM-01 Domain class model
├── component-diagram.puml         ← DM-02 Component architecture
├── deployment-diagram.puml        ← DM-03 Infrastructure topology
├── dfd-level0.puml                ← DM-04 Context DFD
├── dfd-level1.puml                ← DM-05 Level-1 DFD
├── erd.puml                       ← DM-06 Logical ERD
├── seq-authentication.puml        ← DM-07 Sequence: Auth + Password Reset
├── seq-appointment-booking.puml   ← DM-08 Sequence: Booking
├── seq-slot-swap.puml             ← DM-09 Sequence: Slot-swap
├── seq-document-extraction.puml   ← DM-10 Sequence: Extraction
├── seq-medical-coding.puml        ← DM-11 Sequence: Medical coding
├── seq-registration.puml          ← DM-12 Sequence: Patient registration (NEW)
├── seq-patient-intake.puml        ← DM-13 Sequence: Patient intake (NEW)
├── seq-walk-in-booking.puml       ← DM-14 Sequence: Walk-in + No-Show auto-mark (NEW)
├── seq-admin-operations.puml      ← DM-15 Sequence: Admin ops + success metrics (NEW)
├── uc-system-overview.puml        ← DM-UC1 Use case overview
├── uc-booking-flow.puml           ← DM-UC2 Booking use case
└── uc-clinical-intelligence.puml  ← DM-UC3 Clinical use case
```

---

## 10. Traceability to Specification

| Diagram | FR Coverage                                                                       | UC Coverage                      |
| ------- | --------------------------------------------------------------------------------- | -------------------------------- |
| DM-01   | FR-001–FR-008, FR-011–FR-017, FR-021–FR-025, FR-031–FR-036, FR-041–FR-046, FR-051–FR-058, FR-061–FR-065, FR-071–FR-080, FR-091–FR-096, FR-101–FR-105, FR-111–FR-120 | All UCs |
| DM-02   | All FR — component responsibility mapping                                         | All UCs                          |
| DM-03   | NFR-001, NFR-004, NFR-007, AC-1, AC-6, AC-7, AC-9                                | —                                |
| DM-04   | All FR — external boundary flows                                                  | All UCs                          |
| DM-05   | All FR — internal process decomposition                                           | All UCs                          |
| DM-06   | FR-001–FR-008, FR-011–FR-017, FR-021–FR-025, FR-031–FR-036, FR-041–FR-046, FR-051–FR-058, FR-061–FR-065, FR-071–FR-080, FR-091–FR-096, FR-111–FR-120 | All UCs |
| DM-07   | FR-001–FR-008, FR-006, FR-115                                                    | UC-001, UC-002                   |
| DM-08   | FR-011–FR-017, FR-031–FR-036, FR-061–FR-065                                       | UC-003                           |
| DM-09   | FR-021–FR-025, FR-112                                                             | UC-004                           |
| DM-10   | FR-071–FR-080, FR-113                                                             | UC-007                           |
| DM-11   | FR-091–FR-096, FR-114                                                             | UC-008                           |
| DM-12   | FR-001, FR-006, FR-007, FR-115                                                    | UC-001                           |
| DM-13   | FR-041–FR-046                                                                    | UC-005                           |
| DM-14   | FR-051–FR-058, FR-112                                                            | UC-006                           |
| DM-15   | FR-101–FR-105, FR-111–FR-117, FR-118–FR-120                                     | UC-009, UC-010                   |

---

*End of Model Document — Version 1.0.0 | Status: Draft | 2026-05-23*
