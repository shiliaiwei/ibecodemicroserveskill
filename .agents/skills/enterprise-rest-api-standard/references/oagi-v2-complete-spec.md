# OAGi RESTful Web API Design Standard (v2.0) - Comprehensive Specification

## Origin & Governance
* **Organization**: Open Applications Group (OAGi) - REST Working Group
* **Principal Author**: Steffen M. Fohn, Ph.D. (ADP)
* **Contributors**: NIST, Oracle, Land O’Lakes
* **Target Level**: Level 3 of the Richardson Maturity Model (RMM) - Hypermedia as the Engine of Application State (HATEOAS).

---

## 1. Core Architectural Constraints & Postel's Law

### 1.1. Robustness Principle (Postel's Law - Rule R12)
> *"Be conservative in what you send, be liberal in what you accept."*
* **Rule R12**: Clients MUST be designed to ignore data elements that are not recognized.
* This allows servers to introduce new non-breaking optional fields without invalidating existing client decoders.

### 1.2. Backwards Compatibility Rules
An API **maintains** backwards compatibility when:
1. An optional property is added.
2. A new value is added to an enumerated data domain.
3. Metadata of a property is expanded (e.g. reduced minimum size, increased maximum size).
4. An optional operation or query parameter is added.

An API **breaks** backwards compatibility when:
1. A property changes from optional to mandatory.
2. An existing property is removed.
3. A new mandatory property is added.
4. An existing value is removed from an enumeration domain.
5. The metadata of a property is restricted (e.g. changed data type, reduced maximum size).

---

## 2. Resource Identification & URI Format

### 2.1. Structural Syntax (RFC 3986)
```
{scheme}://{host}/{service-domain}/{version}/{resource-path}[?{query-string}]
```
* **Scheme**: `https` strictly enforced (Port 443).
* **Host / Domains**:
  * API Gateway: `api.domain.com` or `domain.com/api`
  * Developer Portal: `developer.domain.com`
* **Service Domain**: Grouping of business capabilities (e.g., `hr`, `finance`, `academic`).
* **API Version**: Major version represented in URI path: `/v1/`, `/v2/`.
* **Path Segments**:
  * Collection Resources: Lowercase plural nouns (`/associates`, `/invoices`).
  * Instance Resources: Singular or UUID identifier (`/associates/121212`).
  * Controller Resources: Procedural operators as lowercase verbs, placed exclusively as the **final segment** (`/associates/121212/hire`, `/currencies/USD/convert`).
* **Path Delimiters & Formatting**:
  * Forward slash `/` indicates hierarchical parent-child relationships.
  * Hyphen `-` separates multi-word path elements (e.g., `emergency-contact`).
  * Underscores `_` are **strictly forbidden** (obscured by hyperlink underlining).
  * Trailing slashes are **strictly forbidden** (`/associates`, not `/associates/`).
  * File extensions (`.json`, `.xml`) are **strictly forbidden**.

---

## 3. Comprehensive HTTP Message Headers Catalog

### 3.1. General Headers
* `Cache-Control`: Directives for caches (`no-cache`, `no-store`, `max-age=<seconds>`, `must-revalidate`).
* `Date`: Origin timestamp in RFC 1123 format (e.g., `Sun, 06 Nov 1994 08:49:37 GMT`).
* `Pragma`: For legacy HTTP 1.0 proxies (`Pragma: no-cache`).

### 3.2. Request Headers
* `Accept`: MIME types with quality values and data masking parameter:
  `Accept: application/json; q=0.9; masked=true, application/xml; q=0.5; masked=false`
  * Default interpretation if `masked` is omitted: `masked=true` (PII protection).
* `Accept-Charset`: Acceptable character sets (e.g., `utf-8`).
* `Accept-Encoding`: Compression encoding (e.g., `gzip, deflate`).
* `Accept-Language`: RFC 5646 language tags (e.g., `en-US, km-KH; q=0.8`).
* `Authorization`: `Bearer <token>` or `Basic <base64>` (over HTTPS only).
* `If-Match`: ETag entity tag for optimistic locking. Returns `412 Precondition Failed` if mismatched.
* `If-None-Match`: ETag validation for caching. Returns `304 Not Modified` if matched.
* `If-Modified-Since` & `If-Unmodified-Since`: RFC 1123 timestamps for cache and update preconditions.
* `Prefer` (RFC 7240): Declares client preferences:
  * `respond-async`: Request asynchronous processing.
  * `wait=<seconds>`: Time client is willing to block synchronously before falling back to async.
  * `return=representation` vs `return=minimal`: Response payload shaping.
  * `/oagi/confirm-message`: Request detailed Confirm Message envelope in response.
  * `/oagi/long-polling`: Request long-polling server push for event delivery.
* `Range` & `If-Range`: Byte range for resumable downloads (`206 Partial Content`).

### 3.3. Response Headers
* `ETag`: Resource state fingerprint/digest (must not be host-specific).
* `Location`: URI of created resource (`201 Created`) or async task result (`303 See Other`).
* `Content-Location`: Direct URL of the returned payload entity body.
* `Retry-After`: Delay in seconds or RFC 1123 datetime before retrying (`202 Accepted` or `503 Service Unavailable`).
* `WWW-Authenticate`: Authentication scheme and realm (`401 Unauthorized`).
* `Allow`: Comma-separated allowed HTTP methods (`OPTIONS` or `405 Method Not Allowed`).

### 3.4. OAGi Custom Headers (Namespace-HeaderName Pattern)
Custom headers communicate operational context without altering HTTP verb semantics:
* `OAGi-MessageID`: Globally unique UUID of the message instance.
* `OAGi-CorrelationID`: Identifier of the originating request being replied to.
* `OAGi-ConversationID`: Identifies a multi-turn business conversation across multiple systems.
* `OAGi-OriginatorID`: System/actor that triggered the request.
* `OAGi-ReferenceID`: Identifier of the specific business workflow task instance.
* `OAGi-ScenarioID`: Business scenario category.
* `OAGi-TaskID`: Business command or event identifier.
* `OAGi-UserID`: User identity for audit logging (non-authentication).
* `OAGi-Allow-CustomOperator`: In `OPTIONS` responses, lists allowed verbs on a controller resource.
* `OAGX-<custom-name>`: Extension pattern for vendor-specific custom headers.

---

## 4. OData 4.0 Query Specification

Collection endpoints support standardized query parameters prefixed with `$`:

### 4.1. Field Selection (`$select`)
Specifies partial projections to reduce network transfer:
```http
GET /hr/v1/associates?$select=personName,address/city,address/postalCode
```
* Delimited by commas `,`.
* Wildcard `*` selects all properties.

### 4.2. Relationship Expansion (`$expand`)
Inlines associated entities in a single round trip with nested query options:
```http
GET /hr/v1/associates?$expand=workAssignments($select=jobTitle,salary;$filter=active eq true)
```

### 4.3. Filter Expressions (`$filter`)
* **Comparison Operators**: `eq`, `ne`, `gt`, `ge`, `lt`, `le`.
* **Logical Operators**: `and`, `or`, `not`.
* **Built-in Functions**: `contains(field, 'value')`.
* **Grouping**: Parentheses `(...)` for operator precedence.
* **Collection Lambda Operators**:
  * `any()`: Returns true if at least one item satisfies condition:
    `$filter=locations/any(loc: loc/cityName eq 'Charlotte')`
  * `all()`: Returns true if all items satisfy condition:
    `$filter=locations/all(loc: loc/status eq 'active')`

### 4.4. Sorting (`$orderby`)
* `property-name [asc|desc]`
* Default is ascending.
* Example: `$orderby=personName/familyName asc,birthDate desc`

### 4.5. Free-Text Search (`$search`)
* Single words, double-quoted phrases `"..."`, and boolean terms:
* Example: `$search="software engineer" AND (Java OR Python)`

---

## 5. Read-Consistent Pagination Architecture

To eliminate pagination drift (missing or duplicate records as rows are inserted or deleted during a client session):

```
Client                                      Server                                  Cache Store
  │                                           │                                          │
  │ 1. GET /items?$top=10                     │                                          │
  │    &resourceSetSaveIndicator=true         │                                          │
  │──────────────────────────────────────────>│                                          │
  │                                           │ 2. Query DB & create frozen snapshot     │
  │                                           │─────────────────────────────────────────>│
  │                                           │    resourceSetID: "SET-9941"             │
  │                                           │<─────────────────────────────────────────│
  │ 3. 200 OK                                 │                                          │
  │    { paginationResponse: {                │                                          │
  │        startSequenceNumber: 1,            │                                          │
  │        returnedNumber: 10,                │                                          │
  │        totalNumber: 25,                   │                                          │
  │        completeIndicator: false,          │                                          │
  │        resourceSetID: "SET-9941" } }      │                                          │
  │<──────────────────────────────────────────│                                          │
  │                                           │                                          │
  │ 4. GET /items?$top=10&$skip=10            │                                          │
  │    &resourceSetID=SET-9941                │                                          │
  │──────────────────────────────────────────>│ 5. Read slice 11..20 from frozen cache   │
  │                                           │─────────────────────────────────────────>│
  │ 6. 200 OK (Consistent Slice)              │<─────────────────────────────────────────│
  │<──────────────────────────────────────────│                                          │
```

---

## 6. Secure Handling of Sensitive & Large Queries

### The Vulnerability:
URLs with sensitive data (SSN, national ID, tax identifier, bank details) leak through:
1. Browser history and autocomplete caches.
2. Proxy, CDN, and load balancer plain-text access logs.
3. HTTP `Referer` headers passed to third-party endpoints.
4. User bookmarks.

### The Solution: Save and Query Resource Set Pattern
1. **Save Query via POST**:
   ```http
   POST /hr/v1/associates/save-resource-set HTTP/1.1
   Host: api.abc.com
   Content-Type: application/x-www-form-urlencoded

   taxIdentifier=987-65-4321&status=active
   ```
2. **Server Responds with Transient Resource Set**:
   ```http
   HTTP/1.1 201 Created
   Location: https://api.abc.com/hr/v1/associates/resource-sets/set-89124
   ETag: "v1-digest"
   ```
3. **Client Retrieves Data Safely**:
   ```http
   GET /hr/v1/associates/resource-sets/set-89124 HTTP/1.1
   Host: api.abc.com
   ```

---

## 7. Hypermedia Controls & HATEOAS Model

### 7.1. The Link Object Schema
Every resource state contains navigational transitions:
* `href`: Target URI template (RFC 6570).
* `rel`: Relation type (`self`, `alternate`, `create`, `describedby`, `edit-form`, `first`, `next`, `previous`, `last`, `search`, `/oagi/invoke`, `/oagi/callback`, `/oagi/processing-status`, `/oagi/request-result`).
* `method`: HTTP verb (`GET`, `POST`, `PUT`, `PATCH`, `DELETE`).
* `targetSchema`: Advisory schema for the response payload.
* `mediaType`: Expected MIME type.

### 7.2. Hypermedia Actions (State-Sensitive Operations)
```json
{
  "actions": [
    {
      "operationCode": "timesheet.approve",
      "confirmationRequiredIndicator": true,
      "commentAllowedIndicator": true,
      "colorCode": "80FF00",
      "links": [
        {
          "rel": "/oagi/invoke",
          "title": "Approve Timesheet",
          "href": "https://api.domain.com/hr/v1/timesheets/123/approve",
          "method": "POST"
        }
      ]
    }
  ]
}
```

---

## 8. Two-Level Confirm Message Envelope Standard

For single and batch operations, provides unified error and progress reporting:

```json
{
  "confirmMessage": {
    "messageID": "urn:uuid:69fe0381-ed80-45ef-b4c7-41e2db362b91",
    "messageDateTime": "2026-09-14T12:00:00Z",
    "requestID": "JOB-BATCH-8921",
    "requestProcessingStatusCode": "completed",
    "requestResultStatusCode": "partiallyFailed",
    "messages": [
      {
        "messageCode": "BATCH_PARTIAL_ERROR",
        "messageTypeCode": "error",
        "message": "1 out of 2 records failed validation."
      }
    ],
    "resourceMessages": [
      {
        "resourceID": "REC-01",
        "resourceResultStatusCode": "failed",
        "messages": [
          {
            "messageCode": "INVALID_POSTAL_CODE",
            "messageTypeCode": "error",
            "message": "Postal code does not exist.",
            "resourcePath": "$.associates[0].address.postalCode"
          }
        ]
      },
      {
        "resourceID": "REC-02",
        "resourceResultStatusCode": "succeeded",
        "messages": [
          {
            "messageCode": "ASSOCIATE_UPDATED",
            "messageTypeCode": "success",
            "message": "Record successfully modified.",
            "resourcePath": "$.associates[1]"
          }
        ]
      }
    ]
  }
}
```

---

## 9. Asynchronous Processing Models

### 9.1. Consumer Polling with `303 See Other`
1. Request: `POST /long-job` with `Prefer: respond-async, wait=5`
2. Server responds: `202 Accepted` + `Link: </jobs/123/status>; rel="/oagi/processing-status"` + `Retry-After: 30`
3. Client polls `/jobs/123/status`:
   * While processing: `200 OK` with `requestProcessingStatusCode: "started"`
   * On completion: `303 See Other` with `Location: /results/123`
4. Client issues `GET /results/123` to fetch final output.

### 9.2. Server Push (Webhook Callback)
* Client passes callback link: `Link: <https://consumer.com/callback>; rel="/oagi/callback"`
* Server returns `202 Accepted`.
* Server executes `POST` to callback with payload and `OAGi-CorrelationID` header.
* Consumer returns `201 Created` or `200 OK`.

### 9.3. Long Polling for Event Notifications
* Client request: `GET /events?topic=orders` with `Prefer: /oagi/long-polling`
* Server suspends socket until an event is ready or timeout occurs.
* Server returns `200 OK` with un-cached payload (`Cache-Control: no-cache, no-store`).
