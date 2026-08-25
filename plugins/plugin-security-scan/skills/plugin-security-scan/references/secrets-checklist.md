# Manual Secrets & Data Leakage Checklist (fallback / supplement to gitleaks and TruffleHog)

## TruffleHog (preferred tool for verification)

Install: `pip install trufflehog3 --break-system-packages` or the standalone binary
from the TruffleHog releases page (check current install instructions — the
project has shipped both a Python and a Go rewrite over time, so verify which one
`trufflehog` resolves to in the environment). Run against the plugin directory and,
separately, against git history if the plugin lives in a repo:

```
trufflehog filesystem <plugin-dir> --json --only-verified
trufflehog git file://<repo-path> --json --only-verified
```

What makes this worth running ahead of/alongside gitleaks: TruffleHog doesn't stop
at "this looks like an AWS key" — for many credential types it actively tests the
credential against the real service (does this key actually authenticate) before
marking it verified. A verified hit is definitive: treat it as Critical immediately,
no placeholder-judgment step needed, since it's been confirmed live. An unverified
pattern match (`--only-verified` omitted, or a credential type TruffleHog can't
verify) still needs the same judgment calls as a gitleaks hit below.

Grep every text file in the plugin (`.md`, `.json`, `.js`, `.ts`, `.py`, `.sh`,
`.yml`, `.yaml`) for the patterns below. Treat matches inside `references/examples/`
or clearly-labeled placeholder text (`~~api-key`, `YOUR_TOKEN_HERE`, `<CHANGE_ME>`,
`xxx`, `sk-example...`) as Informational, not Critical — flag them separately as
"placeholder, verify not a real credential" rather than as a leak.

## Hardcoded credential patterns

- Generic: `api[_-]?key`, `secret`, `password`, `passwd`, `token`, `credential`
  followed by `=` or `:` and a non-placeholder-looking literal value.
- Cloud provider keys: AWS (`AKIA[0-9A-Z]{16}`), GCP service account JSON blobs,
  Azure connection strings (`DefaultEndpointsProtocol=...;AccountKey=...`).
- Private keys: `-----BEGIN (RSA|EC|OPENSSH|PGP) PRIVATE KEY-----`.
- Database connection strings with embedded credentials:
  `postgres://user:password@host`, `mongodb+srv://user:pass@...`.
- Bearer tokens / JWTs hardcoded in source: `eyJ...` long base64-looking strings
  next to `Authorization` headers.
- Webhook URLs with embedded secrets (Slack `hooks.slack.com/services/...`, Discord
  webhook URLs) — anyone with the plugin's source can post to that channel.

## Data exfiltration risks (specific to Claude/Cowork plugins)

- Skill or agent instructions that direct Claude to send file contents, conversation
  history, credentials, or user PII to an external URL/API/MCP server as a side
  effect not central to the skill's stated purpose.
- Hooks (`PostToolUse`, `Stop`, etc.) that log full tool inputs/outputs to a file or
  external endpoint — this can capture secrets or sensitive data incidentally typed
  or read during the session.
- `.mcp.json` servers pointed at domains outside the organization's known/trusted
  list, especially combined with broad scopes or credentials passed via plain env
  vars in the config rather than `${VAR}` expansion from the user's actual
  environment.
- Any instruction telling Claude to read broadly-scoped local files unrelated to the
  skill's job (`~/.ssh/*`, `~/.aws/credentials`, browser profile/cookie directories,
  shell history files) — flag even if no immediate exfiltration path exists, since
  it's an unnecessary privilege that becomes dangerous if combined with any output
  channel.
- Debug/telemetry code that phones home with more than aggregate, non-sensitive
  usage data.

## PII patterns (secondary — relevant if the plugin processes personal data)

- Hardcoded real-looking email addresses, phone numbers, national ID / SSN-shaped
  strings, or full names paired with contact info left in test fixtures or example
  files that shipped with the plugin by mistake.

## Verification before rating Critical

1. Confirm the match isn't a placeholder (see intro).
2. If it looks like a real, working credential, mark **Critical** and recommend:
   - Immediate rotation of that credential.
   - Removing it from the file and from git history (`git filter-repo` /
     BFG Repo-Cleaner) if the plugin has been committed to version control.
   - Moving it to an environment variable referenced via `${VAR_NAME}` in
     `.mcp.json` or loaded at runtime, never committed to the plugin source.
