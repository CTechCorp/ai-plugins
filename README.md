<!--
Generated from public/README.md in C Tech's private development repository. Every file in
this repository is written by the release process. Do not edit it here.
-->

# C Tech AI plugins

> **For C Tech subscribers only.** These files are public so they are easy to install, but
> public availability does not grant a license. They are licensed only to C Tech customers
> with a current EVS subscription, under the C Tech AI Integrations License Agreement in
> [LICENSE.md](LICENSE.md).

AI agent plugins from [C Tech Development Corporation](https://www.ctech.com/), starting
with **EVS Data Tools**. The `evs-data-tools` plugin teaches AI agents to create, read,
validate and convert [Earth Volumetric Studio (EVS)](https://www.ctech.com/products/earth-volumetric-studio/)
data files.

Documentation, with step-by-step install guides, example prompts and help moving from the
old `CTechAISkills` installer: **[ctech.com/ai_help](https://ctech.com/ai_help/)**.

## Skills in `evs-data-tools`

| Skill | File Type | Description |
|-------|-----------|-------------|
| `evs-data` | — | Top-level reference: shared rules, concepts, and routing table |
| `evs-lithology` | — | Shared lithology concepts: materials, ID ordering, simplification |
| `evs-convert` | — | Conversion procedures between EVS data file formats |
| `evs-apdv` | `.apdv` | Analyte Point Data File |
| `evs-aidv` | `.aidv` | Analyte Interval Data File |
| `evs-lpdv` | `.lpdv` | Lithology Point Data File |
| `evs-lsdv` | `.lsdv` | Lithology Screen Data File |
| `evs-pgf` | `.pgf` | PreGeology File |
| `evs-geo` | `.geo` | Borehole Geologic Stratigraphy File |
| `evs-gmf` | `.gmf` | Geologic Stratigraphy Multi-File |

The skills load automatically when a task involves EVS data. When installed as a plugin
they are namespaced, e.g. `/evs-data-tools:evs-data`.

## Install

Quick commands below. For screenshots, organization-wide setup and troubleshooting, see
[Install](https://ctech.com/ai_help/evs-data-tools/install/) in the help.

### Claude Code

> Licensed only to C Tech customers with a current EVS subscription. By installing, you
> agree to the C Tech AI Integrations License Agreement included with the release, also
> published at https://ctech.com/downloads/.

```bash
claude plugin marketplace add CTechCorp/ai-plugins
```

```bash
claude plugin install evs-data-tools@ctech
```

To make the plugin available to a whole team, commit this to the project's
`.claude/settings.json`. Whoever commits it accepts the license for the organization.

```json
{
  "extraKnownMarketplaces": {
    "ctech": { "source": { "source": "github", "repo": "CTechCorp/ai-plugins" } }
  },
  "enabledPlugins": { "evs-data-tools@ctech": true }
}
```

### Claude Desktop and claude.ai

> Licensed only to C Tech customers with a current EVS subscription. By installing, you
> agree to the C Tech AI Integrations License Agreement included with the release, also
> published at https://ctech.com/downloads/.

Customize > Plugins > Add > Add marketplace, then enter `CTechCorp/ai-plugins`.

Team and Enterprise owners can provision the plugin for the whole organization from
Organization settings > Plugins & skills. The person who provisions it accepts the license
for the organization.

### Codex

> Licensed only to C Tech customers with a current EVS subscription. By installing, you
> agree to the C Tech AI Integrations License Agreement included with the release, also
> published at https://ctech.com/downloads/.

```bash
codex plugin marketplace add CTechCorp/ai-plugins
```

```bash
codex plugin add evs-data-tools@ctech
```

### Other agents (Cursor, GitHub Copilot, Gemini CLI, ...)

> Licensed only to C Tech customers with a current EVS subscription. By installing, you
> agree to the C Tech AI Integrations License Agreement included with the release, also
> published at https://ctech.com/downloads/.

```bash
npx skills add CTechCorp/ai-plugins
```

### Microsoft 365 Copilot and Copilot Studio

These platforms allow at most 8 skills per agent and 20,000 characters per `SKILL.md`. To
fit, each [release](https://github.com/CTechCorp/ai-plugins/releases) includes
`evs-data-<version>.zip`: a single skill with the format specs in `references/` and the
license in `LICENSE.md`.

> Licensed only to C Tech customers with a current EVS subscription. By downloading or
> uploading `evs-data-<version>.zip`, you agree to the C Tech AI Integrations License
> Agreement included with the release, also published at https://ctech.com/downloads/.

Upload it in Agent Builder (Microsoft 365 Copilot declarative agents) or Copilot Studio
(Build > Skills > Upload a skill).

## Updates

Each release carries the license version issued with it, in `LICENSE.md`. Updating to a new
release means accepting the license included with that release. When the license changes,
the release notes say so.

You choose when to update:

| Platform | Automatic updates | Turn on or off | Update by hand |
|----------|-------------------|----------------|----------------|
| Claude Code | Off by default for this marketplace | `/plugin` > Marketplaces > `ctech` > Enable or Disable auto-update | `claude plugin update evs-data-tools@ctech` |
| Claude Desktop and claude.ai | Off unless you turn on **Sync automatically** | **Sync automatically** on the `ctech` marketplace | **Check for updates** |
| Organization-wide (Team and Enterprise) | Set by the owner who provisions it | **Sync automatically** on the marketplace, in Organization settings > Plugins & skills | **Check for updates** |
| Codex | Not documented | — | `codex plugin marketplace upgrade ctech`, then start a new session |
| `npx skills` | No | — | `npx skills check`, then `npx skills update` |
| Microsoft 365 Copilot and Copilot Studio | No | — | Upload the new `evs-data-<version>.zip`. In Copilot Studio: the skill's **…** menu > **Replace** |

In Claude Code, `claude plugin marketplace update ctech` refreshes the list of versions
but does not update the installed plugin.

More in [Updates](https://ctech.com/ai_help/evs-data-tools/updates/) in the help.

## License

The skills are licensed under the C Tech AI Integrations License Agreement in
[LICENSE.md](LICENSE.md). They are licensed only to C Tech customers with a current, paid
EVS Fixed, Floating, Enterprise or University Enterprise subscription. Public availability
of these files does not grant a license. See also
[License](https://ctech.com/ai_help/license/) in the help.
