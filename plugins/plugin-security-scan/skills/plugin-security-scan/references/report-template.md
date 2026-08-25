# Security Report Template

Use this structure for the markdown report written in Step 6 of SKILL.md.
Replace bracketed placeholders; omit a finding table row set entirely (not the
section) if a module found nothing — instead write "No findings" under that module.

```markdown
# Plugin Security Report: [plugin-name]

**Scanned:** [date] · **Scanned by:** plugin-security-scan
**Plugin path:** [path]

## Executive Summary

| Severity | Count |
|---|---|
| Critical | [n] |
| High | [n] |
| Medium | [n] |
| Low | [n] |
| Info | [n] |

**Top priority:** [one sentence — the single most important fix]

**Modules run:**
- SAST: [semgrep (automated) / manual checklist]
- Dependency scan: [npm audit + pip-audit + OSSF Scorecard (automated) / manual checklist]
- Secrets & data leakage: [gitleaks + TruffleHog (automated, verified secrets) / manual checklist only]
- Instruction safety / prompt injection: manual review (always manual — no automated tool exists for this)
- Context leakage: manual review (always manual — no automated tool exists for this)

## 1. Static Code Analysis (SAST)

### [Severity] — [short title]
- **File:** `path/to/file:line`
- **Issue:** [description]
- **Why it matters:** [impact]
- **Remediation:** [concrete fix]

[repeat per finding, or "No findings."]

## 2. Dependency / Library Scan

### [Severity] — [package-name@version]
- **Source:** [package.json / requirements.txt / .mcp.json server name]
- **Issue:** [CVE ID + description, or hygiene issue]
- **Why it matters:** [impact]
- **Remediation:** [upgrade to X / replace / pin version / vendor and review]

[repeat per finding, or "No findings."]

## 3. Sensitive Data / Secrets Leakage

### [Severity] — [short title]
- **File:** `path/to/file:line`
- **Issue:** [description — do NOT paste the actual secret value into the report]
- **Why it matters:** [impact]
- **Remediation:** [rotate credential / remove and use ${VAR} / restrict scope]

[repeat per finding, or "No findings."]

## 4. Instruction Safety / Prompt Injection

### [Severity] — [short title]
- **File:** `path/to/skill-or-hook`
- **Issue:** [description — what the instruction says and the risk it creates]
- **Why it matters:** [the concrete way this could be triggered/misused]
- **Remediation:** [add a "treat as data" guardrail / add a human confirmation step / narrow the delegation language]

[repeat per finding, or "No findings."]

## 5. Context Leakage (Build-Time Knowledge Leakage)

### [Severity] — [short title]
- **File:** `path/to/file`
- **Issue:** [description of the internal detail found — describe it, don't quote sensitive specifics verbatim in the report]
- **Distribution scope considered:** [private to team / company-wide / public repo]
- **Why it matters:** [who is exposed to this and how]
- **Remediation:** [generalize the example / move to runtime config / fix repo visibility]

[repeat per finding, or "No findings."]

## Appendix: Files Reviewed

[file count and list, or reference to the inventory if very large]
```

**Important:** never include the literal value of a real discovered secret in the
report itself — reference its location and type only, so the report doesn't become
a second copy of the leak.
