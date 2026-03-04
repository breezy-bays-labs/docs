---
title: Introduction
description: 'Soji (掃除) — Dev workspace observability, maintenance, and organization for agentic development workflows.'
category: overview
status: draft
last_updated: 2026-03-03
depends_on: []
---

# Soji — 掃除

**Dev workspace observability, maintenance, and organization TUI.** Keeps the dojo tidy.

## What Soji Does

Soji provides a **workspace-centric lens** over your development environment. It monitors Claude sessions, cmux workspaces, Docker containers, dev servers, MCP servers, and desktop apps — organized by workspace. It identifies orphaned processes and helps clean them up.

The core abstraction: **Workspace = repo + everything that serves its development.** Every process, service, secret, deployment, and tool instance maps to a workspace.

## Three Pillars

| Pillar | Purpose | Status |
|--------|---------|--------|
| **Observability** | See your workspace state — what's running, what's consuming resources, what belongs where | Active (printf dashboard) |
| **Maintenance** | Clean up stale processes, prune branches, reclaim resources | Active (CLI cleanup) |
| **Organization** | Define workspaces declaratively, automate lifecycle, enforce structure | Planned |

## Why Soji Exists

The [Breezy Bays Labs operating model](/breezy-bays-labs/identity) is one human directing N concurrent AI agent sessions. This model generates significant workspace sprawl:

- Multiple Claude sessions across repos, each spawning agents and MCP servers
- cmux workspaces with terminal and browser surfaces
- Dev servers, Docker containers, and cloud services per project
- Git worktrees and branches accumulating across repos

Without active management, development environments become cluttered, memory-hungry, and hard to reason about. Soji is the tool that keeps the dojo clean so you can focus on the work.

## Current State

Soji is in early development. **M0 (scaffold)** is complete:

- Data gathering from ps, lsof, cmux, docker — all running concurrently via tokio
- Printf-based status dashboard with workspace grouping
- Process cleanup with SIGTERM → SIGKILL escalation
- Claude session detection with model and session name lookup
- cmux workspace/surface detection with git branch tracking

See the [Roadmap](/soji/docs/roadmap) for what's next.

## Tech Stack

| Tool | Purpose |
|------|---------|
| **Rust** | Systems language — performance, safety, single binary distribution |
| **tokio** | Async runtime for concurrent data gathering |
| **ratatui** + **crossterm** | TUI framework (imported, not yet active) |
| **clap** | CLI argument parsing |
| **serde** | Serialization for domain types and future JSON output |

## Architecture

Clean architecture with domain-driven design:

```
src/
  domain/         # Pure data types (Process, Workspace, Surface)
  sources/        # Data gathering (shells out to system tools)
  views/          # Display layer (printf now, ratatui planned)
  actions/        # Cleanup operations
  main.rs         # CLI entry with subcommand routing
```

**Shell-out architecture** — Soji gathers data by running `cmux`, `ps`, `lsof`, `docker` commands rather than using system APIs directly. Simpler, more portable, matches what the tools expose.

**Orchestration wrapper** — Soji doesn't replace any tool. It wraps existing tools (git, cmux, docker, Claude, etc.) and provides a workspace-centric view over all of them.

## Quick Start

```bash
cargo build                  # Debug build
cargo run                    # Show status dashboard
cargo run -- --full          # Full view (all workspaces + all details)
cargo run -- clean           # Kill orphaned processes
cargo run -- clean --force   # Kill without confirmation
```
