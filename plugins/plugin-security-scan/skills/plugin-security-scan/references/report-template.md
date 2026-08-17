# Security Report Template

Use this structure for the markdown report written in Step 4 of SKILL.md.
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
- Dependency scan: [npm audit + pip-audit (automated) / manual checklist]
- Secrets & data leakage: [gitleaks + manual checks (automated) / manual checklist only]

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

## Appendix: Files Reviewed

[file count and list, or reference to the inventory if very large]
```

**Important:** never include the literal value of a real discovered secret in the
report itself — reference its location and type only, so the report doesn't become
a second copy of the leak.
