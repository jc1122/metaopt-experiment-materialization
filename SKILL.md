---
name: metaopt-experiment-materialization
description: "Use when the ml-metaoptimization orchestrator needs to implement an experiment design as code changes. Produces a unified diff patch artifact, immutable artifacts for the batch manifest, and sanity verification notes. Keywords: materialization, code changes, patch artifact, experiment implementation, metaoptimization worker."
---

# metaopt-experiment-materialization

## Overview

Leaf worker skill for the `ml-metaoptimization` orchestrator's **Materialization Lane**.
This is the only worker that writes actual code.
It turns a designed experiment into concrete code changes in an isolated worktree, producing a unified diff patch artifact, immutable artifacts for the batch manifest, and verification notes for the `LOCAL_SANITY` check.

The orchestrator dispatches this skill as an auxiliary slot with `mode = materialization` and `model_class = strong_coder`.

## Runtime Contract

Model class: `strong_coder` (prefer a strong coding model like Opus 4.6 fast; use for code changes, debugging, and conflict resolution).

The orchestrator validates that `mode = materialization` uses `model_class = strong_coder` before dispatch. This skill must not downgrade the model class.

## Input Contract

The orchestrator provides these inputs via the subagent prompt.

### Standard Envelope (provided by orchestrator on every dispatch)

| Field | Type | Description |
|-------|------|-------------|
| `campaign_id` | string | Campaign identifier |
| `current_iteration` | integer | Current iteration number |
| `slot_id` | string | The slot ID dispatching this worker |
| `attempt` | integer | Attempt number for this dispatch (1-indexed) |
| `materialization_mode` | string | One of `"standard"`, `"remediation"`, or `"conflict_resolution"` |

### Mode-Specific Inputs

**Standard mode** (`materialization_mode: "standard"` — during `MATERIALIZE_CHANGESET`):

| Input | Source | Description |
|-------|--------|-------------|
| Experiment design specification | Design phase output | Concrete experiment spec: what to change, where, and why |
| Campaign config | `ml_metaopt_campaign.yaml` | `code_roots`, `data_roots`, `artifacts.exclude` patterns, `execution` config |
| Project codebase | Worktree access | An isolated git worktree for making changes |
| Baseline implementation | Orchestrator context | Reference implementation for comparison |
| Key learnings | `state.json → key_learnings` | Relevant learnings from prior iterations |

**Remediation mode** (`materialization_mode: "remediation"` — during `LOCAL_SANITY` after diagnosis):

| Input | Source | Description |
|-------|--------|-------------|
| `code_guidance` | Diagnosis output | Targeted code change description from diagnosis `fix_recommendation.code_guidance` |
| Original experiment design | `state.selected_experiment.design` | The original design for context |
| Current local changeset | `state.local_changeset` | The current patch state to build upon |
| Diagnosis history | `state.selected_experiment.diagnosis_history` | Prior diagnosis attempts for this experiment |
| Project codebase | Isolated worktree | Worktree with current patch applied |

**Conflict-resolution mode** (`materialization_mode: "conflict_resolution"` — when mechanical patch integration fails):

| Input | Source | Description |
|-------|--------|-------------|
| Conflicting patches | Orchestrator context | The patches that failed to apply cleanly |
| Base worktree state | Integration worktree | The worktree at the point of conflict |
| Experiment design context | `state.selected_experiment.design` | For intent understanding |
| Key learnings | `state.key_learnings` | Relevant learnings |

All three modes produce the same output shape (unified diff patch artifact + metadata).

The experiment design specification is the authoritative guide for what code changes to make in standard mode. The `code_guidance` is the authoritative guide in remediation mode. If the guidance is ambiguous or infeasible, report the blocker clearly instead of guessing.

## Output Contract

This skill must return all of the following to the orchestrator:

### 1. Unified Diff Patch Artifact

Exactly one unified diff patch artifact containing all code changes. The patch must:
- Apply cleanly to the base worktree (`git apply --check` must succeed)
- Be generated from the isolated worktree, not the orchestrator's working tree
- Contain only the changes specified by the experiment design

### 2. Patch Artifact Metadata

Each patch artifact must record:

| Field | Type | Description |
|-------|------|-------------|
| `producer_slot_id` | string | The slot ID of this materialization worker |
| `purpose` | string | Short description of what the patch does |
| `patch_path` | string | Path to the unified diff file in `.ml-metaopt/artifacts/patches/` |
| `target_worktree` | string | Path to the worktree the patch was generated against |

### 3. Immutable Artifact Inputs

Code and data artifacts ready for the orchestrator to package into the batch manifest:
- Code artifact: the patched codebase, packaged according to `artifacts` config
- Data manifest: references to required data, respecting `artifacts.exclude` patterns

### 4. Sanity Verification Notes

Notes describing what `LOCAL_SANITY` should verify:
- Which files were changed and why
- Expected behavior after applying the patch
- Known edge cases or risks
- Suggested sanity command focus areas

### 5. Change Description

A human-readable summary of what changed and why, suitable for inclusion in the iteration report.

## Patch Artifact Contract

The unified diff patch artifact is the primary deliverable. It follows the contract defined in `ml-metaoptimization/references/worker-lanes.md` (Maintenance Lane → Patch artifact contract) and `ml-metaoptimization/references/contracts.md` (Local Changeset Contract).

### Generation Rules

1. Work exclusively in the isolated worktree provided by the orchestrator
2. Generate the patch with `git diff --no-color` from the worktree
3. Ensure the diff is a proper unified diff format
4. Write the patch file to a path under `.ml-metaopt/artifacts/patches/`
5. The patch filename should encode the iteration and slot ID for traceability

### Integration Path

The orchestrator integrates patches mechanically:
1. Applies the patch to a dedicated integration worktree
2. If application conflicts or requires a non-trivial merge, the orchestrator dispatches a `strong_coder` for conflict resolution
3. The patched worktree is then used for `LOCAL_SANITY` verification

### Local Changeset Fields

When the orchestrator persists the materialization output to `state.json → local_changeset`, the following fields are populated:

- `integration_worktree`: path to the integration worktree
- `patch_artifacts[]`: array of patch artifact metadata objects (see above)
- `apply_results[]`: array of `{ patch_path, status }` objects (set by orchestrator after apply)
- `verification_notes`: the sanity verification notes from this skill
- `code_artifact_uri`: URI of the packaged code artifact (set by orchestrator after packaging)
- `data_manifest_uri`: URI of the data manifest (set by orchestrator after packaging)

## Behavioral Rules

1. **Exactly one patch artifact** — produce exactly one unified diff patch. Do not split changes across multiple patches.

2. **Isolated worktree only** — all code changes must be made in the isolated worktree. Never modify the orchestrator's working tree or any other worktree.

3. **Self-contained and testable** — changes must be self-contained so the project's sanity command can verify them independently.

4. **Respect the design spec** — implement only what the experiment design specifies. Do not add unrelated improvements, refactors, or fixes.

5. **Do not modify test infrastructure or CI** — only modify what the experiment design specifies. Leave test harnesses, CI configs, and build infrastructure untouched.

6. **Respect exclude patterns** — honor the campaign's `artifacts.exclude` patterns when producing artifact inputs. Excluded paths must not appear in packaged artifacts.

7. **Clean patch application** — the patch must apply cleanly to the base worktree. Verify with `git apply --check` before returning.

8. **Report blockers, don't guess** — if the design spec is ambiguous, contradictory, or infeasible given the codebase state, report the blocker clearly with specifics. Do not make assumptions about intent.

9. **No model class downgrade** — this skill runs as `strong_coder`. Do not request or accept a weaker model class.

10. **Traceability** — include the `producer_slot_id` in all output metadata so the orchestrator can trace artifacts back to this worker.

11. **Respect the materialization mode** — in standard mode, implement the experiment design. In remediation mode, apply the diagnosis code_guidance to fix the identified issue. In conflict-resolution mode, resolve the merge conflict while preserving the intent of both patches. Do not mix concerns across modes.

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Modifying files outside the isolated worktree | All changes go in the worktree provided by the orchestrator |
| Producing multiple patch files | Combine all changes into exactly one unified diff |
| Including test infrastructure changes | Only modify what the experiment design specifies |
| Generating a patch that doesn't apply cleanly | Run `git apply --check` against the base before returning |
| Guessing when the design spec is unclear | Report the ambiguity as a blocker with specifics |
| Including excluded paths in artifacts | Filter outputs through `artifacts.exclude` patterns |
| Forgetting `producer_slot_id` in patch metadata | Always populate all four metadata fields |
| Making "bonus" improvements beyond the design | Stick to the experiment spec — extras go through ideation |
| Using the orchestrator's working tree | Always use the isolated worktree |
| Omitting sanity verification notes | Always describe what `LOCAL_SANITY` should check |

## References

- `ml-metaoptimization/references/worker-lanes.md` — Materialization Lane contract and Patch artifact contract
- `ml-metaoptimization/references/contracts.md` — Local Changeset Contract and Patch artifact field definitions
- `ml-metaoptimization/SKILL.md` — Orchestrator contract, dispatch invariants, and behavioral guarantees
