## ADDED Requirements

### Requirement: No run is recorded until a model runs

An `agent_runs` row SHALL mean that a model actually ran. No route and no command in this change SHALL insert one, and completing every flow this change provides SHALL leave the table empty. The table exists so that a later batch can record real runs and so that `document_reviews.run_id` has something to reference.

#### Scenario: Nothing writes a run in this change

- **WHEN** the whole flow is exercised — a conversation, messages, a proposal, its confirmation, and a review
- **THEN** `agent_runs` is still empty

#### Scenario: The table carries what a run has to explain

- **WHEN** the `agent_runs` table is inspected
- **THEN** it holds the fields the parent design names: the message it belongs to, workflow and prompt versions, provider, model, latency, token counts, tool calls, and confidence
