# Hybrid Intelligence Squad (HIS)

A replicable GitHub-native framework for coordinating AI agents (Copilot) across issues, PRs, and GitHub Projects.

## Quick Start

1. **Copy** the `.github/` folder to your target repository
2. **Run** the `Bootstrap Agent Squad` workflow from the Actions tab
3. **Edit** `.github/copilot-instructions.md` with your project-specific conventions

## Architecture

```
.github/
├── squad-config.json              # Central config (org, project, labels)
├── agents/
│   ├── coder.agent.md             # Implementation agent
│   ├── orchestrator.agent.md      # Task coordination agent
│   ├── planner.agent.md           # Planning agent
│   └── reviewer.agent.md          # Code review agent
├── actions/
│   ├── assign-copilot/            # Assign Copilot bot to an issue
│   ├── create-phase-issue/        # Create phase issue + automations
│   ├── find-wip-issues/           # Find in-progress agent issues
│   ├── import-external-task/      # Import a single task from Jira/Kanbanize
│   ├── load-squad-config/         # Load squad-config.json as outputs
│   ├── sync-labels/               # Sync labels from JSON
│   ├── sync-milestones/           # Sync milestones from JSON (optional)
│   ├── update-project-status/     # Update issue status in Project V2
│   └── add-to-project/            # Add issue to GitHub Project V2
├── workflows/
│   ├── bootstrap-squad.yml        # One-time setup (manual trigger)
│   ├── on-issue-opened-start-work.yml  # On issue open: assign Copilot + update status
│   ├── on-copilot-pr-transition-to-review.yml  # On PR events: event-driven WIP → Review
│   ├── on-pr-merge-close-issues.yml    # On PR merge: close issue + status → Done
│   ├── create-next-phase-issues.yml     # On issue close: create next phases
│   ├── scheduled-wip-monitor.yml        # Poll WIP issues (fallback, every 15 min)
│   ├── manual-sync-labels.yml         # Manual label sync
│   ├── import-from-external-board.yml # Import tasks from Jira/Kanbanize
│   └── export-completion-report.yml   # Generate CSV/Markdown completion report
└── templates/
    ├── profiles.json              # Available template profiles
    ├── default/                   # Generic 4-phase workflow (Discovery → Verification)
    ├── modernization/             # 8-phase legacy modernization workflow
    ├── feature/                   # 3-phase feature development workflow
    ├── bugfix/                    # 2-phase bug fix workflow
    └── <custom>/                  # Your own custom profiles
        ├── templates-mapping.json # Phase dependency graph + label definitions
        └── *.md                   # Phase issue templates
```

## Configuration

### squad-config.json

All project-specific values live in `.github/squad-config.json`:

```json
{
  "org_login": "your-org",
  "project_number": 1,
  "project_url": "https://github.com/orgs/your-org/projects/1",
  "knowledge_base": {
    "type": "local",
    "name": "",
    "url": "",
    "local_path": ".github/knowledge-base"
  },
  "labels": {
    "wip": "AI-AGENT_WIP",
    "review": "AI-AGENT_REVIEW",
    "trigger": "feature"
  },
  "copilot_bot_login": "copilot-swe-agent",
  "human_reviewers": [],
  "project_status_values": {
    "todo": "Todo",
    "wip": "In Progress",
    "review": "Review",
    "done": "Done"
  },
  "wip_timeout_hours": 24,
  "template_profile": "feature",
  "template_vars": {},
  "workspaces": {
    "source": {
      "label": "",
      "paths": {}
    },
    "target": {
      "label": "",
      "paths": {}
    }
  }
}
```

### Template Profiles

Templates are organized under `.github/templates/<profile>/`. The `template_profile` field in `squad-config.json` selects which set of templates to use. The workflow resolves template files from the profile directory first, falling back to `default/` if not found.

#### Built-in Profiles

| Profile | Phases | Trigger Label | Use Case |
|---------|--------|---------------|----------|
| `default` | 4 (Discovery → Planning → Implementation → Verification) | any | Generic workflow for any project type |
| `modernization` | 8 (Documentation → Foundation → Test Planning → Database → API → Backend → Frontend → Integration Testing) | `modernization` | Legacy system modernization |
| `feature` | 3 (Requirements → Implementation → Testing) | `feature` | New feature development |
| `bugfix` | 2 (Triage → Fix & Verify) | `bugfix` | Bug fix workflows |

#### Creating a Custom Profile

1. Create a directory: `.github/templates/my-profile/`
2. Add a `templates-mapping.json` with your phase definitions and dependency graph
3. Add `.md` template files for each phase
4. Optionally define `milestones` in `templates-mapping.json` to group phases
5. Register the profile in `profiles.json`
6. Set `"template_profile": "my-profile"` in `squad-config.json`
7. Set `"trigger"` in `labels` to match your workflow trigger label

### Template Variables

The `template_vars` field holds arbitrary key-value pairs that are substituted into issue templates via `{{KEY}}` placeholders. Built-in variables include `KB_NAME`, `KB_TYPE`, `KB_URL`, `SOURCE_LABEL`, `TARGET_LABEL`, and workspace-derived paths like `SOURCE_PATH_PROGRAMS`, `TARGET_PATH_BACKEND`, etc. Custom variables like `TARGET_DB` are defined here.

### Milestones (Optional)

Profiles can optionally define milestones in `templates-mapping.json` to group phases into deliverable batches. When present, milestones are automatically synchronized to the repository during bootstrap and phase creation.

Add a top-level `milestones` array and a `milestone` field per template:

```json
{
  "milestones": [
    {
      "title": "Foundation",
      "description": "Discovery and infrastructure - phases 0 through 2",
      "state": "open"
    },
    {
      "title": "Delivery",
      "description": "Implementation and validation - phases 3 through 5",
      "state": "open"
    }
  ],
  "templates": [
    {
      "id": "discovery",
      "milestone": "Foundation",
      "..."
    }
  ]
}
```

When `milestones` is absent or empty, no milestones are created and the framework behaves exactly as before.

### Workspaces (Source/Target)

The `workspaces` field maps source and target directories. Path keys are arbitrary — match them to your project structure. These are flattened into template variables automatically:
- `workspaces.source.paths.programs` → `{{SOURCE_PATH_PROGRAMS}}`
- `workspaces.target.paths.backend` → `{{TARGET_PATH_BACKEND}}`

### copilot-instructions.md

Project-specific conventions for the agents. This is where you define:
- Code style and architecture rules
- Build and test commands
- Pitfalls and constraints

## Workflow Lifecycle

```
Issue opened (with trigger label)
  ↓
on-issue-opened-start-work.yml
  → Assign Copilot bot
  → Add WIP label
  → Set Project status → "In Progress"
  → Enable scheduled-wip-monitor
  ↓
Copilot creates PR and works
  ↓
on-copilot-pr-transition-to-review.yml (event-driven, ~30s)
  → Detect Copilot PR + completed workflow
  → Swap WIP → Review label
  → Set Project status → "Review"
  → Request review from human_reviewers
  ↓ (fallback: scheduled-wip-monitor.yml polls every 15 min)
  ↓
Human reviews and merges PR
  ↓
on-pr-merge-close-issues.yml
  → Set Project status → "Done"
  → Close issue automatically
  ↓
create-next-phase-issues.yml
  → Check dependency graph
  → Create next phase issues
  → Add to Project
  → Loop restarts for next phase
```

### Stuck Detection

If a WIP issue has no completed Copilot workflow after `wip_timeout_hours` (default: 24h), the scheduled-wip-monitor will:
- Add a `stuck` label to the issue
- Comment with diagnostic information

## Replicating to a New Project

1. Copy the entire `.github/` folder
2. Go to **Actions** → **Bootstrap Agent Squad** → **Run workflow**
3. Fill in your org, project number, project URL, **template profile**, and **trigger label**
4. Edit `.github/copilot-instructions.md` for your project conventions
5. Optionally customize or create a new template profile under `.github/templates/`

### Example: Bootstrap for Different Scenarios

| Scenario | `template_profile` | `trigger_label` |
|----------|-------------------|------------------|
| Modernize a legacy app | `modernization` | `modernization` |
| Build a new feature | `feature` | `feature` |
| Fix a bug | `bugfix` | `bugfix` |
| Generic / custom | `default` | your choice |

## Actions Reference

| Action | Purpose |
|--------|---------|
| `assign-copilot` | Assigns copilot-swe-agent bot to an issue via GraphQL |
| `create-phase-issue` | Creates phase issue with optional Copilot assignment, project linking, sub-issue |
| `find-wip-issues` | Paginates issues by label, returns JSON array |
| `import-external-task` | Imports a single task from Jira/Kanbanize as a GitHub Issue (idempotent) |
| `load-squad-config` | Reads squad-config.json and exposes values as step outputs |
| `sync-labels` | Syncs labels from JSON file to repository |
| `sync-milestones` | Syncs milestones from templates-mapping.json to the repository (idempotent, optional) |
| `update-project-status` | Updates issue status in GitHub Project V2 |
| `add-to-project` | Adds an issue to a GitHub Project V2 and optionally sets status |

## Agents Reference

| Agent | Role |
|-------|------|
| **Coder** | Writes code following project conventions |
| **Orchestrator** | Breaks down tasks, delegates to other agents |
| **Planner** | Creates implementation plans, never writes code |
| **Reviewer** | Reviews code for quality, security, conventions |

## External Board Integration (Jira / Kanbanize)

Teams using Jira or Kanbanize can import tasks into the HIS framework without changing their existing workflow. The approach is **Import & Replicate**: tasks are copied from the external board into GitHub Issues, and the HIS automation handles everything from there.

### How It Works

1. **Admin (one-time)**: Configure secrets in repo settings
   - Jira: `JIRA_EMAIL` + `JIRA_API_TOKEN`
   - Kanbanize: `KANBANIZE_API_KEY`
2. **PO**: Go to **Actions** → **Import from External Board** → **Run workflow**
   - Select source (Jira or Kanbanize)
   - Fill in board URL / project key / board ID
   - Click **Run**
3. **Automatic**: The workflow fetches tasks, skips already-imported ones (idempotent), and creates GitHub Issues with the trigger label
4. **HIS takes over**: Each imported issue enters the normal lifecycle (Copilot assigned → WIP → Review → Done)
5. **PO**: Run **Export Completion Report** to download a CSV/Markdown mapping external tasks → GitHub Issues → PRs → status

### Idempotency

Each imported issue contains a hidden marker (`<!-- source: jira:PROJ-123 -->`) that prevents duplicate imports. Running the import workflow multiple times is safe.

### Imported Issues vs Phase Issues

Imported issues are **independent** — they don't trigger phase chaining (`create-next-phase-issues` exits gracefully). They go through a single WIP → Review → Done cycle.

---

## Quick Reference (Day-to-Day)

Simplified process for daily operation:

1. **Issue appears** with trigger label → Copilot is auto-assigned, WIP label applied
2. **Copilot works** → Creates a PR linking to the issue ("Closes #N")
3. **PR detected** → WIP label swapped to REVIEW, human reviewers notified
4. **You review** → Approve and merge the PR
5. **Auto-close + next phase** → Issue closes, next phase issues created automatically

### Status Meanings

| Label / Status | Meaning |
|----------------|--------|
| `AI-AGENT_WIP` | Copilot is actively working on the issue |
| `AI-AGENT_REVIEW` | Copilot finished; waiting for human review |
| `stuck` | WIP issue exceeded timeout (default: 24h) — needs attention |
| Project: "In Progress" | Work underway |
| Project: "Review" | Ready for human review |
| Project: "Done" | PR merged and issue closed |

### Where to Look

- **Issue labels** — Current state (WIP / REVIEW / stuck)
- **Project board** — Status column for all phases
- **Actions tab** — Workflow run history and logs
- **PR** — Copilot's code changes and review status

## Troubleshooting

### Copilot not assigned to issue?
- Verify the issue has the **trigger label** (configured in `squad-config.json` → `labels.trigger`)
- Check `copilot_bot_login` in `squad-config.json` matches the bot name
- Review the `on-issue-opened-start-work` workflow run logs in the Actions tab

### PR not transitioning to Review?
- Ensure the PR body contains `Closes #N` linking to the WIP issue
- Verify `copilot_pr_author` in config matches the PR author name
- Check if the Copilot workflow completed successfully on the PR branch
- The scheduled-wip-monitor (every 15 min) will catch it as a fallback

### Next phase not created after merge?
- Verify the closed issue has the trigger label
- Check the dependency graph in `templates-mapping.json` — all dependencies must be closed
- Look at the `create-next-phase-issues` workflow run logs
- An issue with that phase label may already exist (idempotency check)

### WIP issue stuck for too long?
- After `wip_timeout_hours` (default: 24h), the monitor adds a `stuck` label
- Check the diagnostic comment on the issue for details
- You may need to manually close and recreate the issue, or investigate Copilot logs
