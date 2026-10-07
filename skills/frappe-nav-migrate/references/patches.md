# Patch templates

Put each patch in `{app}/{app}/patches/v16_0/`. Add its dotted path to the right section of
`{app}/{app}/patches.txt`. A patch must be a no-op on a site that is already correct, and on a
site where frappe is older than v16.50.

## Retitle an old Workspace Sidebar (pre_model_sync)

Use this for pitfall 1. It must run before frappe's post_model_sync conversion patches.

```python
import frappe

MODULE = "Abc Erp"  # exact Module Def name


def execute():
	"""Give the old site sidebars of this module the exact module name as their title.

	Frappe v16.50 converts each old Workspace Sidebar into a Sidebar. Its validator compares the
	title with the module with case, but checks Module Def without case, so a title such as
	"ABC ERP" for module "Abc Erp" stops the migrate.
	"""
	if not frappe.db.table_exists("Workspace Sidebar"):
		return

	rows = frappe.get_all(
		"Workspace Sidebar",
		filters={"module": MODULE, "standard": 0},
		fields=["name", "title"],
	)
	for row in rows:
		if row.title != MODULE and row.title.lower() == MODULE.lower():
			frappe.db.set_value("Workspace Sidebar", row.name, "title", MODULE, update_modified=False)
```

Use `frappe.db.set_value`, not `doc.save()`. In v16.50 the `Workspace Sidebar` doctype has
no create or write permission, and its controller is reduced.

## Delete site Sidebar rows that hide the app Sidebar (post_model_sync)

Use this for pitfall 2, only after the user approves the deletion on dev.

```python
import frappe

MODULE = "Abc Module"


def execute():
	"""Delete the site Sidebar that hides the standard Abc Module Sidebar.

	Frappe v16.50 turns an old site sidebar into a site Sidebar row for this module, and a site
	row wins over the Sidebar that the app ships.
	"""
	if not frappe.db.exists("DocType", "Sidebar"):
		return

	if not frappe.db.exists("Sidebar", {"module": MODULE, "standard": 1}):
		return

	for name in frappe.get_all("Sidebar", filters={"module": MODULE, "standard": 0}, pluck="name"):
		frappe.delete_doc("Sidebar", name, ignore_permissions=True)
```

## Guard old navigation code

Put this at the top of a function that writes `Workspace Sidebar` or `Desktop Icon` rows:

```python
	# Frappe v16.50 reads a module Sidebar and a Dock that the app ships, not these site records.
	if frappe.db.exists("DocType", "Dock"):
		return
```

Put this before an old patch imports a fixture file that the app no longer ships:

```python
		if not os.path.exists(path):
			continue
```

## Switch to the Apps screen and merge the site Dock layer (post_model_sync)

Use this for pitfalls 7 and 8.

```python
import frappe

APP = "abc_app"


def execute():
	if not frappe.db.exists("DocType", "Dock"):
		return

	frappe.db.set_single_value("Desktop Settings", "desktop_page", "Apps")
	merge_standard_entries_into_site_layer()
	frappe.clear_cache()


def merge_standard_entries_into_site_layer():
	standard = frappe.db.get_value("Dock", {"app": APP, "standard": 1})
	if not standard:
		return

	layers = frappe.get_all(
		"Dock", filters={"app": APP, "standard": 0, "user": ("in", ("", None))}, pluck="name"
	)
	standard_rows = frappe.get_doc("Dock", standard).items
	for name in layers:
		layer = frappe.get_doc("Dock", name)
		present = {(row.link_type, row.link_to) for row in layer.items}
		missing = [row for row in standard_rows if (row.link_type, row.link_to) not in present]
		if not missing:
			continue

		site_rows = [row.as_dict() for row in layer.items]
		layer.set("items", [])
		# A reference to a standard entry inherits its icon and title from the Dock below it.
		for row in missing:
			layer.append("items", {"link_type": row.link_type, "link_to": row.link_to, "added": 0, "hidden": 0})
		for row in site_rows:
			layer.append(
				"items",
				{field: row.get(field) for field in ("link_type", "link_to", "url", "icon", "title", "added", "hidden")},
			)
		layer.save(ignore_permissions=True)
```

Test it: reset the layer to only its site rows, set `desktop_page` to "Desktop Icons", run
`bench --site X migrate`, then check the layer rows and the setting.
