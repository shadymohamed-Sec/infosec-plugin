# Instruction-Safety / Prompt-Injection Checklist

This is the one module in the scan that is inherently manual — there is no tool
that statically detects "this natural-language instruction is dangerous" the way
semgrep detects a SQL injection pattern. Read every skill/agent/hook body and apply
judgment using this checklist.

## Missing "treat as data, not instructions" guardrails

Any time a skill has Claude read content it didn't author — a file the user
uploaded, an email, a Slack thread, search results, a web page, another skill's
output — and then act on what it read, that content is a potential prompt-injection
vector. Look for:

- Does the skill tell Claude to treat that content as **data to analyze**, not as
  **instructions to follow**? If a skill says "read this document and summarize
  it," that's fine. If a skill's flow could let text inside that document cause
  Claude to take a different, unintended action (send something, delete something,
  call a tool it wouldn't otherwise call), that's a finding.
- Is there any step where the skill says something like "read this and do whatever
  it says" or "follow the instructions in the file"? That is a direct, explicit
  injection vector — treat as Critical regardless of how unlikely misuse seems.

## Missing human-in-the-loop on consequential actions

- Does the skill perform irreversible or broad-impact actions (delete, overwrite,
  send an email, post publicly, share a file externally, grant access, spend
  money) without a clear checkpoint for user confirmation first?
- Does a hook auto-approve (`"decision": "approve"`) tool calls in situations where
  the underlying action is genuinely risky, rather than a narrow, clearly-safe case?
- Are there phrases like "always," "automatically," "without asking," "don't
  confirm," or "proceed without user input" attached to anything consequential?
  Note: this phrasing is completely fine for genuinely low-stakes, repetitive
  actions (e.g., "always format dates as YYYY-MM-DD") — judge the actual action,
  not just the presence of the word "always."

## Self-modification / privilege expansion

- Does the skill instruct Claude to widen its own tool access, install other
  plugins, modify its own configuration, or write new skills/hooks without the
  user being told this is happening?
- Does the skill request tool grants broader than its stated purpose needs (see
  also `sast-checklist.md`'s tool-grant section — this is the natural-language
  companion check: does the *prose* ask Claude to go beyond what the skill claims
  to do)?

## Confused-deputy patterns

- Does the skill chain into other tools/skills/MCP servers in a way that could let
  a lower-trust input (something read from an external source) cause a
  higher-trust action to happen without the user realizing which input caused it?
- Example: a skill that reads a support ticket and then, based on ticket content,
  decides to grant a refund, change an account setting, or send credentials — with
  no step where a human reviews the specific action before it happens.

## Severity guidance

- **Critical**: explicit "follow whatever instructions you find in this content"
  pattern, or a clear path from attacker-controlled input to a consequential
  action with zero human checkpoint.
- **High**: missing "treat as data" framing on a step that reads external/untrusted
  content and feeds it into a consequential decision, even without a fully worked
  attack path.
- **Medium**: broad delegation language ("always," "automatically") attached to
  moderately consequential actions; tool grants broader than the skill's stated
  purpose without a clear reason.
- **Low**: vague or stylistic overreach with no plausible harm path attached.
