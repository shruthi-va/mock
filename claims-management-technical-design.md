# Claims Management System — Technical Design

## 1. Purpose

This document describes a claims management system that lets claimants report and follow incidents, respond to requests for information, and receive decisions. Claims staff can take ownership of incoming claims, investigate and assess them, progress them to settlement or rejection, and monitor team workload and performance.

### Operating context

ABC Insurance APAC processes motor and property claims across six markets. The system must support both lines of business across all six markets while allowing market-specific requirements to be configured. The names of the markets and their detailed regulatory, operational, and product rules have not yet been provided and must not be assumed.

## 2. Goals and scope

### Goals

- Provide a clear, accessible end-to-end claim journey for claimants.
- Give claims staff a reliable queue and tools to assess and resolve claims.
- Keep a traceable record of claim status changes, decisions, evidence, and communications.
- Give team leads timely operational views of workload and performance.
- Support consistent regional operations while accommodating approved differences by market and line of business.

### In scope

- Claim submission and confirmation.
- Claim status and claimant-facing updates.
- Requests for information and claimant responses.
- Staff work queues, claim assignment, review, assessment, and decision recording.
- Settlement progression and rejection communication.
- Team workload and performance reporting.
- Claim classification and operational reporting by market and line of business.
- Configurable market-specific forms, workflows, communications, and business calendars where required.
- Role-based access, audit history, and notifications.

### Out of scope for this design

- Specific insurance product rules, coverage calculations, or fraud models.
- Payment-provider integration and payment execution details.
- Migration from a legacy claims platform.
- A specific cloud provider, programming language, or user-interface framework.

## 3. Assumptions

- A claimant can submit a claim through a web application and can return later to track it.
- Claimants can authenticate to access their own claims. Account recovery and identity-proofing policy will be defined by the business.
- Claims staff and team leads are authenticated workforce users.
- A claim may have multiple incidents, documents, information requests, and communications over its lifetime.
- Each claim belongs to one of the six configured markets and to either the motor or property line of business.
- Market-specific requirements will be provided and approved by ABC Insurance APAC; this design does not infer market names or legal rules.
- A decision is made by an authorized staff member. Automated eligibility or settlement decisions are not assumed.
- “Performance” means operational measures such as queue size, claim age, throughput, and time to milestones; it does not mean an automated staff ranking.

## 4. Users and permissions

| Role | Capabilities |
|---|---|
| Claimant | Create a claim, view their own claims, upload requested information, view claimant-visible updates and decisions, and receive notifications. |
| Claims handler | View claims assigned to them or available in their permitted queue, claim or assign work according to team policy, record assessments, request information, and progress claims. |
| Team lead | View team queues and aggregate performance, assign or reassign work, and access claim records as permitted by team policy. |
| Administrator | Manage workforce access, teams, reference data, and system configuration. Administrative access does not implicitly grant permission to alter claim decisions. |

Authorization must be enforced by the service for every operation. A claimant must never gain access to another claimant’s record by changing an identifier in a URL or API request. Internal notes and sensitive assessment fields must not be exposed through claimant-facing endpoints.

## 5. End-to-end workflows

### 5.1 Report an incident

1. The claimant starts a claim and enters the incident details and required contact information.
2. The system validates required fields and accepts supporting documents where applicable.
3. The claimant reviews and submits the claim.
4. The system creates a claim reference, records the submission time, sets the initial status to `submitted`, and confirms receipt.
5. The claim appears in the appropriate staff intake queue.

Submission must be idempotent or protected against accidental duplicate requests so that retries do not create duplicate claims.

### 5.2 Staff intake and assessment

1. A handler opens the intake queue and filters or sorts claims by relevant operational fields, such as submission time, priority, or age.
2. The handler takes ownership or assigns the claim according to team policy.
3. The handler reviews incident details and evidence, records an assessment, and adds internal notes as needed.
4. The handler either continues assessment, requests more information, or records a decision.
5. Each state change and material action is recorded in the claim history.

### 5.3 Request and provide additional information

1. The handler creates an information request with a claimant-visible message, requested items, and an optional due date.
2. The claim moves to `awaiting_claimant`.
3. The claimant receives a notification, signs in, reviews the request, and submits a response and any attachments.
4. The system records the response and notifies the assigned handler.
5. The handler reviews the response and resumes assessment. If some information is still missing, the handler may create another request.

The request and response must remain linked and visible in the claim history. The system should clearly distinguish requested information from internal notes.

### 5.4 Record a decision and progress to resolution

1. An authorized handler records a decision of `accepted` or `rejected`, the decision date, rationale, and any required claimant-facing explanation.
2. The system validates that required decision fields are present and that the handler has the necessary permission.
3. The claimant is notified and can view the decision and its explanation.
4. An accepted claim proceeds to `settlement_in_progress` and then `settled` when settlement is confirmed.
5. A rejected claim proceeds to `rejected`. Any review or appeal process is configurable and is not otherwise specified here.

Decision corrections should be represented as a new, audited event or a governed amendment, not as an untraceable overwrite.

## 6. Claim lifecycle

Recommended primary statuses:

| Status | Meaning |
|---|---|
| `draft` | Started but not submitted. |
| `submitted` | Received and waiting for staff intake. |
| `assigned` | Owned by a handler and awaiting or undergoing review. |
| `under_review` | Actively being assessed. |
| `awaiting_claimant` | A request for information is open. |
| `decision_recorded` | A decision is recorded and communication or downstream processing is pending. |
| `settlement_in_progress` | Accepted; settlement actions remain. |
| `settled` | Settlement is complete. |
| `rejected` | Claim is rejected and the claimant has been informed. |
| `closed` | No further work is expected; retained for reference. |

Transitions must be validated on the server. Typical transitions are:

`draft → submitted → assigned → under_review`

`under_review → awaiting_claimant → under_review`

`under_review → decision_recorded → settlement_in_progress → settled`

`under_review → decision_recorded → rejected`

`settled → closed` and `rejected → closed`

Reopening, reassignment, cancellation, and appeal paths should be explicitly configured before implementation. Status is the current summary; the event history is the authoritative account of how the claim reached that status.

## 7. Functional requirements

### Claimant experience

- Submit an incident with validated required information and supporting evidence.
- Receive a durable claim reference and submission confirmation.
- View current status, key dates, claimant-visible messages, open information requests, and decisions.
- Respond to each information request and upload supported file types within configured size limits.
- Receive notifications for submission receipt, information requests, status changes, and decisions.
- Clearly distinguish what the claimant can do next from informational status updates.

### Staff experience

- View an intake queue and team or personal work queues.
- Filter and sort by status, assignee, team, priority, submission date, and age.
- Claim, assign, and reassign work subject to role and team permissions.
- View a consolidated claim record, including incident details, evidence, communications, requests, and event history.
- Record internal notes separately from claimant-visible messages.
- Request additional information, assess evidence, and record or update a decision through an auditable action.
- Progress accepted claims through settlement and record settlement completion.
- Record rejection rationale and claimant-facing explanation.

### Team operations

- Show queue counts by status, assignee, team, market, line of business, and age band.
- Show claim inflow, claims resolved, and time-to-milestone measures for a selected period.
- Support drill-down from aggregate measures to authorized claim lists.
- Display metric definitions, reporting period, and last refresh time.
- Allow authorized regional and market-level views while enforcing team and record-level access.

## 8. Proposed logical architecture

The solution can be implemented as a web application backed by a versioned API and a relational database. Components are logical boundaries and do not require separate deployable services initially.

1. **Claimant portal** — submission, claim tracking, information responses, and decisions, with market-appropriate content and forms.
2. **Staff workspace** — intake queues, claim review, assignment, communications, assessment, and resolution, with market and line-of-business filters.
3. **Claims API** — authentication-aware business operations, validation, authorization, workflow transitions, market-specific configuration, and reporting queries.
4. **Relational data store** — claims and normalized related records, with transactions for important state changes.
5. **Document storage** — encrypted object storage for uploaded evidence; database records retain metadata and access-control associations.
6. **Notification worker** — sends email or other configured notifications from durable notification jobs. Notification failure must not undo a successfully committed claim action.
7. **Reporting queries or read model** — aggregates operational measures without allowing reports to bypass record-level authorization.

The API is the authoritative path for business state changes. The UI must not implement workflow rules as its only enforcement layer.

## 9. Core data model

| Entity | Important fields and relationships |
|---|---|
| `Claim` | `id`, `reference`, `claimant_id`, `market_code`, `line_of_business` (`motor` or `property`), `status`, `priority`, `team_id`, `assignee_id`, `submitted_at`, `updated_at`, `closed_at`, optimistic-lock/version field. |
| `Incident` | `id`, `claim_id`, incident date/time, market-aware location, description, incident category, and product-specific attributes. |
| `Document` | `id`, `claim_id`, optional `request_id`, storage key, original filename, media type, byte size, upload status, uploader, and creation time. |
| `InformationRequest` | `id`, `claim_id`, `created_by`, claimant-visible message, requested items, due date, status, created and responded timestamps. |
| `InformationResponse` | `id`, `request_id`, `submitted_by`, response text, submission time, and associated documents. |
| `Assessment` | `id`, `claim_id`, author, assessment details, outcome recommendation, and creation time. |
| `Decision` | `id`, `claim_id`, decision outcome, rationale, claimant-facing explanation, author, decision time, and optional superseded-decision reference. |
| `Settlement` | `id`, `claim_id`, settlement status, amount/currency if applicable, and relevant milestone timestamps. Payment details are subject to product and finance requirements. |
| `ClaimEvent` | `id`, `claim_id`, event type, actor, timestamp, and structured event data for audit and timeline rendering. |
| `Communication` | `id`, `claim_id`, direction, audience, channel, content or secure reference, sender, and timestamp. |
| `Team` / `User` | Workforce identity, team membership, role, and active state. Authentication credentials should be managed by the identity provider, not stored in these records. |

Use generated opaque identifiers for internal records and a separate human-readable claim reference. Store timestamps consistently in UTC and render them in the user's locale. Keep market configuration and market-specific reference data versioned and auditable; do not encode market rules as scattered application conditionals. Avoid storing duplicate status fields in related records when they can be derived safely.

## 10. API outline

Illustrative REST-style endpoints; exact naming and payload schemas should follow the selected API conventions.

| Method and path | Purpose |
|---|---|
| `POST /api/claims` | Create and submit a claim. |
| `GET /api/claims` | List claims visible to the authenticated user; apply server-side authorization and pagination. |
| `GET /api/claims/{claimId}` | Retrieve a role-filtered claim view. |
| `POST /api/claims/{claimId}/documents` | Register or upload a document using an approved upload flow. |
| `POST /api/claims/{claimId}/assignments` | Assign or reassign a claim. |
| `POST /api/claims/{claimId}/information-requests` | Create a claimant information request. |
| `POST /api/information-requests/{requestId}/responses` | Submit a claimant response and associated documents. |
| `POST /api/claims/{claimId}/assessments` | Add an assessment. |
| `POST /api/claims/{claimId}/decisions` | Record a decision. |
| `POST /api/claims/{claimId}/settlement-events` | Record settlement progress or completion. |
| `GET /api/staff/queues` | Retrieve authorized staff queue summaries and paginated work items, filterable by market and line of business. |
| `GET /api/staff/reports/operations` | Retrieve permitted team-, market-, and line-of-business-level operational measures. |

Write operations should validate state transitions, authorization, and required fields. Use pagination and stable sorting for list endpoints. Use an idempotency key for claim submission and other retried operations where duplicate execution would be harmful. Return consistent validation and authorization errors without leaking whether an unauthorized claimant's record exists.

## 11. Workload and performance measures

Recommended initial measures:

- **Open workload:** count of non-closed claims by team, assignee, and status.
- **Unassigned intake:** submitted claims not yet assigned.
- **Age of open claims:** elapsed time since submission, grouped by configured age bands.
- **Throughput:** claims reaching a defined resolved state during the selected period.
- **Time to first assignment:** submission to first assignment.
- **Time to first staff action:** submission to first substantive staff action.
- **Time to decision:** submission to decision, with paused time during claimant-wait states optionally reported separately.
- **Settlement time:** acceptance decision to settlement completion.

Define business calendars per market, treatment of reopened claims, paused intervals, period boundaries, and exclusions before publishing targets. Show aggregate team measures by default; limit individual-level reporting to authorized operational purposes.

## 12. Security, privacy, and audit

- Authenticate claimants and workforce users through an approved identity system.
- Enforce least-privilege, role-based and record-level authorization on every API operation.
- Separate claimant-visible communications from internal notes at both data and API boundaries.
- Encrypt network traffic and stored data, including uploaded documents and backups.
- Validate file size and type, scan uploads for malware, and serve documents only through short-lived authorized links or an equivalent controlled download mechanism.
- Minimize personal data, define retention and deletion policies, and avoid placing sensitive claim details in logs or notification previews.
- Record an immutable or tamper-evident audit trail for access-sensitive actions and all material claim changes, including actor, time, action, and relevant before/after values.
- Apply rate limits and abuse monitoring to public-facing submission, authentication, and upload endpoints.
- Use secure session handling, CSRF protections where applicable, output encoding, and standard web security controls.
- Restrict administrative operations and audit changes to roles, team membership, and workflow configuration.

Applicable privacy, accessibility, records-retention, and insurance regulations must be confirmed for each of the six markets. Market-specific access boundaries and data-residency requirements must be assessed before choosing deployment topology.

## 13. Reliability and operational requirements

The following are proposed targets to confirm with stakeholders:

- **Availability:** 99.9% monthly availability for claimant and staff core journeys, excluding announced maintenance.
- **Performance:** 95th-percentile response time below 2 seconds for ordinary reads and below 3 seconds for standard writes, excluding document transfer and external notification delivery.
- **Durability:** committed claim submissions and workflow changes must not be lost on application restart.
- **Recovery:** define recovery time and recovery point objectives with the business; back up the database and document store and regularly test restoration.
- **Observability:** structured logs, service metrics, traces for key workflows, and alerts for failed submissions, notification backlog, upload failures, and elevated error rates.
- **Accessibility:** target WCAG 2.2 AA for claimant and staff user interfaces, subject to formal assessment.

## 14. Error handling and consistency

- Validate fields and workflow transitions before committing changes; return field-level errors that users can act on.
- Commit the main claim change and its audit event in one database transaction where possible.
- Use a transactional outbox or equivalent durable job mechanism for notifications triggered by committed changes.
- Treat document upload as a staged operation: only associate a document with a claim after upload validation and storage succeed; expose processing status where scanning is asynchronous.
- Use optimistic concurrency or equivalent protection to prevent one handler from silently overwriting another's updates.
- Make retries safe for operations that clients or workers may repeat.
- Surface actionable errors to users and operationally useful error details to staff, without exposing sensitive data.

## 15. Acceptance criteria

1. A claimant can submit a valid claim, receive a reference, and later see that claim without accessing another claimant's data.
2. A staff handler can find an unassigned claim, take ownership, review it, and see the assignment reflected in team workload.
3. A handler can request information; the claimant can respond and attach evidence; the handler can see that response linked to the request.
4. Internal notes are not returned by claimant-facing views or notifications.
5. An authorized handler can record a decision with required rationale; the claimant can see the decision and appropriate explanation.
6. An accepted claim can be progressed to settlement completion; a rejected claim can be closed under configured policy.
7. Invalid status transitions and unauthorized actions are rejected by the API and recorded where required by audit policy.
8. Team leads can view workload and defined performance measures for their authorized teams and markets, with metric definitions and reporting periods.
9. A notification delivery failure does not lose or roll back the committed claim state change and is visible to operations for retry.
10. Claim events show who did what and when for submission, assignment, information requests and responses, decision, and settlement progression.

## 16. Decisions needed before implementation

- Which claim types, incident fields, evidence types, and validation rules are required?
- What claimant identity proofing and authentication options are acceptable?
- What are the assignment, priority, escalation, and service-level policies?
- Are appeals, reconsiderations, cancellations, or reopening required, and what are their rules?
- Which channels are required for claimant notifications, and what content may those channels include?
- What settlement data and external payment or finance integrations are in scope?
- What are the six markets, and which market-specific claim, regulatory, localization, data-residency, and business-calendar requirements apply?
- Which jurisdictional privacy, accessibility, retention, and audit requirements apply in each market?
- What are the approved availability, performance, recovery, and reporting targets?
