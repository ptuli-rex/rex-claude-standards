# Zoho Projects Workflow

## Task Requirement

Before starting any non-trivial feature for **GetDone, ProofUp, BillRoute, IDCore, PayUp, or JobCall**, ensure a Zoho Projects task exists. If none exists, ask the user whether to create one.

- Assign tasks to the current developer
- Use the appropriate project (see project-ids.md)
- Set category to "Product Feature" or "Product Enhancement"

## Task Lifecycle

Tasks move through **task lists** (stages) and **statuses**:

### Task Lists (Stages)

| Task List | Purpose |
|-----------|---------|
| **Backlog** | Not yet prioritized |
| **Discovery** | Currently being scoped |
| **Engineering** | Currently being built |
| **QA & Testing** | Currently being QA'd |
| **Go-Live** | Going live or already live |
| **Admin** | Non-engineering tasks (marketing, etc.) |

### Typical Flow

```
Backlog -> Discovery -> Engineering -> QA & Testing -> Go-Live
```

### Statuses

| Status | ID | Use When |
|--------|-----|----------|
| Open | `2526274000000016068` | Task created, not started |
| In Progress | `2526274000000031001` | Actively being worked on |
| On Hold | `2526274000000031007` | Blocked or paused |
| Completed | `2526274000000069229` | Done (set completion_percentage: 100) |

## API Notes

- **Portal**: rexdotcom (ID: `893715552`)
- **Date format**: ISO 8601 (`2026-04-09T00:00:00.000Z`)
- **Mandatory fields** when creating tasks: `name`, `tasklist`, `start_date`, `category`
- **To complete a task**: set `completion_percentage: 100` (auto-transitions status)
