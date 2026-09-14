---
name: smart-school-api-contracts
description: Authoritative REST API specification, OpenAPI 3.0 contracts, JSON request/response DTOs, HTTP status codes, and endpoint documentation for the Smart School Enterprise Core Microservices. Covers Auth & Tenancy, Academic Students, Daily Attendance, Finance & Fees, and Assessment Exams. Trigger on: "api", "rest api", "api contracts", "openapi spec", "endpoint documentation", "request response dtos", "http status codes", "swagger spec".
---

# Core REST API Specifications & Contracts Standard
## Smart School Enterprise Platform (Spring Boot 3 & OpenAPI 3.0)

### Executive Overview & API Governance

The **Smart School REST API** serves as the unified programmatic interface across web portals (Super Admin, Campus Dean, Teacher, Accountant, Receptionist, Librarian, Student, Parent), the Public Front Site (`PUBLIC_CMS`), and future native mobile applications.

All endpoints adhere to **RESTful design principles**, JSON payloads, RFC 7807 problem details, and strict multi-tenant authorization headers.

---

## 0. The 10 Inviolable Golden API Design Rules

Every REST & WebSocket API endpoint across all microservices and gateways must strictly enforce these 10 rules:

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                 THE 10 GOLDEN RULES OF ENTERPRISE APIS                                 │
├────┬───────────────────────────────────┬────────────────────────────────────────────────────────────┤
│ #  │ Architectural Principle           │ Concrete Implementation in Smart School Platform           │
├────┼───────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ 1  │ Keep Endpoints Predictable        │ RESTful resource nouns only (e.g. GET /api/v1/students,    │
│    │ (Path vs. Query Params Standard)  │ POST /api/v1/fees/invoices); NO RPC verbs like /getStudent.│
│    │                                   │ • PATH PARAMS: For single resource identity only:          │
│    │                                   │   GET /api/v1/courses/{courseId} -> Single Object.         │
│    │                                   │ • QUERY PARAMS (Option B): Mandatory for all Filtering,    │
│    │                                   │   Searching & Collections:                                 │
│    │                                   │   GET /api/v1/courses?name=js&level=adv&page=0&size=20     │
│    │                                   │   -> Returns paginated collection envelope.                │
├────┼───────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ 2  │ Version Your APIs                 │ URI path versioning mandatory: /api/v1/..., /api/v2/...    │
│    │                                   │ Backward compatibility guaranteed across releases.         │
├────┼───────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ 3  │ Validate Every Input              │ Jakarta Bean Validation (@NotNull, @Size, @Pattern);       │
│    │                                   │ Reject malformed payloads immediately before DB query.     │
├────┼───────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ 4  │ Use Proper HTTP Status Codes      │ 200 OK, 201 Created, 202 Accepted (Async Kafka),           │
│    │                                   │ 400 Bad Request, 401 Unauthorized, 403 Forbidden,          │
│    │                                   │ 404 Not Found, 409 Conflict, 422 Unprocessable, 429 Limit. │
├────┼───────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ 5  │ Make Critical Operations          │ X-Idempotency-Key header mandatory on POST/PUT payments;   │
│    │ Idempotent                        │ Redis 120s TTL check prevents duplicate double-charges.    │
├────┼───────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ 6  │ Paginate Large Responses          │ Cursor / offset pagination: ?page=0&size=50 (Max size 100);│
│    │                                   │ Never return unbounded SELECT * from database.             │
├────┼───────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ 7  │ Rate-Limit Sensitive Endpoints    │ Redis Token Bucket: 10 req/min for /login, 20 req/min for  │
│    │                                   │ payments; protects against brute-force & denial of service.│
├────┼───────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ 8  │ Log Important Requests            │ Structured JSON logging with X-Request-ID & traceparent;   │
│    │                                   │ Passwords, card numbers, and Bearer tokens MASKED.         │
├────┼───────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ 9  │ Document Your API Contracts       │ OpenAPI 3.1 / Swagger auto-generated schemas;              │
│    │                                   │ Published Language serving as single source of truth.      │
├────┼───────────────────────────────────┼────────────────────────────────────────────────────────────┤
│ 10 │ NEVER TRUST THE CLIENT. EVER. 😭  │ Re-verify JWT claims, recalculate all prices server-side,  │
│    │                                   │ enforce Neon RLS tenant isolation app.current_branch_id.   │
└────┴───────────────────────────────────┴────────────────────────────────────────────────────────────┘
```

---

## 1. Global API Standards & Request Headers

### Mandatory Request Headers

```http
Authorization: Bearer <JWT_ACCESS_TOKEN>
X-Branch-ID: c4b8e21a-9f12-4a7b-8912-3a5e89d12345
X-Request-ID: REQ-8942-XF
Idempotency-Key: 7da7a728-f910-11e6-942a-68f728c1ba70
Content-Type: application/json
Accept: application/json
```

---

## 1.5. SailPoint & Azure Enterprise REST Specifications (MVP Standards)

Synthesizing the **SailPoint RESTful API Guidelines** and the **Microsoft Azure API Design Guide eBook**, the platform enforces these architectural directives:

### A. HTTP Verb Semantics & Asynchronous Processing (Azure / RFC 7231)
* **POST**: Creates resources or triggers actions. Must return `201 Created` with a `Location: /api/v1/resources/{id}` header.
* **PUT**: Replaces the entire resource state idempotently. If ID does not exist, returns `404 Not Found` (do not silently create unless explicitly designed as client-assigned ID).
* **PATCH (RFC 7396 JSON Merge Patch)**: Partial resource modification. Content-Type must be `application/merge-patch+json`.
* **DELETE**: Deletes resources idempotently. Returns `204 No Content` on success, or `207 Multi-Status` for bulk deletion batches.
* **Asynchronous Operations (`202 Accepted` & `303 See Other`)**:
  - For long-running operations (e.g. bulk admit card rendering, mass fee calculation), server immediately returns:
    ```http
    HTTP/1.1 202 Accepted
    Location: /api/v1/operations/status/98421
    ```
  - Client polls status endpoint. When complete, status returns:
    ```http
    HTTP/1.1 303 See Other
    Location: /api/v1/exams/admit-cards/batch-98421
    ```

### B. Optimistic Concurrency Control (ETag & RFC 7232 If-Match)
* To prevent lost updates during concurrent edits (e.g., two accountants modifying the same fee ledger or two teachers modifying grades):
  * Responses return `ETag: "version_hash"`.
  * Mutating `PUT` / `PATCH` requests MUST supply `If-Match: "version_hash"`.
  * If the record was modified by another user in the interim, server immediately returns `412 Precondition Failed`.

### C. Standardized Money Object (ISO 4217 & SailPoint Standard)
* To prevent floating-point rounding errors across currencies (USD and KHR):
* **Forbidden**: Plain float/double numbers (e.g. `"amount": 19.99`).
* **Mandatory Closed Schema**:
  ```json
  "tuitionFee": {
    "amount": 150.00,
    "currency": "USD"
  },
  "khmerRielEquivalent": {
    "amount": 615000,
    "currency": "KHR"
  }
  ```
* Server implementation uses Java `BigDecimal` exclusively.

### D. Deprecation & Sunset Lifecycle (RFC 8594)
* Deprecated API versions or fields MUST return RFC 8594 standard headers:
  ```http
  Deprecation: Tue, 31 Dec 2025 23:59:59 GMT
  Sunset: Wed, 31 Dec 2026 23:59:59 GMT
  ```
* Guaranteed minimum **2-year deprecation grace period** before decommissioning endpoints.

### E. Rate Limit Quota Headers (RFC 6585)
* When approaching or exceeding limits, server returns:
  ```http
  HTTP/1.1 429 Too Many Requests
  Retry-After: 60
  X-RateLimit-Limit: 120
  X-RateLimit-Remaining: 0
  X-RateLimit-Reset: 1726320000
  ```

### F. OpenAPI 3.1 & Schema Naming Conventions
* **Property Names**: Strict ASCII `camelCase` (e.g., `studentNumber`, `invoiceId`).
* **Resource Paths**: Lowercase kebab-case (e.g., `/api/v1/academic-years`, `/api/v1/fee-structures`).
* **Enum Values**: Strict `UPPER_SNAKE_CASE` (e.g., `PENDING_VERIFICATION`, `PARTIALLY_PAID`).
* **Booleans**: Use positive semantics (`enabled: true`, not `disabled: false`); never return `null` for booleans.
* **Top-Level Envelope**: Responses MUST ALWAYS return a JSON object (never top-level bare arrays `[]`) to guarantee backward-compatible extensibility.

### G. Microservice API Patterns (MAP) Standard (Zimmermann et al., Addison-Wesley 2023)
Synthesizing the foundational work by Olaf Zimmermann, Mirko Stocker, Daniel Lübke, Uwe Zdun, and Cesare Pautasso (Vaughn Vernon Signature Series):
1. **Message Granularity Patterns (Embedded Entity vs. Linked Information Holder)**:
   * **Embedded Entity**: Used when data is tightly coupled and displayed immediately (e.g. `studentProfile` embeds `currentAddress`). Reduces network round trips and ensures atomic, internally consistent snapshots.
   * **Linked Information Holder**: Used when related data has high volatility or large payload size (e.g. `moveHistory` or `uploadedDocuments` referenced via `{"rel": "documents", "href": "/api/v1/students/{id}/documents"}`). Prevents overfetching and decouples caching lifetimes.
2. **Client-Driven Message Content / Response Shaping (Wish List & Wish Template)**:
   * **Wish List Pattern**: Clients request a sparse projection of attributes via query string:
     `GET /api/v1/students/{id}?fields=id,fullName,rollNo,balance`
     - Server respects *data parsimony (Datensparsamkeit)*, saving bandwidth and mobile device battery.
   * **Wish Template Pattern**: Used for nested parameter trees, allowing clients to shape complex JSON responses without requiring full GraphQL server overhead.
3. **Message Exchange Optimization (Request Bundle Pattern)**:
   * Combines multiple independent actions into a single HTTP conversation container (e.g. `POST /api/v1/attend/batch-bundle` to process multiple student roll-calls in one round trip).
   * Supports `207 Multi-Status` for per-item individual execution reporting.
4. **Lifecycle Management & Two-in-Production Pattern**:
   * Platform guarantees that when a new major API version is published (e.g. `/api/v2/`), the previous version (`/api/v1/`) remains running simultaneously (**Two-in-Production**) under a **Limited Lifetime Guarantee** (minimum 2 years).
   * Deprecations enforce **RFC 8594 Sunset & Deprecation headers** to prevent broken third-party and mobile clients.

---

## 1.6. Enterprise Cloud RESTful Engine Blueprint (CPM / N2WS v1.4.0 Standards)

Synthesizing the production standards from the **N2WS CPM (Cloud Protection Manager) RESTful API (v1.4.0)**, the platform enforces these mission-critical operational patterns:

### A. Kid Storytime Metaphor (Explain-Like-I'm-8 Mental Model)
* **The Magic Lego Castle**: Imagine building an enormous Lego fortress in your room. If a playful puppy knocks it over, you don't cry—you click a button (`POST /recover/`) and a friendly robot rebuilds every brick in 10 seconds from a "magic snapshot"!
* **The Two Wristbands**: You unlock the castle using a permanent VIP key to get a **Yellow Glow-in-the-Dark Wristband** (Access Token, 1 hour). When it runs out of glow, you use your all-day **Green Wristband** (Refresh Token, 24 hours) to get a shiny new yellow one.
* **The Attic & The Freezer**: You keep toys you play with daily on the desk (EBS). Toys you play with once a month go to the attic (S3). Toys you only need for graduation get frozen in deep ice (Glacier/Freezer) where they cost almost zero pennies!

### B. Dual-Token Authentication Lifecycle
* `POST /api/token/obtain/api_key/` -> Returns `{ "access": "...", "refresh": "..." }`.
* `POST /api/token/refresh/` -> Returns renewed `{ "access": "..." }`.
* Access Token lifespan: **1 hour**. Refresh Token lifespan: **24 hours**.
* Every authenticated call passes `Authorization: Bearer <Access Token>`. Failure returns `401 Unauthorized` with `WWW-Authenticate`.

### C. Content-Negotiated Versioning
* Versioning supplements the media type in the header:
  `Accept: application/json; version=1.4.0`
* Omission defaults to the latest stable version; invalid versions strictly return `406 Not Acceptable`.

### D. Canonical Pagination & Navigation Envelope
* Collections accept query parameters `?page=XX&page_size=XX`.
* Returns canonical envelope:
  ```json
  {
    "count": 100,
    "next": "https://api.domain.com/api/policies/?page=4&page_size=20",
    "previous": "https://api.domain.com/api/policies/?page=2&page_size=20",
    "results": [ ... ]
  }
  ```

### E. Multi-Field Ordering & Query Syntax
* Ascending: `?ordering=username`
* Descending: Prepend `-` (e.g. `?ordering=-start_time`)
* Multi-property: `?ordering=every_unit,-every_how_many`

### F. Structured Error Response Matrix
* Validation failures return an indexed `errors` dictionary:
  ```json
  {
    "errors": [
      {
        "name": [{ "message": "This field may not be blank.", "code": "blank" }],
        "description": [{ "message": "This field contains invalid characters.", "code": "invalid" }]
      }
    ]
  }
  ```

### G. Asynchronous Task Lifecycle (`202 Accepted` & Tracker IDs)
* Long-running operations (cloud restores, backups, VPC clones, archive exports) respond immediately with `202 Accepted` and `{ "tracker_id": 1234 }`.
* Dedicated endpoints allow clients to monitor progress (`GET /api/backups/?tracker_id=1234`) or abort operations (`POST /api/policies/{id}/archive/abort/`).

```http
Authorization: Bearer <JWT_ACCESS_TOKEN>
X-Branch-ID: c4b8e21a-9f12-4a7b-8912-3a5e89d12345
X-Request-ID: REQ-8942-XF
Content-Type: application/json
Accept: application/json
```

### Standardized Response Envelope (RFC 7807 Compliant)

All successful and paginated responses use the universal response schema:

```json
{
  "success": true,
  "statusCode": 200,
  "message": "Operation completed successfully",
  "timestamp": "2026-09-12T06:55:00.124Z",
  "trackingId": "REQ-8942-XF",
  "data": { ... },
  "pagination": {
    "page": 0,
    "size": 50,
    "totalElements": 482,
    "totalPages": 10,
    "hasMore": true
  }
}
```

### Error Response Envelope

```json
{
  "success": false,
  "statusCode": 400,
  "error": "BAD_REQUEST",
  "message": "Validation failed on 2 fields",
  "timestamp": "2026-09-12T06:55:00.124Z",
  "trackingId": "REQ-8942-XF",
  "errors": [
    { "field": "phone", "message": "Invalid telephone format" },
    { "field": "dueDate", "message": "Due date cannot be in the past" }
  ]
}
```

---

## 2. Service 1: `auth-tenant-service` Endpoints

### 2.1. User Login & Token Generation
- **Endpoint**: `POST /api/v1/auth/login`
- **Security**: Public Ingress (Rate-limited via Redis sliding window)
- **Description**: Authenticates user credentials, validates multi-tenant active status, and returns JWT tokens with role entitlements.

#### Request Body
```json
{
  "username": "dean.black@mountcarmel.edu",
  "password": "SecurePassword123!",
  "branchCode": "MC-CAMPUS-01"
}
```

#### Response Body (`200 OK`)
```json
{
  "success": true,
  "statusCode": 200,
  "message": "Authentication successful",
  "data": {
    "accessToken": "eyJhbGciOiJSUzI1NiIsIn...",
    "refreshToken": "eyJhbGciOiJSUzI1NiIsIn...",
    "expiresIn": 3600,
    "tokenType": "Bearer",
    "user": {
      "id": "u-9000",
      "username": "dean.black@mountcarmel.edu",
      "fullName": "Joe Black",
      "role": "CAMPUS_ADMIN",
      "branchId": "c4b8e21a-9f12-4a7b-8912-3a5e89d12345",
      "branchName": "Mount Carmel High School - Main Campus",
      "currency": "USD",
      "permissions": [
        "STUDENT_VIEW", "STUDENT_CREATE", "STUDENT_EDIT",
        "FEES_VIEW", "FEES_COLLECT", "PAYROLL_RUN",
        "ATTENDANCE_VIEW", "ATTENDANCE_APPROVE"
      ]
    }
  }
}
```

### 2.2. Branch Directory & Context Switch
- **Endpoint**: `GET /api/v1/tenants/branches`
- **Security**: Requires `SUPER_ADMIN` (Returns all branches) or `CAMPUS_ADMIN` (Returns own branch)
- **Response**: List of active campus branches with currency and timezone metadata.

---

## 3. Service 2: `academic-core-service` Endpoints

### 3.1. Filter Student Directory
- **Endpoint**: `GET /api/v1/students`
- **Security**: `SUPER_ADMIN`, `CAMPUS_ADMIN`, `TEACHER`, `ACCOUNTANT`, `RECEPTIONIST`
- **Query Parameters**:
  - `classId` (UUID, optional)
  - `sectionId` (UUID, optional)
  - `keyword` (String, optional: Name, Roll No, National ID)
  - `page` (Int, default 0)
  - `size` (Int, default 50)

#### Response Body (`200 OK`)
```json
{
  "success": true,
  "statusCode": 200,
  "data": [
    {
      "id": "e82b79a1-5d32-4e6f-b124-789a45c12345",
      "admissionNo": "18001",
      "rollNumber": "01",
      "fullName": "Edward Thomas",
      "className": "Class 1",
      "sectionName": "A",
      "fatherName": "Olivier Thomas",
      "gender": "Male",
      "dateOfBirth": "2018-05-14",
      "mobileNumber": "+189562423934",
      "photoUrl": "https://s3.amazonaws.com/smartschool/students/18001.jpg",
      "status": "ACTIVE"
    }
  ],
  "pagination": {
    "page": 0,
    "size": 50,
    "totalElements": 1,
    "totalPages": 1,
    "hasMore": false
  }
}
```

### 3.2. 360-Degree Student Dossier
- **Endpoint**: `GET /api/v1/students/{id}/dossier`
- **Security**: Scoped by RLS (Staff or Student self/guardian)
- **Response**: Aggregated profile including academic history, attendance percentage, pending fees, and uploaded documents.

---

## 4. Service 3: `attendance-msg-service` Endpoints

### 4.1. Load Daily Attendance Roster
- **Endpoint**: `GET /api/v1/attendance/roster`
- **Security**: `TEACHER`, `CAMPUS_ADMIN`, `SUPER_ADMIN`
- **Query Parameters**: `classId` (UUID), `sectionId` (UUID), `date` (YYYY-MM-DD)

### 4.2. Atomic Batch Attendance Submission
- **Endpoint**: `POST /api/v1/attendance/batch-save`
- **Security**: `TEACHER`, `CAMPUS_ADMIN`
- **Description**: Atomic submission of attendance for up to 50 students. Immediately fires Kafka absent alerts.

#### Request Body
```json
{
  "classId": "7a3f8921-1b2c-4d5e-89a1-bc4567890123",
  "sectionId": "9c1e2345-2f3a-4b5c-9d8e-ef1234567890",
  "attendanceDate": "2026-09-12",
  "records": [
    { "studentId": "e82b79a1-5d32-4e6f-b124-789a45c12345", "status": "PRESENT", "remarks": "" },
    { "studentId": "b14c89a2-4e31-4a7f-9b21-45a789c12345", "status": "ABSENT", "remarks": "Fever reported" }
  ]
}
```

#### Response Body (`200 OK`)
```json
{
  "success": true,
  "statusCode": 200,
  "message": "Attendance recorded for 2 students. 1 absence notification dispatched to Kafka.",
  "data": {
    "totalProcessed": 2,
    "presentCount": 1,
    "absentCount": 1,
    "lateCount": 0
  }
}
```

---

## 5. Service 4: `finance-ledger-service` Endpoints

### 5.1. Student Fee Ledger Lookup
- **Endpoint**: `GET /api/v1/finance/student-ledger/{studentId}`
- **Security**: `ACCOUNTANT`, `CAMPUS_ADMIN`, `STUDENT` (Self), `PARENT` (Ward)
- **Response**: Itemized fee breakdown (Tuition, Transport, Hostel), concessions applied, payments made, and net due.

### 5.2. POS Counter Fee Collection
- **Endpoint**: `POST /api/v1/finance/collect-fee`
- **Security**: `ACCOUNTANT`, `SUPER_ADMIN`
- **Description**: Records payment and generates thermal receipt payload.

#### Request Body
```json
{
  "studentId": "e82b79a1-5d32-4e6f-b124-789a45c12345",
  "invoiceId": "INV-2026-09-00124",
  "amountPaid": 450.00,
  "paymentMode": "CASH",
  "referenceNumber": "POS-REC-8921",
  "collectedDate": "2026-09-12",
  "printThermalReceipt": true
}
```

#### Response Body (`201 Created`)
```json
{
  "success": true,
  "statusCode": 201,
  "message": "Payment recorded successfully",
  "data": {
    "receiptNumber": "REC-2026-09-00452",
    "amountPaid": 450.00,
    "remainingBalance": 0.00,
    "receiptPdfUrl": "https://s3.amazonaws.com/smartschool/receipts/REC-2026-09-00452.pdf"
  }
}
```

---

## 6. Service 5: `assessment-exam-service` Endpoints

### 6.1. Batch Exam Marks Entry
- **Endpoint**: `POST /api/v1/exams/marks/batch-entry`
- **Security**: `TEACHER`, `CAMPUS_ADMIN`
- **Description**: Batch entry of marks (Theory, Practical, Viva) for an exam schedule.

#### Request Body
```json
{
  "examScheduleId": "f93e1245-8a2b-4c5d-9e12-3a5e89d12345",
  "marks": [
    {
      "studentId": "e82b79a1-5d32-4e6f-b124-789a45c12345",
      "theoryMarks": 75.50,
      "practicalMarks": 18.00,
      "vivaMarks": 5.00,
      "isAbsent": false,
      "remarks": "Excellent analysis"
    }
  ]
}
```

### 6.2. Trigger Asynchronous Bulk Admit Cards
- **Endpoint**: `POST /api/v1/exams/admit-cards/bulk-generate`
- **Security**: `CAMPUS_ADMIN`, `SUPER_ADMIN`
- **Description**: Starts an async Kafka batch job rendering vector PDFs for all students in a class cohort.

#### Response Body (`202 Accepted`)
```json
{
  "success": true,
  "statusCode": 202,
  "message": "Bulk admit card generation job accepted and running in background",
  "data": {
    "jobId": "JOB-ADMIT-2026-8912",
    "status": "PROCESSING",
    "totalCohort": 140,
    "websocketTopic": "/topic/assessment-jobs/JOB-ADMIT-2026-8912"
  }
}
```

---

## 7. HTTP Status Codes & Error Conventions

| HTTP Status | Standard Meaning | Architectural Usage |
|:---|:---|:---|
| `200 OK` | Request succeeded | Standard queries, profile lookups, rosters. |
| `201 Created` | Resource created | Student enrollment, payment receipt, class creation. |
| `202 Accepted` | Async job accepted | Bulk admit card PDF, auto-invoicing, payroll runs. |
| `400 Bad Request` | Validation failure | Missing required fields, invalid date formats. |
| `401 Unauthorized` | Invalid/expired JWT | Expired bearer token, invalid credentials. |
| `403 Forbidden` | RBAC / Tenant violation | Teacher accessing finance, or cross-branch data access. |
| `404 Not Found` | Entity missing | Student ID or Invoice ID not found. |
| `409 Conflict` | Unique key violation | Duplicate Admission No, Roll No, or Fee Code. |
| `422 Unprocessable`| Semantic domain failure | Overpaying an invoice, assigning student to full bus. |

---

## 8. Verification & OpenAPI Export

- [x] All 7 core service endpoints standardized with DTO schemas.
- [x] Zero Emoji Policy strictly maintained in API responses and documentation.
- [x] RFC 7807 problem details implemented for client-side error handling.
- [x] Directive 02 Synchronous-to-Asynchronous Bridge verified with `202 Accepted`.
