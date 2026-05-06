# Install / Upgrade AOR Skills

> **Usage:** Paste this entire prompt into a **GitHub Copilot CLI** session
> (`copilot` in any terminal, including VS Code's integrated terminal) **or**
> into a **VS Code Copilot CLI session** (Chat view → Session Target →
> "Copilot CLI"). Once installed, skills are automatically available in every
> future Copilot CLI session **and** in VS Code Copilot Chat — no extra steps
> needed. In VS Code Chat, reload the window after install (`Ctrl+Shift+P` →
> "Developer: Reload Window") to pick up the new skills.

---

Please help me install or upgrade skills from the `Agents-On-Rails/aor-skills`
GitHub repository. Follow **every** step below in order and report progress
after each one. Do not combine steps or skip ahead.

---

## Step 1 — Detect OS and resolve install directories

Your environment context includes an `Operating System:` field. Read it:
- If it reads `Windows_NT` → you are on **Windows**. Use PowerShell for all
  shell commands.
- Otherwise → you are on **macOS or Linux**. Use shell (bash/zsh) commands.

If the environment context is unavailable, run a detection command:

**PowerShell (Windows):**
```powershell
$onWindows = ($env:OS -eq 'Windows_NT') -or ($IsWindows -eq $true) -or ($PSVersionTable.Platform -eq 'Win32NT')
Write-Host "onWindows=$onWindows"
```

**Bash (macOS / Linux):**
```bash
echo "OS=$(uname -s)"
```

Based on the result, store these three named variables — you will reference
them by name in every subsequent step:

| Variable | Windows value | macOS / Linux value |
|----------|---------------|---------------------|
| `SKILLS_DIR` | `$env:USERPROFILE\.copilot\skills` | `$HOME/.copilot/skills` |
| `AGENTS_GLOBAL_DIR` | `$env:USERPROFILE\.copilot\agents` | `$HOME/.copilot/agents` |
| `AGENTS_PROJECT_DIR` | `<current working directory>\.claude\agents` | `<current working directory>/.claude/agents` |

> **Note:** `AGENTS_GLOBAL_DIR` makes agents available to VS Code Copilot
> Chat. `AGENTS_PROJECT_DIR` is required by the `/aor-review` skill, which
> discovers agents by globbing `.claude/agents/aor-sme-*.md` relative to the
> working directory. Both will be populated in Step 7.

Report the resolved values of all three variables before continuing.

---

## Step 2 — Determine the version ref

Fetch the tags list — **always use `raw: true`** on every `web_fetch` call
in this prompt:

```
web_fetch(url="https://api.github.com/repos/Agents-On-Rails/aor-skills/tags", raw=true)
```

Parse the returned JSON array:

- **Non-empty array:** Find the entry whose `name` field is the highest
  semantic version (compare `major.minor.patch` numerically; ignore
  pre-release suffixes for ordering). Set:
  - `REF` = that entry's `name` (e.g., `v0.1.0`)
  - `REF_SHA` = that entry's `commit.sha`
- **Empty array:** Set `REF = main` and `REF_SHA = n/a (main branch HEAD)`.

If the fetch returns a **4xx or 5xx status**, stop and report the error with
the full response body before continuing.

Report: `Using ref: <REF>  (SHA: <REF_SHA>)`

---

## Step 3 — Discover available skills and agent files

### 3a — Skills directory listing

Fetch the skills directory listing (`raw: true`):

```
web_fetch(url="https://api.github.com/repos/Agents-On-Rails/aor-skills/contents/.claude/skills?ref=<REF>", raw=true)
```

- **404:** Fall back — retry with `?ref=main`. If still 404, stop and report:
  "Skills directory not found — the repository structure may have changed."
- **4xx / 5xx:** Stop and report the full error.

For each item in the array whose `type` is `"dir"`, that item is one skill.
Note the folder name (e.g., `aor-req`).

### 3b — Fetch each SKILL.md via shell (not web_fetch)

> **Why shell, not `web_fetch`?** `web_fetch` truncates responses at ~5 000
> characters. Several SKILL.md files exceed that limit. Use
> `Invoke-WebRequest` (PowerShell) or `curl` (bash) to fetch file content —
> they return the full body regardless of size.

For each skill folder, fetch the raw SKILL.md into a variable:

**PowerShell:**
```powershell
$remoteSkillContent = @{}
foreach ($skill in $skills) {
    $url = "https://raw.githubusercontent.com/Agents-On-Rails/aor-skills/$REF/.claude/skills/$skill/SKILL.md"
    $remoteSkillContent[$skill] = (Invoke-WebRequest -Uri $url -UseBasicParsing).Content
}
```

**Bash:**
```bash
declare -A remote_skill_content
for skill in "${skills[@]}"; do
    remote_skill_content[$skill]=$(curl -fsSL "https://raw.githubusercontent.com/Agents-On-Rails/aor-skills/$REF/.claude/skills/$skill/SKILL.md")
done
```

From each fetched text, parse the YAML frontmatter (the block between the
first pair of `---` delimiters at the very top). Extract `name:` and
`description:`. If frontmatter is missing or unparseable, use the folder
name as the skill name and "No description available" as the description,
and continue.

### 3c — Agents directory listing

Fetch the agents directory listing (`raw: true`):

```
web_fetch(url="https://api.github.com/repos/Agents-On-Rails/aor-skills/contents/.claude/agents?ref=<REF>", raw=true)
```

Collect the `name` and the raw CDN URL for every item whose `name` ends
in `.md`. Construct raw CDN URLs as:

```
https://raw.githubusercontent.com/Agents-On-Rails/aor-skills/<REF>/.claude/agents/<filename>
```

Store this as `remote_agents[]`.

---

## Step 4 — Check what is already installed locally

For each discovered skill:

1. Check whether `<SKILLS_DIR>/<skill-name>/SKILL.md` exists.
   - Use the `glob` tool with a **forward-slash path** (required by glob on
     all platforms, including Windows):
     `~/.copilot/skills/<skill-name>/SKILL.md`
   - Or use a shell command:
     - **PowerShell:** `Test-Path "$env:USERPROFILE\.copilot\skills\<skill-name>\SKILL.md"`
     - **Bash:** `[ -f "$HOME/.copilot/skills/<skill-name>/SKILL.md" ] && echo exists`

2. If the file exists, read its content.

3. Before comparing, normalize line endings in **both** the local text and
   `remote_skill_content[<skill-name>]`:
   - **PowerShell:** `$text -replace "\r\n", "\n"`
   - **Bash:** `tr -d '\r' <<< "$text"`

4. Compare the normalized strings and classify:
   - ❌ **not installed** — local file not found
   - ✅ **up to date** — normalized strings are equal
   - ⬆️ **upgrade available** — normalized strings differ

---

## Step 5 — Present the skills table

Display:

| Skill | Description | Status |
|-------|-------------|--------|
| `<name>` | `<description>` | ❌ / ✅ / ⬆️ |

Below the table, always show:

> Source: `<REF>` (SHA: `<REF_SHA>`)
>
> Installing or upgrading any skill will also install/upgrade the shared SME
> agent files to:
> - `<AGENTS_GLOBAL_DIR>` (global — VS Code Copilot Chat)
> - `<AGENTS_PROJECT_DIR>` (project-local — required by `/aor-review` in this directory)

**If every skill is already up to date**, add:

> ✅ All skills are already up to date. You can force a re-install to
> overwrite local files with the current remote versions, or cancel.

---

## Step 6 — Ask what to install / upgrade

Use `ask_user` with these choices:

1. **Install / upgrade ALL** — installs or re-installs every skill and all
   agent files, regardless of current status
2. **Choose skills individually** — presents only skills that are "not
   installed" or have an "upgrade available"
3. **Cancel — make no changes**

If the user picks **option 2**, for each skill classified as "not installed"
or "upgrade available", call `ask_user`: "Install/upgrade `<skill-name>`?"
with choices **Yes** / **No**. Do not prompt for up-to-date skills.

If the user picks **option 3**, report "No changes made." and stop.

---

## Step 7 — Install / upgrade chosen skills

Create skill directories and download all files using a single shell loop.
This avoids `web_fetch` truncation and handles both new files and overwrites
in one operation.

**PowerShell:**
```powershell
$REF       = "<REF>"
$skillsDir = "$env:USERPROFILE\.copilot\skills"
$chosen    = @(<comma-separated chosen skill names, e.g. 'aor-req','aor-review'>)

foreach ($skill in $chosen) {
    # Create directory (no-ops if already exists)
    $null = New-Item -ItemType Directory -Force -Path "$skillsDir\$skill"

    # Fetch file listing from Contents API
    $listing = (Invoke-WebRequest -Uri "https://api.github.com/repos/Agents-On-Rails/aor-skills/contents/.claude/skills/$skill`?ref=$REF" -UseBasicParsing).Content |
               ConvertFrom-Json

    foreach ($file in $listing) {
        $content = (Invoke-WebRequest -Uri $file.download_url -UseBasicParsing).Content
        [System.IO.File]::WriteAllText(
            "$skillsDir\$skill\$($file.name)",
            $content,
            [System.Text.UTF8Encoding]::new($false)   # UTF-8 without BOM
        )
        Write-Host "✅ $skillsDir\$skill\$($file.name)"
    }
}
```

**Bash:**
```bash
REF="<REF>"
SKILLS_DIR="$HOME/.copilot/skills"
chosen=(<space-separated chosen skill names, e.g. aor-req aor-review>)

for skill in "${chosen[@]}"; do
    mkdir -p "$SKILLS_DIR/$skill"

    # Fetch file listing and extract download_urls
    curl -fsSL "https://api.github.com/repos/Agents-On-Rails/aor-skills/contents/.claude/skills/$skill?ref=$REF" \
    | python3 -c "import sys,json; [print(f['download_url']) for f in json.load(sys.stdin)]" \
    | while read url; do
        fname=$(basename "$url")
        curl -fsSL "$url" > "$SKILLS_DIR/$skill/$fname"
        echo "✅ $SKILLS_DIR/$skill/$fname"
    done
done
```

If any download or write fails, report the error and use `ask_user` to ask
"Retry or skip `<skill>`?" before continuing.

---

## Step 8 — Install / upgrade agent files

Regardless of which individual skills were selected, install **all** agent
files to **both** agent directories using a single shell loop.

**PowerShell:**
```powershell
$REF          = "<REF>"
$globalDir    = "$env:USERPROFILE\.copilot\agents"
$projectDir   = "$(Get-Location)\.claude\agents"

$null = New-Item -ItemType Directory -Force -Path $globalDir
$null = New-Item -ItemType Directory -Force -Path $projectDir

# Fetch agent listing from Contents API
$agentListing = (Invoke-WebRequest -Uri "https://api.github.com/repos/Agents-On-Rails/aor-skills/contents/.claude/agents?ref=$REF" -UseBasicParsing).Content |
                ConvertFrom-Json | Where-Object { $_.name -like "*.md" }

foreach ($agent in $agentListing) {
    $content = (Invoke-WebRequest -Uri $agent.download_url -UseBasicParsing).Content
    foreach ($dir in @($globalDir, $projectDir)) {
        [System.IO.File]::WriteAllText("$dir\$($agent.name)", $content, [System.Text.UTF8Encoding]::new($false))
        Write-Host "✅ $dir\$($agent.name)"
    }
}
```

**Bash:**
```bash
REF="<REF>"
GLOBAL_DIR="$HOME/.copilot/agents"
PROJECT_DIR=".claude/agents"

mkdir -p "$GLOBAL_DIR" "$PROJECT_DIR"

curl -fsSL "https://api.github.com/repos/Agents-On-Rails/aor-skills/contents/.claude/agents?ref=$REF" \
| python3 -c "import sys,json; [print(f['name'], f['download_url']) for f in json.load(sys.stdin) if f['name'].endswith('.md')]" \
| while read name url; do
    content=$(curl -fsSL "$url")
    printf '%s' "$content" > "$GLOBAL_DIR/$name"  && echo "✅ $GLOBAL_DIR/$name"
    printf '%s' "$content" > "$PROJECT_DIR/$name" && echo "✅ $PROJECT_DIR/$name"
done
```

---

## Step 9 — Report results

List every file written with its full local path. Then output:

```
✅ Skills installed / upgraded  : <comma-separated skill names, or "none">
✅ Agents installed / upgraded  : <comma-separated agent filenames, or "none">
📁 Skills directory             : <SKILLS_DIR>
📁 Global agents directory      : <AGENTS_GLOBAL_DIR>
📁 Project agents directory     : <AGENTS_PROJECT_DIR>
🏷️  Source ref                   : <REF>  (SHA: <REF_SHA>)

Each skill's SKILL.md documents its slash commands and usage.

⚠️  To activate in Copilot CLI   : start a new session (exit and relaunch).
⚠️  To activate in VS Code Chat  : reload the window
     (Ctrl+Shift+P → "Developer: Reload Window").

ℹ️  /aor-review uses agents from .claude/agents/ relative to your working
    directory. To use it in a different project, run this installer from
    that project's root directory.
```

If any files failed to write, list them explicitly and note that those
skills or agents may be partially installed.
