# workspace — Multi-Repo Workspace Manager

An installable [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview)
plugin for AI-assisted development, geared towards long-term tasks across multiple repositories. Install it,
then from anywhere run `/workspace:setup-environment` to scaffold a workspace:
declare the repos you need (via a **domain** or a custom config), and the
plugin clones and organizes them, layers per-repo Claude context on top, and
gives you structured project workspaces for long-running tasks.

Ships with bundled domains for common scenarios (the included ones target
OpenShift components) — or build your own for any domain.

## Install

From inside Claude Code:

```text
/plugin marketplace add openshift-eng/edge-tooling
/plugin install workspace@edge-tooling
```

## Quick Start

```text
/workspace:setup-environment
```

The skill walks you through everything: it asks **where** to create the
workspace, lets you pick a **domain** (or an external one by git URL), clones
the repos, distributes context files, and generates the workspace's root
`CLAUDE.md`. When it finishes, launch `claude` from inside the new workspace
directory for future sessions.

To build a workspace from an arbitrary set of repos instead of a bundled
domain, use `/workspace:create-domain`.

## Way of Working

A workspace is a long-lived directory. Projects inside it track individual
tasks — a bug fix, a feature, an investigation — across as many Claude Code
sessions as they need.

**The loop:**

```text
                    ┌────────────────────────────┐
                    │                            │
                    ▼                            │
/new-project ──▶ <work> ──▶ /update-project ──▶ /resume-project
                                │
                                ▼
                          /close-project
```

| Step | What happens |
|------|-------------|
| **new-project** | Creates a `projects/<name>/CLAUDE.md` with the task description, checklist, and links to relevant repos. Optionally sets up a git worktree for isolated work. |
| **work** | Normal development. The project CLAUDE.md keeps you oriented — Claude loads it when you resume. |
| **update-project** | Records what the session accomplished: checked-off items, decisions, blockers. Run this before ending a session. |
| **resume-project** | Reloads the project context in a new session. The SessionStart hook also surfaces your recent projects automatically. |
| **close-project** | Marks the project done and cleans up any worktrees it created. |

For single-session tasks you can skip the loop — just work directly. Projects
pay off when a task spans multiple sessions or you need to context-switch
between efforts.

**Handoffs.** When you want the next session to pick up exactly where you
left off, use `/workspace:handoff` before `/clear`. The next session
auto-resumes that project instead of showing the project list.

## Skills

**Core workflow** — the project lifecycle:

| Skill | Purpose |
|-------|---------|
| `/workspace:new-project` | Start a project for a task (bug, feature, docs, analysis) |
| `/workspace:update-project` | Record session progress into the project docs |
| `/workspace:resume-project` | Reload a project's context and continue |
| `/workspace:close-project` | Mark a project done and clean up worktrees |
| `/workspace:handoff` | Arm a handoff so the next `/clear` auto-resumes |
| `/workspace:consolidate-project` | Archive completed checklist items from a large project |

**Workspace setup** — usually one-time:

| Skill | Purpose |
|-------|---------|
| `/workspace:setup-environment` | Create or refresh a workspace from a domain |
| `/workspace:create-domain` | Build a custom domain from arbitrary repos |
| `/workspace:update-domain` | Feed lessons from a project back into the domain's context files |

A SessionStart hook surfaces your recent projects whenever you launch Claude
Code inside a workspace (it stays silent elsewhere). After
`/workspace:handoff`, that same hook instead resumes the handed-off
project on your next `/clear`.

## Concepts

- **Plugin root** — where the plugin ships (read-only). You never edit here.
- **Workspace root** — the directory you choose during setup. It holds
  `dev-env.yaml`, `repos/`, `projects/`, and any workspace-local `domains/`.
- **Domain** — a reusable config: a `dev-env.yaml` repo list plus optional
  per-repo context, supplemental CLAUDE.md files, and docs.

### Domains

- **Bundled** — ship with the plugin (`tnf`, `lvm-operator`, `example`).
- **External** — installed from a git URL into your workspace's `domains/`:
  `/workspace:setup-environment` → "External domain (git URL)". A URL may
  include a `#subdir` fragment for packs holding multiple domains.
- **Authoring** — a domain is a directory with `domain.yaml` (name +
  description), `dev-env.yaml` (repos), and optional `context/<repo>.md`,
  `supplemental/<repo>.md`, `docs/`, and `settings.local.json.tpl`. Workspace
  domains shadow bundled ones of the same name.

## dev-env.yaml

Generated in your workspace by the setup skills. Schema:

```yaml
domain:                    # auto-recorded by setup; enables refresh-domain
  name: <domain-name>
  source: bundled          # 'bundled' or the external git URL
  # ref / subdir            # (external only)

repos:
  - name: my-repo          # identifier and directory name under repos/
    url: https://github.com/org/my-repo.git
    branch: main
    category: development  # docs | development | testing | deployment | troubleshooting
    summary: "Brief description of the repo's role"
    directory: my-repo     # (optional) overrides the directory name
```

Repos are cloned with `--filter=blob:none` (blobless): full structure visible,
blob contents fetched on demand.

## Requirements

- Git (2.27+ for blobless clones)
- Python 3.9+ with **PyYAML** (`pip3 install pyyaml`), *or* `yq` — needed to
  parse `dev-env.yaml`/project frontmatter. The project tooling reports a clear
  message if PyYAML is missing.
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code/overview)

## Upgrading from the pre-plugin version

Earlier versions were cloned-and-run and installed a SessionStart hook into
your workspace's `.claude/settings.local.json`. The hook now ships with the
plugin, so **remove the stale `hooks` block** from that file to avoid a
duplicate/failing hook:

```jsonc
// delete this block from .claude/settings.local.json
"hooks": {
  "SessionStart": [ { "hooks": [ { "type": "command",
    "command": "\"$CLAUDE_PROJECT_DIR\"/scripts/recent-projects.py" } ] } ]
}
```

The old `/dev-env-setup` and `/project:*` commands are replaced by the
`/workspace:*` skills above.

## Developing the plugin

See [CLAUDE.md](CLAUDE.md) for the layout, dev loop
(`claude --plugin-dir .` + `/reload-plugins`), and test command
(`bash tests/test_setup.sh`).
