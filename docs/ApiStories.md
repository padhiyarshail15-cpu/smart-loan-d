# API Stories — Loan Approval Workflow

Derived from [UserStories.md](UserStories.md). Scope: **Backend API contracts** for the loan approval workflow. These stories define the endpoints, request/response schemas, and acceptance criteria that support the UI user stories.

**Base URL:** `/api/v1`

**Status flow:** `Pending → In BA Review → (Sent to Underwriter → Approved / Rejected by Underwriter) | Rejected by BA (eligibility) | Documents Requested (BA only)`

**Authentication:** All endpoints require a valid JWT token with `role` claim (`BA` or `Underwriter`). Requests without valid auth return 401 Unauthorized. Role-specific access is enforced per endpoint.

**Legend:** Priority = High / Medium / Low · Estimate left blank for the team.

---

## Common Models & Schemas

### Application
```
{
  "application_id": string,
  "customer_id": string,
  "applicant_name": string,
  "applicant_email": string,
  "applicant_mobile": string,
  "loan_type": string,
  "loan_amount": number,
  "loan_tenure": number,
  "status": string (enum: Pending, In BA Review, Sent to Underwriter, Approved, Rejected, Documents Requested),
  "submitted_date": ISO 8601 datetime,
  "created_at": ISO 8601 datetime,
  "updated_at": ISO 8601 datetime
}
```

### Document
```
{
  "document_id": string,
  "application_id": string,
  "document_type": string,
  "file_name": string,
  "file_url": string,
  "uploaded_date": ISO 8601 datetime,
  "size_bytes": number
}
```

### BA Review
```
{
  "review_id": string,
  "application_id": string,
  "ba_user_id": string,
  "ba_name": string,
  "recommendation": string (enum: Accept, Reject),
  "comments": string,
  "review_date": ISO 8601 datetime
}
```

### Status History Entry
```
{
  "history_id": string,
  "application_id": string,
  "actor_id": string,
  "actor_name": string,
  "actor_role": string (enum: BA, Underwriter),
  "action": string,
  "status_from": string,
  "status_to": string,
  "reason": string (optional),
  "timestamp": ISO 8601 datetime
}
```

---

## Foundation

### AS-01 — Get Current User
- **Priority:** High · **Estimate:** _TBD_
- **API Story:** Return authenticated user details including name and role for dashboard initialization.
- **Endpoint:** `GET /auth/user`
- **Authorization:** Required (BA or Underwriter)
- **Request:** (No body)
- **Response (200):**
  ```
  {
    "user_id": string,
    "name": string,
    "role": string (enum: BA, Underwriter),
    "email": string
  }
  ```
- **Error Responses:**
  - 401: Invalid or expired token
  - 500: Server error
- **Acceptance Criteria:**
  - [ ] Endpoint returns 200 with valid user object when authenticated.
  - [ ] User role is correctly set (BA or Underwriter).
  - [ ] Returns 401 when token is missing or invalid.
  - [ ] Response includes user_id, name, role, and email.
- **Related User Stories:** US-01
- **Out of Scope:** Auth provider integration, token issuance.

---

## Application Queue & Details

### AS-02 — Get Application List (Filtered by Role & Status)
- **Priority:** High · **Estimate:** _TBD_
- **API Story:** Return paginated list of applications with role-based and status-based filtering so that BAs see Pending/In BA Review and Underwriters see Sent to Underwriter applications.
- **Endpoint:** `GET /applications?status=Pending&limit=20&offset=0`
- **Authorization:** Required (BA or Underwriter)
- **Query Parameters:**
  - `status` (optional): Application status filter
  - `limit` (optional, default=20): Page size
  - `offset` (optional, default=0): Pagination offset
- **Request:** (No body)
- **Response (200):**
  ```
  {
    "data": [Application, Application, ...],
    "total": number,
    "limit": number,
    "offset": number
  }
  ```
- **Error Responses:**
  - 400: Invalid query parameters
  - 401: Unauthorized
  - 500: Server error
- **Acceptance Criteria:**
  - [ ] BA role sees applications with status `Pending` or `In BA Review` by default.
  - [ ] Underwriter role sees applications with status `Sent to Underwriter` by default.
  - [ ] Pagination works correctly (limit, offset).
  - [ ] Total count returned for calculating page count.
  - [ ] Returns empty data array when no applications match filter.
  - [ ] Returns 400 if limit or offset are invalid.
- **Related User Stories:** US-03, US-07
- **Out of Scope:** Advanced search, sorting (handled client-side).

### AS-03 — Get Application Details with Documents
- **Priority:** High · **Estimate:** _TBD_
- **API Story:** Return full application details including borrower info, loan details, and list of uploaded documents so reviewers can assess the application.
- **Endpoint:** `GET /applications/:id`
- **Authorization:** Required (BA or Underwriter)
- **Request:** (No body)
- **Response (200):**
  ```
  {
    "application": Application,
    "documents": [Document, Document, ...],
    "ba_review": BAReview (if exists),
    "history": [StatusHistoryEntry, ...]
  }
  ```
- **Error Responses:**
  - 401: Unauthorized
  - 403: Forbidden (user's role cannot access this application)
  - 404: Application not found
  - 500: Server error
- **Acceptance Criteria:**
  - [ ] Returns full Application object with all required fields.
  - [ ] Returns array of Documents with document_type, file_name, uploaded_date.
  - [ ] Includes BA review details if present (recommendation, comments, reviewer name, review date).
  - [ ] Includes status history timeline.
  - [ ] Returns 404 if application_id does not exist.
  - [ ] Returns 403 if BA tries to access application not in their queue.
- **Related User Stories:** US-02, US-04, US-08
- **Out of Scope:** Document file streaming (assumed separate endpoint).

---

## BA Review Actions

### AS-04 — BA Send Application to Underwriter
- **Priority:** High · **Estimate:** _TBD_
- **API Story:** Allow BA to forward a complete application to Underwriter with recommendation and comments, validating that both fields are provided and application status transitions correctly.
- **Endpoint:** `POST /applications/:id/ba-review/send-to-underwriter`
- **Authorization:** Required (BA only)
- **Request Body:**
  ```
  {
    "recommendation": string (enum: Accept, Reject) — required,
    "comments": string (non-empty, max 2000 chars) — required
  }
  ```
- **Response (200):**
  ```
  {
    "application_id": string,
    "status": "Sent to Underwriter",
    "message": "Application forwarded to Underwriter"
  }
  ```
- **Error Responses:**
  - 400: Validation error (missing recommendation, empty comments, or invalid state)
  - 401: Unauthorized
  - 403: Forbidden (user is not BA)
  - 404: Application not found
  - 409: Conflict (application status not In BA Review)
  - 500: Server error
- **Acceptance Criteria:**
  - [ ] Recommendation field is required (must be Accept or Reject).
  - [ ] Comments field is required and must be non-empty (no whitespace-only strings).
  - [ ] Comments max length is 2000 characters.
  - [ ] Returns 400 if recommendation is missing or invalid.
  - [ ] Returns 400 if comments are empty or exceed max length.
  - [ ] Application status transitions from `In BA Review` → `Sent to Underwriter`.
  - [ ] Status history entry is created with actor = current BA, action = "Sent to Underwriter".
  - [ ] Returns 409 if application status is not `In BA Review` or `Pending`.
  - [ ] Returns 403 if user role is not BA.
- **Related User Stories:** US-04, US-05
- **Out of Scope:** Email notifications to Underwriter.

### AS-05 — BA Reject Application (Eligibility Failure)
- **Priority:** High · **Estimate:** _TBD_
- **API Story:** Allow BA to reject an application for failing basic eligibility checks with mandatory reason, validating input and transitioning status.
- **Endpoint:** `POST /applications/:id/ba-review/reject`
- **Authorization:** Required (BA only)
- **Request Body:**
  ```
  {
    "eligibility_failure_reason": string (non-empty, max 1000 chars) — required,
    "comments": string (optional, max 2000 chars)
  }
  ```
- **Response (200):**
  ```
  {
    "application_id": string,
    "status": "Rejected",
    "rejected_by": "BA",
    "message": "Application rejected"
  }
  ```
- **Error Responses:**
  - 400: Validation error (missing or empty reason)
  - 401: Unauthorized
  - 403: Forbidden (user is not BA)
  - 404: Application not found
  - 409: Conflict (application status invalid)
  - 500: Server error
- **Acceptance Criteria:**
  - [ ] Eligibility failure reason is required and non-empty.
  - [ ] Reason max length is 1000 characters.
  - [ ] Comments field is optional, max 2000 characters if provided.
  - [ ] Returns 400 if reason is missing or empty.
  - [ ] Application status transitions to `Rejected`.
  - [ ] Status history records actor = BA, reason = eligibility_failure_reason.
  - [ ] Returns 409 if application status is not eligible for rejection (e.g., already Rejected or Approved).
  - [ ] Returns 403 if user role is not BA.
- **Related User Stories:** US-04, US-05
- **Out of Scope:** Email notification to applicant.

### AS-06 — BA Request Documents
- **Priority:** Medium · **Estimate:** _TBD_
- **API Story:** Allow BA to request specific documents from applicant with optional note, updating application status to Documents Requested.
- **Endpoint:** `POST /applications/:id/ba-review/request-documents`
- **Authorization:** Required (BA only)
- **Request Body:**
  ```
  {
    "requested_documents": [
      {
        "document_type": string (e.g., "Pay Stub", "Bank Statement", "Tax Return"),
        "is_custom": boolean,
        "custom_label": string (optional, used if is_custom=true)
      }
    ], — required, at least 1 document
    "applicant_note": string (optional, max 1000 chars)
  }
  ```
- **Response (200):**
  ```
  {
    "application_id": string,
    "status": "Documents Requested",
    "message": "Documents requested from applicant"
  }
  ```
- **Error Responses:**
  - 400: Validation error (no documents provided, invalid types)
  - 401: Unauthorized
  - 403: Forbidden (user is not BA)
  - 404: Application not found
  - 409: Conflict (application status invalid)
  - 500: Server error
- **Acceptance Criteria:**
  - [ ] Requested documents array is required with at least 1 item.
  - [ ] Each document has document_type and optional custom_label (if is_custom=true).
  - [ ] Applicant note is optional, max 1000 characters.
  - [ ] Returns 400 if documents array is empty or missing.
  - [ ] Returns 400 if custom_label is missing when is_custom=true.
  - [ ] Application status transitions to `Documents Requested`.
  - [ ] Status history entry records actor = BA, action = "Requested Documents".
  - [ ] Returns 409 if application status is not `In BA Review` or `Pending`.
  - [ ] Returns 403 if user role is not BA.
- **Related User Stories:** US-06
- **Out of Scope:** Email notification to applicant; custom document upload interface.

---

## Underwriter Decision Actions

### AS-07 — Underwriter Approve Application
- **Priority:** High · **Estimate:** _TBD_
- **API Story:** Allow Underwriter to approve an application with optional comments, transitioning status to Approved and recording the decision.
- **Endpoint:** `POST /applications/:id/underwriter-decision/approve`
- **Authorization:** Required (Underwriter only)
- **Request Body:**
  ```
  {
    "decision_comments": string (optional, max 2000 chars)
  }
  ```
- **Response (200):**
  ```
  {
    "application_id": string,
    "status": "Approved",
    "approved_by": string (Underwriter name),
    "message": "Application approved"
  }
  ```
- **Error Responses:**
  - 400: Validation error
  - 401: Unauthorized
  - 403: Forbidden (user is not Underwriter)
  - 404: Application not found
  - 409: Conflict (application status not Sent to Underwriter)
  - 500: Server error
- **Acceptance Criteria:**
  - [ ] Underwriter can approve only applications with status `Sent to Underwriter`.
  - [ ] Decision comments are optional, max 2000 characters if provided.
  - [ ] Application status transitions to `Approved`.
  - [ ] Status history records actor = Underwriter, action = "Approved", timestamp.
  - [ ] Returns 409 if application status is not `Sent to Underwriter`.
  - [ ] Returns 403 if user role is not Underwriter.
  - [ ] Returns 404 if application does not exist.
- **Related User Stories:** US-09
- **Out of Scope:** Email notification to applicant; disbursement initiation.

### AS-08 — Underwriter Reject Application
- **Priority:** High · **Estimate:** _TBD_
- **API Story:** Allow Underwriter to reject an application with mandatory rejection reason, transitioning status to Rejected and recording the decision.
- **Endpoint:** `POST /applications/:id/underwriter-decision/reject`
- **Authorization:** Required (Underwriter only)
- **Request Body:**
  ```
  {
    "rejection_reason": string (non-empty, max 1000 chars) — required,
    "decision_comments": string (optional, max 2000 chars)
  }
  ```
- **Response (200):**
  ```
  {
    "application_id": string,
    "status": "Rejected",
    "rejected_by": "Underwriter",
    "message": "Application rejected"
  }
  ```
- **Error Responses:**
  - 400: Validation error (missing or empty rejection_reason)
  - 401: Unauthorized
  - 403: Forbidden (user is not Underwriter)
  - 404: Application not found
  - 409: Conflict (application status not Sent to Underwriter)
  - 500: Server error
- **Acceptance Criteria:**
  - [ ] Rejection reason is required and non-empty (no whitespace-only strings).
  - [ ] Reason max length is 1000 characters.
  - [ ] Decision comments are optional, max 2000 characters if provided.
  - [ ] Returns 400 if rejection_reason is missing or empty.
  - [ ] Application status transitions to `Rejected`.
  - [ ] Status history records actor = Underwriter, action = "Rejected", reason = rejection_reason.
  - [ ] Returns 409 if application status is not `Sent to Underwriter`.
  - [ ] Returns 403 if user role is not Underwriter.
  - [ ] Returns 404 if application does not exist.
- **Related User Stories:** US-09
- **Out of Scope:** Email notification to applicant; reason categorization.

---

## Application Status & History

### AS-09 — Get Application Status History
- **Priority:** Medium · **Estimate:** _TBD_
- **API Story:** Return complete status history/timeline for an application showing all state transitions, actor, timestamps, and reasons for visibility into the workflow progression.
- **Endpoint:** `GET /applications/:id/history`
- **Authorization:** Required (BA or Underwriter)
- **Request:** (No body)
- **Response (200):**
  ```
  {
    "application_id": string,
    "history": [
      {
        "history_id": string,
        "actor_name": string,
        "actor_role": string (BA or Underwriter),
        "action": string,
        "status_from": string,
        "status_to": string,
        "reason": string (optional),
        "timestamp": ISO 8601 datetime
      }
    ]
  }
  ```
- **Error Responses:**
  - 401: Unauthorized
  - 403: Forbidden (user's role cannot access this application)
  - 404: Application not found
  - 500: Server error
- **Acceptance Criteria:**
  - [ ] Returns complete history array ordered by timestamp (oldest first).
  - [ ] Each entry includes actor name, role, action, status transitions, reason (if applicable).
  - [ ] Rejection reasons are included in history for all rejections (BA and Underwriter).
  - [ ] Returns 404 if application does not exist.
  - [ ] Returns 403 if user is not authorized to view this application.
  - [ ] Empty history array if no transitions have occurred (initial Pending state).
- **Related User Stories:** US-10
- **Out of Scope:** Editing or deleting history entries.

---

## Cross-Cutting Concerns

### AS-10 — Error Handling & Validation
- **Priority:** High · **Estimate:** _TBD_
- **API Story:** All endpoints validate input consistently and return appropriate HTTP status codes with descriptive error messages.
- **Acceptance Criteria:**
  - [ ] All text inputs validated for length (max limits enforced per field).
  - [ ] Enum fields validated (status, role, recommendation, etc.).
  - [ ] Required fields return 400 with field name and reason when missing.
  - [ ] All error responses include a message field for client display.
  - [ ] 401 returned for missing/invalid auth token.
  - [ ] 403 returned for role-based access violations.
  - [ ] 404 returned for missing resources.
  - [ ] 409 returned for invalid state transitions (e.g., rejecting already-approved app).
  - [ ] 500 returned for unhandled server errors with generic message (details in logs).
- **Related User Stories:** US-04, US-05, US-09, US-12
- **Out of Scope:** Rate limiting, request logging.

### AS-11 — Idempotency & Duplicate Prevention
- **Priority:** Medium · **Estimate:** _TBD_
- **API Story:** Ensure that repeated submissions of the same action (e.g., double-click Send to Underwriter) do not cause duplicate state transitions.
- **Acceptance Criteria:**
  - [ ] Mutating endpoints (POST actions) are idempotent: second submission of same action returns success without re-applying the change.
  - [ ] Client receives consistent response structure whether first or repeated submission.
  - [ ] Status history should not record duplicate entries for the same action.
- **Related User Stories:** US-05, US-09
- **Out of Scope:** Explicit idempotency key handling (handled via status checks).

---

## Traceability Matrix

User Story → API Story coverage.

| User Story | API Stories |
|---|---|
| US-01: Role-Based Navigation | AS-01 |
| US-02: Shared Application Detail | AS-03 |
| US-03: BA Pending Review Queue | AS-02 |
| US-04: BA Review Screen | AS-03, AS-04, AS-05, AS-06 |
| US-05: BA Confirmation & Toast | AS-04, AS-05 |
| US-06: BA Request Documents | AS-06 |
| US-07: Underwriter Queue | AS-02, AS-07 |
| US-08: Underwriter Decision Context | AS-03 |
| US-09: Underwriter Final Decision | AS-07, AS-08 |
| US-10: Status Timeline | AS-09 |
| US-11: Toast Notifications | (UI-only, no API required) |
| US-12: Form Validation | AS-10 |
