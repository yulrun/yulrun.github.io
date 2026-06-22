---
layout: default
title: 1. Introduction & Quick Start
parent: GodotGAS - Addon
nav_order: 4
---

# 1. Introduction & Quick Start

Welcome to the GodotGAS framework! This guide will walk you through installing the plugin, understanding the foundational concepts, and getting your first entity ready to use the Gameplay Ability System.

## What is GodotGAS?

GodotGAS is a data-driven architecture that standardizes how entities in your game interact. Instead of writing custom damage scripts, stun timers, and overlapping buff logic on every character, GodotGAS provides a unified pipeline:

* **Abilities** generate data (e.g., "I am casting a fireball").
* **Effects** apply mathematical modifications or state changes (e.g., "Take 50 Damage" or "Apply Stun").
* **Tags** act as the connective tissue, dictating rules (e.g., "You cannot cast Fireball if you have the Status.Stunned tag").
* **The Ability System Component (ASC)** simply processes and routes this data.

By decoupling your logic, your character controllers stay clean, and your designers can build complex interactions purely through data resources.

---

## Installation

Installing GodotGAS is as simple as adding any standard Godot plugin.

1. Download the latest `v1.0.0` release of GodotGAS.
2. Extract the `addons/GodotGAS` folder into your Godot project's `addons/` directory.
3. Open your project in **Godot 4.6+**.
4. In the top menu, navigate to **Project** -> **Project Settings** -> **Plugins**.
5. Check the **Enable** box next to GodotGAS.

> **Note:** Enabling the plugin automatically registers two highly optimized Autoloads: `GameplayTagManager` and `GameplayCueManager`. It also activates the custom GodotGAS Editor Dashboard in your bottom panel.

---

## Your First ASC (Hello World)

The heart of the framework is the **Ability System Component** (ASC). Any entity in your game that needs health, mana, buffs, or abilities needs an ASC. It is a standard Godot `Node` that acts as the brain for that specific entity.

### Step-by-Step Setup

1. Open the scene of your player or enemy character (e.g., a `CharacterBody3D` or `CharacterBody2D`).
2. Add a new child node and search for `AbilitySystemComponent`.
3. Add it to your character's scene tree.

### The Lifecycle & Teardown

Because the ASC manages deep references—such as active effects, attributes, and object-pooled visual cues—it is critical to cleanly tear it down before the parent node is deleted or freed. GodotGAS includes a dedicated `cleanup()` method exactly for this purpose.

Here is an example of a basic character script utilizing the ASC:

```gdscript
extends CharacterBody3D

@onready var ability_system: AbilitySystemComponent = $AbilitySystemComponent

func _ready() -> void:
    # The ASC is initialized automatically on _ready().
    # It is now ready to receive tags, attributes, and effects!
    pass

func _exit_tree() -> void:
    # CRITICAL: Safely teardown the ASC before this character is deleted.
    # This prevents memory leaks, dangling references, and ensures
    # any active pooled Object Cues are returned safely to the Cue Manager.
    if ability_system:
        ability_system.cleanup()
```
