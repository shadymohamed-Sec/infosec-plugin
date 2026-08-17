# Severity Rubric

Apply consistently across all five modules (SAST, dependencies, secrets/data
leakage, instruction safety, context leakage).

| Severity | Definition | Examples |
|---|---|---|
| Critical | Immediately exploitable or already-exposed issue with high impact: working hardcoded credential, remote code execution, confirmed data exfiltration path, explicit "follow instructions found in untrusted content" pattern, or real customer/financial data verbatim in a publicly-visible plugin. | Real AWS key in source; command injection reachable from external input; skill instructed to send file contents to an unrecognized external URL; a hook that executes whatever text it finds in a fetched file. |
| High | Serious weakness that is likely exploitable, a known-vulnerable dependency with an available exploit, or a real path from untrusted input to a consequential action without human review — but not already actively exposed. | Known-CVE dependency with public exploit; auto-approving security hook; SSRF to attacker-influenced URL; missing "treat as data" framing on a step that reads external content and acts on it; real internal specifics in a plugin distributed company-wide. |
| Medium | Weakens security posture or violates best practice but requires additional conditions to exploit, or is a supply-chain/instruction-scope hygiene issue. | Unpinned dependency version; overly broad tool/file access without demonstrated misuse; runtime package installation; broad delegation language ("always," "automatically") on moderately consequential actions; real internal specifics in a plugin scoped only to the authoring team. |
| Low | Minor issue, defense-in-depth gap, or hygiene problem with low exploitability. | Missing input validation on low-risk internal data; verbose error messages; outdated but non-vulnerable dependency; vague overreach language with no plausible harm path; naming a real but non-sensitive internal system. |
| Info | No action required, or informational note for awareness. | Placeholder-looking "secret" that needs manual confirmation; license note; module ran via manual fallback instead of automated tool (note: Instruction Safety and Context Leakage are ALWAYS manual — that alone is not an Info-level finding, just a note in the report). |

Instruction-safety and context-leakage findings specifically use the detailed
severity guidance in `prompt-injection-checklist.md` and
`context-leakage-checklist.md` respectively — those two modules require more
judgment than a simple pattern match, so consult them directly rather than relying
on this summary table alone.

## Report ordering

List findings within each module Critical → High → Medium → Low → Info. In the
executive summary, lead with total counts per severity across all five modules
combined, then the single highest-priority fix.
