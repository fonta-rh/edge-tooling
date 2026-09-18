---
name: update-project
description: Update project documentation from what was accomplished in this session
argument-hint: [name-or-number]
context: fork
---

# Update Project Documentation

Update a project's documentation based on what was accomplished in the
current conversation. Dispatch a fork to do the work, then act on its
report.

## Scope Rules

**Update:** files under `projects/<name>/` in the workspace — CLAUDE.md
(index, checklists, progress) and detail files (investigation notes, test
results, plans, etc.).

**NEVER touch during this command:**
- The `status:` frontmatter field — use `/workspace:close-project` to change it
- Memory files (`memory/MEMORY.md`, `memory/project_*.md`)
- Internal session tasks (TaskCreate / TaskUpdate)
- Repo source files under `repos/`

## Step 1: Dispatch the Fork

Check `$ARGUMENTS` for a trailing `auto` token (this is how the
auto-update cron job identifies itself — see `/workspace:auto-update`).
If present, strip it and remember this run is **loop-driven**; it
determines cron behavior in the final step. A manual invocation (no
`auto` token) is never loop-driven, regardless of whether a loop happens
to be running.

Dispatch a fork (Agent tool, `subagent_type: "fork"`) to do Steps 2-4
below, passing the remaining argument (if any) as the project name. Do
not use a fresh agent and do not set a `model` override — a fresh agent
or a different model has no conversation context and forces a cold
re-read of everything already in context.

Give the fork this directive: You already have the conversation and the
project CLAUDE.md in context. Do not re-read files that are in context.
Apply all edits with the fewest tool calls possible. Do not re-read files
to verify. Target 4 to 6 turns. Do not call CronCreate, CronList, or
CronDelete — cron management happens in the main session after you
return.

The fork's only output is a one-line report: `updated: <what>` or
`nothing`.

## Step 2: Resolve Project (fork)

Use the project already loaded in this conversation (from
`/workspace:resume-project` or any earlier project interaction). If
`$ARGUMENTS` has a token, use that as the project name instead.

If no project is in context and no argument was given, report `nothing`
and stop.

## Step 3: Identify Updates (fork)

Review the conversation history and identify:

1. **Checklist items completed** — `- [ ]` items now done.
2. **New checklist items** — work discovered or queued.
3. **Detail file updates** — new findings, test results, or analysis to
   add to existing detail files, or new detail files to create.
4. **New detail files in Reference Files table** — files created in
   `projects/<name>/` not yet registered in CLAUDE.md's table.
5. **Progress entries** — milestones or outcomes to append.
6. **`last-active` timestamp** — always update `last-active: <YYYY-MM-DDTHH:MM>`
   in the frontmatter to the current date and time when any other update is
   applied. If the field does not exist yet, add it after the `status:` line.

If nothing to update, report `nothing` and stop.

## Step 4: Apply Edits and Report (fork)

Apply the updates identified in Step 3 directly: only edit files under
the project directory, never change the `status:` frontmatter field, use
the Edit tool for existing files and Write for new files, and edit each
file individually — do not rewrite entire files.

Report exactly one line: `updated: <brief summary of what changed>`.

## After the Fork Returns (main session)

Confirm the fork's result to the user (one line: what was updated, or
"Nothing to update."). If the session produced durable domain-level
knowledge (not just project status), suggest `/workspace:update-domain`.

**Cron management only applies when this run is loop-driven** (Step 1).
A manual invocation stops here — never call CronCreate, CronList, or
CronDelete for a manual run, even if a loop happens to be active.

If loop-driven:
- Fork reported `updated: ...` — arm a new one-shot cron: compute the
  date/time 50 minutes from now (`date`) and call CronCreate with
  `recurring: false`, cron pinned to that exact
  minute/hour/day-of-month/month (not `*/50` — cron reads that as
  minutes 0 and 50), prompt `/workspace:update-project <name> auto`.
- Fork reported `nothing` — do not re-arm. Tell the user: "Notes are
  current. Auto-update stopped. Run `/workspace:auto-update` to
  re-enable."
- Fork failed outright — tell the user: "Update failed: [error summary].
  Run `/workspace:update-project` manually to retry." Do not re-arm.
