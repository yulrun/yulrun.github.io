---
layout: default
title: Wiring to the Game (UI & Input)
parent: GodotGAS - Addon
nav_order: 4
---

# Wiring it to the Game (UI & Input)

GodotGAS operates as a completely decoupled "black box" of combat math and state. It does not read directly from Godot's `InputMap`, nor does it know what a `ProgressBar` is. 

To make your game playable, you must wire the `AbilitySystemComponent` (ASC) to your Player Controller and your UI framework.

---

## 1. Input Handling (Casting Abilities)

Instead of hardcoding inputs directly inside an ability script (e.g., `if Input.is_action_just_pressed("ui_accept")`), GodotGAS uses an **Input ID** routing system. This allows you to dynamically rebind abilities to different action bar slots at runtime!

### Step 1: Bind the Ability to a Slot (ID)
When you grant an ability to a player, assign it an Integer ID. Think of this ID as an "Action Bar Slot" (e.g., Slot 1, Slot 2).

```gdscript
@onready var asc = $AbilitySystemComponent
var fireball_ability = preload("res://abilities/ga_fireball.tscn").instantiate()

func _ready():
    # Grant the ability to the ASC
    asc.grant_ability(fireball_ability)
    
    # Bind the Fireball ability to Input ID '1'
    asc.bind_ability_to_input(fireball_ability, 1)
```

### Step 2: Route Hardware Input to the ASC
In your Player script, listen for hardware inputs and forward the corresponding ID to the ASC. The ASC will automatically find whichever ability is bound to that ID and trigger its activation pipeline.

```gdscript
func _input(event: InputEvent) -> void:
    # If the user presses the "Skill_1" key...
    if event.is_action_pressed("Skill_1"):
        # Tell the ASC that Input ID 1 was pressed
        asc.ability_local_input_pressed(1)
        
    elif event.is_action_released("Skill_1"):
        # Tell the ASC that Input ID 1 was released (Useful for charging attacks!)
        asc.ability_local_input_released(1)
```

---

## 2. Health Bars (The Attribute Changed Signal)

Never put UI-updating logic inside Godot's `_process(delta)` function to check if health changed. This is highly inefficient. Instead, listen to the ASC's data broadcasts.

Whenever an attribute's value successfully changes, the ASC emits the `attribute_changed` signal.

```gdscript
extends ProgressBar

@export var target_asc: AbilitySystemComponent

func _ready() -> void:
    # Connect to the ASC's data broadcast
    target_asc.attribute_changed.connect(_on_attribute_changed)

func _on_attribute_changed(attribute_name: String, old_value: float, new_value: float, spec: GameplayEffectSpec) -> void:
    # We only care about the health attribute for this UI bar
    if attribute_name == "health":
        value = new_value
```

---

## 3. Buff and Debuff Icons

When a player receives a Damage Over Time (DoT) effect or a temporary Buff, you want to show an icon on the screen with a duration sweep.

The ASC automatically manages arrays of active effects and broadcasts exactly when they are added or removed.

```gdscript
extends HBoxContainer

@export var target_asc: AbilitySystemComponent

func _ready() -> void:
    target_asc.active_effect_added.connect(_on_buff_added)
    target_asc.active_effect_removed.connect(_on_buff_removed)

func _on_buff_added(active_effect: ActiveGameplayEffect) -> void:
    var effect_def = active_effect.get_effect_def()
    
    # You can read the resource name, or custom icon exports you add to your GameplayEffects!
    print("UI: Spawn a new icon for ", effect_def.resource_name)
    
    # You can also read the duration to start a UI sweep
    var duration = effect_def.duration

func _on_buff_removed(active_effect: ActiveGameplayEffect) -> void:
    print("UI: Remove the icon for ", active_effect.get_effect_def().resource_name)
```

---

## 4. Floating Combat Text (The Narrative of Combat)

**The Polling Trap:** Do not use the `attribute_changed` signal to spawn Floating Combat Text! If a player dodges an attack, or has a shield that completely blocks the damage, their Health changes by `0`. The attribute signal will never fire, and the player won't know they dodged!

Instead, listen to the **Lifecycle of the Effect**. When the ASC fully processes an incoming attack, it emits an `effect_received` signal along with the `GameplayEffectSpec`.

By combining this with **Dynamic Tags** injected by your Execution Calculations, the UI knows exactly what happened.

```gdscript
extends Node2D

@export var target_asc: AbilitySystemComponent

func _ready() -> void:
    # Listen for incoming attacks/heals/effects
    target_asc.effect_received.connect(_on_effect_received)

func _on_effect_received(source_asc: AbilitySystemComponent, spec: GameplayEffectSpec) -> void:
    # 1. Did we Dodge/Miss entirely?
    if spec.has_tag(GameplayTags.Event_Combat_Missed):
        spawn_floating_text("Miss!", Color.GRAY)
        return
        
    # 2. Get the actual damage dealt (calculated in the ExecCalc or Modifiers)
    var damage_taken = spec.calculated_deltas.get("health", 0.0)
    
    # Ignore if this effect didn't touch health (e.g., a pure mana drain)
    if damage_taken == 0.0:
        return
        
    # 3. Was it a Critical Hit?
    if spec.has_tag(GameplayTags.Event_Combat_Critical):
        spawn_floating_text(str(abs(damage_taken)) + "!", Color.ORANGE_RED)
    else:
        # Standard Hit
        spawn_floating_text(str(abs(damage_taken)), Color.WHITE)
```

### Summary
By utilizing these four methods (`bind_ability_to_input`, `attribute_changed`, `active_effect_added`, and `effect_received`), your UI and Player Controllers remain lightweight, responsive, and completely disconnected from the heavy math occurring inside the GodotGAS framework.
