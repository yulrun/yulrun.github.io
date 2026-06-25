---
layout: default
title: API Reference (ASC Signals)
parent: GodotGAS - Addon
nav_order: 5
---

# API Reference (ASC Signals)

The `AbilitySystemComponent` (ASC) acts as the central event bus for your entity's combat state. Because the framework is fully decoupled, your custom game logic (UI, Animation State Machines, AI, Passives) should rely heavily on listening to these signals rather than checking variables every frame.

Below is the complete list of signals emitted by the `AbilitySystemComponent`, their arguments, and common use cases.

---

## State & Data Signals

These signals fire whenever the core mathematical or tag state of the entity changes.

### `attribute_changed`
**Fired when:** An attribute's `current_value` is mathematically modified by a `GameplayEffect`.
* **Arguments:**
    * `attribute_name` *(String)*: The name of the stat (e.g., "health").
    * `old_value` *(float)*: The value before the math was applied.
    * `new_value` *(float)*: The final clamped value after the math.
    * `effect_spec` *(GameplayEffectSpec)*: The payload that caused the change.
* **Use Case:** Updating UI progress bars (Health, Mana, Stamina) or triggering a "Death" state if `new_value <= 0`.

### `tag_added`
**Fired when:** A specific `GameplayTag` is added to the entity for the first time (its internal stack count goes from 0 to 1).
* **Arguments:**
    * `tag` *(StringName)*: The tag that was added (e.g., `Status.Stunned`).
* **Use Case:** Forcing an AnimationTree transition. (e.g., If `Status.Stunned` is added, switch to the dizzy animation state).

### `tag_removed`
**Fired when:** A specific `GameplayTag` is completely purged from the entity (its internal stack count drops to 0).
* **Arguments:**
    * `tag` *(StringName)*: The tag that was removed.
* **Use Case:** Restoring an entity's default state. (e.g., If `Status.Stunned` is removed, return to the Idle animation).

### `tag_count_changed`
**Fired when:** An existing tag receives another stack, or drops a stack but remains active (e.g., going from 2 stacks of burning down to 1 stack).
* **Arguments:**
    * `tag` *(StringName)*: The tag being modified.
    * `new_count` *(int)*: The new integer stack count.
* **Use Case:** Updating a tiny number on a UI debuff icon to show how many stacks of a DoT the player currently has.

---

## Effect Lifecycle Signals

These signals fire specifically when `Duration` or `Infinite` effects are attached to or expire from the ASC.

### `active_effect_added`
**Fired when:** A new ongoing effect (Buff/Debuff/Cooldown) successfully attaches to the entity.
* **Arguments:**
    * `active_effect` *(ActiveGameplayEffect)*: The live instance of the effect.
* **Use Case:** Spawning a Buff Icon in the UI. You can read `active_effect.get_effect_def().duration` to start a radial cooldown sweep on the icon.

### `active_effect_removed`
**Fired when:** An ongoing effect reaches the end of its duration naturally, or is forcefully cleansed/purged.
* **Arguments:**
    * `active_effect` *(ActiveGameplayEffect)*: The instance that just expired.
* **Use Case:** Finding and deleting the specific Buff Icon associated with this effect from the UI container.

---

## Combat Narrative Signals

These signals are the core of your feedback loop, telling the entity *what just happened* during an interaction.

### `effect_received`
**Fired when:** This ASC successfully processes an incoming effect cast by someone else (or itself). This is the **Defender's** perspective.
* **Arguments:**
    * `source_asc` *(AbilitySystemComponent)*: The ASC of the attacker/caster.
    * `spec` *(GameplayEffectSpec)*: The payload containing the damage/healing data and dynamic tags (like Critical Hit).
* **Use Case:** Spawning **Floating Combat Text** (Damage Numbers, "Miss!", "Blocked!"). Also used for reactive passive abilities, like "Thorns" (dealing damage back to the `source_asc`).

### `effect_applied_to_target`
**Fired when:** This ASC successfully casts an effect onto another entity. This is the **Attacker's** perspective.
* **Arguments:**
    * `target_asc` *(AbilitySystemComponent)*: The ASC of the enemy you just hit.
    * `spec` *(GameplayEffectSpec)*: The payload you sent them.
* **Use Case:** Triggering "Hit Markers" in the UI, or triggering "Life Leech" passives (e.g., "Heal yourself for 10% of the damage you just dealt").

### `gameplay_event_received`
**Fired when:** A global, tag-based event is sent directly to this ASC.
* **Arguments:**
    * `event_tag` *(StringName)*: The specific event (e.g., `Event.Environment.TrapSprung`).
    * `payload` *(GameplayEffectContext)*: The context of who triggered the event.
* **Use Case:** Bypassing normal input. The ASC will automatically check if it has any granted `GameplayAbilities` listening for this specific `event_tag` and immediately activate them if it does.

---

## Input & Gatekeeping Signals

These signals relate to the player physically trying to use abilities.

### `ability_activation_failed`
**Fired when:** The player presses a button to cast an ability, but the ASC rejects the attempt based on its gatekeeping rules.
* **Arguments:**
    * `ability` *(GameplayAbility)*: The specific ability node that failed.
    * `reason` *(ActivationError Enum)*: The strict reason (e.g., `ON_COOLDOWN`, `MISSING_TAG`, `INSUFFICIENT_RESOURCES`).
    * `payload` *(Dictionary)*: Context data (e.g., which specific tag blocked it, or how much mana it was missing).
* **Use Case:** Flashing the screen red, playing a "dull thud" error sound, or popping up standard MMO text like *"Not enough Mana!"* or *"That ability is still on cooldown."*
