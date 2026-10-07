# frappe-nav-migrate

A Claude Code skill that moves custom Frappe apps to the Frappe v16.50 navigation.

Frappe v16.50 replaces `Workspace Sidebar` and `Desktop Icon` with three parts:

- an Apps screen tile, from `add_to_apps_screen` in `hooks.py`;
- a `Dock`, the left rail that lists the app's modules;
- one `Sidebar` per module.

The skill writes these files for each app, runs `bench migrate`, guides a browser check of the result, and writes the patches that give a live site the same state on
`bench migrate`. It also lists 10 problems found in real migrations, with the fix for each.

## Install

As a Claude Code plugin:

```
/plugin marketplace add rahmed-dev/frappe-nav-migrate
/plugin install frappe-nav-migrate@frappe-nav-migrate
```

Or copy the skill folder by hand:

```bash
git clone https://github.com/rahmed-dev/frappe-nav-migrate
cp -r frappe-nav-migrate/skills/frappe-nav-migrate ~/.claude/skills/
```

## Requirements

A Frappe bench with Frappe v16.50 or later. No other dependency.

## Use

Ask Claude Code, in a bench with Frappe v16.50 or later:

- "Move my custom apps to the new navigation."
- "My app icon is missing on the desktop after the update."
- "Migrate fails in `frappe.patches.v16_0.convert_sidebars` with 'is another module's name'."

Run `bench update` first. The skill does not upgrade the bench.

## Contents

| Path | Contents |
|---|---|
| `skills/frappe-nav-migrate/SKILL.md` | The model, the pre-checks, the steps per app, the pitfalls, the patch rules, the browser check, the safety rules |
| `skills/frappe-nav-migrate/references/patches.md` | Patch templates for live sites |

## License

MIT
