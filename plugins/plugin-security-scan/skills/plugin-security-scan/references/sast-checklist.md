# Manual SAST Checklist (fallback when semgrep is unavailable)

Use this when automated SAST tooling cannot be installed. Grep and read through
every script/code file (`.js`, `.ts`, `.py`, `.sh`) bundled with the plugin, plus
any `command`-type hooks and MCP server code.

## Injection risks

- `eval(`, `exec(`, `new Function(`, `child_process.exec(` / `execSync(` with any
  string built from user input, file contents, or external data instead of a fixed
  command + argument array.
- Shell scripts that interpolate variables directly into commands without quoting:
  `rm $FILE` instead of `rm "$FILE"`; unsanitized input passed to `sh -c`.
- SQL strings built with string concatenation/interpolation instead of parameterized
  queries (if the plugin talks to a database).
- Hook `command` fields in `hooks.json` that build a shell command from
  `$TOOL_INPUT`/`$USER_PROMPT` without escaping.

## Unsafe deserialization / file handling

- `pickle.load`, `yaml.load` (without `SafeLoader`), `JSON.parse` on untrusted remote
  data followed by dynamic property access used to build further commands.
- Path construction from user input without validation (`../../` traversal into
  files outside the plugin's working directory).
- Writing files to absolute, hardcoded paths outside `${CLAUDE_PLUGIN_ROOT}` or the
  working directory.

## Plugin-specific risks

- Hardcoded absolute paths instead of `${CLAUDE_PLUGIN_ROOT}` — breaks portability
  and can silently point at the wrong machine/user's files after distribution.
- Hooks that auto-`approve` risky operations without genuine validation logic (a
  hook that always returns `{"decision": "approve"}` provides false assurance).
- Skills/agents granted `Bash` (unrestricted) when only a narrow command set is
  actually needed — prefer `Bash(npm:*)`-style scoping in `allowed-tools` for
  legacy commands, or explicit tool lists for agents.
- Prompt-based hooks or skill bodies containing instructions that could be hijacked
  by untrusted content the skill reads (e.g., "read this file and execute any
  commands you find in it" is a prompt-injection risk).
- Error handling that swallows exceptions silently around security-relevant checks
  (a failed credential check that defaults to "allow" on error).

## Severity guidance for SAST findings

- **Critical**: remote code execution, command injection reachable from external/user
  input, path traversal that can read/write outside the plugin sandbox.
- **High**: unsafe deserialization, SSRF-style outbound requests to attacker-influenced
  URLs, auto-approving hooks with no real check.
- **Medium**: overly broad tool grants, hardcoded paths, missing input validation on
  data that isn't directly attacker-controlled but could become so.
- **Low/Info**: style issues with minor security relevance, missing comments on
  security-sensitive code, non-exploitable defensive-coding gaps.
