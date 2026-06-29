---
type: concept
title: "Plugin Distribution"
created: 2026-06-29
updated: 2026-06-29
tags: [plugins, distribution, marketplace, community, official, submission, validation]
source_count: 1
sources:
  - sources/clean/plugins.md
---

# Plugin Distribution

Once a plugin works locally, distribution means sharing it through a marketplace — a private one for your team, or one of Anthropic's two public marketplaces for the wider community.

## Prepare to share

When your plugin is ready:

1. **Add documentation**: include a `README.md` with installation and usage instructions.
2. **Choose a versioning strategy**: decide whether to set an explicit `version` or rely on the git commit SHA. See [Plugin Versioning](./plugin-versioning.md).
3. **Create or use a marketplace**: distribute through a plugin marketplace for installation.
4. **Test with others**: have team members test the plugin before wider distribution.

To keep a plugin internal to your team, host the marketplace in a private repository.

## The two public marketplaces

Anthropic maintains two public marketplaces for Claude Code plugins:

- **`claude-plugins-official`** — a curated set of plugins maintained by Anthropic. Registered automatically the first time you start Claude Code interactively. A non-interactive script that runs before that first launch must add it explicitly with `claude plugin marketplace add anthropics/claude-plugins-official`.
- **`claude-community`** — the public community marketplace where third-party submissions land after review. Users add it with `/plugin marketplace add anthropics/claude-plugins-community` and install from it as `@claude-community`.

## Submit to the community marketplace

To submit a plugin for community-marketplace review, use one of the in-app forms:

- **claude.ai**: [claude.ai/admin-settings/directory/submissions/plugins/new](https://claude.ai/admin-settings/directory/submissions/plugins/new)
- **Console**: [platform.claude.com/plugins/submit](https://platform.claude.com/plugins/submit)

> The claude.ai form requires a Team or Enterprise organization and directory management access; organization Owners have this access by default. Individual authors who aren't part of a Team or Enterprise organization can use the Console form instead.

Run [`claude plugin validate`](./plugin-cli-commands.md#plugin-validate) locally before you submit. The review pipeline runs the same check on every submission, along with automated safety screening.

### How approval propagates

> Approved plugins are pinned to a specific commit SHA in the `anthropics/claude-plugins-community` catalog, and CI bumps the pin automatically as you push new commits to your repository. The public catalog syncs nightly from the review pipeline, so there can be a delay between approval and your plugin appearing in `marketplace.json`.

This commit-SHA pinning is the same mechanism that drives [versioning](./plugin-versioning.md) when no explicit `version` is set.

## The official marketplace is curated separately

> The official marketplace, `claude-plugins-official`, is curated separately. Anthropic decides which plugins to include at its discretion. There is no application process, and the submission form does not add plugins to the official marketplace.

If Anthropic lists your plugin in the official marketplace, your CLI can prompt Claude Code users to install it (a "plugin hint").

## Related concepts

- [Creating Plugins](./creating-plugins.md) — the authoring hub
- [Plugin Development Workflow](./plugin-development-workflow.md) — testing before submission, including `--plugin-url` trust rules
- [Plugin Versioning](./plugin-versioning.md) — explicit `version` vs commit-SHA, which governs catalog pinning
- [Plugin CLI Commands](./plugin-cli-commands.md) — `plugin validate`, `plugin install`, `marketplace add`
- [Plugin Installation Scopes](./plugin-installation-scopes.md) — where installed plugins land for the people who add your marketplace
