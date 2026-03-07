# Command Dashboard v2 — Workspace-Wide Installer

> A plug-and-play module by Alex McConnachie | [lightleads.com.au](https://lightleads.com.au)
> Workspace-wide scanning by mindm

<!-- MODULE METADATA
module: command-dashboard
version: v2
status: RELEASED
released: 2026-03-06
requires: []
phase: 4
category: Utility
complexity: moderate
api_keys: 0
setup_time: 10-15 minutes
-->

---

## FOR CLAUDE

You are helping a user install the Command Dashboard v2 — a workspace-wide scanner that discovers commands, skills, and agents across multiple projects, global config, and shared libraries.

**Behavior:**
- Assume the user is non-technical unless they tell you otherwise
- Explain what you are doing at each step in plain English BEFORE doing it
- Celebrate small wins ("Found 9 global commands and 7 projects! Let's scan them all.")
- If something fails, do not dump error logs — explain the problem simply and suggest the fix
- Never skip verification steps — if a check fails, stop and help the user fix it
- Use encouraging language throughout — they are building something real

**Pacing:**
- Do NOT rush. Pause after major milestones.
- After workspace discovery: "Found X projects with commands/skills. Ready to scan?"
- After scanning each scope: "Global: X commands. Shared: Y commands, Z skills, W agents. Projects: ..."
- After building the dashboard: "Your dashboard is ready. Let me open it so you can see everything."

**Error handling:**
- If no workspace root is findable, ask the user to specify one
- If a command file is unreadable, skip it and note what was skipped at the end
- If the template file is missing, check the module folder structure
- Never say "check the logs" — find the problem and explain it

---

## OVERVIEW

The Command Dashboard v2 scans your **entire workspace** — global commands, shared libraries, and every project — to build a single interactive dashboard. Search, filter by project or stream, click any card for details, and see workflow chains and automated services.

**What you'll have when done:**
- A fully populated interactive dashboard HTML file
- A `cmd-dash-config.json` file for future rescans via `/cmd-dash-update`
- A global `/cmd-dash-update` command installed in `~/.claude/commands/`

**Scope:** Global commands + shared library (commands, skills, agents) + all discovered projects

**Setup time:** 10-15 minutes
**Cost:** Free — everything runs locally, no API keys needed

---

## SCOPING

### Quick Path (RECOMMENDED)

Ask: "Want to go with smart defaults (full workspace scan, standard output path), or walk through options?"

**Smart defaults:**
- Scan everything: global commands + shared library + all discovered projects
- Output to `{this-module}/outputs/command-dashboard.html`
- Page title: "Workspace Dashboard"
- AI name: "Claude"
- Include services panel and workflow chains if detected
- Theme: GitHub Dark (default template colours)
- Font: Modern (Inter + JetBrains Mono)
- Logo: None

### Custom Path

Walk through each option:
1. **Output path**: Where to save the dashboard HTML file
2. **Page title**: Custom title for the browser tab and header
3. **AI name**: Name used in the skills banner (default: "Claude")
4. **Services panel**: Include or exclude automated services detection
5. **Workflow chains**: Include or exclude workflow chain detection
6. **Exclude paths**: Any directories to skip during scanning
7. **Colour scheme** — Offer presets:
   - GitHub Dark (default — template colours as-is)
   - Navy & Red (navy bg `#1a2332`, navy+red gradient border, gradient title, glow hover)
   - Midnight (deep black bg `#0a0a0a`, cyan accents, subtle glow)
   - Forest (dark green bg `#0d1a0d`, green accents, earthy tones)
   - Custom (user provides hex values for bg, surface, border, accent colours)
8. **Logo** (optional) — User provides a path to an image file. Read the file, convert to base64, and save as `outputs/logo-base64.txt`. The base64 string is embedded as an `<img>` tag in the dashboard.
9. **Font style** — Offer presets:
   - Modern (Inter + JetBrains Mono — clean, default)
   - Classic (Space Grotesk display + IBM Plex Sans body + JetBrains Mono)
   - Minimal (system fonts — no external loading)
   - Rounded (Nunito + JetBrains Mono — friendly)

---

## PREREQUISITES

### Workspace Root

Ask the user: "Where is the root of your workspace? This is the directory that contains your projects."

Common patterns:
- `~/builds/active-projects/` (multiple project directories inside)
- `~/code/` or `~/projects/`
- A single parent directory containing project subdirectories

### Template File

Verify the dashboard template exists in this module:
```
templates/dashboard-template.html
```

### Output Directory

Create the output directory if needed:
```bash
mkdir -p outputs
```

---

## INSTALL

### Step 1: Discover Workspace

**1a. Find all projects with Claude commands or skills:**

Starting from the workspace root, recursively find directories (max depth 4) that contain `.claude/commands/` or `.claude/skills/`:

```bash
find {workspace-root} -maxdepth 4 -type d -name "commands" -path "*/.claude/commands" 2>/dev/null
find {workspace-root} -maxdepth 4 -type d -name "skills" -path "*/.claude/skills" 2>/dev/null
```

For each match, the **project** is the parent of `.claude/`. Extract the project name from the directory name (last path segment).

**1b. Detect shared library:**

Look for a directory at or near workspace root level that has both `commands/` and `skills/` subdirectories (or `agents/` too), or is named `shared*`, `common*`, or `library*`. This is distinct from project-level `.claude/` directories — shared libraries typically live at the workspace root and have a flat `commands/`, `skills/`, `agents/` structure rather than `.claude/commands/`.

If found, confirm with the user: "I found what looks like a shared library at `{path}`. Include it?"
If not found, ask: "Do you have a shared command/skill library? If so, where is it? If not, we'll skip this."

**1c. Check global commands:**

```bash
ls ~/.claude/commands/ 2>/dev/null
```

If found, confirm: "Found global commands in `~/.claude/commands/`. Include them?"

**1d. Present discovery results:**

Show the user what was found:
```
Workspace Discovery:
  Global commands:  ~/.claude/commands/ (X .md files)
  Shared library:   {path} (Y commands, Z skills, W agents)
  Projects (N):
    - project-name-1 (X commands, Y skills)
    - project-name-2 (X commands)
    ...

Ready to scan everything?
```

Wait for confirmation. Let them exclude any projects if needed.

---

### Step 2: Scan Global Commands

Scan all `.md` files in `~/.claude/commands/`:

**For each command file, extract these fields:**

```
id         — "global--{command-id}" where command-id is derived from file path:
              strip prefix and .md suffix, replace / with -, remove leading -
              Example: plan.md → global--plan
              Example: cmd-dash-update.md → global--cmd-dash-update

name       — the command invocation name: /{path without .md}
              Example: plan.md → /plan

project    — "global"
scope      — "global"

stream     — classify based on path and content:
              - Files in ops/ subdirectory → "ops"
              - Files in content/ subdirectory → "content"
              - Everything else → "meta"

mode       — "interactive", "automated", or "one-shot" (see classification guide below)

tagline    — 5-8 word summary from headings or first descriptive sentence
desc       — 2-4 sentence description synthesised from the file
when       — when to use this command
input      — what the command takes as input (or "None")
output     — what the command produces
outputPath — file system path where output is saved (or "")
chains     — other command IDs this leads into (or [])
tags       — 3-6 keyword tags
```

**Mode classification guide:**
- `"interactive"`: asks questions, has Q&A flow, collects user input, multi-step wizard
- `"automated"`: runs end-to-end without questions, pipeline, generates output
- `"one-shot"`: quick single action, displays info, captures one thing

Store all in a JavaScript array and show the user: "Found **X global commands**: [list names]. Look right?"

---

### Step 3: Scan Shared Library

If a shared library was detected in Step 1:

**3a. Scan shared commands** (`{shared-path}/commands/*.md`):
- Same extraction fields as Step 2
- `id` format: `"shared--{command-id}"`
- `project`: `"shared-library"`
- `scope`: `"shared"`

**3b. Scan shared skills** (`{shared-path}/skills/*/`):
- Look for each skill directory and its main `.md` file
- Extract fields:

```
id         — "shared--{skill-dir-name}"
name       — human-readable name from directory name or heading
project    — "shared-library"
scope      — "shared"
domain     — "integration", "content-mkt", "web", or "skill-meta"
tagline    — 4-6 word summary
desc       — 2-3 sentence description
trigger    — what causes activation (comma-separated task types)
context    — what data/reference material is loaded
```

**Domain classification guide:**
- `"integration"`: connects to external APIs, services, tools, CRM, analytics
- `"content-mkt"`: content creation, marketing, ads, social media, copywriting
- `"web"`: web development, design, diagrams, visual output
- `"skill-meta"`: meta/utility skills, workspace management

**3c. Scan shared agents** (`{shared-path}/agents/*/` or `{shared-path}/agents/*.md`):
- Extract fields:

```
id         — "shared--{agent-name}"
name       — human-readable name from file/directory name or heading
project    — "shared-library"
scope      — "shared"
category   — classify: "architecture", "review", "testing", "build", "docs", "security", "cleanup"
desc       — what the agent does
whenToUse  — when to use this agent
tools      — comma-separated list of tools available
model      — suggested model: "opus", "sonnet", or "haiku" (or "" if unspecified)
```

Show the user: "Shared library: **X commands**, **Y skills**, **Z agents**. Look right?"

---

### Step 4: Scan Each Project

For each project discovered in Step 1:

**4a. Scan project commands** (`.claude/commands/**/*.md`):
- Same extraction fields as Step 2
- `id` format: `"{project-name}--{command-id}"`
- `project`: `"{project-name}"` (directory name of the project)
- `scope`: `"project"`

**4b. Scan project skills** (`.claude/skills/*/`):
- Same extraction fields as Step 3b
- `id` format: `"{project-name}--{skill-id}"`
- `project`: `"{project-name}"`
- `scope`: `"project"`

After scanning ALL projects, show summary:
```
Project scan complete:
  project-a: 12 commands, 5 skills
  project-b: 8 commands, 3 skills
  project-c: 4 commands
  ...
Total: X commands, Y skills across N projects

Look right?
```

Wait for confirmation.

---

### Step 5: Detect Automated Services (optional)

Check for background services running on the system.

**On macOS, check launchd agents:**
```bash
ls ~/Library/LaunchAgents/*.plist 2>/dev/null
```

For each plist found, determine: service name, what it does, running vs scheduled, schedule details.

**On Linux, check crontab:**
```bash
crontab -l 2>/dev/null
```

**On Windows, check scheduled tasks:**
```powershell
Get-ScheduledTask | Where-Object {$_.State -ne 'Disabled'} | Select-Object TaskName, State
```

Build HTML snippets for the services panel using `svc-item` structure (running icon `&#9679;` or scheduled icon `&#9201;`).

If no services found: "No automated services detected. I'll leave that panel out."

---

### Step 6: Define Workflow Chains (optional)

Ask: "Do you have multi-step workflows? For example: 'Plan → Build → Deploy' or 'Research → Write → Publish'?"

If yes, collect: label (1 word, max 8 chars), ordered list of command IDs, optional note text between steps.

Build HTML using `wf-row` structure with `wf-node` spans (class `m`/`o`/`c` matching stream).

If no workflows AND no services: remove the entire `top-panels` grid from the template.

---

### Step 7: Write Config File

Write `cmd-dash-config.json` alongside the output HTML:

```json
{
  "version": 2,
  "generatedAt": "{ISO timestamp}",
  "templatePath": "{absolute path to templates/dashboard-template.html}",
  "outputPath": "{absolute path to output HTML}",
  "configPath": "{absolute path to this config file}",
  "globalCommandsPath": "{absolute path to ~/.claude/commands/}",
  "scanRoots": ["{absolute path to workspace root}"],
  "excludePaths": [],
  "sharedLibraryPath": "{absolute path to shared library, or empty string}",
  "includeArchived": false,
  "includeServices": true,
  "includeWorkflows": true,
  "pageTitle": "{chosen title}",
  "aiName": "{chosen AI name}",
  "theme": {
    "preset": "github-dark",
    "font": "modern",
    "logoPath": "",
    "customColors": null
  }
}
```

Use OS-native absolute paths. This file enables `/cmd-dash-update` to rescan without re-asking questions.

---

### Step 8: Populate Template

Read `templates/dashboard-template.html` from this module folder.

**Make these replacements:**

| Marker | Replace With |
|--------|-------------|
| `__PAGE_TITLE__` | The chosen page title (appears in `<title>` and `<h1>`) |
| `__AI_NAME__` | The chosen AI name (appears in skills banner) |
| `/* __COMMANDS_DATA__ */` + sample array | Real commands array from Steps 2-4 |
| `/* __SKILLS_DATA__ */` + sample array | Real skills array from Steps 3-4 |
| `/* __AGENTS_DATA__ */` + sample array | Real agents array from Step 3 |
| `/* __DASH_CONFIG__ */` + sample object | Real dashConfig with generatedAt, projectCount, pageTitle, aiName |
| `<!-- __WORKFLOW_HTML__ -->` + sample rows | Real workflow HTML from Step 6 (or remove `top-panels` grid) |
| `<!-- __SERVICES_HTML__ -->` + sample items | Real services HTML from Step 5 (or remove `svc-box`) |
| `/* __FONT_IMPORT__ */` + default import | Google Fonts `@import` URL for chosen font preset (or remove import line for `minimal`) |
| `/* __THEME_OVERRIDES__ */` | CSS `:root { }` block overriding colour variables from theme preset (or empty for `github-dark`) |
| `<!-- __LOGO_IMG__ -->` | `<img src="data:image/...;base64,..." alt="Logo">` tag (or remove line if no logo) |
| `<!-- __PROJECT_PILLS__ -->` | Dynamic — built from data at runtime (no replacement needed) |

**Theme preset CSS overrides:**

Each preset generates a `:root { }` block injected at `/* __THEME_OVERRIDES__ */`. The block overrides CSS variables from the default `:root`. A matching `html.light { }` override is also generated for presets that define light-mode variants. If the preset is `custom`, use the hex values from `config.theme.customColors`.

**Font preset replacements:**

| Preset | `@import` URL | `--font` | `--font-display` | `--mono` |
|--------|---------------|----------|-------------------|----------|
| `modern` | Inter + JetBrains Mono (default — no change needed) | `'Inter', …sans-serif` | `'Inter', …sans-serif` | `'JetBrains Mono', …monospace` |
| `classic` | Space Grotesk + IBM Plex Sans + JetBrains Mono | `'IBM Plex Sans', …sans-serif` | `'Space Grotesk', …sans-serif` | `'JetBrains Mono', …monospace` |
| `minimal` | Remove the `@import` line entirely | `system-ui, …sans-serif` | `system-ui, …sans-serif` | `ui-monospace, …monospace` |
| `rounded` | Nunito + JetBrains Mono | `'Nunito', …sans-serif` | `'Nunito', …sans-serif` | `'JetBrains Mono', …monospace` |

**Important replacement notes:**
- The `commands`, `skills`, and `agents` arrays each have comment markers (`/* __COMMANDS_DATA__ */`) on the line above them. Replace from the marker comment through the closing `];` of the array.
- The `dashConfig` object has a marker comment above it. Replace from the marker through the closing `};`.
- For `__PAGE_TITLE__` and `__AI_NAME__`, do a global find-and-replace (they appear multiple times).
- Project pills are built dynamically by the JavaScript `buildProjectPills()` function — no static replacement needed. The `<!-- __PROJECT_PILLS__ -->` comment is just a marker for reference.

**Write the populated file** to the output path (default: `outputs/command-dashboard.html` in this module).

---

### Step 9: Install Global Update Command

Copy the `/cmd-dash-update` command to the user's global commands:

```bash
cp {this-module}/cmd-dash-update.md ~/.claude/commands/cmd-dash-update.md
```

If `cmd-dash-update.md` doesn't exist in this module, create it with the content specified in the project's `cmd-dash-update.md` file.

Tell the user: "Installed `/cmd-dash-update` globally. You can run it any time to rescan your workspace and regenerate the dashboard."

---

### Step 10: Update CLAUDE.md (optional)

If the workspace has a `CLAUDE.md` file at the root, suggest adding:

```markdown
### Command Dashboard

Workspace-wide interactive dashboard at `{output-path}` showing all commands, skills, agents, workflow chains, and automated services across all projects.

Run `/cmd-dash-update` to rescan the workspace and regenerate the dashboard.
```

Ask if they want to add it. If yes, append. If no, skip.

---

## TEST

### Open the Dashboard

Tell the user to open the generated HTML file:

- **macOS**: `open {output-path}`
- **Linux**: `xdg-open {output-path}`
- **Windows**: `start {output-path}`

### Visual Checks

Walk through with the user:

1. **Scope grouping** — Are items grouped as Global > Shared > Projects?
2. **Project pills** — Does clicking a project filter show only its items?
3. **Stream pills** — Do stream pills compose with project filter?
4. **Search** — Does search find commands across all projects?
5. **Commands tab** — Correct count, grouped by scope then stream?
6. **Skills tab** — Correct count, grouped by scope then domain?
7. **Agents tab** — Visible? Shows shared library agents?
8. **Detail panel** — Click a card. Shows project association?
9. **Project badge** — Cards show subtle project origin tag?
10. **Stats** — Topbar numbers update correctly when filtering?
11. **Workflow chains** — If defined, flow diagrams display correctly?

If everything looks good: "Your workspace dashboard is live! Run `/cmd-dash-update` any time to refresh it."

---

## WHAT'S NEXT

1. **Bookmark it** — Pin the HTML file in your browser. It works offline.

2. **Keep it in sync** — Run `/cmd-dash-update` after adding new commands, skills, or projects. It reads the saved config and rescans without re-asking questions.

3. **Customise the look** — CSS variables at the top control all colours. Change `--accent-blue`, `--accent-green`, etc.

4. **Share it** — The template is portable. Anyone can clone this repo, run the installer, and get their own workspace dashboard populated with their data.

---

> Created by Alex McConnachie | [lightleads.com.au](https://lightleads.com.au)
> Workspace-wide scanning by mindm
