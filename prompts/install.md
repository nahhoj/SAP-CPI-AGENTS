# SAP CPI Groovy — Environment Setup Task List

## ⚙️ Config Variables (set these before running)

```bash
JAVA_VERSION="17.0.19"        # Microsoft Build of OpenJDK
GROOVY_VERSION="2.4.21"       # Primary target runtime
GROOVY_ALT_VERSION="4.0.29"   # Secondary/reference version
```

Use these variables everywhere a version/path is referenced below —
never hardcode a version number directly in a command.

---

## Rules
- Run steps **in order**. If any step fails/can't be verified → **stop, report the exact error**, do not proceed or guess.
- **Idempotent**: each step checks current state before acting — safe to re-run after a partial failure.

---

## 1. Detect OS
```bash
uname -s        # Darwin=macOS, Linux=Linux
```
```powershell
$IsWindows
```
Use result to pick the correct command/path variant in later steps.

## 2. Check/Install VS Code
```bash
code --version
```
If missing: download OS-appropriate installer from https://code.visualstudio.com/Download, install (use silent flags if available), re-verify with `code --version`. Fail → stop, report.

## 3. Check/Install VS Code Extension
```bash
code --list-extensions | grep -i sap-cpi-groovy-script
```
If missing: `code --install-extension johancalderon.sap-cpi-groovy-script`, then re-verify. Still missing → stop, report.
(Marketplace ref only: https://marketplace.visualstudio.com/items?itemName=JohanCalderon.sap-cpi-groovy-script)

## 4. Download JDK + Groovy

⛔ **ABSOLUTE PROHIBITION — applies to this entire step and Step 5:**
**NEVER install** Java or Groovy using any package manager or installer
tool. This includes, without limitation: `brew`, `brew install`, `apt`,
`apt-get`, `yum`, `dnf`, `choco`, `winget`, `sdkman`, `asdf`, or any GUI
installer. **Do not run an install command for Java or Groovy under any
circumstance — even if one of these tools is already available on the
system, and even if the user asks for it.**

**Step 4a — Check each package individually before downloading anything.**
Run this command:
```bash
ls ~/programs
```
Then evaluate the following 3 conditions **one by one**:

| Package | Condition to check | If folder found | If folder NOT found |
|---|---|---|---|
| JDK | Does a folder named exactly `jdk-${JAVA_VERSION}*` exist? | **SKIP downloading JDK.** Go to Step 4b. | **Download JDK** in Step 4c. |
| Groovy (primary) | Does a folder named exactly `groovy-${GROOVY_VERSION}*` exist? | **SKIP downloading Groovy primary.** Go to Step 4b. | **Download Groovy primary** in Step 4c. |
| Groovy (alt) | Does a folder named exactly `groovy-${GROOVY_ALT_VERSION}*` exist? | **SKIP downloading Groovy alt.** Go to Step 4b. | **Download Groovy alt** in Step 4c. |

**Exception:** if the user explicitly asked for a clean reinstall, ignore this table and download all 3 packages regardless of what's found.

**Step 4b — If ALL 3 packages were found:** skip Step 4c and Step 5 entirely for those packages, and go directly to Step 6 using the existing folders. If at least one package was NOT found, continue to Step 4c for only the missing package(s).

**Step 4c — Download only the missing package(s):**
```bash
mkdir -p ~/tmp-cpi-setup
```
**Before downloading**, confirm exact version + URL are still valid on the listing pages (don't assume a hardcoded URL is current):
- JDK `$JAVA_VERSION`: https://learn.microsoft.com/en-us/java/openjdk/download (pick OS/arch archive)
- Groovy `$GROOVY_VERSION` binary `.zip`: https://groovy.jfrog.io/ui/native/dist-release-local/groovy-zips
- Groovy `$GROOVY_ALT_VERSION` binary `.zip`: same URL as above

Download both into `~/tmp-cpi-setup/`. Version unresolvable → stop, report which package.

## 5. Extract to `~/programs`
No `JAVA_HOME` env var. Create `~/programs` if missing. **Skip re-extraction** if a matching folder already exists (`jdk-$JAVA_VERSION*`, `groovy-$GROOVY_VERSION*`, etc.) unless user wants a clean reinstall.

```bash
tar -xzf ~/tmp-cpi-setup/<java-archive> -C ~/programs
unzip ~/tmp-cpi-setup/<groovy-archive>.zip -d ~/programs
```
```powershell
Expand-Archive -Path "$HOME\tmp-cpi-setup\<archive>.zip" -DestinationPath "$HOME\programs" -Force
```
Record exact resulting folder names (e.g. `jdk-$JAVA_VERSION+9`, `groovy-$GROOVY_VERSION`) — needed as-is in Step 6.

## 6. Configure VS Code `settings.json`
- Windows: `%APPDATA%\Code\User\settings.json`
- macOS: `~/Library/Application Support/Code/User/settings.json`
- Linux: `~/.config/Code/User/settings.json`

**Before writing**, verify real setting IDs via Extensions panel → gear icon → Extension Settings; if they differ from below, use the real ones and note the discrepancy.

**Write these keys into the JSON object.** If the file doesn't exist, create it with `{}` first. If the file already has other, unrelated keys, keep them untouched. **If `groovy.java.home` or `groovy.groovy.home` already exist in the file, overwrite their values with the ones below — do not skip them.**

Use the **root folder** of each extracted package (the folder that directly contains `bin/`, not `bin` itself):
- `groovy.java.home` → root folder of the extracted JDK (version `$JAVA_VERSION`)
- `groovy.groovy.home` → root folder of the extracted Groovy. **Default to `$GROOVY_VERSION`** (the primary target runtime) **unless the user explicitly says to use `$GROOVY_ALT_VERSION` instead.**

```json
{
  "groovy.java.home": "<absolute path to extracted JDK root, version $JAVA_VERSION>",
  "groovy.groovy.home": "<absolute path to extracted Groovy root, version $GROOVY_VERSION unless user specified $GROOVY_ALT_VERSION>"
}
```

## 7. Download agent + skill (raw content only)
Use raw file URLs, not `github.com/.../blob/...` (returns HTML, not content):
- Agent: raw URL for `agents/sap-cpi-groovy.md`
- Skill: raw URL for `skills/sap-cpi-groovy-best-practice/SKILL.md`

Save to `~/tmp-cpi-setup/`. Non-200 or HTML instead of markdown → stop, report.

## 8. Place files in OpenCode config (global: `~/.config/opencode/`)
1. Skill → `~/.config/opencode/skills/sap-cpi-groovy-best-practice/SKILL.md` (create dirs as needed)
2. Agent → `~/.config/opencode/agents/sap-cpi-groovy.md` (create dirs as needed)

## 9. Cleanup
**Delete only**: entire `~/tmp-cpi-setup/` (archives + original agent/skill copies).
**Never delete**: `~/programs/<jdk-folder>`, `~/programs/<groovy-folder>`, `~/.config/opencode/agents/.../sap-cpi-groovy.md`, `~/.config/opencode/skills/.../SKILL.md`, `~/.config/opencode/opencode.json`/`.jsonc`.
Validate the JSON/JSONC config is well-formed before marking this step complete.

---

## Key safeguards preserved
- Stop-and-report on any unverifiable/failed step (no silent workarounds)
- Idempotency checks before install/extract/create
- Prefer existing `opencode.json`/`.jsonc` over creating a duplicate
- Version/URL freshness must be confirmed, not assumed
