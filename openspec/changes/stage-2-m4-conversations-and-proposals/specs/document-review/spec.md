## ADDED Requirements

### Requirement: A review may name the run that produced it

`document_reviews` SHALL carry a nullable `run_id` referencing `agent_runs`, so a review written by an automated run can say which run it was. The column SHALL be nullable because a review written by a person has no run behind it, and a review with no run SHALL NOT be given one.

#### Scenario: A human review carries no run

- **WHEN** a review is recorded by the operator command
- **THEN** its `run_id` is null

#### Scenario: A run that does not exist cannot be cited

- **WHEN** a review is stored with a `run_id` no row carries
- **THEN** the database refuses the row
