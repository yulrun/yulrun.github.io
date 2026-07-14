---
layout: default
title: DataTableRowHandle
parent: Godot DataTables - Addon
nav_order: 3
---

# DataTableRowHandle
{: .no_toc }

Accessing data in code using raw strings (e.g., `get_item("sword_01")`) is a common source of typos and runtime crashes. 

To solve this, the addon includes the `DataTableRowHandle`, a specialized Resource that provides a seamless, error-free bridge between your gameplay code and your DataTables.

<details open markdown="block">
  <summary>
    Table of contents
  </summary>
  {: .text-delta }
1. TOC
{:toc}
</details>

---

## The Inspector Plugin
Instead of exporting strings in your Nodes, export a `DataTableRowHandle`.

```gdscript
@export var starting_weapon: DataTableRowHandle
```

When you look at your Node in the Godot Inspector, the plugin will render a clean two-step dropdown interface:
1. **Select Table:** Pick the specific DataTable resource.
2. **Select Row:** A dropdown automatically populates with all valid `row_id` entries inside that table.

If you delete or rename a row in the DataTable editor, the Inspector will instantly flag the handle as invalid, allowing you to catch broken references long before you run the game.

---

## API Reference

Once you have a handle, extracting data is highly streamlined.

### `get_row()`
Returns the exact instantiated `DataStructure` object for the row. This provides perfect IDE autocomplete for all your custom fields.

```gdscript
var weapon_data = starting_weapon.get_row()
if weapon_data:
    print("Damage: ", weapon_data.base_damage)
```

### `get_value(property_name: StringName, array_index: int = -1)`
Safely fetches a specific value. If the property is an Array (like a leveling curve) and you provide an `array_index`, the handle will safely extract that specific element, clamping to the max size if you exceed the array bounds.

```gdscript
# Get the spell's damage at Level 5 (Index 4)
var dmg = spell_handle.get_value("damage_curve", 4)
```

### `get_row_as_dictionary(numeric_only: bool = false, array_index: int = -1)`
The ultimate integration tool. This unpacks the entire strongly-typed row into a standard Godot Dictionary.

If `numeric_only` is set to `true`, the handle strips out all Strings, Booleans, and Objects, strictly returning floats. This is the **perfect bridge** for feeding stat overrides into standalone gameplay frameworks (like GodotGAS) without creating hard dependencies.

```gdscript
# GodotGAS Integration Example
# Flattens all stats in the row at "Level 10" (Index 9) into a pure math dictionary
var stat_overrides: Dictionary = enemy_handle.get_row_as_dictionary(true, 9)

# Result: {"max_health": 500.0, "speed": 12.5, "armor": 50.0}
attributes.apply_override_dict(stat_overrides)
```
