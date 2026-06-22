---
layout: default
title: 3. Core Architecture (ASC & Payloads)
parent: GodotGAS - Addon
nav_order: 6
---

# 3. Core Architecture: The ASC & Payloads

At the heart of GodotGAS is a strict, decoupled data pipeline. In traditional game development, a fireball ability might directly find a target and reduce its health variable. In GodotGAS, abilities **never** modify stats directly. Instead, they generate data payloads that are routed through the Ability System Component (ASC).

This decoupling allows you to easily intercept, modify, multiply, or cancel interactions (e.g., dodging, blocking, or damage reflection) without writing complex inter-node dependencies.

---

## The Payload Pipeline

When an interaction occurs, data flows through three distinct wrappers before the ASC ever processes a stat change. The flow is always: **TargetData -> Context -> Spec**.

### 1. GameplayAbilityTargetData
This is the foundation of the payload. It strictly contains the data regarding *who* or *what* was hit. If your ability overlaps with three enemies in an area of effect, the `GameplayAbilityTargetData` resource captures and holds the array of those target `Node` references.

### 2. GameplayEffectContext
The Context wraps the `TargetData` and adds the critical "Who" and "How" of the interaction. This ensures the target always knows exactly where the damage or buff came from.
* **Instigator:** The overarching entity that activated the ability (e.g., the Player Character).
* **Causer:** The physical entity that caused the effect (e.g., a thrown Fireball projectile node). If there is no separate projectile or physical causer, it defaults to the Instigator.

*Example from the framework's standard initialization:*
```gdscript
# Context initialized with the Player (Instigator) and the Fireball (Causer)
var context = GameplayEffectContext.new(player_node, fireball_node)
context.target_data = hit_targets
```

### 3. GameplayEffectSpec
This is the final, fully-baked package. The Spec marries the `GameplayEffectContext` with a specific `GameplayEffect` resource (which contains the actual definitions for stat changes, like "-50 Health"). 

The Spec locks in the level of the ability at the exact time of casting and evaluates dynamic data (like calculating total duration) so the target's ASC knows exactly what to execute.

---

## The Ability System Component (ASC)

The ASC is the central brain of your entity. While it holds your active tags and Attribute Sets, it contains **no hardcoded game logic**. It acts purely as a state manager and payload router.

Once a `GameplayEffectSpec` is generated, it is passed to the target's ASC. The ASC then processes it systematically:
1. **Validates Tags:** It checks if the target is immune to the effect based on its current active Gameplay Tags.
2. **Applies Modifiers:** It unpacks the Spec and routes the mathematical changes to the underlying Attributes (triggering the `pre_attribute_change` clamps we discussed earlier).
3. **Fires Cues:** It sends the payload's visual/audio tags to the `GameplayCueManager` to spawn hit effects.
4. **Updates State:** It applies any new granted tags or status durations to the entity.

By standardizing this pipeline, any entity equipped with an ASC can interact with any other entity seamlessly—whether they are a player, a boss monster, or a destructible barrel.

---

*In the next section, we will look closer at the global Managers handling the connective tissue: Tags and Cues.*
