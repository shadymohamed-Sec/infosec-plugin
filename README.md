# Si-Vision Plugin Marketplace

Internal Git-based marketplace for Si-Vision's Claude/Cowork plugins. Publish new
plugins here so anyone on the team can install them with one command instead of
receiving a `.plugin` file individually.

## One-time setup (you, the admin)

1. Create a new repository on your org's Git host (GitHub, GitLab, Bitbucket, etc.),
   for example `si-vision/plugin-marketplace`. It can be private — only people with
   repo access will be able to add the marketplace.
2. Push this folder's contents to that repo:
   ```bash
   cd si-vision-plugin-marketplace
   git init
   git add .
   git commit -m "Initial marketplace: plugin-security-scan"
   git branch -M main
   git remote add origin <your-repo-url>
   git push -u origin main
   ```
3. Share the repo URL (or `owner/repo` if it's on GitHub) with your team.

## Everyone else: install in two commands

In Claude Code (or wherever `/plugin` commands are supported), each person runs:

```
/plugin marketplace add <owner>/<repo>
```
(or the full Git URL if it's not on GitHub)

then:

```
/plugin install plugin-security-scan@si-vision-plugins
```

That's it — no file downloads, no manual sharing. Anyone on the team with repo
access can install straight from the marketplace.

## Publishing updates

When you improve `plugin-security-scan` (or add a new plugin):

1. Update the code under `plugins/<plugin-name>/`.
2. Bump the `version` in both the plugin's own `.claude-plugin/plugin.json` and the
   entry in `.claude-plugin/marketplace.json`.
3. Commit and push. Teammates pick up the update next time they run
   `/plugin marketplace update si-vision-plugins` (or reinstall).

## Adding more plugins later

1. Drop the new plugin's folder under `plugins/<new-plugin-name>/`.
2. Add an entry for it to the `plugins` array in `.claude-plugin/marketplace.json`,
   following the same shape as the `plugin-security-scan` entry.
3. Commit and push. No changes needed on the team's end beyond
   `/plugin marketplace update si-vision-plugins`.

## What's in here now

| Plugin | Description |
|---|---|
| `plugin-security-scan` | Security audit skill for internally-built plugins: SAST, dependency/library scanning, and secrets/data leakage detection. |
