---
layout: default
title: Import & Export
parent: Godot DataTables - Addon
nav_order: 4
---

# Import & Export
{: .no_toc }

While the internal Godot DataTables editor is highly robust, there are times when it is simply faster to balance large datasets using external spreadsheet tools like Microsoft Excel or Google Sheets.

The addon provides a fully integrated I/O pipeline supporting both **CSV** and **JSON** formats.

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## Exporting Data

If you want to edit your table externally, the safest workflow is to export it first to get the correct formatting template.

1. Open your table in the **DataTable** dock.
2. Click the **Export** button in the toolbar.
3. Choose your format (`.csv` or `.json`) and save location.

*Note: Enums are exported as their underlying integer value (e.g., 0, 1, 2), ensuring math and database queries remain fast and lightweight outside the engine.*

## Importing Data

When you are ready to bring your balanced data back into Godot:

1. Click the **Import** button in the toolbar.
2. Select your modified `.csv` or `.json` file.
3. **Conflict Resolution:** If the tool detects that row IDs in your import file already exist in the active table, a popup will appear asking how you want to handle the conflict:
    * **Overwrite Duplicates:** The external data will completely replace the existing row data.
    * **Skip Duplicates:** The existing row is protected, and only brand-new rows from the file are added.

### Type Safety During Import
Godot DataTables performs aggressive sanitization during the import process. It reads your compiled Data Structure schema and attempts to automatically cast the raw CSV/JSON strings back into native Godot objects (Colors, Vectors, Booleans, etc.). 

If a piece of external data absolutely cannot be cast to your required schema type, the tool will safely substitute the property's default value to prevent engine crashes.
