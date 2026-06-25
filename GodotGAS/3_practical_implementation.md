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

### Step 2: Configuring the Ability Node
1. Create a new Node in your project and attach your `ga_fireball.gd` script. Save it as `ga_fireball.tscn`.
2. Select the node and look at the Inspector. You will see the GodotGAS "Ability Rules" and "Ability Mechanics" categories.
3. **Gatekeeping:** Assign a tag to `Activation Blocked Tags` (e.g., `Status.Stunned`). If the player is stunned, the ASC will automatically block the cast.
4. **Costs:** Drag a `GameplayEffect` resource that subtracts Mana into the `Cost Effect` slot.

### Step 3: Granting the Ability
To give this ability to your player, you must grant it to their ASC. You can do this in the player's `_ready()` function:

```gdscript
@onready var asc = $AbilitySystemComponent
var fireball_ability = preload("res://abilities/ga_fireball.tscn").instantiate()

func _ready():
    asc.grant_ability(fireball_ability)
```

---

## 3. Building the Damage Effect

When the Fireball hits an enemy, it shouldn't just subtract health directly. It needs to apply a `GameplayEffect`.

### Step 1: Create the Resource
1. Right-click in your FileSystem and select **New Resource -> GameplayEffect**.
2. Save it as `ge_fireball_damage.tres`.

### Step 2: Configure the Math
1. Open the resource in the Inspector.
2. Set **Duration Policy** to `Instant` (Damage happens immediately).
3. Under **Modifiers**, add a new `GameplayEffectModifier`.
4. Set the **Attribute Name** exactly as it appears in your Attribute Set (e.g., `health`).
5. Set the **Operation** to `Add`.
6. Set the **Magnitude** to `-50.0` (Negative numbers subtract!).

*Note: For complex, scaling damage (e.g., `Damage = Attacker.Magic - Target.FireResist`), you would leave Modifiers empty and instead assign a custom `GameplayExecutionCalculation` script.*

---

## 4. The Projectile Payload (TargetData)

How does the Fireball pass the `ge_fireball_damage.tres` to the Enemy's ASC? Through `TargetData`.

Inside your Fireball projectile script, when an `Area2D` detects an overlap with an enemy:

```gdscript
func _on_body_entered(body: Node):
	# 1. Package the hit data
	var target_data = GameplayAbilityTargetData.new()
	target_data.append_node(body)
	
	# 2. Tell the ability to apply the effect to whoever we hit!
	# (owning_ability and damage_effect were passed during fireball.setup())
	owning_ability.apply_effect_to_targets(damage_effect, target_data)
	
	queue_free()
```

The `apply_effect_to_targets` function is a built-in framework helper. It automatically grabs the enemy's ASC, wraps your `GameplayEffect` in a live `GameplayEffectSpec`, passes along who cast it, and executes the math safely.
