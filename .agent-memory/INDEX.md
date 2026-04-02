---
schema_version: 2
last_consolidated: never
_consolidation_in_progress: false
---

# Memory Index

Lightweight pointer index for `.agent-memory/topics/`. Each line maps a topic ID to its file.
Agents read this index first and load topic files on demand.

Format: `- <topic-id> | <type> | <date> | <confidence> | <path>`

## Entries

<!-- Add index entries below. Keep this file under 200 lines. Write topic file first, then add the entry here. -->
