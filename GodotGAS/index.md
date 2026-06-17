---
title: GodotGAS
nav_order: 1
---

# Chapter 1: The Godot-Native Architecture

GodotGAS is a robust, data-driven framework inspired by Unreal Engine’s Gameplay Ability System (GAS), deeply optimized for Godot 4’s Node and Resource architecture. 

The core philosophy is **composition and decoupling**: separating game logic (math, cooldowns, state) from presentation (particles, audio, animations).

## The Component Hierarchy
To give any entity (Player, Enemy, or destructible barrel) access to the framework, add an **AbilitySystemComponent (ASC)** as a child node.

* **AbilitySystemComponent (ASC):** The central "Brain" of the entity. It manages granted abilities, processes incoming gameplay effects, tracks active tags, and broadcasts state changes via signals (`tag_added`, `attribute_changed`).
* **The Data Layer (Resources):** Designers create specific `AttributeSets`, `GameplayEffects`, and `GameplayAbilities` in the inspector, which are fed into the ASC.

## The Tag Paradigm (Compile-Time over Runtime)
GodotGAS uses a **Static Generation** approach for tags, avoiding heavy runtime string parsing.

1. **GameplayTagRegistry:** A central resource (`default_tag_registry.tres`) stores all project tags.
2. **Custom Editor Inspector:** A plugin (`gameplay_tag_inspector_plugin.gd`) replaces default string fields with a custom, searchable Tree UI.
3. **Auto-Generation:** When tags are modified, `gameplay_tag_generator.gd` writes a static class file (`gameplay_tags.gd`).

Instead of the engine wasting memory parsing strings at runtime, your code accesses tags as pre-compiled constants (e.g., `GameplayTags.Status_Stunned`). 

> **v1.0 Evolution Note: Networking & Multiplayer**
> *In v0.5.0, the ASC and Tag system are designed for single-player or simple authority-based multiplayer. As we approach v1.0 and beyond, this tag infrastructure will integrate with Godot's `MultiplayerSynchronizer` to support Prediction and Rollback (allowing local clients to predict tag application before the server confirms it).*

# Chapter 2: Attributes & Modifiers

GodotGAS handles attribute math safely, ensuring values are clamped and cleanly reverted when buffs expire.

## Attribute Sets
Attributes are grouped into modular Resource scripts inheriting from `AttributeSet` (e.g., `HealthAttributeSet`). Each attribute is an `AttributeData` object, which separates:
* **Base Value:** The permanent, unbuffed stat.
* **Current Value:** The temporary, buffed/debuffed stat used for gameplay math.

## Safe Math (The Firewall)
Before an attribute is changed, the ASC routes the new value through `pre_attribute_change()` inside the `AttributeSet`.

```gdscript
# Example: Inside health_attribute_set.gd
func pre_attribute_change(attribute_name: String, proposed_value: float) -> float:
	if attribute_name == "health":
		return clampf(proposed_value, 0.0, max_health.current_value)
	return proposed_value```

### Chapter 3: Abilities & State
# Chapter 3: Abilities & State Management

Abilities (`GameplayAbility`) define the specific actions an entity can take. They are intrinsically linked to the ASC to ensure costs and cooldowns are respected.

## The Activation Pipeline
When an entity attempts to use an ability, it calls `try_activate()`. The ASC acts as a gatekeeper:

1. **Tag Blocking:** Checks if the ASC has any tags listed in the ability's `activation_blocked_tags` (e.g., blocking a spell if the caster has `Status.Silenced`).
2. **Cost Validation:** Checks if the entity has enough resources (Mana, Stamina) to pay the `cost_effect`.
3. **Execution:** If passed, the cost is deducted, and the ability's `activate()` logic runs.
4. **Cooldown:** Upon success, the `cooldown_effect` is applied to the ASC.

## State Cleansing & Memory Management
When a Duration or Infinite effect is applied, the ASC creates an `ActiveGameplayEffect` object in memory. This object stores the exact mathematical delta applied to the character.

If a character is "Cleansed" (e.g., drinking an antidote), the ASC finds the Active Effect, removes its granted tags, and perfectly subtracts the stored mathematical delta, restoring the character's original state.

> **v1.0 Evolution Note: TargetData & Gameplay Events**
> *In v0.5.0, abilities are activated manually via `try_activate()`. The upcoming architecture will introduce **TargetData** (spatial payloads containing enemy references/hitboxes) and **GameplayEvents**. This will allow the ASC to broadcast global events (e.g., `Event.Damage.Taken`), enabling passive abilities to trigger automatically without manual activation.*

# Chapter 4: Enterprise Visuals & Object Pooling

In an action-heavy game, if 50 entities cast a spell simultaneously, instantiating 50 complex particle systems and audio streams can cause severe frame drops (hitching) due to the Garbage Collector. 

GodotGAS solves this by strictly decoupling the mathematical engine (ASC) from the visual presentation (Cues), utilizing an Enterprise-grade Object Pooling system.

## The GameplayCueManager (Autoload)
Visuals and Audio are handled by a global Autoload called the `GameplayCueManager`. 

The `AbilitySystemComponent` does not know or care about visual effects. When an ability executes, the ASC simply broadcasts a signal to the void:
`GameplayCueManager.execute_cue("Cue.Fireball", location)`

## Object Pooling Pipeline
When the `GameplayCueManager` receives a request, it does not use Godot's `load()` or `instantiate()` functions at runtime.

1. **Pre-Loading:** The Manager holds a dictionary of pre-instantiated `GameplayCueNotify` scenes in memory.
2. **Waking Up:** It grabs an invisible, inactive node from the pool, moves it to the target location, and activates its visual/audio logic.
3. **Recycling:** When the effect finishes (e.g., an explosion dissipates), the `GameplayCueNotify` does not call `queue_free()`. Instead, the Manager catches it, hides it, and returns it to the pool for the next cast.

This guarantees that no matter how chaotic the screen gets, visual effects will not cause frame rate stuttering.

> **v1.0 Evolution Note: Complex Cue Parameters**
> *In the v0.5.0 baseline, cues are triggered at basic locations. In future updates, as the TargetData payload is introduced, Cues will be able to receive rich context (e.g., attaching a burning particle effect directly to a specific bone on a 3D model, or scaling the explosion size based on the ability's level).*

# Chapter 5: The Road to v1.0 (Future Roadmap)

GodotGAS v0.5.0 establishes the core mathematical loop and data architecture. The following modules are planned to bring the framework to commercial-grade (v1.0) status.

## Phase 1: Target Data (Spatial Payloads)
Currently, abilities operate in a vacuum. **TargetData** will introduce a spatial payload system. When an ability is cast, it will generate a payload containing what it hit (Entity references, Hitboxes, Raycast coordinates) and pass that data securely into the Gameplay Effect.

## Phase 2: Gameplay Events (Passive Abilities)
Right now, abilities must be triggered manually via `try_activate()`. The Gameplay Event pipeline will allow the ASC to broadcast global event packets (e.g., `Event.Damage.Taken`). Passive abilities will listen for these specific tags and trigger automatically, enabling mechanics like "Thorns" or "Retaliation". *(Note: This requires Target Data to function).*

## Phase 3: Execution Calculations (ExecCalcs)
While `GameplayEffectModifiers` handle basic math (Add, Multiply), `ExecCalcs` will allow developers to write custom scripts that capture variables from BOTH the Attacker and the Defender. 
*Example: `Damage = (Attacker.Strength * 1.5) - Defender.Armor`*

## Phase 4: Input Binding
A Quality-of-Life wrapper that allows developers to map Godot's native Input Map directly to Gameplay Abilities, streamlining the controller/keyboard setup.

##Looking Beyond: Multiplayer (v2.0)
Multiplayer Prediction and Rollback algorithms require deeply modifying how state is stored frame-by-frame. To ensure the single-player/co-op core remains lightning-fast and bug-free, highly competitive esports networking (Client Prediction) is intentionally deferred to GodotGAS v2.0.
