# Manual Dependency / Library Review Checklist (fallback when npm audit / pip-audit / Scorecard are unavailable)

## OSSF Scorecard (upstream maintenance-hygiene signal)

Install: `go install github.com/ossf/scorecard/v4@latest` (requires Go) or use the
prebuilt Docker image (`docker run gcr.io/openssf/scorecard ...`) if Go isn't
available — check current install docs, the project ships both. Run against each
direct dependency's source repository:

```
scorecard --repo=github.com/<org>/<project> --format json
```

This checks things a CVE database can't tell you in advance: does the project have
branch protection, are CI dependencies pinned, is there a security policy, does the
project show signs of active maintenance, does it have a history of responding to
vulnerability reports. A low score doesn't mean the package is currently
compromised — it means the conditions that let a repo get compromised (like a
maintainer account takeover leading to a malicious release) are more present than
they should be. Treat a notably low score (check the project's documented scoring
bands, since these are periodically revised) as a Medium finding on its own, and as
a reason to weight any other finding about that dependency more heavily.

If Scorecard can't be installed, fall back to the manual signals below.

## For each declared dependency (package.json, requirements.txt, pyproject.toml, Pipfile)

- **Version pinning**: is the version pinned or range-bound (`^`, `~`, `>=`) in a way
  that could silently pull in a newer, unreviewed release? Unpinned direct
  dependencies in an internal plugin are a Medium finding.
- **Known-vulnerable versions**: cross-check the package name + version against
  public advisories (npm/PyPI advisory databases, GitHub Security Advisories) if a
  scanner truly cannot run in this environment — note in the report that this was a
  manual, best-effort check, not a full CVE database sweep.
- **Maintenance signal**: last publish date, open issue count, whether the package is
  archived/deprecated. Heavily unmaintained packages handling security-relevant work
  (auth, crypto, parsing untrusted input) are a Medium/High finding depending on role.
- **Typosquatting**: does the package name closely resemble a popular package
  (`reqeusts` vs `requests`, `crossenv` vs `cross-env`)? Treat any near-miss as a
  Critical finding pending verification — this is a common supply-chain attack.
- **Scope of what the package can do**: does it need filesystem access, network
  access, or shell execution to do its stated job? A "formatting" library that also
  makes outbound network calls is worth flagging even without a known CVE.

## Runtime installation risk

If any script or hook installs packages at runtime (`pip install X`, `npm install X`
inside a hook or skill script rather than a committed manifest + lockfile):

- Flag this as a Medium/High finding regardless of what package is installed — runtime
  installs bypass review, are not reproducible, and can silently change behavior if
  the package is updated or compromised upstream between runs.
- Recommend vendoring the dependency or moving it into a committed, lockfile-pinned
  manifest reviewed at plugin-build time instead.

## MCP server dependencies

- For `stdio` MCP servers bundled with the plugin, treat their `command`/`args` the
  same as any other executable code — review what package/runtime they invoke.
- For `sse`/`http` MCP servers, there's no local dependency to scan, but confirm the
  `url` uses HTTPS and points to a domain the organization recognizes and trusts.

## License check (secondary, lower priority)

- Flag copyleft licenses (GPL, AGPL) on bundled code if the plugin will be
  distributed outside the org, since that may create obligations. Not a security
  finding — report separately as an Info-level note if relevant.
