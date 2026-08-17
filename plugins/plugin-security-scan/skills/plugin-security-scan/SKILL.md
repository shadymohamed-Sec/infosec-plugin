---
name: plugin-security-scan
description: >
  This skill should be used when the user asks to "scan a plugin for security issues",
  "audit this plugin", "check this plugin for vulnerabilities", "review plugin security",
  "is this plugin safe", "check for data leakage in this plugin", "run a SAST scan on
  this plugin", or "check this plugin's dependencies/libraries for vulnerabilities".
  It performs a three-part security review of an internally-built Claude/Cowork plugin:
  static code analysis (SAST), third-party library/dependency scanning, and sensitive
  data / secrets leakage detection, and produces a written security report.
metadata:
  version: "0.1.0"
  author: "Si-Vision Security Team"
---

# Plugin Security Scan

Perform a structured security review of a Claude/Cowork plugin directory (skills,
agents, hooks, MCP configs, and any bundled scripts). Produce a single markdown
report with findings grouped by module and ranked by severity. Never silently skip
a module — if a tool is unavailable, say so in the report and fall back to the
manual checklist for that module.

## Step 0: Locate and inventory the target

1. Determine the plugin directory the user means. If ambiguous, ask which plugin/path
   to scan (or check `mnt/.local-plugins`, `mnt/.plugins`, `~/.claude/plugins/synced`,
   or an uploaded/attached folder).
2. Confirm `.claude-plugin/plugin.json` exists — if not, warn the user this may not be
   a valid plugin directory, but continue scanning whatever code is present.
3. Build a file inventory: `find <plugin-dir> -type f \( -name "*.md" -o -name "*.json" \
   -o -name "*.js" -o -name "*.ts" -o -name "*.py" -o -name "*.sh" -o -name "*.yml" \
   -o -name "*.yaml" \) -not -path "*/node_modules/*"`. Note total file count and any
   package manifests found (`package.json`, `requirements.txt`, `pyproject.toml`,
   `Pipfile`, `.mcp.json`).
4. Create a scratch directory for scan output, e.g. `/tmp/scan-<plugin-name>/`.

## Step 1: SAST — static code analysis

Goal: find insecure coding patterns in any executable code the plugin ships (hook
scripts, MCP server code, utility scripts referenced by skills/agents/hooks).
Markdown-only skills with no code have nothing to SAST — note that and skip to Step 2.

1. Check for semgrep: `which semgrep || pip install semgrep --break-system-packages`.
2. If semgrep is available, run it with the auto config plus security-focused rulesets
   appropriate to the languages present:
   ```
   semgrep --config auto --config p/security-audit --config p/secrets \
     --json --output <scratch>/semgrep.json <plugin-dir>
   ```
   Scope to relevant languages if the plugin is small (`--config p/javascript`,
   `--config p/python`, `--config p/bash`, etc.) to keep the scan fast.
3. If semgrep cannot be installed (no network / restricted environment), fall back to
   the manual checklist in `references/sast-checklist.md` and clearly mark the report
   section as "manual review — automated SAST unavailable."
4. Parse results and keep findings with severity ERROR/WARNING; deduplicate by
   rule + file + line. Drop pure style/INFO findings unless they touch security rules.
5. Pay special attention (per `references/sast-checklist.md`) to plugin-specific risks:
   command injection in hook `command` strings, unsafe use of `eval`/`exec`/`subprocess`
   with untrusted input, hardcoded absolute paths instead of `${CLAUDE_PLUGIN_ROOT}`,
   hooks that auto-approve (`"decision": "approve"`) without real validation, and
   overly broad `allowed-tools` / tool grants (e.g. unrestricted `Bash` on a skill that
   only needs `Read`).

## Step 2: Dependency / library scanning

Goal: flag vulnerable, abandoned, or unnecessarily risky third-party packages the
plugin pulls in (via `.mcp.json` stdio servers, `package.json`, `requirements.txt`,
`pyproject.toml`, or scripts that `pip install`/`npm install` at runtime).

1. For each manifest found:
   - `package.json` → `npm audit --json` (run inside the plugin dir; if no lockfile,
     run `npm install --package-lock-only` first so audit has a lockfile to check).
   - `requirements.txt` / `pyproject.toml` → `pip install pip-audit --break-system-packages`
     then `pip-audit -r requirements.txt -f json` (or `pip-audit` against the resolved
     environment for `pyproject.toml`).
   - No manifest but scripts run `pip install X` / `npm install X` inline → extract
     those package names and check them individually with the same tools, and flag
     runtime installation itself as a finding (supply-chain risk — pinned, reviewed
     dependencies are safer than installing at run time).
2. If a scanner can't be installed, fall back to `references/dependency-checklist.md`
   for manual review (check for unpinned versions, typosquatting risk, packages with
   very low download counts or no recent maintenance, and licenses incompatible with
   internal use).
3. Also manually check every `.mcp.json` server entry: is it a well-known/reputable
   server, does it require credentials passed as plain env vars vs. `${VAR}`
   expansion, is the URL (for `sse`/`http` servers) pinned to HTTPS and a domain the
   org recognizes.
4. Record CVE IDs, severity, affected package + version, and fixed version (if any)
   for every finding.

## Step 3: Sensitive data / secrets leakage scan

Goal: catch hardcoded secrets, PII, or exfiltration paths that would leak
organizational or user data.

1. Check for gitleaks: `which gitleaks`. If unavailable and installable, install it
   (e.g. via the project's package manager or a static binary); otherwise fall back to
   the regex checklist in `references/secrets-checklist.md`.
2. If gitleaks is available: `gitleaks detect --source <plugin-dir> --no-git \
   --report-format json --report-path <scratch>/gitleaks.json`.
3. Regardless of tool availability, also manually grep for plugin-specific leakage
   patterns per `references/secrets-checklist.md`, including:
   - Hardcoded API keys, tokens, passwords, private keys, connection strings.
   - Skill/agent/hook instructions that tell Claude to send file contents, credentials,
     or user data to an external URL, webhook, or third-party MCP server without the
     user's explicit awareness.
   - `.mcp.json` entries pointing at non-HTTPS or unfamiliar external domains.
   - Logging or debug code that writes full request/response bodies (which may
     contain secrets or PII) to persistent files or external services.
   - Overly broad file access (skills reading arbitrary paths outside their working
     directory, e.g. `~/.ssh`, `~/.aws`, browser cookie stores) without a clear
     legitimate purpose tied to the skill's stated function.
4. Treat any verified secret (not just a pattern match) as CRITICAL and recommend
   immediate rotation of that credential in addition to removing it from the plugin.

## Step 4: Aggregate and report

1. Deduplicate findings across tools/modules that flag the same root cause.
2. Assign severity per `references/severity-rubric.md`: Critical, High, Medium, Low, Info.
3. Write a markdown report to `<scratch>/security-report-<plugin-name>-<date>.md` using
   the structure in `references/report-template.md`: executive summary with counts by
   severity, then one section per module (SAST, Dependencies, Secrets/Data Leakage)
   listing each finding with file, line (if applicable), description, why it matters,
   and a concrete remediation step. Note explicitly which modules ran with real tools
   vs. manual fallback.
4. Deliver the report to the user as a file (this is a deliverable they'll want to
   keep/share — treat it accordingly rather than only pasting a summary in chat).
5. In the chat response, give a short prose summary (no more than a few sentences):
   overall risk level, count of Critical/High findings, and the single most important
   thing to fix first. Do not repeat the full finding list in chat — that's what the
   report file is for.
6. If nothing was found in a module, still say so explicitly ("no hardcoded secrets
   detected") rather than omitting the module — an omitted module reads as "not
   checked."

## Notes on scope and judgment

- This skill reviews plugins your own teams built, not arbitrary untrusted code from
  the internet — calibrate remediation advice accordingly (e.g. a skill instructing
  Claude to call an internal company API is fine; the same call to an unfamiliar
  external domain is a finding).
- False positives happen, especially with SAST auto-config. Use judgment: a
  regex-detected "secret" that is clearly a placeholder (`~~api-key`, `YOUR_TOKEN_HERE`,
  `<CHANGE_ME>`) is not a real leak — note it as informational, not Critical.
- If the user wants to re-scan after fixes, re-run the same three steps and diff
  against the previous report's finding list to confirm closure.
