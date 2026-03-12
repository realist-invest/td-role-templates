# Epic Development Workflow — Implementation Plan

## Goal

Implement a multi-agent epic development workflow using the TD harness and Sidecar
Agent-Driven Mode. The workflow takes pre-defined epics through a full cycle:
PRD + architecture → implementation plan → staged implementation → staged tests → documentation.

This plan targets AI coding agents working in the `td-role-templates` repo
(`github.com/realist-invest/td-role-templates`), which is the public registry that
`td role seed` fetches from.

## Workflow Summary

```
Pre-conditions (given):
  - High-level project scope defined
  - Modules/epics listed in descending priority order
  - High-level project architecture document exists

Per-epic loop (harness-managed per TD issue):
  1. epic-planning   (architect)   → PRD + domain architecture
  2. architecture-review (critic)  → challenge + bounce until approved
  3. plan-creation   (architect)   → staged implementation plan
  4. plan-review     (critic)      → challenge + bounce until approved
  5. implementation  (implementer) → stage-by-stage coding
  6. impl-review     (reviewer)    → challenge per stage + bounce until all pass
  7. testing         (tester)      → stage-by-stage tests
  8. test-review     (reviewer)    → challenge per stage + bounce until all pass
  9. documentation   (architect)   → align architecture, write guidelines, update docs

→ Repeat for next epic (Orchestrator creates next epic issue when current one completes)
```

## Repository Deliverables

| File | Status | Description |
|------|--------|-------------|
| `harness.flows.example.yaml` | ✅ Created | Full harness flow definition |
| `critic/v1.md` | ✅ Created | Critic role behavioral prompt |
| `orchestrator/v1.md` | ✅ Created | Orchestrator role behavioral prompt |
| `manifest.yaml` | ✅ Updated | Added `critic` and `orchestrator` roles |
| `prompt-and-harness-workflow.md` | ✅ This file | Implementation plan |

---

## Stage 1 — Verify Role Prompt Quality

**Goal:** Ensure the four existing role prompts plus the two new ones are complete and correct
for this workflow.

### Deliverables

**1.1** Verify `architect/v2.md` covers epic-planning responsibilities:
- PRD authorship (`docs/epics/{issue_id}/prd.md`)
- Domain architecture design (`docs/epics/{issue_id}/architecture.md`)
- Implementation plan creation (`docs/epics/{issue_id}/implement-plan.md`)
- Documentation stage: align architecture.md to actual implementation, write guidelines.md,
  update existing repo docs
- Harness-aware workflow: `td usage --new-session` → `td start` → `td context` →
  produce artifacts → `td harness advance --to <next>`

**1.2** Verify `implementer/v1.md` covers stage-by-stage implementation:
- Read `docs/epics/{issue_id}/implement-plan.md` at session start
- Read exec-plan to find the current incomplete stage
- Implement only the current stage — do not skip ahead
- Mark stage complete in exec-plan before handoff
- Run build: `td verify run <id> --kind build -- <build-command>`
- Link changed files: `td link <id> <file> --role implementation`
- Advance: `td harness advance <id> --to impl-review`

**1.3** Verify `reviewer/v1.md` covers multi-stage review loop:
- Read exec-plan to check which stage was just implemented
- Check completeness of that stage only — not all stages at once
- If stage passes: check if more stages remain in exec-plan
  - If more stages remain: bounce (`td harness advance <id> --to implementation`)
  - If all stages done: advance (`td harness advance <id> --to testing`)
- If stage fails: bounce with specific feedback

**1.4** Verify `tester/v1.md` covers stage-by-stage test authorship:
- Read exec-plan to find the current untested stage
- Write tests for that stage
- Run: `td verify run <id> --kind test -- <test-command>`
- Advance: `td harness advance <id> --to test-review`

**1.5** Verify `critic/v1.md` (new) is complete — review against the criteria in Stage 2.

**1.6** Verify `orchestrator/v1.md` (new) is complete — review per-cycle protocol.

### Acceptance Criteria

- [ ] Each role prompt references harness commands (`td harness advance`, `td usage --new-session`)
- [ ] Implementer and tester prompts reference exec-plan for stage tracking
- [ ] Reviewer prompt handles both "stage done, more remain → bounce" and "all stages done → advance"
- [ ] Critic prompt uses `td review approve/reject` + `td harness advance`
- [ ] All prompts include structured handoff guidance (--done/--remaining/--decision/--uncertain)
- [ ] All prompts are under 4 KB (TD hard limit)

---

## Stage 2 — Update Role Prompts Where Needed

**Goal:** Edit any role prompts that fail Stage 1 acceptance criteria.

### If `architect/v2.md` needs updates

Add a **Documentation Stage** section describing the `documentation` harness stage:

```markdown
## Documentation Stage (terminal)

When in the `documentation` harness stage:
1. `td context <issue-id>` — review full implementation history
2. Read `docs/epics/{issue_id}/architecture.md` — compare to actual implementation
3. Update architecture.md to reflect what was actually built (not target design)
4. Write `docs/epics/{issue_id}/guidelines.md`:
   - Key design decisions made during implementation
   - Patterns introduced and when to use them
   - Known limitations and future work
5. Search for existing repo docs affected by this epic's changes and update them
6. `td log --decision "Architecture aligned to implementation: [key deltas]"`
7. `td harness advance <issue-id> --to documentation` is the terminal stage —
   when gates pass, the harness marks the run complete automatically
```

### If `implementer/v1.md` needs updates

Add exec-plan stage-tracking section and harness-specific commands.

### If `reviewer/v1.md` needs updates

Add multi-stage loop logic:
```markdown
## Multi-Stage Review Loop

The implementation plan has N stages. You see one stage at a time.

After reviewing a stage:
1. Check exec-plan: `td exec-plan show <issue-id>`
2. If current stage fails quality bar:
   - `td review reject <id> --note "Stage [N]: [specific issue]"`
   - `td harness advance <id> --to implementation`  ← bounce
3. If current stage passes AND more stages remain incomplete:
   - `td review approve <id> --note "Stage [N] approved. Stage [N+1] next."`
   - `td harness advance <id> --to implementation`  ← bounce to continue
4. If current stage passes AND all stages are complete:
   - `td review approve <id> --note "All [N] stages approved. Ready for testing."`
   - `td harness advance <id> --to testing`  ← advance
```

### Acceptance Criteria

- [ ] All six role prompts pass Stage 1 criteria after any edits
- [ ] `git diff` shows only targeted changes to affected files
- [ ] Prompt sizes: `wc -c <role>/v1.md` — all under 4096 bytes

---

## Stage 3 — Validate harness.flows.example.yaml

**Goal:** Confirm the flow file is syntactically correct and semantically sound.

### Deliverables

**3.1** Validate YAML syntax:
```bash
python3 -c "import yaml; yaml.safe_load(open('harness.flows.example.yaml'))" && echo OK
```

**3.2** Check against harness spec rules:
- [ ] `version: 1` present
- [ ] `defaults.orchestrator_agent_type` is set
- [ ] `defaults.escalation_role` is set
- [ ] All stages have `id`, `role`, and at least one of `next` or `terminal: true`
- [ ] All `bounce_back_to` targets exist as stage IDs in the same flow
- [ ] All `next` targets exist as stage IDs in the same flow
- [ ] Terminal stage (`documentation`) has no `bounce_back_to`
- [ ] Terminal stage has `terminal: true`
- [ ] No stage ID appears more than once
- [ ] All `checks_required` values are from the known gate catalog:
  - `decision_log_present`, `exec_plan_present`, `code_changes_linked`
  - `build_passed`, `tests_passed`, `review_submitted`
  - `reviewer_session_not_involved`, `file_exists:<path>`
- [ ] `{issue_id}` template variable is used correctly in artifact paths
- [ ] `max_bounces` on `impl-review` and `test-review` is large enough for
  multi-stage loops (currently 15 — supports up to 5 stages × 3 iterations each)

**3.3** Check role assignments in the flow match supported role names:
- [ ] `architect`, `critic`, `implementer`, `reviewer`, `tester` — all in `manifest.yaml`

**3.4** Review orchestrator_prompt readability:
- [ ] Stage-to-role table is accurate
- [ ] Per-cycle protocol matches sidecar-dev orchestration architecture
- [ ] `{projectSlug}` placeholder documented

### Acceptance Criteria

- [ ] YAML parses without errors
- [ ] All stage graph checks pass
- [ ] Gate names match TD's known gate catalog
- [ ] Bounce limits are sufficient for expected plan sizes (10+ stages)

---

## Stage 4 — Project Setup Documentation

**Goal:** Write a concise setup guide so teams can use this workflow in their projects.

### Deliverables

**4.1** Create `WORKFLOW-SETUP.md` in this repo:

```markdown
# Epic Development Workflow — Project Setup

## 1. Initialize TD in your project
  td init

## 2. Fetch role prompts from this registry
  td role define architect
  td role define critic
  td role define implementer
  td role define reviewer
  td role define tester
  td role define orchestrator

## 3. Assign agent types to roles
  td role assign claude    architect
  td role assign claude    critic
  td role assign codex     implementer
  td role assign claude    reviewer
  td role assign claude    tester
  # orchestrator agent type is set in harness.flows.yaml defaults

## 4. Copy the harness flow definition
  cp .../td-role-templates/harness.flows.example.yaml .todos/harness.flows.yaml
  # Edit as needed (change agent types, adjust iteration limits, add your own flows)

## 5. Create pre-conditions
  # Create the high-level scope, module list, and architecture docs in your repo
  # e.g. docs/scope.md, docs/modules.md, docs/architecture.md

## 6. Create and enroll epics
  td create "Epic: <name>" --type epic --label epic --priority P1
  td start <epic-issue-id>
  # harness auto-selects epic_development flow via selection: issue_types: [epic]

## 7. Activate Sidecar Agent-Driven Mode
  sidecar
  # Press H in harness tab, or ? → Activate Harness
```

**4.2** Document artifact directory convention in `WORKFLOW-SETUP.md`:

```markdown
## Artifact Paths

The flow uses {issue_id} template variables:
  docs/epics/<issue_id>/prd.md           # Epic PRD
  docs/epics/<issue_id>/architecture.md  # Domain architecture
  docs/epics/<issue_id>/implement-plan.md # Staged implementation plan
  docs/epics/<issue_id>/guidelines.md    # Post-implementation guidelines

Agents create these files during their harness stages. The harness gates check
file_exists before allowing stage advancement.
```

### Acceptance Criteria

- [ ] `WORKFLOW-SETUP.md` exists and covers all setup steps
- [ ] Artifact path convention is documented
- [ ] Agent type assignments for all 5 worker roles are shown
- [ ] Sidecar activation steps are accurate

---

## Stage 5 — Manifest and README Update

**Goal:** Finalize the registry so `td role templates` and `td role define` work for new roles.

### Deliverables

**5.1** Verify `manifest.yaml` contains `critic` and `orchestrator` with correct structure:
```yaml
critic:
  description: "Challenge designs and plans before implementation"
  latest: v1
  versions:
    v1: { summary: "Architecture and plan challenger; structured approve/reject workflow" }
orchestrator:
  description: "Coordinate worker agents through TD harness dispatch cycles"
  latest: v1
  versions:
    v1: { summary: "Per-cycle dispatch loop; role-to-session routing via routed handoffs" }
```

**5.2** Verify file paths exist for all manifest entries:
```bash
for role in architect critic implementer orchestrator reviewer tester; do
  echo -n "$role: "
  ls $role/ 2>/dev/null || echo "MISSING"
done
```

**5.3** Update `README.md` — add entries for `critic` and `orchestrator` in the
Repository Structure section.

**5.4** Update `README.md` — add reference to `harness.flows.example.yaml` in the
Repository Structure section.

### Acceptance Criteria

- [ ] `manifest.yaml` has valid YAML with all 6 roles
- [ ] All manifest entries have corresponding `<role>/v1.md` files
- [ ] README mentions all 6 roles and the harness example file
- [ ] `td role templates` (if run against this registry) would list all 6 roles

---

## Completion Criteria

The workflow is fully implemented when:

- [ ] All 6 role prompt files exist and are under 4 KB
- [ ] `harness.flows.example.yaml` passes all validation checks in Stage 3
- [ ] `manifest.yaml` lists all 6 roles with correct file paths
- [ ] `WORKFLOW-SETUP.md` guides teams through end-to-end setup
- [ ] A test epic issue can be created, enrolled in the flow, and dispatched by
  `td harness dispatchable` in a project that has copied the flow file

---

## Role Reference

| Role | File | Stage Ownership |
|------|------|----------------|
| architect | `architect/v2.md` | epic-planning, plan-creation, documentation |
| critic | `critic/v1.md` | architecture-review, plan-review |
| implementer | `implementer/v1.md` | implementation |
| reviewer | `reviewer/v1.md` | impl-review, test-review |
| tester | `tester/v1.md` | testing |
| orchestrator | `orchestrator/v1.md` | Sidecar dispatch coordination (not a harness stage role) |

## Architecture Notes

### How `impl-review` and `test-review` handle multiple plan stages

The harness bounce mechanism drives the multi-stage loops:

```
implementation → impl-review
                 ├─ current stage fails    → bounce → implementation (fix stage)
                 ├─ stage passes, more remain → bounce → implementation (next stage)
                 └─ all stages pass        → advance → testing
```

The exec-plan (`td exec-plan show <id>`) is the ground truth for "which stage is current"
and "are all stages done". The reviewer reads it to determine whether to bounce or advance.
The implementer reads it to determine which stage to work on next.

Max bounces on `impl-review` = 15 supports: 5 stages × (1 pass through + 2 quality bounces) = 15.
Increase `max_bounces_per_edge` in your project's harness.flows.yaml if plans have more stages.

### Why `critic` is a separate role from `reviewer`

- `critic` challenges **designs and plans** before code is written — it is adversarial and
  skeptical by default (reject unless clearly sound)
- `reviewer` challenges **implementation and tests** after code is written — it evaluates
  correctness, completeness, and quality against the plan

Separating them allows different agents to specialize and prevents the same session from
reviewing its own planning outputs.

### How the Orchestrator fits

The Orchestrator is not a harness stage role — it is a Sidecar-level coordination agent.
Its role prompt (`orchestrator/v1.md`) is served from this registry via `td role define orchestrator`
but its behavioral identity is set by the `orchestrator_prompt` field in `harness.flows.yaml`
at a per-flow level. Both are used: the role prompt sets persistent identity, the flow prompt
sets per-flow coordination instructions.
