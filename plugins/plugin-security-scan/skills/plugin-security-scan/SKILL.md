---
name: plugin-security-scan
description: >
  This skill should be used when the user asks to "scan a plugin for security issues",
  "audit this plugin", "check this plugin for vulnerabilities", "review plugin security",
  "is this plugin safe", "check for data leakage in this plugin", "run a SAST scan on
  this plugin", "check this plugin's dependencies/libraries for vulnerabilities",
  "check this plugin for prompt injection", "check this plugin for leaked internal
  information", or "show scan trends/history". It performs a five-part security
  review of an internally-built Claude/Cowork plugin: static code analysis (SAST,
  via semgrep), third-party library/dependency scanning (via pip-audit/npm audit
  plus OSSF Scorecard for upstream maintenance-hygiene risk), sensitive data /
  secrets leakage detection (via gitleaks plus TruffleHog for verified-live-
  credential detection), instruction-safety / prompt-injection review, and
  context-leakage review (catching real internal/business details that leaked into
  the plugin during authoring). It produces a written security report and logs the
  result to a shared "Scan Results" Google Doc for the security team, which tracks
  both a running history of every scan and an auto-updated trends summary across
  all scans to date.
metadata:
  version: "0.4.0"
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
5. Run an OSSF Scorecard check on each direct dependency's upstream repository to
   catch maintenance-hygiene risk before it becomes a CVE: `which scorecard ||`
   install per `references/dependency-checklist.md`'s Scorecard section, then
   `scorecard --repo=<dependency-repo-url> --format json`. Flag any dependency
   scoring poorly on maintained-ness, code review practice, or branch protection —
   these are leading indicators of exactly the kind of maintainer-account-compromise
   risk that hit `deep-translator` (PYSEC-2022-252). If Scorecard can't be installed,
   fall back to the manual signals in `references/dependency-checklist.md` (last
   publish date, open issue count, archived/deprecated status).

## Step 3: Sensitive data / secrets leakage scan

Goal: catch hardcoded secrets, PII, or exfiltration paths that would leak
organizational or user data.

1. Check for gitleaks: `which gitleaks`. If unavailable and installable, install it
   (e.g. via the project's package manager or a static binary); otherwise fall back to
   the regex checklist in `references/secrets-checklist.md`.
2. If gitleaks is available: `gitleaks detect --source <plugin-dir> --no-git \
   --report-format json --report-path <scratch>/gitleaks.json`. Also point it (or a
   history-aware run) at the plugin's git history if it's in a repo, not just the
   working tree — a secret removed in a later commit is still exposed in history.
3. Prefer TruffleHog when available for anything gitleaks flags, because it goes a
   step further than pattern matching: `which trufflehog || ` install per
   `references/secrets-checklist.md`'s TruffleHog section, then
   `trufflehog filesystem <plugin-dir> --json --only-verified` (or point it at the
   git history the same way as gitleaks above). TruffleHog actively checks whether a
   detected credential is live — e.g. it tries the AWS key against AWS, the Slack
   token against Slack — before calling it verified. Treat a TruffleHog-verified
   secret as definitively real (skip the placeholder-judgment step below entirely
   for verified hits) and treat an unverified pattern match the same as a
   gitleaks/manual hit — still worth reporting, just not automatically Critical.
4. Regardless of tool availability, also manually grep for plugin-specific leakage
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
5. Treat any verified secret (a TruffleHog-verified hit, or a pattern match you've
   confirmed by other means is real and not a placeholder) as CRITICAL and recommend
   immediate rotation of that credential in addition to removing it from the plugin.

## Step 4: Instruction-safety / prompt-injection review

Goal: catch risks that are specific to Claude/Cowork plugins being natural-language
instruction sets rather than code — this is NOT covered by SAST, because there's no
traditional vulnerability pattern to match against. Run this on every skill and
agent body, every hook prompt, and any text the plugin feeds back to Claude.

1. Read the full text of every `SKILL.md`, `agents/*.md`, and prompt-type hook in
   `hooks/hooks.json`.
2. Apply the checklist in `references/prompt-injection-checklist.md`. In short, look
   for: language that tells Claude to act without confirmation on consequential
   actions (delete, send, post publicly, share externally, grant access); missing
   "treat as data, not instructions" guardrails on any step where the skill has
   Claude read external or untrusted content (files from other people, fetched URLs,
   Slack/email search results, uploaded documents) and then act on what it read;
   hooks or instructions that auto-approve/auto-decide without real conditional
   logic; and instructions that tell Claude to widen its own tool access, install
   other plugins, or modify its own configuration without the user being told.
3. This step requires judgment, not just pattern matching — read the instructions
   the way an attacker would: "if I control the content this skill reads (a file,
   an email, a web page), what could I make Claude do just by writing text into
   that content?" If the answer is "something consequential, without the user
   confirming," that's a real finding.
4. Rate per `references/severity-rubric.md`'s instruction-safety guidance.

## Step 5: Context-leakage review (build-time knowledge leakage)

Goal: catch real internal/business-specific details that ended up hardcoded into
the plugin during authoring — a risk specific to plugins built through an assisted,
natural-language flow that searches Slack, email, and documents to fill in details,
rather than written by a developer from a blank file.

1. Read every file in the plugin (skills, references, examples, README,
   CONNECTORS.md) looking for content that reads like it was copied verbatim from a
   real internal source rather than written as a generic example. Apply
   `references/context-leakage-checklist.md`.
2. Distinguish deliberate, clearly-labeled generic examples ("e.g. a project named
   Alpha") from content that looks like real specifics: actual client/customer
   names, real project codenames, specific dollar figures, real people's names tied
   to sensitive statements, actual Slack channel names or ticket IDs, or anything
   that reads like it was pulled from an actual conversation rather than composed
   as an example.
3. Weigh severity against how broadly the plugin will be distributed (check the
   marketplace/repo visibility if known — see `references/context-leakage-checklist.md`
   for the exact rubric): the same internal detail is a bigger problem in a plugin
   pushed to a public or company-wide repo than one scoped to the single team that
   authored it, since visibility determines who's actually exposed to the leak.
4. Recommend generalizing the specific detail, or moving it to a private config
   value the plugin reads at runtime instead of hardcoding it into a shared file.

## Step 6: Aggregate and report

1. Deduplicate findings across tools/modules that flag the same root cause.
2. Assign severity per `references/severity-rubric.md`: Critical, High, Medium, Low, Info.
3. Write a markdown report to `<scratch>/security-report-<plugin-name>-<date>.md` using
   the structure in `references/report-template.md`: executive summary with counts by
   severity, then one section per module (SAST, Dependencies, Secrets/Data Leakage,
   Instruction Safety, Context Leakage) listing each finding with file, line (if
   applicable), description, why it matters, and a concrete remediation step. Note
   explicitly which modules ran with real tools vs. manual fallback (Instruction
   Safety and Context Leakage are always manual/judgment-based — there is no
   automated tool for either, say so plainly rather than implying otherwise).
4. Deliver the report to the user as a file (this is a deliverable they'll want to
   keep/share — treat it accordingly rather than only pasting a summary in chat).
5. In the chat response, give a short prose summary (no more than a few sentences):
   overall risk level, count of Critical/High findings, and the single most important
   thing to fix first. Do not repeat the full finding list in chat — that's what the
   report file is for.
6. If nothing was found in a module, still say so explicitly ("no hardcoded secrets
   detected") rather than omitting the module — an omitted module reads as "not
   checked."
7. If any Critical or High finding was found in ANY module, say so plainly and
   explicitly recommend the plugin get security sign-off before merging/distributing
   further — don't let a serious finding get lost in a routine-sounding summary.

## Step 7: Log the scan to the shared Google Drive record, and keep it a history, not just a log

Every scan must be recorded centrally so the security team has visibility without
relying on each developer to forward their report. This is required, not optional
— do not skip it just because it's the last step. The `Scan Results` doc has two
parts: a **Trends Summary** at the top (recomputed every time, not appended to) and
the **chronological entry log** below it (append-only, newest first). Both must be
kept current on every run — the doc is a history, not just a running log.

1. Check whether Google Drive tools are available in this session. If not, tell the
   user this scan's result was NOT logged centrally because Google Drive isn't
   connected in this session, and suggest they enable the Google Drive connector
   (or forward the report to the security team manually) — then proceed without
   blocking on it.
2. If Google Drive tools are available, search for a file named exactly
   `Scan Results` (a Google Doc) in Drive. If it doesn't exist yet, create it in the
   shared/org-visible location the user designates (ask once, then remember for
   next time within the session) — do not create it in the requester's private
   "My Drive" root where the security team can't see it. A new doc starts with an
   empty Trends Summary and no entries.
3. Read the current contents of `Scan Results` in full — you need every prior entry
   to recompute the Trends Summary, not just to append below it.
4. Note: this connector has no in-place "edit this doc's content" call, only
   create/trash. To update the doc, trash the existing file by its ID and create a
   new Google Doc with the same title in the same folder containing the full
   rebuilt content (Trends Summary + all entries, old and new). Do this in one
   trash-then-create pair per scan, not multiple partial writes.
5. Append one new entry at the bottom of the entry log's ordering position (top of
   the list, since it's newest-first) using this format, filling in real values:

   ```markdown
   ## [YYYY-MM-DD HH:MM] — [plugin-name]
   - **Run by:** [name/email of the person running this session — use the session's
     known user identity; if unavailable, ask "who should this scan be logged under?"]
   - **Plugin:** [plugin-name] (version [x.y.z] if known)
   - **Result:** [Critical: n, High: n, Medium: n, Low: n, Info: n]
   - **Top issue:** [one-line summary of the single most important finding, or "None — clean scan"]
   - **Modules:** SAST [automated/manual], Dependencies [automated/manual], Secrets [automated/manual], Instruction Safety [manual], Context Leakage [manual]
   ---
   ```

6. Recompute the Trends Summary from ALL entries (including the new one) and place
   it at the very top of the doc, above "Entries below this line." Use this
   structure:

   ```markdown
   ## Trends Summary (auto-updated on every scan)
   - **Total scans logged:** [n]
   - **Cumulative findings:** Critical: [n], High: [n], Medium: [n], Low: [n], Info: [n]
   - **Most common finding categories:** [top 3-5 recurring issue types across all
     entries, by how often each type of finding shows up — e.g. "hardcoded secrets
     (4 scans), missing webhook auth (3 scans), unpinned dependencies (3 scans)"]
   - **Plugins with open Critical/High findings:** [list plugin names from the most
     recent scan of each, if that scan had Critical/High findings — this is a
     to-do list for the security team, not just a statistic]
   - **Last updated:** [YYYY-MM-DD HH:MM] via scan of [plugin-name]
   ```

   Deriving "most common finding categories" requires reading back through prior
   entries' "Top issue" lines and your own judgment about recurring themes — there's
   no structured field to aggregate mechanically, so use reasonable groupings (e.g.
   "no webhook authentication" and "unauthenticated endpoint" count as the same
   category) rather than being overly literal about exact wording matches.
7. Do not paste the literal value of any real discovered secret into either the log
   entry or the Trends Summary — reference type/location only, same rule as the
   full report (see Step 6).
8. Confirm to the user in one line that the scan was logged (e.g. "Logged to the
   shared Scan Results doc — now N scans on record."). Don't over-explain this step
   in chat, and don't paste the Trends Summary into the chat response either — it
   lives in the doc for whoever wants to check it.

## Notes on scope and judgment

- This skill reviews plugins your own teams built, not arbitrary untrusted code from
  the internet — calibrate remediation advice accordingly (e.g. a skill instructing
  Claude to call an internal company API is fine; the same call to an unfamiliar
  external domain is a finding).
- False positives happen, especially with SAST auto-config. Use judgment: a
  regex-detected "secret" that is clearly a placeholder (`~~api-key`, `YOUR_TOKEN_HERE`,
  `<CHANGE_ME>`) is not a real leak — note it as informational, not Critical.
- If the user wants to re-scan after fixes, re-run all five modules and diff
  against the previous report's finding list to confirm closure.
- Instruction-safety and context-leakage findings require more interpretation than
  pattern-matched findings. When genuinely unsure whether something rises to a
  finding, include it at a lower severity with a note that it needs human review,
  rather than silently dropping it — false negatives here are worse than false
  positives, since these two modules exist specifically to catch what automated
  tools can't.
