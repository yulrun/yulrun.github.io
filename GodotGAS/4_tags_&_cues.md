---
layout: default
title: 4. Tags & Cues
parent: GodotGAS - Addon
nav_order: 7
---

# 4. Tags & Cues

In GodotGAS, game logic and presentation are entirely separated. The logic is driven by **Tags**, and the visual/audio feedback is handled by **Cues**. Because these elements are used constantly across every entity in your game, they are managed by highly optimized global Autoloads.

---

## Gameplay Tags

Gameplay Tags are the connective tissue of the entire framework. Instead of using boolean flags like `is_stunned = true` or `is_burning = false`, GodotGAS uses a hierarchical string-based tagging system. 

### The `GameplayTagManager` Autoload
Once you create tags in the Editor Dashboard, they are loaded into the runtime `GameplayTagManager`. 

* **StringName Optimization:** Under the hood, the manager converts and caches all tags as Godot `StringName` variables. This means tag comparisons (e.g., checking if an entity has `Status.Stunned`) are executed as lightning-fast C++ pointer comparisons rather than slow string evaluations.
* **Hierarchical Logic:** Tags use a strict dot-notation format (e.g., `Ability.Skill.Fireball`, `Status.Effect.Burning`). This hierarchy allows you to easily categorize and search for tags.

Whenever an Ability or Effect executes, the ASC cross-references these tags to determine if the action is allowed (e.g., "Cancel this ability if the caster has the `Status.Stunned` tag").

---

## Gameplay Cues

A "Cue" is any audio, visual, or UI event that needs to play in response to a gameplay event (e.g., a hit spark, a damage number, or a camera shake). 

If you instantiated and destroyed a particle system every single time a machine gun fired, your game would suffer from severe garbage collection lag. To solve this, GodotGAS uses a custom **Variant-based Object Pool** managed by the `GameplayCueManager`.

### How to Create a Cue

1. **Design your Scene:** Create a standard Godot scene (e.g., a `Node3D` with a `GPUParticles3D` and an `AudioStreamPlayer3D`).
2. **Attach the Script:** Attach a script to the root node that extends `GameplayCueNotify`.
3. **Map the Cue:** Open the GodotGAS Dashboard, go to the Cue Manager, and link your new scene to a Gameplay Tag (e.g., `Cue.Combat.Hit.Fire`).

### The Golden Rule: Never Call `queue_free()`

Because Cues are object-pooled, they are created once and kept in memory. When a Cue is done playing, you **must not delete it**. Deleting it destroys the pool! Instead, you tell the Manager to put it back to sleep by emitting the `cue_finished` signal.

Here is an example of a perfectly structured Cue script:

```gdscript
extends GameplayCueNotify

# Note: Because this object is pooled, _ready() will only ever fire ONCE 
# in its entire lifetime. Do not use _ready() to reset visual state!

func execute_cue() -> void:
    # 1. Play your effects
    $GPUParticles3D.emitting = true
    $AudioStreamPlayer3D.play()
    
    # 2. Wait for them to finish
    await $AudioStreamPlayer3D.finished
    
    # 3. CRITICAL: Return to the Object Pool! Do NOT call queue_free()!
    cue_finished.emit()
```

### Variant-Based Safety
Behind the scenes, the `GameplayCueManager` stores pooled objects as untyped `Variant` types. This cleverly bypasses Godot's static analyzer, allowing you to freely mix 2D Nodes, 3D Nodes, and standard Base Nodes in the exact same Object Pool without throwing compiler errors.

---

*In the next section, we will dive into Gameplay Effects & Attributes, and look at the framework's elegant solution for Effect Stacking.*
