# SAP CPI Groovy — Environment Setup Task List (corrected)

Perform the following tasks in order.

---

## 1. Detect operating system

Detect the operating system currently running: Windows, macOS, or Linux.
Use the detected OS to select the correct command/path variant in every
step below.

---

## 2. Check / install VS Code

Check if VS Code is installed (e.g. `code --version`).

If not installed, download and install it from:
- https://code.visualstudio.com/Download

(Select the OS-appropriate installer: `.exe` for Windows, `.zip`/`.dmg`
for macOS, `.deb`/`.rpm`/tarball for Linux.)

---

## 3. Install the VS Code extension

Install the "SAP CPI Groovy Script" extension using the VS Code CLI
(preferred over manual marketplace install, since it's scriptable and
verifiable):

```
code --install-extension johancalderon.sap-cpi-groovy-script
```

Verify installation:

```
code --list-extensions | grep -i sap-cpi-groovy-script
```

(Extension marketplace page for reference only, not for direct download:
https://marketplace.visualstudio.com/items?itemName=JohanCalderon.sap-cpi-groovy-script)

---

## 4. Download Java JDK and Groovy

**Before downloading, verify the exact version numbers and direct
download URLs are still valid** — these listing/archive pages change
over time and require OS/architecture-specific selection:

- Java JDK 17 (Microsoft Build of OpenJDK): https://learn.microsoft.com/en-us/java/openjdk/download
  - Select the archive matching the detected OS and architecture
    (x64/aarch64).
- Groovy 4.x binary release: https://groovy.apache.org/download.html
  - Use the "Binary Release" `.zip` (not source, not installer) matching
    a current 4.x version.

Download both archives into a temporary working folder, e.g.
`~/tmp-cpi-setup/`.

---

## 5. Extract into `~/programs`

Create `~/programs` if it doesn't exist.

Extract each archive into it:

**macOS / Linux:**
```
mkdir -p ~/programs
tar -xzf <java-archive> -C ~/programs
unzip <groovy-archive>.zip -d ~/programs
```

**Windows (PowerShell):**
```
New-Item -ItemType Directory -Force -Path "$HOME\programs"
Expand-Archive -Path <java-archive>.zip -DestinationPath "$HOME\programs"
Expand-Archive -Path <groovy-archive>.zip -DestinationPath "$HOME\programs"
```

After extraction, confirm the resulting folder names (they typically
include the version number, e.g. `~/programs/jdk-17.0.19+9`,
`~/programs/groovy-4.0.29`) — you'll need these exact paths in Step 6.

---

## 6. Configure VS Code settings.json

Locate `settings.json` for the detected OS:

- **Windows:** `%APPDATA%\Code\User\settings.json`
- **macOS:** `~/Library/Application Support/Code/User/settings.json`
- **Linux:** `~/.config/Code/User/settings.json`

Before adding keys, **verify the exact setting IDs** against the "SAP
CPI Groovy Script" extension's published configuration schema (Extensions
panel → gear icon → Extension Settings, or its marketplace "Feature
Contributions" tab) — do not assume the key names without checking, as
they may differ from what's listed below.

Add (or update) the following keys, using the **root folder** of each
extracted package (the folder that directly contains `bin/` — not the
`bin` folder itself):

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

**Use the raw file URLs** — the `github.com/.../blob/...` URLs return an
HTML wrapper page, not the file content, and must not be used for
download:

- Agent: https://raw.githubusercontent.com/nahhoj/SAP-CPI-AGENTS/master/agent/sap-cpi-groovy.md
- Skill: https://raw.githubusercontent.com/nahhoj/SAP-CPI-AGENTS/master/skill/sap-cpi-groovy-best-practice/SKILL.md

Save both to the temporary working folder from Step 4
(e.g. `~/tmp-cpi-setup/`).

---

## 8. Place the agent and skill in the OpenCode config directory

Determine which `opencode.json` you are configuring (see Step 9) — this
determines the base directory for relative `prompt` paths. This guide
assumes the **global** OpenCode config at `~/.config/opencode/`.

1. Copy the **skill** file as-is to:
   ```
   ~/.config/opencode/skill/sap-cpi-groovy-best-practice/SKILL.md
   ```

2. For the **agent** file: open the downloaded `sap-cpi-groovy.md` and
   **remove its YAML frontmatter block** (everything between the first
   `---` and the second `---`, including both delimiter lines). Keep
   only the body content starting at `# SAP CPI Groovy Agent`.

   This is required because `mode`, `model`, `temperature`, and
   `permission` are defined per-agent in `opencode.json` (Step 9), not
   inside the prompt file. Leaving the frontmatter in would create
   duplicate/conflicting configuration between the two agent entries
   that share this file.

3. Save the stripped body content to:
   ```
   ~/.config/opencode/agent/prompts/sap-cpi-groovy.md
   ```

---

## 9. Add both agents to `opencode.json`

Edit `~/.config/opencode/opencode.json` (create it if it doesn't exist,
using OpenCode's standard top-level structure). Under the `"agent"` key,
add:

```json
{
  "agent": {
    "sap-cpi-groovy": {
      "description": "SAP CPI Groovy specialist for SAP Cloud Integration",
      "mode": "primary",
      "hidden": false,
      "model": "opencode/deepseek-v4-flash-free",
      "temperature": 0.3,
      "prompt": "agent/prompts/sap-cpi-groovy.md",
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
      "prompt": "agent/prompts/sap-cpi-groovy.md",
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

If `opencode.json` already has other top-level keys (e.g. `instructions`,
other agents), merge this into the existing `"agent"` object — do not
overwrite the whole file.

**Note:** the `prompt` path (`agent/prompts/sap-cpi-groovy.md`) is
relative to the location of this `opencode.json` file. Since this is the
global config at `~/.config/opencode/opencode.json`, the resolved path is
`~/.config/opencode/agent/prompts/sap-cpi-groovy.md` — matching Step 8.

---

## 10. Clean up temporary downloads

Delete **only** the temporary installer/archive files — never the
installed program folders or the final config files:

**Delete:**
- The temporary working folder from Steps 4 and 7 (e.g. `~/tmp-cpi-setup/`),
  including the Java archive, Groovy archive, and the original
  (frontmatter-included) copies of the agent/skill files.

**Do NOT delete:**
- `~/programs/<jdk-folder>` and `~/programs/<groovy-folder>` — these are
  the actual installation, required at runtime.
- `~/.config/opencode/agent/prompts/sap-cpi-groovy.md`
- `~/.config/opencode/skill/sap-cpi-groovy-best-practice/SKILL.md`
- `~/.config/opencode/opencode.json`

---

## Summary of fixes applied vs. the original task list

| # | Issue | Fix |
|---|---|---|
| 7/8 | GitHub `blob` URLs return HTML, not file content | Switched to `raw.githubusercontent.com` URLs |
| 7/8 | Agent frontmatter would conflict with Step 9's `opencode.json` config | Step 8 now explicitly strips frontmatter before saving as prompt file |
| 4 | JDK/Groovy links were listing/index pages, not direct downloads, with unverified version numbers | Reworded to require OS/arch selection and version verification before download |
| 2/3/6 | No OS-specific paths/commands given | Added Windows/macOS/Linux variants throughout |
| 3 | Marketplace page isn't a script-friendly install method | Switched to `code --install-extension` CLI |
| 6 | Setting key names (`groovy.java.home` etc.) unverified | Added explicit instruction to confirm against the extension's real config schema first |
| 8 | Skill folder naming convention (`skill/` vs `skills/`) unconfirmed | Flagged as assumption; verify against current OpenCode docs before relying on it |
| 9 | Duplicate step numbering ("9." twice); unclear which `opencode.json` | Renumbered 9/10; explicitly scoped to global config with path-resolution note |
| 9 | Risk of overwriting existing `opencode.json` content | Added explicit merge instruction |
| 10 | "Delete all files downloaded" was ambiguous and could delete the actual install | Explicit keep/delete lists |
