# Next Implementation Idea: Hybrid Prompt System

## Goal

Use a hybrid prompt system with:

1. **Base prompts as versioned files** in `.todos/roles/<role>.md`
2. **Small flow-specific supplements** stored in `.todos/harness.flows.yaml`
3. **Runtime composition** of base prompt + supplement + live TD context

This avoids duplicating full prompt files per flow while keeping role behavior flow-aware.

## Core Model

The effective prompt for an agent should be composed from:

1. **Base role prompt**
   - Stable role identity and constraints
   - TD-centric communication rules
   - Generic workflow discipline

2. **Flow-specific supplement**
   - What this role means in the active flow
   - Stage ownership in that flow
   - Flow-specific expectations, bounce targets, and completion criteria

3. **Runtime TD context**
   - Current issue
   - Current stage
   - Handoff message
   - Logs, links, verification evidence

## Important Separation of Concerns

- **Base prompt** = how the role behaves everywhere
- **Flow supplement** = what the role does in this workflow
- **Handoff** = issue-specific work message

Do **not** use handoffs to carry long-lived workflow policy.

## Where Supplements Should Live

Store supplements directly in `.todos/harness.flows.yaml`.

Recommended shape:

- `defaults.orchestrator_prompt`
- `flows.<flow>.role_prompts.<role>`
- optional `flows.<flow>.stages[].prompt` for very specific stage guidance

Supplements should stay short and operational.

## How Agents Should Receive Supplements

### Workers

Best injection point: **when the worker is bound to a specific issue**.

Recommended command behavior:

- `td usage --new-session`
  - loads base role prompt
  - shows inbox / pending handoffs
  - may show a short preview of active-flow guidance

- `td start <id>`
  - resolves issue ownership
  - resolves active flow
  - resolves current stage
  - binds the session to the issue

- `td context <id>`
  - includes the authoritative flow supplement for the current role/stage
  - includes handoff, logs, links, and verification evidence

### Orchestrator

The orchestrator is not bound to a single issue at startup.

Recommended model:

- base prompt from `.todos/roles/orchestrator.md`
- harness-wide supplement from `defaults.orchestrator_prompt`
- per-flow guidance from the active flow definitions in `.todos/harness.flows.yaml`

## Recommended Precedence Order

When composing instructions, later layers should override earlier ones:

1. Base role prompt
2. Harness-wide supplement
3. Flow-specific role supplement
4. Stage-specific supplement
5. Live issue context / handoff

## Why `td context` Is the Best Place

If only one place gets supplement injection first, make it `td context <id>`.

Why:

- by then the issue is known
- the active flow is known
- the current stage is known
- the role-specific supplement can be chosen accurately

This gives agents precise, issue-bound guidance without bloating base prompts.

## Design Rules

- Keep base prompts reusable and stable
- Keep YAML supplements short and flow-specific
- Do not repeat base prompt content inside YAML
- Do not store full derived prompts as canonical files
- Prefer runtime composition over copying prompt text into multiple places

## Minimal First Implementation

1. Keep base prompts in the role files as they are now
2. Keep orchestrator supplements in YAML as already implemented
3. Add worker-role supplements in YAML only where needed
4. Extend `td context <id>` to include the active supplement block for the current role
5. Later, optionally extend `td usage --new-session` to preview the active-flow guidance

## Sparse Supplement Matrix

Not every role needs a supplement in every flow.

Example:

- `solo-implement`: orchestrator, implementer
- `implement-and-review`: orchestrator, implementer, reviewer
- `implement-review-test`: orchestrator, implementer, reviewer, tester
- `epic-development`: orchestrator, architect, critic
- `epic_work_item_execution`: orchestrator, implementer, reviewer, tester

## Expected Outcome

This approach gives:

- one role identity per agent type
- flow-aware behavior without prompt duplication
- cleaner versioning for base prompts
- smaller and more maintainable YAML supplements
- better alignment between TD state, harness flows, and injected agent instructions
