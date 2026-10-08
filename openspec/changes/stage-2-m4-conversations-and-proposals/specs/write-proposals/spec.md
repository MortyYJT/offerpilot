## ADDED Requirements

### Requirement: A proposal belongs to exactly one subject and one conversation

Every `proposals` row SHALL be scoped to one `clients` row and one `conversations` row of that same subject. Another subject's proposal SHALL be reported as absent rather than as forbidden, and a proposal SHALL never be confirmable by a subject that does not own it.

#### Scenario: Reading another subject's proposal

- **WHEN** a request presents subject A's cookie and asks for a proposal owned by subject B
- **THEN** the response is 404 and nothing about the proposal is returned

#### Scenario: Confirming another subject's proposal

- **WHEN** a request presents subject A's cookie and tries to confirm subject B's proposal
- **THEN** the response is 404 and subject B's state is unchanged

### Requirement: A proposal names what it would change and with what

A proposal SHALL carry a `kind` from `update_profile`, `update_roadmap`, `update_application`, `create_task`, `archive_document`, and a `payload` describing the change. A kind outside that set SHALL be refused. The `payload` SHALL NOT carry fields the server owns — `origin`, `status`, ids, timestamps — and a payload that does is refused rather than accepted and quietly ignored.

#### Scenario: An unknown kind is refused

- **WHEN** a proposal is created with a kind outside the five
- **THEN** the write is refused and no proposal is stored

#### Scenario: A payload carrying a server-owned field is refused

- **WHEN** a proposal's payload names `origin` or `status`
- **THEN** the write is refused

#### Scenario: A proposal may name the message that carried it

- **WHEN** a proposal is created against one of the conversation's own messages
- **THEN** the proposal carries that message id and serves it

#### Scenario: A proposal cannot name another conversation's message

- **WHEN** a proposal names a message that belongs to a different conversation
- **THEN** the write is refused and no proposal is stored

### Requirement: The applicant can read their own proposals

`GET /api/proposals` SHALL serve the caller's proposals, newest first, with their kind, risk, status, payload and the conversation they belong to. A resolved proposal SHALL remain readable rather than disappearing: what was proposed and what the applicant decided are both part of the record.

#### Scenario: A resolved proposal is still served

- **WHEN** a proposal has been rejected and the applicant reads their proposals
- **THEN** it is served with the status `rejected`

#### Scenario: The list is the caller's own

- **WHEN** two subjects each hold a proposal
- **THEN** each sees only their own

### Requirement: Confirming a proposal applies it through the service that owns that kind

Confirming SHALL apply the payload by calling the existing service for that kind — the profile write, the roadmap task write, the applications replacement, or the document archive — in one transaction, and SHALL then record the resolution. The proposal machinery SHALL NOT write the target tables itself, so every rule those services enforce keeps applying: ownership, `origin`, the uniqueness constraints, and the audit entries.

#### Scenario: Confirming a portfolio proposal goes through the applications service

- **WHEN** a proposal of kind `update_application` is confirmed
- **THEN** the portfolio is replaced by that service and its refusal rules apply unchanged

#### Scenario: A confirmed roadmap proposal writes its audit entries

- **WHEN** a proposal of kind `create_task` is confirmed
- **THEN** the resulting rows carry the history entry the roadmap service writes for a change

#### Scenario: A payload the owning service refuses leaves the proposal pending

- **WHEN** confirming a proposal whose payload the owning service refuses
- **THEN** the response is a mapped 4xx, the target state is unchanged, and the proposal is still `pending`

### Requirement: A proposal is resolved once

Confirming or rejecting SHALL record who resolved it and when, and SHALL set the status to `confirmed` or `rejected`. A resolution of a proposal that is already resolved SHALL be refused with 409 rather than applied twice or answered as a success.

#### Scenario: Confirming twice is refused

- **WHEN** an already confirmed proposal is confirmed again
- **THEN** the response is 409 and nothing is applied a second time

#### Scenario: Rejecting applies nothing

- **WHEN** the applicant rejects a proposal
- **THEN** the status becomes `rejected`, the target state is unchanged, and the proposal stays readable

### Requirement: A proposal built on state that has since changed expires

A proposal SHALL only be confirmed while the state its kind depends on has not been modified since the proposal was created: `profiles.updated_at` for `update_profile`, the newest `roadmap_tasks.updated_at` for `update_roadmap` and `create_task`, the newest `applications.updated_at` for `update_application`, and `documents.updated_at` for `archive_document`. A confirmation arriving after that state moved SHALL be refused with 409 and SHALL set the status to `expired`, because applying it would be acting on a picture of the applicant that is no longer true.

#### Scenario: A stale proposal cannot be confirmed

- **WHEN** the profile is edited after a profile proposal was created, and that proposal is then confirmed
- **THEN** the response is 409, the profile is unchanged by the proposal, and the proposal is `expired`

#### Scenario: A fresh proposal is unaffected

- **WHEN** a proposal is confirmed and nothing it depends on has changed since
- **THEN** it is applied and becomes `confirmed`

### Requirement: Risk is served and does not change the path

Every proposal SHALL carry a `risk` of `low` or `high` and it SHALL be served. In this change every proposal requires the applicant's confirmation: the low-risk operations named in the parent design already have their own direct routes, so there is no path by which a proposal is applied without a confirmation.

#### Scenario: Every proposal waits for the applicant

- **WHEN** a proposal is created, whatever its risk
- **THEN** it is `pending` and nothing has been applied

### Requirement: This change removes nothing

No route in this change SHALL delete a proposal. A delete request SHALL be answered with 405.

#### Scenario: A delete request is refused

- **WHEN** a caller sends `DELETE /api/proposals/{id}`
- **THEN** the response is 405 and the proposal is unchanged
