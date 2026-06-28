---
layout: default
title: Practical Implementation
parent: GodotGAS - Addon
nav_order: 3
---

# Practical Implementation (How-To)

Now that you understand the core concepts and have configured your tags and stats in the Editor Dashboard, it is time to build a working combat loop.

This guide will walk you through the foundational steps of GodotGAS: Setting up a Character, creating a Fireball Ability, configuring its Costs/Cooldowns, applying a Damage Effect, and writing complex Execution Calculations.

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

`gdscript
extends GameplayAbility

@export var projectile_scene: PackedScene
@export var damage_effect: GameplayEffect

func _activate_ability() -> bool:
	# 1. Pay the mana cost and trigger the cooldown automatically
	commit_ability()
	
	# 2. Trigger the casting audio/visuals via the Cue Manager
	execute_cue(GameplayTags.Cue_SFX_Fireball_Cast)
	
	# 3. Spawn the physical projectile
	var fireball = projectile_scene.instantiate()
	fireball.global_position = owner_asc.get_parent().global_position
	
	# Pass the ability reference and the damage data down to the fireball
	fireball.setup(self, damage_effect) 
	
	get_tree().current_scene.add_child(fireball)
	
	return true
`

### Step 2: Granting the Ability
1. Create a new Node in your project, attach your `ga_fireball.gd` script, and save it as `ga_fireball.tscn`.
2. To give this ability to your player, you must grant it to their ASC. You can do this in the player's `_ready()` function:

`gdscript
@onready var asc = $AbilitySystemComponent
var fireball_ability = preload("res://abilities/ga_fireball.tscn").instantiate()

func _ready():
    asc.grant_ability(fireball_ability)
`

---

## 3. Configuring Costs and Cooldowns

In GodotGAS, you do not write hardcoded timers for cooldowns, nor do you manually subtract mana in your scripts. Both of these mechanics are driven purely by `GameplayEffect` resources. When your script calls `commit_ability()`, the ASC automatically applies these effects to the caster.

### Creating the Cost Effect
1. Create a new `GameplayEffect` resource and name it `ge_fireball_cost.tres`.
2. Set the **Duration Policy** to `Instant`.
3. Under **Modifiers**, add a new modifier. 
4. Set the **Attribute Name** to `mana` (or whichever resource your spell uses).
5. Set the **Operation** to `Add`, and the **Magnitude** to `-20.0`.
   
### Predictive Resource Costs
The ASC actively predicts dynamic resource costs. If your spell's mana cost is dynamically modified by an Execution Calculation (e.g., a "Blood Magic" passive that doubles mana costs), the `can_afford_cost()` function generates a mock Spec, runs the math predictively, and correctly blocks the cast if the mutated cost exceeds the player's current mana.

### Creating the Cooldown Effect
1. Create another `GameplayEffect` resource and name it `ge_fireball_cooldown.tres`.
2. Set the **Duration Policy** to `Duration`.
3. Set the **Duration** to `5.0` (seconds).
4. Under **State Management -> Granted Tags**, add a tag to represent the cooldown, such as `State.Cooldown.Fireball`.
   * *How it works:* The ASC applies this effect and grants the tag for exactly 5 seconds. If the player tries to cast Fireball again, the framework sees the tag and blocks the cast until the 5 seconds are up.

### Linking them to the Ability
Select your `ga_fireball.tscn` node. In the Inspector, under **Ability Mechanics**, drag and drop your `ge_fireball_cost.tres` into the `Cost Effect` slot, and `ge_fireball_cooldown.tres` into the `Cooldown Effect` slot.

---

## 4. Building the Damage Effect

When the Fireball hits an enemy, it needs to apply a separate `GameplayEffect` to deal damage.

### Step 1: Create the Resource
1. Create a new `GameplayEffect` resource and save it as `ge_fireball_damage.tres`.

### Step 2: Configure the Math
1. Open the resource in the Inspector.
2. Set **Duration Policy** to `Instant` (Damage happens immediately).
3. Under **Modifiers**, add a new `GameplayEffectModifier`.
4. Set the **Attribute Name** exactly as it appears in your Attribute Set (e.g., `health`).
5. Set the **Operation** to `Add`.
6. Set the **Magnitude** to `-50.0` (Negative numbers subtract!).

---

## 5. The Projectile Payload (TargetData)

How does the physical fireball projectile actually pass the `ge_fireball_damage.tres` to the Enemy's ASC? It uses the framework's **TargetData** architecture.

Here is an example of what the script on your `FireballProjectile.tscn` (an `Area2D` or `Area3D`) should look like. Notice how it receives the data from the ability in the `setup()` function, and then packages the enemy it hits into a `GameplayAbilityTargetData` object:

`gdscript
extends Area2D

var owning_ability: GameplayAbility
var damage_effect: GameplayEffect

# Called by ga_fireball.gd right before adding this projectile to the scene
func setup(ability: GameplayAbility, effect: GameplayEffect) -> void:
	owning_ability = ability
	damage_effect = effect

func _on_body_entered(body: Node) -> void:
	# 1. Package the entity we just hit into a standard payload
	var target_data = GameplayAbilityTargetData.new()
	target_data.append_node(body)
	
	# 2. Tell the original ability to apply the effect to whoever we hit!
	if owning_ability and damage_effect:
		owning_ability.apply_effect_to_targets(damage_effect, target_data)
		
	# 3. Destroy the projectile
	queue_free()
`

---

## 6. Advanced Math: Execution Calculations

While static Modifiers are great for basic logic, complex games require dynamic formulas (e.g., `Damage = (Attack * 1.5) - Armor`). Furthermore, you might want to dynamically alter a spell's cooldown based on a "Haste" stat.

This is where **Execution Calculations (ExecCalcs)** come in. When an ExecCalc runs, it intercepts the live `GameplayEffectSpec` *before* any math is applied. You have two primary workflows:

### Workflow A: The Mutator (Changing Rules)
You can directly alter the blueprint of the effect by mutating the `GameplayEffectSpec` payload. This allows you to dynamically scale cooldown durations, tick periods, or base modifier magnitudes. You return an empty dictionary since you are just tweaking the Spec's rules.

`gdscript
class_name CalcCooldownOverride extends GameplayExecutionCalculation

func execute(spec: GameplayEffectSpec, target_asc: AbilitySystemComponent) -> Dictionary:
	# Example: Cooldown Reduction based on Haste
	var haste_stat = target_asc.get_attribute("haste")
	var haste = haste_stat.current_value if haste_stat else 0.0
	
	# Reduce duration by 1% for every point of Haste
	spec.duration *= maxf(0.1, 1.0 - (haste / 100.0))
	
	return {}
`

### Workflow B: The Calculator (Combat Math)
If you want to perform cross-entity math, you use the Calculator workflow. Instead of mutating the spec, you calculate the math and return the flat numerical damage as a Dictionary. The ASC automatically merges this into the final damage pool.

`gdscript
class_name CalcPhysicalDamage extends GameplayExecutionCalculation

func execute(spec: GameplayEffectSpec, target_asc: AbilitySystemComponent) -> Dictionary:
	var instigator = spec.context.instigator
	var source_asc = instigator.get_node_or_null("AbilitySystemComponent")
	
	if not source_asc or not target_asc:
		return {}
		
	var attack_power = source_asc.get_attribute("attack").current_value
	var armor = target_asc.get_attribute("defence").current_value
	
	var final_damage = maxf(1.0, (attack_power * 1.5) - armor) 
	
	# Inject dynamic tags for the UI to read later!
	if final_damage > 100:
		spec.inject_tag(GameplayTags.Example_Event_Damage_Critical)
	
	# Return the exact attribute we want to modify, and the flat amount
	return {"health": -final_damage}
`

### Applying the ExecCalc
1. Go back to your `GameplayEffect` resource file.
2. Under the **Attribute Modifiers -> Executions** array, add a new element.
3. Click the dropdown and select **New CalcPhysicalDamage** (or whatever you named your script). 

Now, whenever the effect is applied, GodotGAS will automatically route the payload into your custom math script before touching the enemy's attributes!
