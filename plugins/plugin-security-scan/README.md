# Plugin Security Scan

A Cowork plugin for Si-Vision's security team to audit internally-built plugins
before they're rolled out to other teams.

## Overview

Runs a three-part security review of a Claude/Cowork plugin directory and produces
a single markdown report with findings ranked by severity:

1. **Static code analysis (SAST)** — insecure coding patterns in any bundled
   scripts, hook commands, or MCP server code (command injection, unsafe eval/exec,
   path traversal, overly broad tool grants, auto-approving hooks).
2. **Dependency / library scan** — vulnerable, unpinned, unmaintained, or
   typosquatted third-party packages pulled in via `package.json`,
   `requirements.txt`/`pyproject.toml`, or MCP server dependencies.
3. **Sensitive data / secrets leakage** — hardcoded credentials, private keys,
   webhook URLs, and instructions or hooks that could exfiltrate file contents,
   credentials, or user data to unrecognized external destinations.

## Components

| Component | Purpose |
|---|---|
| Skill: `plugin-security-scan` | On-demand security audit of a plugin directory; produces a markdown report |

## Setup

The skill prefers real scanning tools and will attempt to install them on demand:

- [semgrep](https://semgrep.dev/) (`pip install semgrep`) for SAST
- `npm audit` (bundled with npm) and [pip-audit](https://pypi.org/project/pip-audit/) for dependency scanning
- [gitleaks](https://github.com/gitleaks/gitleaks) for secrets scanning

If any tool can't be installed in the current environment (no network access,
restricted sandbox), the skill falls back to the manual checklists in
`skills/plugin-security-scan/references/` and clearly marks which modules ran
manually in the report.

No credentials or environment variables are required to use this plugin.

## Usage

Ask Claude to scan a plugin, for example:

- "Scan the `standup-prep` plugin for security issues"
- "Audit this plugin before we roll it out"
- "Check this plugin for hardcoded secrets and vulnerable dependencies"
- "Is this plugin safe to install?"

Point Claude at the plugin's directory (or attach/upload it) if it isn't already
installed locally. Claude will inventory the plugin's files, run all three modules,
and deliver a markdown security report plus a short chat summary of the top
findings.

## Customization

This plugin has no `~~` placeholders — it doesn't reference any org-specific tools,
so there's nothing to customize before use. If your org has an internal list of
"trusted" external domains/MCP servers, consider adding it to
`skills/plugin-security-scan/references/secrets-checklist.md` so the scan can flag
unrecognized destinations more precisely.
