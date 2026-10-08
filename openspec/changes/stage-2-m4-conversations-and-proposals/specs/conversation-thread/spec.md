## ADDED Requirements

### Requirement: A conversation belongs to exactly one subject

Every `conversations` row SHALL be scoped to one `clients` row, and every message SHALL be reachable only through a conversation the calling subject owns. Another subject's conversation or message SHALL be reported as absent rather than as forbidden.

#### Scenario: Reading another subject's conversation

- **WHEN** a request presents subject A's cookie and asks for a conversation owned by subject B
- **THEN** the response is 404 and no message is returned

#### Scenario: Listing returns only the caller's conversations

- **WHEN** two subjects each hold a conversation and one of them asks for the list
- **THEN** only that subject's conversation is served

### Requirement: Messages are numbered without gaps

A message SHALL carry a `seq` that starts at 1 within its conversation and increases by exactly one, enforced by a unique constraint on `(conversation_id, seq)`. The number SHALL be chosen under a lock on the conversation, so two messages arriving at once cannot skip a number or collide, and neither can be resolved by renumbering afterwards.

#### Scenario: The first message is number 1

- **WHEN** the first message is appended to a conversation
- **THEN** its `seq` is 1

#### Scenario: Two messages arriving at once

- **WHEN** two messages are appended to one conversation at the same time
- **THEN** both are stored and their numbers are consecutive

### Requirement: The applicant can read and extend their own conversation

`GET /api/conversations` SHALL serve the caller's conversations, newest first, and `GET /api/conversations/{id}` SHALL serve one conversation with its messages in order. `POST /api/conversations/{id}/messages` SHALL append a message written by the applicant. A first-time subject SHALL be given a conversation on first write rather than being refused for not having one.

#### Scenario: A conversation starts empty

- **WHEN** a conversation is created and read before anything is said
- **THEN** it is served with no messages

#### Scenario: The applicant's own message is stored and served

- **WHEN** the applicant posts a message
- **THEN** it is stored with `role = user` and served back in order

### Requirement: Every assistant-side message states whether a person or a model wrote it

A message whose role is `assistant` SHALL carry a non-empty `origin` in its metadata, and the interface SHALL show which origin it is. A message with no origin SHALL NOT be presented as having been written by a model.

#### Scenario: An operator-written message reads as written by a person

- **WHEN** the applicant reads a conversation containing an assistant message whose origin is `operator`
- **THEN** the message is labelled as entered by a person

#### Scenario: An unknown origin is not presented as a model's

- **WHEN** an assistant message carries an origin this build does not know
- **THEN** the interface shows that the origin is unknown and does not claim a model wrote it

#### Scenario: An assistant message with no origin is refused

- **WHEN** something tries to store an assistant message with no origin
- **THEN** the write is refused

### Requirement: This change removes nothing

No route in this change SHALL delete a conversation or a message. A delete request against either SHALL be answered with 405.

#### Scenario: A delete request is refused

- **WHEN** a caller sends `DELETE /api/conversations/{id}`
- **THEN** the response is 405 and the conversation and its messages are unchanged
