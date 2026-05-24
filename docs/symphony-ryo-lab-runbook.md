# Symphony-Ryo Fork Lab Runbook

This fork validates `openai/symphony` against the `Symphony-Ryo` workflow before any production-default adoption.

## Safety Policy

- Use one test Linear issue at a time.
- Keep `agent.max_concurrent_agents: 1` during first validation.
- Keep final merge human-controlled.
- Do not enable auto-merge.
- Route the workflow with `tracker.assignee: "me"` and verify only one issue is in the active candidate set before starting.

## Build

```bash
cd ~/src/symphony-ryo-lab/elixir
mise trust
mise install
mise exec -- mix setup
mise exec -- mix build
```

## Run

```bash
export LINEAR_API_KEY=...
cd ~/src/symphony-ryo-lab/elixir
mise exec -- ./bin/symphony --port 4000 ../WORKFLOW.md
```

## First Smoke Target

- Linear issue: RYO-143
- Project: Symphony-Ryo
- Team: RYO
- State: Todo
- Assignee: me
- Workspace root: `~/workspaces/symphony-ryo-lab`

## Verify

- Dashboard opens at `http://localhost:4000`.
- Exactly one test Linear issue is claimed.
- Workspace is created under the configured workspace root.
- No final merge happens automatically.

## First Smoke Result

- Date: 2026-05-24
- Linear issue: RYO-143
- Workspace path: not created
- PR created: intentionally no
- Validation run: dashboard started at `http://127.0.0.1:4000/`; polling did not claim an issue
- Final merge remained human-controlled: yes
- Blockers: `LINEAR_API_KEY` was not present in the process environment, so Symphony logged `Linear API token missing in WORKFLOW.md`
