---
title: User Stories
description: 'Full user journeys and stories for Soji — the scenarios that drive product decisions.'
category: product
status: draft
last_updated: 2026-03-04
depends_on:
  - vision
---

# User Stories

> Soji's primary user is an individual developer running an agentic workflow — one human directing N concurrent AI agent sessions across multiple repos. All stories are written from that perspective.

---

## Core Journeys

### Journey 1: Morning Startup — "What state am I in?"

**Context**: Developer opens their terminal after being away (overnight, a few days, a week). Multiple workspaces may have things running or leftover.

**Trigger**: `soji` with no arguments.

**Desired experience**:
1. Opens to a workspace dashboard — one card per workspace showing health at a glance.
2. At a glance: which workspaces have active processes, which have stale/orphaned ones, which are clean.
3. Red flags are obvious — high memory usage, orphaned processes, broken environment vars.
4. From here the developer can drill in or take action without leaving the TUI.

**User stories**:
- As a developer, I want to see all my workspaces and their health status in a single view so I can quickly orient myself.
- As a developer, I want to see if any processes are orphaned or detached without having to run multiple commands.
- As a developer, I want memory usage to be visible per workspace so I know where resources are going.

---

### Journey 2: Context Switch — "Is this workspace ready to work in?"

**Context**: Developer finishes a task in one workspace and wants to switch to another. They want to know if the target workspace is in a good state before context-switching.

**Trigger**: Navigating to a workspace in the TUI or `soji status <workspace>`.

**Desired experience**:
1. Single workspace detail view: processes, ports, env vars, secrets, git state.
2. Green checkmarks where things are healthy; warnings where something needs attention.
3. Can see: "Claude session from 2 days ago, detached" and kill it before switching.
4. Can see: secrets are loaded ✓, direnv is active ✓, dev server is running on :3000 ✓.

**User stories**:
- As a developer, I want to see the full environment state of a workspace before I start working in it.
- As a developer, I want to know if my env vars and secrets are properly loaded so I don't spend 20 minutes debugging a missing env var.
- As a developer, I want to clean up stale Claude sessions from the last time I worked in this workspace.

---

### Journey 3: Port Conflict — "Something is using my port"

**Context**: Developer tries to start a dev server and gets `Error: address already in use :3000`.

**Trigger**: Port conflict error, or proactively opening Port Manager before starting work.

**Desired experience**:
1. Open Port Manager (`p` or tab to port manager view).
2. See all listening ports, grouped by workspace.
3. Find port 3000 — it's a node process from yesterday, workspace-bound to `kata`, started 18h ago.
4. Press `k` to kill it. Confirmation prompt. Killed. Port free.
5. Optional: developer can see the exact command that was running so they know what it was.

**User stories**:
- As a developer, I want to see what process is holding a port before I kill it so I don't accidentally kill something important.
- As a developer, I want to see the full command that spawned a process, not just the process name.
- As a developer, I want to kill a port-holding process in one action from a TUI without remembering lsof syntax.
- As a developer, I want ports to show which workspace they belong to so I have context before acting.

---

### Journey 4: End of Day Cleanup — "Tidying up before I stop"

**Context**: Developer is wrapping up for the day and wants to leave a clean environment.

**Trigger**: `soji clean` or opening the maintenance view.

**Desired experience**:
1. See a list of processes/resources that are candidates for cleanup.
2. Each item shows: what it is, how long it's been stale, what workspace it belonged to, how much memory it's using.
3. Developer reviews the list, toggles off anything they want to keep, confirms.
4. Gets a summary: "killed 4 processes, freed 1.8 GB, 2 Docker containers pruned."
5. Workspace feels clean. Developer can close laptop without anxiety.

**User stories**:
- As a developer, I want to see all orphaned and detached processes in one place so I can clean them up quickly.
- As a developer, I want to review each cleanup candidate before it's killed — not a "nuke everything" button.
- As a developer, I want to see how much memory I'll reclaim from cleanup before I do it.
- As a developer, I want cleanup to be safe — no accidental kills, always confirm.

---

### Journey 5: "What's eating my memory?"

**Context**: Computer is running slow. Developer wants to identify and eliminate resource hogs.

**Trigger**: General sluggishness, or periodic review.

**Desired experience**:
1. Open Soji. Dashboard shows memory breakdown across workspaces.
2. One workspace has an unusually high footprint — drill in.
3. Find: a Docker container from a project that's been inactive for a week, consuming 2.4 GB.
4. Find: three Claude agents that are running but no longer have active sessions.
5. Kill selectively, reclaim resources, see the total freed at the end.

**User stories**:
- As a developer, I want to see resource usage organized by workspace, not just raw PID lists.
- As a developer, I want to identify which process types (Claude, Docker, dev servers) are consuming the most memory.
- As a developer, I want to kill resource hogs individually without taking down an entire workspace.

---

### Journey 6: New Workspace Setup — "Starting a project from scratch"

**Context**: Developer is starting work on a new project or picking up a long-abandoned repo.

**Trigger**: `soji up <workspace>` or a new workspace wizard.

**Desired experience**:
1. Soji checks if a cmux workspace exists for this repo — creates one if not.
2. Loads the workspace's direnv config (`.envrc`).
3. Provisions secrets via `ks inject` (or prompts if secrets aren't set up).
4. Starts declared services from `soji.toml` (dev server, Docker services).
5. Developer gets a green "workspace ready" state — all surfaces open, all services running, all env loaded.
6. They can start working immediately.

**User stories**:
- As a developer, I want to spin up a full workspace environment with one command instead of manually running 5 different tools.
- As a developer, I want to know if my workspace setup succeeded or if something failed to start.
- As a developer, I want workspace setup to be idempotent — running `soji up` on an already-running workspace should be a no-op or gracefully handle what's already started.

---

### Journey 7: Environment Debugging — "Why is my code broken?"

**Context**: Something is failing in the application. Developer suspects a missing or wrong environment variable.

**Trigger**: Unexpected runtime error that might be environment-related.

**Desired experience**:
1. Open workspace detail view.
2. Navigate to Environment section.
3. See all env vars loaded for this workspace, their source (direnv, `.env`, `ks`), and whether they're marked as secrets.
4. Immediately see: `STRIPE_SECRET_KEY` is marked as a ks secret but shows `not loaded`.
5. Can run `ks inject` (or Soji wraps it) to reload without leaving the TUI.
6. Variable now shows `loaded ✓`. Problem diagnosed and fixed.

**User stories**:
- As a developer, I want to see which env vars are loaded in my workspace and where they came from.
- As a developer, I want to immediately see if any expected secrets are missing from the environment.
- As a developer, I want to re-provision secrets from within Soji without dropping to a shell.

---

### Journey 8: "What is this unknown process?"

**Context**: Developer sees an unfamiliar entry in the Port Manager (port 8080, unknown command).

**Trigger**: Regular Soji usage — noticed while doing something else.

**Desired experience**:
1. Cursor on the unknown row. Press `Enter`.
2. Detail pane opens: full command is `java -jar /Users/.../old-service.jar`, CWD is a repo from 3 months ago, running for 6 days.
3. Developer recognizes it as a leftover from a demo project.
4. Presses `k` to kill it. Confirms. Gone.

**User stories**:
- As a developer, I want to identify unfamiliar processes without having to remember `lsof`, `ps`, and `kill` syntax.
- As a developer, I want to see the full command and working directory of any process so I understand what spawned it.
- As a developer, I want to act on a process immediately once I understand it — no context switching.

---

### Journey 9: Weekly Review — "How has my environment evolved?"

**Context**: Developer does a weekly check-in on the state of their dev environment.

**Trigger**: `soji diff --weekly` or opening the history view.

**Desired experience**:
1. See what changed since last week: new workspaces created, old ones cleaned up, memory usage trend.
2. See what has been running longest without cleanup.
3. See if any workspace has grown significantly in resource usage.
4. Get a summary suitable for a quick review — not drowning in details.

**User stories**:
- As a developer, I want to see a weekly summary of my dev environment so I can catch resource creep early.
- As a developer, I want to see which workspaces have accumulated the most stale processes over time.

*(Note: This journey depends on M5 snapshot history — deferred.)*

---

### Journey 10: Soji as a Shared Infrastructure Layer

**Context**: Developer has services like OrbStack, Ollama, and a local Postgres that serve multiple workspaces. They want to understand what shared infrastructure is running.

**Trigger**: Switching workspaces, or a general system health check.

**Desired experience**:
1. In the Port Manager or a separate "shared services" panel, see all shared infrastructure.
2. OrbStack: running ✓. Ollama: running on :11434 ✓. Postgres: running on :5432 ✓.
3. These are clearly distinguished from workspace-bound processes — they belong to the org layer, not any single workspace.
4. Developer can see at a glance that all shared infrastructure is healthy before starting work.

**User stories**:
- As a developer, I want shared services (OrbStack, Ollama, databases) to be clearly separated from workspace-specific processes.
- As a developer, I want to see the health of shared infrastructure in a single view without it cluttering my workspace views.
- As a developer, I want to configure what counts as "shared infrastructure" so Soji doesn't misclassify things.

---

## Smaller Stories

These don't have full journeys but represent discrete capabilities:

- As a developer, I want mouse support so I can click on entries in the TUI without learning all keyboard shortcuts.
- As a developer, I want `Enter` to always mean "show me more" — never trigger a destructive action accidentally.
- As a developer, I want to select multiple processes and kill them in one action instead of one-by-one.
- As a developer, I want Soji to gracefully work even when cmux/Docker/ks aren't installed — degrade, don't break.
- As a developer, I want to jump to a workspace by pressing its number key (1–9) so navigation is fast.
- As a developer, I want a keybindings help overlay (`?`) so I don't have to remember everything.
- As a developer, I want `soji` to feel fast — sub-second startup, data that refreshes without feeling janky.
