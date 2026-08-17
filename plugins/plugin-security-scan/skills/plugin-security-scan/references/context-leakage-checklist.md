# Context-Leakage Checklist (Build-Time Knowledge Leakage)

Plugins built through an assisted, natural-language flow (e.g. a plugin-authoring
skill that searches Slack, email, and internal docs to fill in organization-specific
details during customization) can end up carrying real internal specifics into a
file that's meant to be shared broadly. This module is manual/judgment-based like
the instruction-safety review — there is no tool that distinguishes "realistic
example" from "actual internal detail" automatically.

## What to look for

Read every file in the plugin (skill bodies, references, examples, README,
CONNECTORS.md) for content that reads like it was copied from a real source rather
than composed as a generic illustration:

- Real client, customer, or partner names used as if they were specific examples,
  rather than a generic placeholder ("Acme Corp") or the plugin author's own
  company name used appropriately.
- Real project codenames, internal initiative names, or product names that aren't
  public.
- Specific dollar figures, headcounts, revenue numbers, or other business metrics
  that look like they came from an actual document rather than a round, obviously
  illustrative number.
- Real employees' or contacts' names appearing alongside sensitive statements
  (performance details, compensation, health/personal information, disciplinary
  matters).
- Actual internal Slack channel names, ticket/issue IDs, workspace IDs, or
  repository names that reference a real internal system rather than a
  category-level placeholder (`~~project tracker`, `#your-team-channel`).
- Text that reads like it was lifted verbatim from a real conversation or document
  — specific phrasing, dates, and details that are too precise to be a made-up
  example ("as discussed in the July 12 sync, the Meridian account renewal...").

## What is NOT a finding

- The plugin author's own company name, used appropriately as the plugin's brand.
- Clearly-labeled generic examples, especially ones using well-known placeholder
  conventions ("Acme Corp," "Jane Doe," "example.com," round numbers like
  "$10,000").
- `~~category` placeholders per the plugin's own `CONNECTORS.md` — these are
  intentional and expected.
- References to genuinely public information (a company's public product names,
  publicly announced figures).

## Weighing severity: distribution scope matters

The same internal detail is a bigger problem the more broadly the plugin will be
seen. Check how the plugin is being distributed (ask if not obvious from context —
e.g. "is this repo private to your team, company-wide, or public?") and weigh
accordingly:

- **Critical**: real customer/financial/personal data, verbatim, in a plugin hosted
  in a public repository or otherwise available outside the organization.
- **High**: real internal specifics (codenames, real employee names with sensitive
  context, specific figures) in a plugin distributed company-wide (e.g. a shared
  internal marketplace) beyond the team that authored it.
- **Medium**: real internal specifics in a plugin scoped only to the small team
  that authored it and already had legitimate access to that information.
- **Low**: internal tool/process references with no specific sensitive detail
  attached (e.g. naming a real but non-sensitive internal system).

## Remediation guidance

- Replace the specific detail with a generic, clearly-labeled example.
- If the value is genuinely needed for the plugin to function (a real workspace ID,
  a real project name the plugin must reference to work), move it to a
  configuration value the plugin reads at runtime (an environment variable, a
  per-user config file) rather than hardcoding it into a file that gets committed
  and distributed.
- If the repo hosting the plugin is more broadly visible than the sensitivity of
  its content warrants (e.g. public repo, internal-only content), recommend fixing
  the repo's visibility in addition to fixing the content.
