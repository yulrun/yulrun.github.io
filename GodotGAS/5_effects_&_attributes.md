---
layout: default
title: 5. Gameplay Effects & Attributes
parent: GodotGAS - Addon
nav_order: 8
---

# 5. Gameplay Effects & Attributes

In GodotGAS, **Attributes** are the raw numbers (e.g., Health, Mana, Stamina), and **Gameplay Effects** are the rules and mathematical operations that modify those numbers. By keeping these two systems separated, designers can build vast arrays of abilities without ever writing custom damage scripts.

---

## Attribute Sets

As discussed in the Dashboard section, your stats are generated as `AttributeSet` resources using the `AttributeData` wrapper. 

When you initialize an entity, you assign its `AttributeSet` directly to its Ability System Component (ASC). Once registered, the ASC native `_apply_modifiers()` function handles all incoming changes automatically, routing them through your generated `pre_attribute_change()` clamps.

---

## Gameplay Effects

A `GameplayEffect` is a highly configurable data Resource. It defines *how* an interaction happens. When designing an Effect, you configure three primary components:

### 1. Duration Policy
* **Instant:** Permanently alters a base stat (e.g., taking 50 damage, drinking a health potion).
* **Has Duration:** Applies a temporary change that reverts when the timer expires (e.g., a 5-second movement speed buff).
* **Infinite:** Applies a change that lasts until explicitly removed by a tag or ability (e.g., an equipable armor aura).

### 2. Modifiers
Modifiers are simple mathematical operations applied to specific attributes. For example, a basic fireball effect might have a Modifier defined as:
* **Attribute:** `health`
* **Operation:** `ADD`
* **Magnitude:** `-50.0`

### 3. Execution Calculations (Advanced Math)
Sometimes, simple addition or multiplication isn't enough. If you need a formula like `Damage = (BaseDamage * 1.5) - TargetArmor`, you use an Execution Calculation (`GameplayExecutionCalculation`). 

This is a custom Resource script you attach to the Effect. It intercepts the `GameplayEffectSpec` right before application, grabs the Instigator's stats and the Target's stats, runs your complex math, and returns a flat dictionary of modifiers for the ASC to apply.

---

## The Effect Stacking Pattern (Critical)

One of the most complex challenges in any RPG or MOBA is dealing with effect stacking. If a player casts "Fire Shield" three times in a row, does the duration stack? Does the armor value stack? Or does it just refresh?

**GodotGAS uses a beautifully simple, data-driven approach to stacking policies.** You do not need to write complex timer-reset code or write a custom stacking manager. Instead, you utilize the Effect's built-in tag arrays.

### How to Prevent Stacking:
To prevent an effect from stacking, you simply use the **`application_ignore_tags`** array combined with the **`granted_tags`** array.

1. In your `GameplayEffect` resource, you set the **Granted Tags** to include the tag you want the target to receive (e.g., `Status.Buff.Shield`). As long as the effect is active, the target has this tag.
2. In that exact same `GameplayEffect` resource, you add `Status.Buff.Shield` to the **Application Ignore Tags** array.

**The Result:** When the ASC receives the `GameplayEffectSpec` for the *first* time, it applies the effect successfully and grants the `Status.Buff.Shield` tag. 
If the player tries to cast it a *second* time, the ASC checks the Application Ignore rules, sees that the target already possesses the `Status.Buff.Shield` tag, and instantly safely rejects the new application. 

This pattern allows designers to create completely custom stacking, blocking, and refreshing policies entirely in the inspector, without writing a single line of logic.
