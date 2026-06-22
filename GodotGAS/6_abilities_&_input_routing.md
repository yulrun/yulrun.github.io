---
layout: default
title: 6. Gameplay Abilities & Input Routing
parent: GodotGAS - Addon
nav_order: 9
---

# 6. Gameplay Abilities & Input Routing

While Tags, Attributes, and Effects manage the *state* and *math* of your game, **Gameplay Abilities** define the *actions*. An ability can be anything from a standard melee attack or a fireball, to a passive aura or an item interaction. 

GodotGAS ensures that your abilities remain highly modular and entirely decoupled from your hardware inputs.

---

## Granting & Binding Abilities

Before an entity can cast an ability, it must be "granted" to their Ability System Component (ASC). This is typically done during the entity's initialization.

When you grant an ability, the ASC holds it in memory, ready to be activated. However, to trigger it via a button press, you must bind it to an **Input ID** (an integer). This integer-based approach makes it incredibly easy to build dynamic hotbars or remappable skill slots using a standard `enum`.

```gdscript
# Example of binding a granted ability to an input slot (e.g., Slot 0)
ability_system.bind_ability_to_input(fireball_ability, 0)
```

---

## Decoupled Hardware Input

A common mistake in Godot development is hardcoding hardware inputs directly inside action scripts (e.g., `if Input.is_action_just_pressed("ui_accept"): cast_fireball()`). This creates rigid code and makes remapping keys or building AI controllers extremely difficult.

GodotGAS completely decouples hardware input from the ability execution.

### How Input Routing Works:
1. **The Controller Listens:** Your standard Godot character script (e.g., `CharacterBody3D`) listens for hardware inputs.
2. **Routing to the ASC:** When a button is pressed, the character script does not call the ability directly. Instead, it passes the integer Input ID to the ASC.
3. **The ASC Triggers the Ability:** The ASC looks at its list of active abilities, finds the one mapped to that specific Input ID, and triggers its internal pressed/released events.

```gdscript
# Inside your Player Character script

func _unhandled_input(event: InputEvent) -> void:
    if event.is_action_pressed("combat_attack_1"):
        # The hardware input is converted into a generic ASC command for Slot 0
        ability_system.ability_local_input_pressed(0)
        
    elif event.is_action_released("combat_attack_1"):
        # Releasing is also routed, allowing for complex behaviors like charge-ups
        ability_system.ability_local_input_released(0)
```

### Charge-ups & Channeling
Because the ASC separately routes both `pressed` and `released` events, your `GameplayAbility` scripts can easily handle complex behaviors natively. You can write an ability that charges up over time and unleashes a massive payload strictly when the ASC notifies it that the input has been released.

---

## Event-Driven Waking

Not all abilities are triggered by buttons. Some are passive or reactive (e.g., "Thorns" armor that damages attackers when you get hit).

GodotGAS supports **Event-Driven Abilities**. Instead of waiting for an input, an ability can be configured to listen for a specific `GameplayTag` payload broadcasted to the ASC. When the ASC receives an event matching the ability's trigger tag (e.g., `Event.Combat.TookDamage`), it instantly wakes the ability up, passing along the context (who hit you, how much damage) so the ability can react accordingly.

---

## Building a Basic Ability

To create a new ability, you create a script extending `GameplayAbility` and override its core virtual functions. The most important is the activation logic, where you typically construct your `TargetData`, build your `GameplayEffectContext`, generate a `GameplayEffectSpec`, and apply it to the target's ASC.

Once the ability finishes its animation, logic, or channeling, you **MUST** explicitly end the ability to tell the ASC to clean up the state and remove any active execution tags.
