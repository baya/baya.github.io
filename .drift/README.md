# Drift

A decision-driven development loop for this repository. Drift turns one instruction into a disciplined pipeline of artifacts, then converges on code.

## The loop

rfc -> adr -> spec -> human-views -> do -> verify -> (feedback) -> rfc -> ...

Every stage transition (advance) requires HUMAN review and approval: the artifact produced in the current stage is presented for review, and the pipeline only moves on after the human approves.

## Stage semantics

- RFC = Exploration (search space): what is possible? Explore the options.
- ADR = Decision: which direction do we choose? Commit to one.
- Spec = Constraint (intermediate representation): exactly what must be built.
- Human Views = Projection: the internal decision model rendered as human-readable, reviewable documents.
- DO = Execution: turn the spec into code.
- Verify = Observation: test reality and feed the result back into the next RFC.

RFC + ADR + Spec form the agent-internal decision model. Human Views are the cognitive interface that converts that model into something a human can understand, review, and modify.

## Intelligence mapping

Explore -> Decide -> Constrain -> Project -> Execute -> Observe -> Evolve

## Human views (12)

- prd — PRD (产品视角 Product)
- tech-design — Tech Design (工程视角 Engineering)
- api-doc — API Doc (接口视角 Interface)
- test-plan — Test Plan (验证视角 Verification)
- architecture — Architecture Diagram (结构视角 Structure)
- data-model — Data Model Document
- threat-model — Threat Model / Security Document
- migration — Migration Plan
- runbook — Operations / Runbook
- user-guide — User Guide
- changelog — Change Log / Evolution History
- decision-review — Decision Review Report

## Directory structure

.drift/
  README.md      (this methodology)
  drift.json     (ledger / state machine)
  rfc/           (numbered RFC documents)
  adr/           (numbered decision records)
  spec/          (numbered specs)
  human-views/   (one file per view, latest wins)
  do/            (numbered execution records)
  verify/        (numbered verification/feedback records)

## Drift tools

drift_init    start a session (writes this directory + ledger)
drift_record  write an artifact at the current stage
drift_advance  request human review of the current stage, then advance on approval
drift_status  inspect the current state