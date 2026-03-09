# Command Dashboard v2

> See every command, skill, and agent across your entire workspace at a glance — searchable, filterable, always in sync.

## What This Does

- **Workspace-wide scanning** — discovers commands and skills across all projects, global `~/.claude/commands/`, and shared libraries
- Interactive single-page dashboard with three tabs: Commands, Skills, Agents
- **Native commands** — optionally includes Claude Code's built-in slash commands as a quick reference
- **Copy to Global** — one-click clipboard copy to promote project commands to global scope
- **Global agents** — discovers agents from `~/.claude/agents/` alongside shared library agents
- **Scope-first grouping** — items organised by Global > Shared Library > Per-Project, with stream/domain sub-groups
- Two-tier filtering: filter by project, then by stream/domain — filters compose together
- Tab toggle, instant search, collapsible sections, detail panel with project association
- Dark theme, responsive layout, keyboard shortcuts (`/` to search, `Esc` to close)
- Portable and whitelabel-ready — anyone can clone this repo and get a dashboard populated with their own workspace data

## What You Need

- A computer (Mac, Linux, or Windows)
- Claude Code installed
- A workspace with one or more projects containing `.claude/commands/` (skills and agents are optional)

## How to Install

1. Give this folder to Claude Code
2. Say: "Read INSTALL.md and help me set this up"
3. Follow along — Claude discovers your workspace and builds the dashboard

**Estimated setup time:** 10-15 minutes

## Keeping It Updated

After adding, removing, or modifying commands, skills, or projects:

```
/cmd-dash-update
```

This global command reads the saved config and rescans everything without re-asking setup questions. New projects in your workspace are picked up automatically.

## Running Cost

Free — everything runs locally. No API keys, no external services, no subscriptions.

## What's Inside

| File | Purpose |
|------|---------|
| `INSTALL.md` | Installation guide (Claude reads this) |
| `templates/dashboard-template.html` | Dashboard template with theme placeholders — populated during install |
| `cmd-dash-update.md` | Global rescan command — copied to `~/.claude/commands/` during install |
| `outputs/command-dashboard.html` | Generated dashboard (created during install, gitignored) |
| `outputs/cmd-dash-config.json` | Scan config with theme settings for `/cmd-dash-update` (gitignored) |
| `.gitignore` | Keeps `outputs/` untracked so `git pull` never conflicts with your data |

## Portability

This module is designed to work for anyone:

- **No hardcoded paths** — all paths are discovered during install and saved to `cmd-dash-config.json`
- **Cross-platform** — works on macOS, Linux, and Windows
- **White-label theming** — choose a colour scheme (GitHub Dark, Navy & Red, Midnight, Forest, or custom hex), font style (Modern, Classic, Minimal, Rounded), and optional logo — all saved in config and reapplied on every rescan
- **Copy to Global** — copies a shell command to clipboard; works in any terminal (macOS/Linux) or Git Bash (Windows)
- **Config-driven updates** — `/cmd-dash-update` uses the saved config, so rescans just work
- **Safe to `git pull`** — the template is tracked, but all user data lives in `outputs/` (gitignored). After pulling upstream updates, run `/cmd-dash-update` to regenerate with the new template and your existing config

Clone this repo, run the installer, and get a branded dashboard that reflects your workspace — not someone else's.

---

> Created by Alex McConnachie | [lightleads.com.au](https://lightleads.com.au)
> Workspace-wide scanning, UI enhancements, and Copy to Global by Nick Martin ([@mindmatter-aia](https://github.com/mindmatter-aia)) | [PatriotAgentic LLC](https://www.patriotagentic.com)
