---
name: memory-management
description: Durable project-memory rules for `.agent-memory/` plus session-memory boundaries.
user-invocable: false
---

# Skill: Memory Management

This skill defines the rules for interacting with the `.agent-memory/` directory. It ensures that the project's long-term "brain" remains consistent, clean, and useful.

---

## 1. Directory Structure

- `.agent-memory/`
  - `INDEX.md`: Lightweight pointer index (~150 chars per line, always loaded in agent context). Maps topic IDs to files. Never stores facts -- only locations.
  - `topics/`: On-demand topic files with full decision/pattern details. Loaded only when needed.
  - `archive/`: Aged or outdated topic files.

### INDEX.md Schema

The index is a routing map, not a knowledge store. It must stay under 200 lines and ~25KB.

Header block (always present at top of INDEX.md):

    ---
    schema_version: 2
    last_consolidated: <ISO date or "never">
    _consolidation_in_progress: false
    ---

Entry format (one per line after the header):

    - <topic-id> | <type: decision | error-pattern | onboarding> | <date: YYYY-MM-DD> | <confidence: high | medium | low> | topics/<topic-id>.md

Rules:
- Write the topic file first, then update the index. Never the reverse.
- Each line must be ~150 characters or less.
- The index never contains facts, inferences, or detailed rationale -- only pointers.
- If the index approaches 200 lines, trigger a consolidation cycle (see Dream Phase).

### Topic File Format

Each topic file in `topics/` follows the entry shapes defined in Section 5, with these additional metadata fields in `memory_meta`:

- `confidence: high | medium | low` -- how verified is this knowledge
- `last_verified: <ISO date>` -- when an agent last confirmed this against the codebase
- `stale: true | false` -- set by consolidation when `last_verified` is older than 30 days

Topic files are loaded on-demand. Agents read the index first and load only topics relevant to the current task.

---

## 2. Durable vs Session Memory

- Durable project knowledge belongs in `.agent-memory/`.
- `vscode/memory` is session memory only. It may hold current-plan notes, transient routing hints, or short-lived user preferences.
- Never treat `vscode/memory` as canonical project truth.
- If a detail must survive across sessions or be shared with future agents, write it into `.agent-memory/`.

---

## 3. Durable vs Runtime Artifacts

- Temporary execution notes, brainstorm pads, command scratchpads, and transient reports do NOT belong in `.agent-memory/`.
- Put transient state in runtime files such as `/.tmp/`, session memory, task-local notes, or other execution artifacts.
- If a detail is only useful for the current run, keep it out of durable memory.
- Planning artifacts such as draft epics, tentative feature breakdowns, and plan deltas stay in session memory unless they harden into durable operating rules, architecture decisions, or recurring constraints.

---

## 4. Formatting Rules

### Markdown Precision

- Use clear headers for each entry.
- Separate **Facts** from **Inferences** whenever there is any uncertainty.
- Keep statements durable and repo-specific; do not paste long narrative or chat-style prose.
- Date every entry in ISO format (YYYY-MM-DD).

### Atomic Updates

- Never overwrite the entire file unless performing Garbage Collection.
- Append new entries to the bottom or merge into existing relevant sections.
- Ensure no duplicate entries exist for the same problem/decision.

---

## 5. Entry Shapes

### Decision Topic Format

Prefer this shape for durable decisions:

```md
## <Decision Title> — YYYY-MM-DD

### Facts
- Verified repo facts with file/path references.

### Inferences
- Marked assumptions or interpretations that still need validation.

### Decision
- The durable rule, invariant, or operating choice.

### Consequences
- What this changes, constrains, or requires going forward.

### Citations
- File paths that justify the decision.

### memory_meta
- timestamp: YYYY-MM-DD
- author: <agent>
```

### Onboarding Snapshot

For onboarding / project familiarization runs, append a compact baseline snapshot:

```md
## Onboarding Snapshot — YYYY-MM-DD

### Facts
- Major modules / packages
- Run / build / test commands
- Key conventions and invariants
- Top risks or TODOs worth remembering

### Inferences
- Only if necessary, and clearly marked
```

### Error-Pattern Topic Format

Prefer this shape for recurring issues:

```md
## <Pattern Title> — YYYY-MM-DD

### Reproduction Signal
- Test name, stack trace, failing command, or clear repro steps.

### Root Cause
- The actual repeated failure mode.

### Fix
- What resolved it.

### Prevention
- Guardrail: test, lint, invariant, review rule, or coding constraint.

### Citations
- File paths or commands that support the pattern.

### memory_meta
- timestamp: YYYY-MM-DD
- author: <agent>
```

### Topic File Naming

Topic files use kebab-case IDs matching their index entry: `topics/<topic-id>.md`

Examples:
- `topics/milestone-support.md`
- `topics/jq-boolean-pattern.md`
- `topics/csv-injection-fix.md`

---

## 6. Conflict Resolution (Multi-Hive)

- In Multi-Hive mode, always work on the local branch copy.
- If a merge conflict occurs in memory files, prioritize the most descriptive and recent information.
- Use bullet points to list alternative approaches if consensus is not possible.
- Use `/delegate` and background sessions for execution isolation, not as a substitute for durable memory writes.

---

## 7. Memory Sync After Meaningful Changes

After any non-trivial feature, bugfix, refactor, onboarding scan, review-driven change, or CI/dependency update:

1. Write or update the relevant topic file in `.agent-memory/topics/`, then update `INDEX.md`. Never update the index without a corresponding topic file.
2. Run this checklist:
   - create or update a topic file in `topics/` for new invariants, decisions, onboarding snapshots, or behavior changes worth keeping
   - create or update a topic file in `topics/` for repeatable failure modes with clear fix and prevention guardrails
   - keep only durable knowledge; move scratch notes, temporary plans, and verbose reports out of `.agent-memory/`
   - separate `Facts` from `Inferences` when the statement is not fully verified from the repo
   - avoid duplicate entries; merge into an existing entry if it describes the same rule or pattern
   - re-read the updated file and verify the new entry still matches the codebase and does not contradict higher-priority memory
   - if a file grows too large or stale, archive the oldest low-value entries to `.agent-memory/archive/`
3. Ensure the new entry is concise, non-duplicative, and still true after the change.
4. If the memory has drifted from reality, fix the stale entry immediately while context is fresh.

---

## 8. Smart Garbage Collection (Archiving)

- **Audit Trigger**: Perform a check when INDEX.md approaches 200 lines or any topic file exceeds 300 lines.
- **Archiving Logic**:
  - Move the oldest 20% of topic files to `.agent-memory/archive/`.
  - Name archived files as `archive/<topic-id>-YYYY-MM-DD.md`.
  - Remove the corresponding line from INDEX.md. Optionally leave a tombstone comment if the topic is referenced by other active topics.

---

## 9. Drift Recovery

If `.agent-memory/` is stale, contradictory, or mostly wrong:

1. Use the smallest recovery level that restores trust:
   - cosmetic drift: fix stale wording, duplication, missing sections, and obvious contradictions in place
   - structural drift: run a fresh onboarding / architecture scan and supersede stale entries using verified repo facts
   - rebuild from fresh evidence: if most memory is untrusted, rebuild the baseline from a fresh repo analysis and archive obsolete entries
2. Prefer repair over deletion; preserve history unless the content is clearly misleading and superseded.

---

## 10. Memory Semantics: Best Known Context

Memory entries are treated as the best available context, not absolute truth. Agents must verify critical facts against the actual codebase before relying on them for decisions.

Rules:
- High-confidence entries verified within 30 days may be trusted without re-verification for routine tasks.
- Medium or low-confidence entries must be verified before use in architectural decisions or critical fixes.
- Stale entries (last_verified > 30 days) should be re-verified on first access or flagged for consolidation.
- If verification reveals a contradiction, update the topic file immediately and note the correction in memory_meta.
- Never treat memory as a substitute for reading the actual code when precision matters.

---

## 11. Dream Phase (Periodic Consolidation)

The Dream Phase is a proactive maintenance cycle that prevents memory drift, staleness, and contradiction accumulation. It is triggered and orchestrated by the Orchestrator (see Step 7.5 in `orchestrator.agent.md`).

### Trigger Conditions

A Dream Phase should run when any of these are true:

1. INDEX.md has more than 10 entries that have not been reviewed since the last consolidation.
2. The `last_consolidated` field in INDEX.md is older than 7 days.
3. The Orchestrator or user explicitly requests consolidation.

### Lock Protocol

1. Before starting, set `_consolidation_in_progress: true` in the INDEX.md header.
2. After completing (success or handled failure), set `_consolidation_in_progress: false` and update `last_consolidated` to the current ISO date.
3. If a consolidation lock has been active for more than 1 hour without progress, any agent may force-release it by setting `_consolidation_in_progress: false`.
4. Do not run Dream Phase concurrently with Step 7 memory writes.

### Phases

#### Orient
- List all files in `.agent-memory/topics/`.
- Read `INDEX.md` fully.
- Identify index entries pointing to missing topic files (orphaned pointers).
- Identify topic files not referenced in the index (orphaned topics).

#### Audit (Read-Only)
- For each active topic, check `last_verified` and `confidence` from its `memory_meta`.
- Cross-reference key facts against the actual codebase where feasible.
- Flag topics as:
  - `stale`: `last_verified` older than 30 days
  - `contradicted`: fact no longer matches codebase
  - `duplicated`: multiple topics cover the same decision/pattern
  - `orphaned`: topic file exists but is not in the index, or index entry points to missing file

#### Consolidate (Write)
- Merge duplicate topics into a single topic file. Preserve the most complete and recent information.
- Update contradicted topics with corrected facts. Note the correction in `memory_meta`.
- Mark stale topics with `stale: true` in their `memory_meta`.
- Convert any relative dates to absolute ISO dates (YYYY-MM-DD).
- Archive low-value or superseded topics to `.agent-memory/archive/<topic-id>-YYYY-MM-DD.md`.
- Add orphaned topic files to the index, or archive them if they are outdated.
- Remove orphaned index pointers.

#### Prune
- If INDEX.md exceeds 180 lines, archive the oldest 20% of low-access topics.
- Ensure INDEX.md stays under 200 lines after all operations.
- Ensure no topic file exceeds 300 lines; split large topics if necessary.

### Output
After consolidation, the executing agent must report:
- Number of topics audited, merged, archived, and flagged as stale.
- Updated `last_consolidated` date.
- `Dream Phase Complete: <summary>`.

---

## 12. Transaction Verification (Critical)

After every write or modification to `.agent-memory/`:

- **Write Order**: Confirm topic file was written before index was updated. If index references a non-existent topic file, the transaction has failed.
- **Read-Back**: You MUST read the file back to verify the entry was correctly appended/merged.
- **Consistency Check**: Ensure the new entry doesn't contradict existing high-priority decisions.
- **Success Report**: Explicitly state `Memory Transaction Successful: <reason>` in the output.
