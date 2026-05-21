# UI Development User Stories — Loan Approval Workflow

Derived from [StoryPlan.md](StoryPlan.md). Scope: **UI development only** (presentation, interaction, client-side validation). Backend contracts, persistence, and auth implementation are out of scope unless noted.

**Status flow:** `Pending → In BA Review → (Sent to Underwriter → Approved / Rejected by Underwriter) | Rejected by BA (eligibility) | Documents Requested (BA only)`

**BA role:** The BA acts as a **gatekeeper**, not a final decision maker. From the review screen the BA can: **Send to Underwriter** (when the application is complete and passes basic eligibility), **Reject** (when the application fails basic eligibility checks), or **Request Documents** (when the application is incomplete). Final approval is always made by the Underwriter.

**Legend:** Priority = High / Medium / Low · Estimate left blank for the team.

---

## Foundation

### US-01 — Role-Based Navigation Shell
- **Priority:** High · **Estimate:** _TBD_
- **User Story:** As a reviewer (BA or Underwriter), I want to land on a role-appropriate dashboard after sign-in so that I only see screens relevant to my role.
- **Acceptance Criteria:**
  - [ ] App shell renders header with current user name and role badge ("BA" or "Underwriter").
  - [ ] Left nav shows role-specific links: BA → "Pending Review"; Underwriter → "Awaiting Decision".
  - [ ] Unauthorized route access redirects to the user's role dashboard.
  - [ ] Sign-out action is visible from the header.
- **UI Notes:** Top app bar + side nav; use role badge color (BA = blue, Underwriter = purple).
- **Dependencies:** None.
- **Out of Scope:** Auth provider integration, token handling.

### US-02 — Shared Loan Application Detail View
- **Priority:** High · **Estimate:** _TBD_
- **User Story:** As a reviewer, I want a consistent application detail view (borrower info + documents) so that I can review the same data regardless of my role.
- **Acceptance Criteria:**
  - [ ] Displays borrower personal details, employment, requested loan amount, and term.
  - [ ] Lists uploaded documents with type, filename, upload date, and a "View" action.
  - [ ] Shows current application status as a badge.
  - [ ] Layout is responsive (desktop + tablet).
- **UI Notes:** Two-column layout (details left, documents right); reuse as a card/panel inside BA and Underwriter screens.
- **Dependencies:** US-01.
- **Out of Scope:** Document preview/PDF viewer implementation (link out is fine).

---

## BA Flow

### US-03 — BA Pending Review Queue
- **Priority:** High · **Estimate:** _TBD_
- **User Story:** As a BA, I want a queue of submitted loan applications pending initial review so that I can pick the next one to work on.
- **Acceptance Criteria:**
  - [ ] Table lists applications with columns: App ID, Applicant Name, Submitted Date, Loan Amount, Status.
  - [ ] Only applications with status `Pending` / `In BA Review` are shown.
  - [ ] Row click opens the BA review screen for that application.
  - [ ] Empty state message renders when the queue is empty.
  - [ ] Loading and error states are visible.
- **UI Notes:** Sortable by Submitted Date (default desc); paginated.
- **Dependencies:** US-01.
- **Out of Scope:** Server-side filtering, advanced search.

### US-04 — BA Review Screen (Gatekeeper Actions)
- **Priority:** High · **Estimate:** _TBD_
- **User Story:** As a BA acting as a gatekeeper, I want to add review comments and either forward a complete application to the Underwriter, reject it for failing basic eligibility, or request additional documents — so that only review-ready applications reach the Underwriter.
- **Acceptance Criteria:**
  - [ ] Embeds the shared application detail view (US-02).
  - [ ] Review comments textarea supports multi-line input and shows a character counter.
  - [ ] Recommendation radio group: ○ Accept | ○ Reject — single-select; **required only when Sending to Underwriter**.
  - [ ] Action buttons visible: **Send to Underwriter**, **Reject**, **Request Documents**.
  - [ ] **Send to Underwriter** requires: recommendation radio selected **and** at least one non-empty comment; inline errors otherwise.
  - [ ] **Reject** is for failed basic eligibility checks; clicking it reveals a mandatory **eligibility-failure reason** textarea; submit disabled until non-empty.
  - [ ] **Request Documents** opens the form defined in US-06.
  - [ ] Form preserves entered comments on validation errors.
- **UI Notes:** Send to Underwriter = primary; Reject = destructive (red); Request Documents = secondary. Place recommendation radios above the action bar; use accessible labels.
- **Dependencies:** US-02, US-12.
- **Out of Scope:** Auto-save / draft persistence; automated eligibility evaluation (BA judges manually).

### US-05 — BA Action Confirmation & Toast (Send to Underwriter / Reject)
- **Priority:** High · **Estimate:** _TBD_
- **User Story:** As a BA, I want a confirmation dialog summarizing my action (forward or reject) before it is committed so that I avoid mistakes.
- **Acceptance Criteria:**
  - [ ] Clicking **Send to Underwriter** opens a modal showing: selected recommendation, comments summary, applicant name.
  - [ ] Clicking **Reject** opens a modal showing: applicant name, eligibility-failure reason, comments summary.
  - [ ] Each modal has **Confirm** and **Cancel** actions.
  - [ ] On confirm of Send to Underwriter: status becomes `Sent to Underwriter`; success toast ("Application forwarded to Underwriter").
  - [ ] On confirm of Reject: status becomes `Rejected` (actor = BA); success toast ("Application rejected").
  - [ ] After success, the BA is returned to the queue (US-03) and the application is removed from their pending list.
  - [ ] On failure, an error toast is shown and the modal stays open.
- **UI Notes:** Send modal uses info/warning accent; Reject modal uses destructive accent. Disable Confirm during in-flight submit.
- **Dependencies:** US-04, US-11.
- **Out of Scope:** Audit log UI.

### US-06 — BA "Request Documents" Form
- **Priority:** Medium · **Estimate:** _TBD_
- **User Story:** As a BA, I want to request additional documents from the applicant so that I can complete my review.
- **Acceptance Criteria:**
  - [ ] "Request Documents" button opens a modal/form.
  - [ ] Form lists common document types as checkboxes (e.g., Pay Stub, Bank Statement, Tax Return).
  - [ ] User can add custom document items via free-text input + "Add" button.
  - [ ] At least one document must be selected/added to submit.
  - [ ] Optional note textarea for the applicant.
  - [ ] Submit shows a success toast and updates application status to `Documents Requested`.
- **UI Notes:** Reuse modal pattern from US-05.
- **Dependencies:** US-04, US-11, US-12.
- **Out of Scope:** Email notification to applicant.

---

## Underwriter Flow

### US-07 — Underwriter Awaiting Decision Queue
- **Priority:** High · **Estimate:** _TBD_
- **User Story:** As an Underwriter, I want a queue of applications forwarded by BA so that I can pick the next one for final decision.
- **Acceptance Criteria:**
  - [ ] Table lists applications with columns: App ID, Applicant Name, BA Recommendation, Forwarded Date, Loan Amount.
  - [ ] Only applications with status `Sent to Underwriter` are shown.
  - [ ] BA recommendation column shows a colored badge (green = Accept, red = Reject).
  - [ ] Row click opens the underwriter decision screen.
  - [ ] Empty, loading, and error states are visible.
- **UI Notes:** Default sort by Forwarded Date desc.
- **Dependencies:** US-01.
- **Out of Scope:** Bulk actions.

### US-08 — Underwriter Decision Screen (BA Context Panel)
- **Priority:** High · **Estimate:** _TBD_
- **User Story:** As an Underwriter, I want to see the BA's recommendation and comments alongside the application so that I can make an informed decision.
- **Acceptance Criteria:**
  - [ ] Embeds the shared application detail view (US-02).
  - [ ] BA recommendation is displayed prominently as a badge: green "Recommended: Accept" or red "Recommended: Reject".
  - [ ] BA comments are shown in a clearly labeled, read-only panel.
  - [ ] Reviewer name (BA) and forwarded timestamp are displayed.
- **UI Notes:** Place BA context panel at the top of the screen above action buttons.
- **Dependencies:** US-02, US-07.
- **Out of Scope:** Edit/override BA comments.

### US-09 — Underwriter Final Decision Actions
- **Priority:** High · **Estimate:** _TBD_
- **User Story:** As an Underwriter, I want to Approve or Reject with a confirmation step so that I make a deliberate final decision.
- **Acceptance Criteria:**
  - [ ] Action buttons visible: **Approve**, **Reject**.
  - [ ] Clicking **Reject** reveals a mandatory rejection reason textarea; submit disabled until non-empty.
  - [ ] Each action opens a confirmation dialog summarizing the decision (and rejection reason if applicable).
  - [ ] On confirm, status updates accordingly: `Approved` / `Rejected`.
  - [ ] Success toast shows on completion; error toast on failure.
  - [ ] Underwriter is returned to the queue (US-07) and the application is removed from their pending list.
- **UI Notes:** Approve = primary button (green); Reject = destructive (red).
- **Dependencies:** US-08, US-11, US-12.
- **Out of Scope:** Multi-step approval workflows beyond a single decision; requesting additional documents at the Underwriter stage (handled only by BA via US-06).

---

## Cross-Cutting

### US-10 — Application Status Timeline
- **Priority:** Medium · **Estimate:** _TBD_
- **User Story:** As a reviewer, I want to see the application's status history so that I understand where it is in the workflow.
- **Acceptance Criteria:**
  - [ ] Status badge reflects current state: `Pending`, `In BA Review`, `Sent to Underwriter`, `Approved`, `Rejected` (by BA or Underwriter), `Documents Requested` (BA-only).
  - [ ] Vertical timeline lists state transitions with timestamp and actor (BA/Underwriter); rejections show which actor rejected and the reason.
  - [ ] Timeline is visible on both BA and Underwriter detail screens.
- **UI Notes:** Use semantic colors consistent with action buttons.
- **Dependencies:** US-02.
- **Out of Scope:** Editing past timeline entries.

### US-11 — Toast Notification System
- **Priority:** High · **Estimate:** _TBD_
- **User Story:** As a reviewer, I want consistent in-app toast notifications so that I get clear feedback after actions.
- **Acceptance Criteria:**
  - [ ] Shared toast component supports `success`, `error`, and `info` variants.
  - [ ] Toasts auto-dismiss after ~4 seconds and can be dismissed manually.
  - [ ] Toasts stack and do not block primary UI.
  - [ ] Accessible: ARIA live region announces messages.
- **UI Notes:** Position top-right; max 3 visible at once.
- **Dependencies:** US-01.
- **Out of Scope:** Persistent notification center.

### US-12 — Form Validation Patterns
- **Priority:** High · **Estimate:** _TBD_
- **User Story:** As a developer, I want a shared validation pattern for mandatory radios and text fields so that BA and Underwriter forms behave consistently.
- **Acceptance Criteria:**
  - [ ] Required radio groups show an inline error when no option selected on submit.
  - [ ] Required textareas show inline error when empty/whitespace on submit.
  - [ ] Submit buttons remain disabled until validation passes OR show errors on click — pattern is consistent across screens.
  - [ ] Error messages use a single accessible style (icon + red text, `aria-describedby`).
- **UI Notes:** Centralize in a shared form-field component.
- **Dependencies:** None.
- **Out of Scope:** Async/server-side validation.

---

## Traceability Matrix

StoryPlan acceptance criteria → User Story coverage.

| StoryPlan AC | Covered by |
|---|---|
| BA sees queue of submitted applications pending initial review | US-03 |
| BA can open an application and view borrower details and documents | US-02, US-04 |
| BA can add or edit review comments (text area) | US-04 |
| BA selects recommendation via radio: Accept / Reject | US-04 |
| Radio selection is mandatory before sending to Underwriter | US-04, US-12 |
| BA acts as gatekeeper (forward / reject / request documents) | US-04 |
| BA action buttons: Send to Underwriter, Reject, Request Documents | US-04 |
| BA Reject requires mandatory eligibility-failure reason | US-04, US-05, US-12 |
| "Send to Underwriter" validates recommendation + at least one comment | US-04, US-05 |
| "Request Documents" opens form to specify needed documents | US-06 |
| Confirmation dialog shows recommendation + comments summary (or rejection reason) | US-05 |
| Success toast displays after forwarding or BA rejection | US-05, US-11 |
| Underwriter sees queue of applications forwarded by BA | US-07 |
| Underwriter views application + BA recommendation badge + comments | US-08 |
| BA recommendation prominently displayed as colored badge | US-07, US-08 |
| Underwriter actions: Approve, Reject | US-09 |
| Reject requires mandatory rejection reason | US-09, US-12 |
| Confirmation dialog before any final action | US-09 |
| Application status updates after action | US-09, US-10 |
| Success/error toast notifications after submission | US-09, US-11 |
| Status flow: Pending → In BA Review → (Sent to Underwriter → Approved/Rejected by UW) \| Rejected by BA (eligibility) \| Documents Requested (BA only) | US-10 |
