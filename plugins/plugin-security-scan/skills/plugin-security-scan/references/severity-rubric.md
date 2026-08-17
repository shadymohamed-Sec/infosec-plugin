# Severity Rubric

Apply consistently across all three modules (SAST, dependencies, secrets/data leakage).

| Severity | Definition | Examples |
|---|---|---|
| Critical | Immediately exploitable or already-exposed issue with high impact: working hardcoded credential, remote code execution, confirmed data exfiltration path. | Real AWS key in source; command injection reachable from external input; skill instructed to send file contents to an unrecognized external URL. |
| High | Serious weakness that is likely exploitable or a known-vulnerable dependency with an available exploit, but not already actively exposed. | Known-CVE dependency with public exploit; auto-approving security hook; SSRF to attacker-influenced URL. |
| Medium | Weakens security posture or violates best practice but requires additional conditions to exploit, or is a supply-chain hygiene issue. | Unpinned dependency version; overly broad tool/file access without demonstrated misuse; runtime package installation. |
| Low | Minor issue, defense-in-depth gap, or hygiene problem with low exploitability. | Missing input validation on low-risk internal data; verbose error messages; outdated but non-vulnerable dependency. |
| Info | No action required, or informational note for awareness. | Placeholder-looking "secret" that needs manual confirmation; license note; module ran via manual fallback instead of automated tool. |

## Report ordering

List findings within each module Critical → High → Medium → Low → Info. In the
executive summary, lead with total counts per severity across all three modules
combined, then the single highest-priority fix.
