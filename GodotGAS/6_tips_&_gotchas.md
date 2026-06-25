---
layout: default
title: Important Tips & Gotchas
parent: GodotGAS - Addon
nav_order: 6
---

# Important Tips & Gotchas

When transitioning to a Gameplay Ability System architecture, certain paradigms behave differently than standard Godot game development. Here are the most critical "Gotchas" to keep in mind to prevent memory leaks, broken logic, and engine crashes.

---

## 1. Death and Node Cleanup (The Memory Leak Trap)

When an entity in your game dies (e.g., Health drops to 0), you cannot simply call `queue_free()` on the CharacterBody. 

If that entity has an active 5-minute buff, or is in the middle of channeling a spell, destroying the parent node without telling the ASC will cause memory leaks and leave orphaned visual Cues stuck on the screen forever.

**The Solution:**
You must manually call `cleanup()` on the ASC before destroying the node. A great way to handle this is by listening to the `attribute_changed` signal.

```gdscript
func _on_attribute_changed(attribute_name: String, old_val: float, new_val: float, spec) -> void:
    if attribute_name == "health" and new_val <= 0.0:
        die()

func die() -> void:
    # 1. Play death animation
    # 2. Tell the ASC to forcefully abort all abilities and drop all buffs
    asc.cleanup()
    
    # 3. Now it is safe to remove the entity
    queue_free()
```

---

## 2. The `await` Trap in Abilities

Godot 4's `await` keyword is incredibly useful for timing animations (e.g., `await get_tree().create_timer(1.0).timeout`). However, coroutines in Godot **do not magically stop** if an ability is aborted.

Imagine this scenario:
1. Your player casts a 2-second \"Meteor\" spell.
2. At 1.0 second, an enemy stuns the player. The ASC correctly calls `abort_ability()`.
3. The ability is marked as inactive... but the `await` timer keeps running in the background!
4. At 2.0 seconds, the timer finishes, and the rest of your code spawns the Meteor anyway, even though the player is stunned.

**The Solution:**
Whenever you use `await` inside your `_activate_ability()` function, you **must** check `is_active` immediately after it resolves.

```gdscript
func _activate_ability() -> bool:
    commit_ability()
    
    # Wait for the casting animation to finish
    await animation_player.animation_finished
    
    # GOTCHA CHECK: Were we stunned/interrupted while we were waiting?
    if not is_active:
        return false 
        
    # We survived the cast time, spawn the projectile!
    spawn_meteor()
    return true
```

---

## 3. Transient Nodes as Context Causers

When creating a `GameplayEffectContext` payload, you must assign an `instigator` (the true owner, like the Player) and a `causer` (the physical thing causing the effect, like a Sword or Projectile).

**Do not pass transient abilities as the causer.** If your Fireball ability script passes `self` into the context, and then the Fireball ability is deleted... the Context is now holding a dangling, dead reference. If an Execution Calculation tries to read data from the causer later, Godot will crash with a Null Instance error.

**The Solution:**
Always map the `causer` back to a persistent node, like your main Character/Avatar, or a permanent Weapon node. The `apply_effect_to_targets()` helper method does this automatically, but keep it in mind if constructing Contexts manually!

---

## 4. Clamping Attributes

Modifiers blindly apply math. If you apply a "+50 Health" effect to a player who is only missing 5 Health, the effect doesn't inherently know about "Max Health".

**The Solution:**
All value clamping must be done inside your generated `AttributeSet` scripts using the `pre_attribute_change()` virtual function.

```gdscript
# Inside your generated PlayerStats.gd
func pre_attribute_change(attribute_name: String, proposed_value: float) -> float:
    if attribute_name == "health":
        # Ensure health never drops below 0 or goes above max_health
        return clamp(proposed_value, 0.0, max_health.current_value)
        
    return proposed_value
```
If you forget to clamp here, your player will quickly end up with 150/100 Health!
