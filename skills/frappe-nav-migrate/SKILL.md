---
name: frappe-nav-migrate
description: Convert custom Frappe apps to the Frappe v16.50 navigation (Apps screen tile, Dock rail, one Sidebar per module), and write the patches that make live sites match. Use for "new navigation", "v16.50 navigation", "app icon missing on desktop", "apps screen tile", "dock", "module sidebar", "convert workspace_sidebar", "desktop_icon is deprecated", "convert-sidebar-fixtures", or a migrate that fails in frappe.patches.v16_0.convert_* with "is another module's name". Does not upgrade the bench itself; run `bench update` first.
license: MIT
metadata:
  version: 1.0.0
---

# frappe-nav-migrate

Frappe v16.50 replaced the v16.0 navigation. This skill moves custom apps to the new model,
checks the result in a browser, and writes patches so a live site gets the same database state
on `bench migrate`.

## 1. The model

The hierarchy is **App > Module > DocTypes, Reports, Pages, Workspaces**.

| Part | What the app ships | Doctype |
|---|---|---|
| Apps screen tile | `add_to_apps_screen` in `hooks.py` | none (boot reads the hook) |
| Dock (left rail, lists the app's modules) | `{app}/{app}/dock/{app}/{app}.json` | `Dock`, `Dock Item` |
| Sidebar (one per module) | `{app}/{app}/{module_folder}/sidebar/{scrub(name)}/{scrub(name)}.json` | `Sidebar`, `Sidebar Item` |

Deprecated, no longer imported: `{app}/{app}/workspace_sidebar/*.json` and
`{app}/{app}/desktop_icon/*.json` (doctypes `Workspace Sidebar`, `Desktop Icon`, now an inert
archive).

Facts from the frappe source. Verify them on the bench before you rely on them, because
frappe changes fast:

- A module with no shipped Sidebar gets one that frappe generates from its contents. Nothing
  breaks, but the links are not curated.
- An app with no Dock has no rail. Its sidebar gets a switcher in the header.
- `mount_on: "<host app>"` in a Dock makes a **companion app**. Its Dock entries go on the
  host's rail, and the app never gets its own tile, even with `add_to_apps_screen`.
- A tile's click target is, in order: the hook's `route`, then the first Dock entry, then the
  first navigable module (`app_landing_route` in
  `frappe/public/js/frappe/ui/sidebar/sidebar.js`). Leave `route` out unless a clear landing
  exists. `"/desk"` sends the user to the desk root, not into the app.
- With no `logo`, the desk draws a letter icon. Logo paths are `/assets/{app}/...`.
- `bench --site X convert-sidebar-fixtures --app A [--dry-run]` converts old
  `workspace_sidebar/` files to Sidebar files.

Reference files in frappe: `frappe/dock/frappe/frappe.json`,
`frappe/users/sidebar/users/users.json`, `frappe/desk/doctype/sidebar/sidebar.py`
(validators), `frappe/desk/doctype/sidebar/convert_fixtures.py`, `frappe/boot.py`
(`get_apps` / app_data), `frappe/desk/RETIRING.md`, `frappe/utils/boilerplate.py` (dock
comment), `frappe/patches/v16_0/convert_*.py`.

## 2. Before you start

1. Confirm `frappe.__version__` is 16.50 or later and every site is migrated: the Patch Log has
   `frappe.patches.v16_0.convert_sidebars` on every site.
2. List custom apps and their sites. Skip apps with no doctypes. Skip third-party apps.
3. Ask the user, with AskUserQuestion, which apps to convert and which are companions
   (`mount_on`) of which host.
4. Plan one agent or one pass **per site**, not per app. Two migrates on one site at the same
   time collide. Different sites can run in parallel.
5. Give each parallel agent its own scratch folder. Agents that share a scratch folder overwrite
   each other's scripts.

## 3. Per app

1. Read the app's `modules.txt`, doctypes (skip child tables), reports, pages and workspaces.
2. Read the site's existing **site Sidebar rows** for the app's modules (standard 0, app NULL;
   the v16.50 migration made them from old site sidebars). Reuse their links in the curated
   Sidebar.
   ```sql
   select name, module, standard, app from tabSidebar where module in (...);
   select parent, type, link_type, link_to, label from `tabSidebar Item` where parenttype='Sidebar' and parent in (...);
   ```
3. Write one Sidebar per user-facing module. Copy the field shape of the frappe reference file.
   - Set `title` and `name` **exactly** equal to the module name, with the same case. See
     pitfall 1.
   - Use section rows (`type: "Section Break"`) and `child: 1` rows the way frappe's larger
     sidebars do.
   - Leave out a module the user wants to delete later. Do not give it a Sidebar or a Dock
     entry.
4. Write the Dock: one item per Sidebar (`link_type: "Sidebar"`, `link_to` = Sidebar name).
   Add `mount_on` for a companion.
5. Add `add_to_apps_screen` to `hooks.py` (not for a companion): `name`, `title`, and
   optionally `logo`, `route` and `has_permission`. Match the file's indent style.
6. Old files: run `convert-sidebar-fixtures --dry-run`, then run it for real. **Read the
   output.** It can pick a title that is not the module name, and it can add stray fields and a
   `link_type` on section rows. Fix the output by hand, then delete `workspace_sidebar/` and
   `desktop_icon/`.
7. Icons: use only names that exist (`frappe/public/icons/`, Lucide names used in frappe JSON).
   Frappe's duotone set is small, so Lucide is often the match.
8. Run `bench --site X migrate`. It must exit 0.
9. Check the database: a Sidebar row (standard 1, app set) and a Dock row with its items.
10. Check the result in a browser (section 6).

## 4. Pitfalls found in practice

1. **Title case bug, migrate fails.** `Sidebar.validate_title_is_its_own` compares
   `title == module` with case, then checks `Module Def` existence in MariaDB without case. An
   old site sidebar titled "ABC ERP" in module "Abc Erp" fails frappe's conversion patch with
   `<b>ABC ERP</b> is another module's name, and a sidebar's title is what the desk calls it
   by.` Fix the data: set the old `Workspace Sidebar.title` to the exact module name, then
   migrate again. For live sites, ship it as a `pre_model_sync` patch (template in
   `references/patches.md`).
2. **A site row hides the app Sidebar.** When a module has a site Sidebar row (standard 0) and
   a standard one, the desk shows the site row. Ask the user before you delete anything. List
   the links that exist only in the site row first. Delete by name with
   `frappe.delete_doc("Sidebar", name)`, then ship a post_model_sync patch for live sites.
3. **Old navigation code makes the old rows again.** Search the app for `Workspace Sidebar`,
   `Desktop Icon` and `get_app_level_directory_path`. These can be in patches, `after_install`,
   `after_migrate` or setup modules. Guard each one:
   - Skip when `frappe.db.exists("DocType", "Dock")`.
   - Skip a fixture file that no longer exists.
   An old patch that has not run on a live site yet will run with the new code.
4. **Maintenance mode stays on after a failed `bench update`.** The flag is in
   `sites/common_site_config.json` (`maintenance_mode`, `pause_scheduler`). After you fix the
   cause:
   1. Migrate the remaining sites.
   2. Run `bench build`.
   3. Run `bench set-config -g maintenance_mode 0` and `bench set-config -g pause_scheduler 0`.
   4. Restart `bench start`, because the running process caches the flag and returns
      `SessionStopped`.
   Do not clear a `pause_scheduler` that a single site set in its own site_config before the
   update.
5. **The tile does nothing in the user's browser, but works headless.** The cause is the old
   desk JS in the browser cache. Tell the user to hard reload (Ctrl+Shift+R), or to clear the
   site data.
6. A page error `removeChild … not a child of this node` on some HR pages comes from frappe or
   hrms, not from the app. Check a baseline before you blame the app.

7. **A site Dock layer hides the app Dock.** frappe's patch `convert_custom_sidebars` can save a
   site layer (`Dock`, standard 0, user "") for site sidebars that users made. A saved layer
   is the whole rail, so the app's standard entries disappear. Fix: put the standard entries
   first in the layer as references (`link_type`, `link_to`, `added` 0, no icon or title), and
   keep the site entries after them. Ship it as a post_model_sync patch.
8. **The site stays on the old icon grid.** frappe's patch `keep_existing_sites_on_desktop_icons`
   sets Desktop Settings `desktop_page` to "Desktop Icons" on an upgraded site. Migrate then
   removes the app's Desktop Icon rows, because the app no longer ships them, so the grid shows no
   app icon. Ask the user, then ship a patch that sets `desktop_page` to "Apps".
9. **App code reads the old doctypes.** Search for `Workspace Sidebar` and `Desktop Icon` in
   `has_permission` hooks and other runtime code, not only in patches. One app's
   `check_app_permission` read `Workspace Sidebar` and had to read `Sidebar`.
10. **`route: "/desk"` or `app_home = "/desk"`** outranks the Dock and opens the desk root. Remove
   them so the tile opens the first Dock entry.

## 5. Patches for live sites

Every database change you make by hand on a dev site needs a patch, so a live site gets it on
`bench migrate`. Templates are in `references/patches.md`.

| Hand change on dev | Patch | Section |
|---|---|---|
| Old `Workspace Sidebar` retitled (pitfall 1) | retitle to exact module name | `[pre_model_sync]`: runs before frappe's post_model_sync conversion |
| Site Sidebar row deleted (pitfall 2) | delete site rows of the module when a standard one exists | `[post_model_sync]`: runs after frappe's conversion and after model sync imports the app's Sidebar |
| Site Dock layer merged (pitfall 7) | put standard entries first in the site layer | `[post_model_sync]` |
| Desktop switched to Apps (pitfall 8) | `set_single_value("Desktop Settings", "desktop_page", "Apps")` | `[post_model_sync]` |
| Old nav code guarded (pitfall 3) | code guard, no patch | — |

Order of a migrate: all apps' pre_model_sync patches, then `sync_all` (imports Sidebar and Dock
files), then all apps' post_model_sync patches. Within one section, frappe runs first, then the
other apps in `apps.txt` order.

Test each delete patch on dev:
1. Insert a temporary site row with a unique test name.
2. Run the patch's `execute()` in `bench --site X console`.
3. Check that the test row is gone and the standard Sidebar is still there.
Delete only what you created, filtered by name.

## 6. Browser check

Ask the user to check each app in a browser, logged in as a user with access to the app:

1. Open `/desk`. The Apps screen shows the app's tile, with its logo or a letter icon.
2. Click the tile. The URL is inside the app, not the desk root.
3. Check the left rail. It lists the app's Dock entries.
4. Check the sidebar. It shows the curated links of the module.
5. Click a few sidebar links. Each one opens with no error.

If the tile does nothing, see pitfall 5 before you change code.

Check the server side yourself in `bench --site X console`:

```python
frappe.get_hooks("add_to_apps_screen")  # the tile hooks of all apps
frappe.get_all("Dock", fields=["name", "app", "standard", "user"])
frappe.get_all("Sidebar", filters={"app": "<app>"}, fields=["name", "module", "standard"])
frappe.db.get_single_value("Desktop Settings", "desktop_page")  # "Apps" shows the tiles
```

When the bench sets `serve_default_site: true` in `sites/common_site_config.json`, every
request goes to `default_site`. To check another site, start a second server with
`bench --site X serve --port <free port>`, and stop it after the check.

## 7. Rules

- Do not edit frappe, erpnext or hrms. If a frappe bug blocks you, report the exact error and
  file:line, and fix the data on the site instead.
- Do not commit, push or deploy unless the user asks.
- Delete only site records you created or the user named. Never bulk-delete. Never delete
  Deleted Document or Error Log rows.
- Ask every user decision with AskUserQuestion: apps in scope, companions, deleting site
  rows, deleting a module from the navigation.
