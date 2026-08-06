# SAP CPI Groovy — Environment Setup Task List

Perform the following tasks in order. If any step fails or cannot be
verified, **stop and report the exact error to the user** — do not
proceed to later steps that depend on it, and do not silently guess.

This list is safe to re-run: each step checks current state before
acting, so a partial/failed prior run can be resumed without manual
cleanup.

---

## 1. Detect operating system

Run the OS-detection command appropriate to your execution shell:

**Bash (macOS/Linux):**
```bash
uname -s
# Darwin => macOS, Linux => Linux
```

**PowerShell (Windows):**
```powershell
$IsWindows   # or: [System.Environment]::OSVersion.Platform
```

Store the result and use it to select the correct command/path variant
in every step below.

---

## 2. Check / install VS Code

Check if VS Code is installed:

```bash
code --version
```

If the command is not found:

1. Download the OS-appropriate installer from:
   https://code.visualstudio.com/Download
   (`.exe` Windows / `.zip`,`.dmg` macOS / `.deb`,`.rpm`,tarball Linux)
2. Install it using the OS-standard method (silent/unattended flags if
   available, e.g. `/VERYSILENT` on the Windows installer).
3. Re-run `code --version` to confirm success. If it still fails, stop
   and report — do not proceed to Step 3.

---

## 3. Install the VS Code extension

Check if already installed:

```bash
code --list-extensions | grep -i sap-cpi-groovy-script
```

If not listed, install it:

```bash
code --install-extension johancalderon.sap-cpi-groovy-script
```

Re-run the `--list-extensions` check to confirm. If it's still not
listed after install, stop and report the error output — do not proceed.

(Marketplace page for reference only, not for direct download:
https://marketplace.visualstudio.com/items?itemName=JohanCalderon.sap-cpi-groovy-script)

---

## 4. Download Java JDK and Groovy

Create the temporary working folder first (if it doesn't already exist):

```bash
mkdir -p ~/tmp-cpi-setup
```
```powershell
New-Item -ItemType Directory -Force -Path "$HOME\tmp-cpi-setup"
```

**Before downloading, confirm the exact version and direct file URL are
still valid** by checking the listing page — do not assume a hardcoded
URL is current:

- Java JDK 17.0.19 (Microsoft Build of OpenJDK): https://learn.microsoft.com/es-es/java/openjdk/older-releases or https://learn.microsoft.com/en-us/java/openjdk/download
  — select the archive matching the OS/architecture detected in Step 1.
- Groovy 4.0.29 binary release: https://groovy.apache.org/download.html
  — use the "Binary Release" `.zip` (not source, not installer/SDK).

**If a listed version cannot be confirmed as currently available,**
stop and report which package/version could not be resolved — do not
substitute an unverified URL.

Download both archives into `~/tmp-cpi-setup/`.

---

## 5. Extract into `~/programs`

Create `~/programs` if it doesn't exist. Before extracting, check
whether a folder matching the package (e.g. `jdk-17*`, `groovy-4*`)
already exists there — if so, skip re-extraction for that package
(idempotency) unless the user asked for a clean reinstall.

**macOS / Linux:**
```bash
mkdir -p ~/programs
tar -xzf ~/tmp-cpi-setup/<java-archive> -C ~/programs
unzip ~/tmp-cpi-setup/<groovy-archive>.zip -d ~/programs
```

**Windows (PowerShell):**
```powershell
New-Item -ItemType Directory -Force -Path "$HOME\programs"
Expand-Archive -Path "$HOME\tmp-cpi-setup\<java-archive>.zip" -DestinationPath "$HOME\programs" -Force
Expand-Archive -Path "$HOME\tmp-cpi-setup\<groovy-archive>.zip" -DestinationPath "$HOME\programs" -Force
```

After extraction, record the exact resulting folder names (they
typically include the version number, e.g. `~/programs/jdk-17.0.19+9`,
`~/programs/groovy-4.0.29`) — required as-is in Step 6.

---

## 6. Configure VS Code settings.json

Locate `settings.json` for the OS detected in Step 1:

- **Windows:** `%APPDATA%\Code\User\settings.json`
- **macOS:** `~/Library/Application Support/Code/User/settings.json`
- **Linux:** `~/.config/Code/User/settings.json`

**Before adding keys, verify the exact setting IDs** against the "SAP
CPI Groovy Script" extension's published configuration (VS Code:
Extensions panel → gear icon on the installed extension → Extension
Settings). If the setting IDs below don't match what the extension
actually exposes, use the extension's real IDs instead and note the
discrepancy when reporting completion.

Merge the following keys into the existing JSON object (create the file
with `{}` first if it doesn't exist; do not overwrite other existing
settings). Use the **root folder** of each extracted package — the
folder that directly contains `bin/`, not `bin` itself:

```json
{
  "groovy.java.home": "<absolute path to extracted JDK root>",
  "groovy.groovy.home": "<absolute path to extracted Groovy root>"
}
```

Example (macOS/Linux):
```json
{
  "groovy.java.home": "/Users/<you>/programs/jdk-17.0.19+9",
  "groovy.groovy.home": "/Users/<you>/programs/groovy-4.0.29"
}
```

---

## 7. Download the agent and skill files (raw content, not GitHub HTML pages)

**Use the raw file URLs** — `github.com/.../blob/...` URLs return an
HTML wrapper page, not the file content:

- Agent: https://github.com/nahhoj/SAP-CPI-AGENTS/blob/master/agent/prompts/sap-cpi-groovy.md
- Skill: https://github.com/nahhoj/SAP-CPI-AGENTS/blob/master/skill/sap-cpi-groovy-best-practice/SKILL.md

Save both to `~/tmp-cpi-setup/` (created in Step 4). If either fetch
returns a non-200 status or HTML content instead of markdown, stop and
report — do not proceed to Step 8 with bad content.

---

## 8. Place the agent and skill in the OpenCode config directory

This guide targets the **global** OpenCode config at `~/.config/opencode/`.

1. Copy the **skill** file as-is to:
   ```
   ~/.config/opencode/skills/sap-cpi-groovy-best-practice/SKILL.md
   ```
   (create parent directories as needed)

2. For the **agent** file: open the downloaded `sap-cpi-groovy.md` and
   **remove its YAML frontmatter block** — everything between the first
   `---` and the second `---` line, including both delimiter lines.
   Keep only the body content starting at `# SAP CPI Groovy Agent`.

   This is required because `mode`, `model`, `temperature`, and
   `permission` are defined per-agent in `opencode.json` (Step 9), not
   inside the prompt file. Leaving the frontmatter in creates
   duplicate/conflicting configuration between the two agent entries
   that share this file.

3. Save the stripped body content to:
   ```
   ~/.config/opencode/prompts/sap-cpi-groovy.md
   ```
   (create parent directories as needed)

---

## 9. Add both agents to the OpenCode config

**First, determine which config file to use — do not create a second
one if one already exists:**

```bash
ls ~/.config/opencode/opencode.jsonc ~/.config/opencode/opencode.json 2>/dev/null
```

- If `opencode.jsonc` exists, edit that file.
- Else if `opencode.json` exists, edit that file.
- Else, create `opencode.json` (the current recommended default when
  starting fresh — comments aren't needed for this config).

Merge the following into the `"agent"` object of whichever file you're
using. **Do not overwrite the whole file** — if it already has other
top-level keys or agents, preserve them and merge this in.

```json
{
  "agent": {
    "sap-cpi-groovy": {
      "description": "SAP CPI Groovy specialist for SAP Cloud Integration",
      "mode": "primary",
      "hidden": false,
      "model": "opencode/deepseek-v4-flash-free",
      "temperature": 0.3,
      "prompt": "prompts/sap-cpi-groovy.md",
      "permission": {
        "read": "allow",
        "edit": "allow",
        "bash": "allow",
        "glob": "allow",
        "grep": "allow",
        "webfetch": "allow"
      }
    },
    "sap-cpi-groovy-plan": {
      "description": "SAP CPI Groovy planning specialist — reviews and designs without writing or executing",
      "mode": "primary",
      "hidden": false,
      "model": "opencode/deepseek-v4-flash-free",
      "temperature": 0.2,
      "prompt": "prompts/sap-cpi-groovy.md",
      "permission": {
        "read": "allow",
        "edit": "deny",
        "bash": "deny",
        "glob": "allow",
        "grep": "allow",
        "webfetch": "allow"
      }
    }
  }
}
```

**Note:** the `prompt` path (`agents/prompts/sap-cpi-groovy.md`) is
relative to the location of the config file you just edited. Since this
is the global config at `~/.config/opencode/`, the resolved path is
`~/.config/opencode/agents/prompts/sap-cpi-groovy.md` — matching Step 8.

After saving, validate the file is syntactically valid JSON/JSONC before
finishing (e.g. `cat <file> | python3 -m json.tool` for `.json`, or
OpenCode's own config validation if available). If invalid, stop and
report — do not leave a broken config in place.

---

## 10. Clean up temporary downloads

Delete **only** the temporary installer/archive files — never the
installed program folders or the final config files:

**Delete:**
- The entire `~/tmp-cpi-setup/` folder, including the Java archive,
  Groovy archive, and the original (frontmatter-included) copies of the
  agent/skill files.

**Do NOT delete:**
- `~/programs/<jdk-folder>` and `~/programs/<groovy-folder>` — the
  actual installation, required at runtime.
- `~/.config/opencode/agents/prompts/sap-cpi-groovy.md`
- `~/.config/opencode/skills/sap-cpi-groovy-best-practice/SKILL.md`
- `~/.config/opencode/opencode.json` (or `.jsonc`, whichever was used)

---

## Summary of fixes applied (this revision)

| # | Issue | Fix |
|---|---|---|
| 9 | Blindly created `opencode.json` even if `.jsonc` already existed | Added explicit check-and-prefer-existing-file logic before edit/create |
| 4, 6 | "Verify" steps had no defined fallback if verification failed | Added explicit stop-and-report instruction on verification failure |
| 5, 3, 9 | No idempotency — re-running could error or duplicate work | Added existence checks before install/extract/create actions |
| 1 | OS detection had no concrete command | Added literal `uname -s` / `$IsWindows` commands |
| 2, 3, 7, 9 | No defined behavior on failure (download/install/fetch errors) | Added explicit "stop and report, do not proceed" at each risk point |
| 4 | Temp folder was referenced but never explicitly created | Added explicit `mkdir` as the first action in Step 4 |
| 9 | No post-write validation of the JSON/JSONC config | Added a validation check before considering Step 9 complete |
