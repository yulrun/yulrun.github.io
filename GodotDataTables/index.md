---
layout: default
title: Godot DataTables - Addon
nav_order: 2
parent: "<img src='/assets/images/icon_labs.png' style='height: 21px; vertical-align: middle; margin-right: 4px; margin-bottom: 4px;'>Labs"
has_children: true
heading_only: false
---

<div align="center">
  <img src="../assets/images/godot-datatables-medium.png" alt="Godot DataTables Logo">
</div>

# GodotDataTables Framework

**Current Version:** [v1.0.0](https://github.com/yulrun/godot-data-tables-addon)

This plugin brings Unreal Engine's renowned "DataTable" paradigm directly into Godot 4.7+. Say goodbye to relying on generic JSON/CSV files that break static typing and lack native engine integration. Godot DataTables provides a highly scalable, visual workflow for managing complex datasets—all written natively in GDScript.

## Core Features in v1.0.0

### Visual Schema Generator
Define your dataset blueprints (e.g., "ItemData", "SpellStats") visually in the new Data Structure editor dock. The tool automatically compiles your visual layout into fully documented, strictly-typed `.gd` script files, ensuring massive runtime performance benefits and flawless IDE autocomplete.

### Native Engine Integration
Move beyond simple strings and floats. Godot DataTables seamlessly supports native Godot types:
* Drag-and-drop `Texture2D`, `PackedScene`, and custom instantiable `Resource` files directly into your spreadsheet cells.
* Full visual editor support for `Color`, `Vector2`, `Vector3`, and `bool` checkboxes.

### Native Enumerator & Array Support
* **Enums:** Visually define custom dropdown options in the schema builder. These compile directly into native `@export_enum` properties and render as clickable dropdowns in your spreadsheet.
* **Arrays:** Full support for editing Godot 4 strictly-typed arrays directly within the grid, featuring a dedicated popup editor with data validation, clamping, and drag-and-drop reordering.

### DataTableRowHandle (Inspector Plugin)
Eliminate "magic strings" and typo-related crashes. The `DataTableRowHandle` is a specialized Resource that draws a sleek, two-step dropdown (`Table` -> `Row ID`) directly in the Godot Inspector, safely linking your databases to your game logic.

### Decoupled System Bridge (GodotGAS Ready)
Includes a built-in `get_row_as_dictionary(numeric_only)` helper. This dynamically unpacks your strongly-typed row into a primitive Godot Dictionary, strictly extracting and casting numbers. It is the perfect bridge for feeding base stat overrides directly into standalone ability systems (like GodotGAS) without creating hard compile-time dependencies.

### Dedicated Editor Spreadsheet Dock
A highly reactive, polished spreadsheet workspace embedded directly in the Godot Editor.
* **Safety First:** Robust Lock/Revert safety mechanics prevent accidental data loss. Unsaved row modifications are highlighted instantly.
* **Quality of Life:** Features advanced text filtering, column sorting, and 1-click duplicate/delete row actions.

### Import & Export Pipeline
Prefer to balance your game's stats in Excel? Safely bridge external data by parsing standard CSV or JSON files directly into your strongly-typed Godot tables, or export your existing tables for external viewing.
