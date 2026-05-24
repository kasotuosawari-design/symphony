---
tracker:
  kind: linear
  api_key: $LINEAR_API_KEY
  project_slug: "symphony-ryo"
  assignee: "me"
  active_states:
    - Todo
  terminal_states:
    - Done
    - Closed
    - Cancelled
    - Canceled
    - Duplicate
polling:
  interval_ms: 10000
workspace:
  root: ~/workspaces/symphony-ryo-lab
hooks:
  after_create: |
    git clone https://github.com/kasotuosawari-design/Symphony-Ryo.git .
    git status --short --branch
agent:
  max_concurrent_agents: 1
  max_turns: 3
codex:
  command: /Applications/Codex.app/Contents/Resources/codex --config 'model="gpt-5.5"' app-server
  approval_policy: never
  thread_sandbox: workspace-write
  turn_sandbox_policy:
    type: workspaceWrite
---

You are running the first Symphony-Ryo fork-lab smoke test.

Scope:

1. Work only on Linear issue `{{ issue.identifier }}`.
2. Confirm the workspace exists and the `kasotuosawari-design/Symphony-Ryo` repository cloned correctly.
3. Do not create, push, merge, or close a pull request during this smoke run.
4. Do not modify production files.
5. Stop at a safe human-review state and report what was observed.

Safety boundary:

- Final merge is always human-controlled.
- This run is only for proving Linear polling, per-issue workspace creation, Codex app-server startup, and dashboard visibility.
