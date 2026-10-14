---
name: openspec-workflow
description: Create, validate, apply and archive an OpenSpec change proposal (spec-driven development). Use when the request mentions proposal, change, spec, plan, or introduces a new capability, a breaking change or an architecture shift. For a plain bug fix that restores specified behaviour, fix it directly instead.
disable-model-invocation: true
argument-hint: describe the change to propose
---
# OPENSPEC WORKFLOW

Spec-driven development over `openspec/`. Specs are the truth of what IS built; changes are proposals
for what SHOULD change. Read `openspec/AGENTS.md` for the full format before writing anything.

## 1. Explore before proposing

```bash
openspec list                # active changes
openspec list --specs        # existing capabilities
```

Read `openspec/project.md`, then only the specs the change touches (the map is in `AGENTS.md`, section
*Spec-Driven Development* — read the row, not every file). Check `openspec/changes/` for a proposal
that already covers the same ground.

## 2. Decide: proposal or direct fix

Propose when the change adds a capability, breaks something, moves architecture, or changes
behaviour. Skip the proposal for a bug that restores what the specs already say, a typo, formatting, a
comment, or tests for existing behaviour.

If the change is a bug against a spec: fix it directly and note the spec in the report. If it is a bug
the specs never covered: that gap is the proposal.

## 3. Scaffold the change

`change-id` is kebab-case and verb-led (`add-`, `update-`, `remove-`, `refactor-`); append `-2` if taken.

```
openspec/changes/<change-id>/
├── proposal.md      ## Why / ## What Changes / ## Impact (affected specs, affected code)
├── tasks.md         ## 1. <group>  + `- [ ] 1.1 ...`
├── design.md        only if a decision needs justifying (cross-cutting, new pattern,
│                    migration or concurrency complexity). Omit otherwise.
└── specs/<capability>/spec.md   one file per affected capability
```

Deltas use `## ADDED|MODIFIED|REMOVED|RENAMED Requirements`. Every requirement needs at least one
`#### Scenario:`. `MODIFIED` replaces the requirement wholesale: paste the whole existing block from
`openspec/specs/<capability>/spec.md` and edit it, or the archiver drops the scenarios you omitted.

## 4. Validate before asking

```bash
openspec validate <change-id> --strict
```

Fix every error and warning. A proposal that does not validate in strict mode is not finished.

## 5. Approval gate

Present the proposal and stop. Do not write implementation code until the user approves it.

## 6. Implement

Work `tasks.md` in order, marking each `- [ ]` as `- [x]` as it lands. Follow `AGENTS.md`, the zone
rules and the skills for whatever you touch. If the plan turns out to be wrong, stop and change the
proposal — do not let the code drift away from it.

## 7. Archive after the merge

```bash
openspec archive <change-id> --yes
```

That folds the deltas into `openspec/specs/`. Check the result stayed within the budget of the
affected spec (~60 lines / ~500 words): archiving **adds** text, and an oversized spec is a signal
that something was duplicated instead of specified. Commit the archive in its own PR.