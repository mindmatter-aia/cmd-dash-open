---
description: Rescan workspace and regenerate the Command Dashboard
---

# /cmd-dash-update

Reads `cmd-dash-config.json`, rescans all configured paths, and regenerates the dashboard HTML with fresh data.

## Steps

1. **Find config** — Look for `cmd-dash-config.json` in these locations (in order):
   - Path provided as argument: `/cmd-dash-update {path-to-config}`
   - Current working directory: `./cmd-dash-config.json`
   - Alongside the dashboard output: `./outputs/cmd-dash-config.json`
   - Search parent directories (up to 4 levels)
   - If not found, ask the user where it is

2. **Read config** — Load `cmd-dash-config.json` and the template from `templatePath`. If `templatePath` is invalid or the file doesn't exist, fall back to `templates/dashboard-template.html` relative to the config file's directory. Read `globalAgentsPath` from config (may be absent in older configs — treat as empty string).

3. **Scan global commands** — Scan `globalCommandsPath` for all `.md` files
   - Set `scope: "global"`, `project: "global"`
   - Extract: id, name, stream, mode, tagline, desc, when, input, output, outputPath, chains, tags

3b. **Scan global agents** — If `globalAgentsPath` is set and non-empty:
   - Scan for `.md` files and subdirectories in `globalAgentsPath`
   - Set `scope: "global"`, `project: "global"`
   - Extract: id, name, category, desc, whenToUse, tools, model

4. **Scan shared library** — If `sharedLibraryPath` is set and non-empty:
   - Scan `commands/` for `.md` files → `scope: "shared"`, `project: "shared-library"`
   - Scan `skills/*/` for skill directories → same scope/project
   - Scan `agents/*/` or `agents/*.md` for agents → same scope/project

5. **Scan each project** — For each directory in `scanRoots`:
   - Find all subdirectories (max depth 4) containing `.claude/commands/` or `.claude/skills/`
   - Skip anything in `excludePaths`
   - Scan commands and skills with `scope: "project"`, `project: "{dir-name}"`

6. **Populate template** — Replace all placeholder markers in the template:
   - `__PAGE_TITLE__` → `pageTitle` from config
   - `__AI_NAME__` → `aiName` from config
   - `/* __COMMANDS_DATA__ */` + array → real commands data
   - `/* __SKILLS_DATA__ */` + array → real skills data
   - `/* __AGENTS_DATA__ */` + array → real agents data
   - `/* __DASH_CONFIG__ */` + object → real config metadata
   - Workflow and services HTML if `includeWorkflows`/`includeServices` are true

   **Theme injection** (read from `config.theme`):
   - Determine CSS variable overrides from `theme.preset` (or `theme.customColors` if preset is `custom`), including `--glow` for card hover effect
   - Determine Google Fonts `@import` URL from `theme.font` preset
   - `/* __FONT_IMPORT__ */` + default import line → correct `@import` for chosen font (or remove for `minimal`)
   - `/* __THEME_OVERRIDES__ */` → `:root { }` block with CSS variable overrides (empty for `github-dark`)
   - If `theme.logoPath` exists and the file is readable, regenerate `outputs/logo-base64.txt` from the image file
   - `<!-- __LOGO_IMG__ -->` → `<img src="data:image/...;base64,..." alt="Logo">` (or remove if no logo)
   - If the config has no `theme` block (older config), use defaults: `github-dark` preset, `modern` font, no logo

7. **Write output** — Overwrite `outputPath` with the populated HTML

8. **Update config** — Update `generatedAt` timestamp in `cmd-dash-config.json`

9. **Report summary**:
   ```
   Dashboard updated: X commands, Y skills, Z agents across N projects
   Output: {outputPath}
   ```

## Important

- Do NOT re-ask scoping questions — use the saved config
- New projects appearing in `scanRoots` are picked up automatically
- Projects that lost their `.claude/commands/` directory show 0 items (not removed from config)
- The `id` format `{project}--{command-id}` prevents collisions between projects that share command names like `/prime` or `/implement`
