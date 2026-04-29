# roadway-integrator

A Claude Code, Codex, and **Cursor** plugin that onboards a customer's codebase to [Roadway's](https://roadwayai.com) attribution pipeline.

When invoked, it detects the customer's CRM, analytics, auth, and billing stack, then dispatches focused sub-skills to wire everything up.

## What it does

The top-level skill (`skills/roadway-integrator/SKILL.md`) acts as a router. It:

1. Detects the stack (HubSpot/Salesforce, Segment/GA4, Stripe, signup entry point, HTML host).
2. Confirms the findings with the user.
3. Dispatches the relevant sub-skills in order.
4. Tracks progress in `.roadway/onboarding.md` so a later session can resume where the previous one left off.

## Sub-skills

| Skill | Purpose |
|---|---|
| `install-roadway-tag` | Adds the `analytics.roadwayai.com/tag.js` script to the HTML host. |
| `setup-hubspot-crm` | Creates HubSpot properties + hidden form fields. |
| `setup-salesforce-crm` | Adds custom fields on Contact/Lead. |
| `setup-segment-identify` | Wires `analytics.identify()` at signup. |
| `setup-ga4-user-id` | Wires GA4 `user_id` at signup. |
| `setup-stripe-metadata` | Writes IDs onto Stripe Customer metadata. |

If the customer has no Segment or GA4 (directly or via GTM), the router exits early and tells the user to contact their Roadway rep — none of the sub-skills can do useful work without an analytics pipeline to carry anonymous IDs.

## Layout

```
roadway-integrator/
├── AGENTS.md              # repo guidance for agents working on the plugin (same router as Cursor rule)
├── .cursor-plugin/        # Cursor plugin manifest (skills + rules)
├── .codex-plugin/         # Codex plugin manifest
├── .claude-plugin/        # Claude plugin manifest
├── rules/                 # Cursor rule(s); roadway-integrator.mdc mirrors AGENTS router
├── assets/                # marketplace logo (see assets/logo.svg)
├── skills/                # router skill + sub-skills, one directory each
└── references/            # shared docs the skills read on demand
```

### Claude Code

Install as a [Claude Code plugin](https://docs.claude.com/en/docs/claude-code/plugins) via the `roadway` marketplace:

```
/plugin marketplace add RoadwayAI/roadway-integrator
/plugin install roadway-integrator@roadway
```

The bundle is defined by `.claude-plugin/plugin.json` (skills under `skills/`); `.claude-plugin/marketplace.json` registers the marketplace entry. Once installed, mention Roadway onboarding in any session and Claude will invoke the `roadway-integrator` router skill.

### Cursor

This repo ships a [Cursor plugin](https://cursor.com/docs/plugins) manifest at `.cursor-plugin/plugin.json`. It loads skills from `skills/` and rules from `rules/` (including `roadway-integrator.mdc`, which mirrors the router in `AGENTS.md`).

#### Install from the marketplace (Coming soon)

We're working on publishing it on Cursor's marketplace, for now install it locally (see bellow).

#### Install locally (clone or develop against this repo)

1. Clone this repository.
2. Symlink the plugin root into Cursor’s local plugins directory:

   ```bash
   ln -s /absolute/path/to/roadway-integrator ~/.cursor/plugins/local/roadway-integrator
   ```

3. Restart Cursor, or run **Developer: Reload Window** from the Command Palette.

4. Confirm the plugin loaded: Cursor Settings → Rules should list the `roadway-integrator` rule; skills appear under Agent skills / manual invocation. In agent chat you can run `/roadway-integrator` (and related skill names).

## Further reading

- [Required IDs for attribution](https://docs.roadwayai.com/required-ids-for-attribution)
- [Roadway attribution tag source](https://github.com/RoadwayAI/roadway-attribution-tag)
