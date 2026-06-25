---
layout: default
title: Practical Implementation
parent: GodotGAS - Addon
nav_order: 3
---

# Practical Implementation (How-To)

Now that you understand the core concepts and have configured your tags and stats in the Editor Dashboard, it is time to build a working combat loop.

This guide will walk you through the three foundational steps of GodotGAS: Setting up a Character, creating a Fireball Ability, and applying a Damage Effect.

---

## 1. Setting up a Character

Any entity that engages in combat (Player, Enemy, Boss, or Destructible Prop) requires an `AbilitySystemComponent` (ASC) and an `AttributeSet`.

### Step 1: Add the ASC
1. Open your character's scene (e.g., `player.tscn`).
2. Add a new child node to the root and search for **AbilitySystemComponent**.
3. Name it exactly `AbilitySystemComponent` (this is a standard convention, though not strictly required).

### Step 2: Link an Attribute Set
1. In the Inspector for your new ASC, locate the `Attribute Sets` array.
2. Add an element to the array.
3. Click the empty slot and select **New [YourGeneratedSetName]** (e.g., `New PlayerStats`).
   * *Note: If you do not see your custom stats, ensure you generated the script via the GodotGAS Dashboard!*

Your character is now fully initialized and ready to receive buffs, take damage, and cast abilities.

---

## 2. Creating an Ability (Fireball)

Abilities are discrete Nodes that handle input, animation, and logic.

### Step 1: The Script
Create a new script called `ga_fireball.gd` and have it extend `GameplayAbility`.

```gdscript
extends GameplayAbility

@export var projectile_scene: PackedScene
@export var damage_effect: GameplayEffect

func _activate_ability() -> bool:
	# 1. Pay the mana cost and trigger the cooldown
	commit_ability()
	
	# 2. Trigger the casting audio/visuals via the Cue Manager
	execute_cue(GameplayTags.Cue_SFX_Fireball_Cast)
	
	# 3. Spawn the physical projectile
	var fireball = projectile_scene.instantiate()
	fireball.global_position = owner_asc.get_parent().global_position
	
	# Pass the damage data to the fireball so it knows what to do when it hits
	fireball.setup(self, damage_effect) 
	
	get_tree().current_scene.add_child(fireball)
	
	return true
```
