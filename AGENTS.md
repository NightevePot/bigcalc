# BigCalcs Project Notes

This file is a durable knowledge note for Codex when working on this project.

## Conceptual Architecture

The calculator has several orthogonal dimensions. Names can change later; the
important part is to preserve these concepts in the data model.

### Source Areas

Source areas answer: where does a stat or rule come from?

- Character base area: the character's preset panel and class template. This
  includes base main stat, attack, element strength, attack enhancement, class
  type, and the default skill template.
- Baseline condition area: the version/build baseline, especially equipment and
  set data. This area is strongly tied to the calculator's data version and may
  change often when equipment is updated.
- Assumption preset area: low-attention defaults that are usually assumed in a
  discussion, such as support-buff size, wedding-room toggle, and global skill
  efficiency assumptions. Hide these in normal mode unless advanced/admin mode
  is enabled.
- Comparison project area: the variable currently being studied, such as adding
  one equipment piece, changing one line, or enabling one conditional effect.

Normal user mode should mostly select presets and edit a few exposed inputs.
Advanced/admin mode should allow editing the underlying presets and detailed
rules.

### Damage Zones

Damage zones answer: how does a stat or rule combine mathematically?

- Attack zone: additive sources, then multiply by the applicable three-attack
  value. Support-provided attack often lives here as a condition because it is
  very large.
- Main-stat zone: additive sources, then use
  `1 + total_main_stat * 0.004`.
- Element zone: additive final element-strength input, then use
  `1.05 + effective_element_strength * 0.0045`.
- Attack-enhancement zone: additive attack-enhancement sources, plus attack
  enhancement amplification, then use
  `1 + total_attack_enhance * (1 + attack_enhance_amp)`.
- Skill-damage zone: itemized multiplicative lines,
  `(1 + line_a) * (1 + line_b) * ...`. Do not flatten this into one summed
  value because each line may have its own target, condition, and source.
- Special-coefficient zone: multiplicative coefficients that should not be
  classified as skill damage. Examples include the equipment-effect conversion
  coefficient `10`, class passive coefficient `1.48`, or other branch/scope
  specific factors.

Three-attack, main stat, element strength, and attack enhancement may be entered
as multiple rows, but they aggregate additively inside their own zones. Skill
damage and special coefficients aggregate multiplicatively as line items.

### Modifiers

Most inputs should be represented internally as modifiers:

```text
modifier = {
  source_area,
  damage_zone,
  value,
  combine_rule,
  target_selector,
  branch_scope,
  condition,
  visibility_mode
}
```

- `source_area`: character base, baseline condition, assumption preset, or
  comparison project.
- `damage_zone`: attack, main stat, element, attack enhancement, skill damage,
  or special coefficient.
- `target_selector`: all skills, one skill group, one skill component, a level
  range, a tag, all equipment effects, or one equipment effect.
- `branch_scope`: skill branch, equipment-effect branch, or both.
- `condition`: optional trigger rule.
- `visibility_mode`: normal, advanced, or admin.

The final calculation is built by collecting all modifiers from the active
source areas, routing them into damage zones, applying target selectors, and
then summing the resulting skill/effect components.

## DNF 115 Damage Formula

Use this as the base model for damage calculations unless the user gives a newer rule.

Treat total damage as separate skill and equipment-effect branches. Do not
implement it as `(skill_data + effect_data * 10) * shared_multipliers`, because
equipment effects do not receive every attribute that normal skills receive.

```text
total_damage =
  sum(per_skill_damage_i for i in skills)
  + sum(per_effect_damage_j for j in equipment_effects)

per_skill_damage_i =
  base_skill_value_i
  * common_skill_coefficient
  * skill_specific_coefficient_i

per_effect_damage_j =
  base_effect_value_j
  * common_effect_coefficient
  * effect_specific_coefficient_j

common_skill_coefficient =
  skill_attack_multiplier
  * skill_main_stat_multiplier
  * skill_element_multiplier
  * attack_enhance_multiplier
  * global_skill_damage_product

common_effect_coefficient =
  effect_attack_multiplier
  * effect_main_stat_multiplier
  * effect_element_multiplier
  * attack_enhance_multiplier
  * global_skill_damage_product

skill_specific_coefficient_i =
  applicable_fake_skill_attack_product_i
  * applicable_level_skill_attack_product_i
  * other_skill_specific_bonus_product_i
```

The old compact form is still useful for mental arithmetic when all attributes
apply equally, but the calculator implementation should keep these branches
separate from the start.

Where:

- `base_skill_value_i`: base value for a skill entry. It may be actual damage, normalized share, or skill equivalent percent after skill level, class passives/buffs, talisman, runes, and cast count are already accounted for.
- `base_effect_value_j`: base value for an equipment-effect entry. Approximate as `effect_data * 10`, with trigger count and cooldown accounted for when calculating DPS.
- `total_attack`: the applicable attack stat: physical attack, magical attack, or independent attack.
- `total_main_stat`: the applicable strength or intelligence.
- `effective_element_strength`: the final element-strength value to use in calculation.
- `total_attack_enhance`: attack enhancement expressed as a decimal multiplier value. Example: `220000% = 2200`.
- `attack_enhance_amp`: attack enhancement amplification as decimal. Example: `50% = 0.5`.
- `skill_damage_bonus_i`: each independent skill damage line as decimal. Example: `500% = 5`.
- `applicable_fake_skill_attack_product_i`: fake skill attack lines that directly modify the matching skill panel/equivalent percent. This applies only to matching normal skill entries.
- `applicable_level_skill_attack_product_i`: level-range skill attack lines that match the current skill. This applies only to matching normal skill entries.
- `other_skill_specific_bonus_product_i`: bonuses that only affect a specific skill or skill group, such as a line that gives only skill A `+50%` damage.
- `skill_attack_multiplier`, `skill_main_stat_multiplier`, and `skill_element_multiplier`: the normal skill branch versions of the attack, main-stat, and element areas.
- `effect_attack_multiplier`, `effect_main_stat_multiplier`, and `effect_element_multiplier`: the equipment-effect branch versions. These may intentionally exclude attributes that effects do not receive.

## Skill Entry Model

The calculator should support up to 50 skill entries. Each entry represents one
damage component such as skill A, B, C, D, and so on.

This 50-entry capacity is a backend/model reserve, not a requirement that users
usually fill 50 detailed rows. Most practical discussions should be expressed as
skill groups on top of the detailed model. For example, when studying a
`level-80 skill specialization +40%`, the visible model can be:

- `level-80 skill`: usually one specific skill entry or one focused skill group.
- `other skills`: the sum of all remaining skill entries, including unused/empty
  reserved entries.

In other words, the user-facing calculator will often look like a `general
skills + specialized skills` structure, while the calculation engine still keeps
the ability to expand those groups into up to 50 entries when a complex
description requires it.

Skills may have a display-level parent and calculation-level components. For
example, display skill `B` may contain components `B1`, `B2`, and `B3`. A
modifier that targets `B` should cascade to `B1`, `B2`, and `B3` by default, and
the output should still display the merged result as `B = B1 + B2 + B3`. Only
add exclusions or overrides when a component explicitly has different behavior.

Each skill entry should have:

- Name.
- Base value.
- Optional cast count or share weight, if not already included in base value.
- Level or level range, for matching level-range skill attack.
- Tags, for matching skill-specific bonuses.
- Skill-specific multiplier list.

Final skill damage is the sum of all skill entries after their own coefficients
are applied:

```text
total_skill_damage = sum(base_skill_value_i * coefficient_i)
```

Common coefficients can be factored out only when they apply to every skill
entry equally. Skill-specific bonuses must stay inside the sum.

Actual damage values and normalized share values are equivalent for relative
gain calculations if they use the same baseline:

```text
sum(actual_damage_i * relative_multiplier_i)
```

is equivalent to:

```text
baseline_total_damage * sum(share_i * relative_multiplier_i)
```

where `share_i = actual_damage_i / baseline_total_damage`. The normalized-share
mode only changes output scale; it should produce the same relative gain. When a
new damage source is added, such as an equipment effect not represented in the
original shares, it must be entered in the same normalized unit or as an actual
value with a known baseline.

## Attribute Blocks

Think of the formula as blocks that can be refined independently:

- Data block: skill entries/components and equipment-effect entries.
- Attack block: applicable three-attack from all active sources.
- Main-stat block: strength or intelligence from all active sources.
- Element block: final element strength from all active sources.
- Attack-enhancement block: attack enhancement and attack-enhancement amplification.
- Skill-damage block: global skill damage, level-range skill attack, fake skill attack, and skill-specific equipment lines.
- Special-coefficient block: non-skill-damage multipliers such as class passives, effect conversion, or other special factors.

As the calculator grows, refine each block instead of collapsing everything into
one shared multiplier.

## Multiplicative Areas

- Three-attack, main stat, element strength, attack enhancement, and skill damage are separate multiplicative areas.
- Main stat is additive inside its own area:
  `main_stat_multiplier = 1 + main_stat * 0.004`.
- Element strength is additive inside its own area:
  `element_multiplier = 1.05 + effective_element_strength * 0.0045`.
- Attack enhancement and attack enhancement amplification use:
  `attack_enhance_multiplier = 1 + attack_enhance * (1 + attack_enhance_amp)`.
- Skill damage lines multiply with each other:
  `(1 + bonus_a) * (1 + bonus_b) * ...`.
- Special coefficients also multiply as line items, but they are kept outside
  the skill-damage zone for semantic and targeting reasons.

## Skill Damage Types

Track these separately:

- Global skill damage: applies to overall damage and multiplies as a skill-damage line.
- Level-range skill attack: applies only to matching skill levels, multiplies with other matching skill-damage lines, and does not apply to equipment special effects.
- Fake skill attack: directly changes the matching skill panel/equivalent percent, multiplies with other matching fake-skill lines, and does not apply to equipment special effects.

Conditional items such as attack-speed shoes or tiered set effects usually determine the numeric value of one skill-damage line. After the condition is resolved, treat the resulting line as normal skill damage.

## Equipment Special Effects

- Equipment special effects can usually receive most global damage multipliers, but must be calculated through their own branch.
- Approximate their skill-equivalent contribution as `effect_data * 10`.
- Approximate special-effect DPS as:
  `effect_data * 10 / effect_cooldown`.
- Special effects are added to total damage after their own branch is calculated.
- Special effects do not receive level-range skill attack or fake skill attack.
- Special effects currently do not receive wedding-room stats.
- Multiple special effects dilute each other because they add in the same data area.

## Wedding Room Attribute Scope

Wedding-room stats may exist as an independent option:

- `+10` main stat.
- `+20` three-attack.
- `+8` element strength.

These stats apply to the normal skill branch, but currently do not apply to
equipment-effect damage. Keep a dedicated interface for them instead of merging
them into unconditional total attack, total main stat, or total element strength.

## Common Marginal Gain Calculations

Use ratio of new multiplier to old multiplier.

```text
gain = new_multiplier / old_multiplier - 1
```

Examples verified from the discussion:

- 7000 main stat + 700 main stat:
  `(1 + 7700 * 0.004) / (1 + 7000 * 0.004) - 1 = 9.66%`.
- 500 element strength + 15 element strength:
  `(1.05 + 515 * 0.0045) / (1.05 + 500 * 0.0045) - 1 = 2.05%`.
- 120% attack enhancement + 15% attack enhancement:
  `(1 + 1.35) / (1 + 1.20) - 1 = 6.82%`.
- 4000 attack + 40000 attack:
  `(4000 + 40000) / 4000 = 11x`, or `1000%` more damage.
- 500% skill damage replaced by 600% skill damage:
  `(1 + 6.00) / (1 + 5.00) - 1 = 16.67%`.
- 220000% attack enhancement + 3000% attack enhancement:
  `(1 + 2230) / (1 + 2200) - 1 = 1.36%`.
- Same case with 50% attack enhancement amplification:
  `(1 + 2230 * 1.5) / (1 + 2200 * 1.5) - 1 = 1.36%`, nearly unchanged because amplification scales both old and new attack enhancement.

## Calculator Structure Recommendation

Do not treat the early section names as fixed UI labels. The important
architecture is:

```text
active_configuration =
  character_base_area
  + baseline_condition_area
  + assumption_preset_area
  + comparison_project_area

calculation =
  route all modifiers into six damage zones
  -> apply target selectors to skill/effect entries
  -> calculate component results
  -> aggregate display skills/groups/effects
```

Common user-facing mode:

- Select a character/class template.
- Select or keep a baseline equipment/build preset.
- Keep assumption presets mostly hidden.
- Edit the comparison project through limited fields or dropdown presets.
- Show aggregated outputs such as specialized skill, other skills, equipment
  effects, total damage, and relative gain.

Advanced/admin mode:

- Edit character panel and class skill templates.
- Expand skill groups into detailed skill entries/components.
- Edit baseline equipment and version-bound preset data.
- Edit low-attention assumptions such as support-buff size, wedding-room toggle,
  and skill efficiency.
- Edit raw modifiers, their damage zones, target selectors, and conditions.

The engine should support detailed calculation while the normal output remains
aggregated. For example, a complex skill may calculate as `B1 + B2 + B3`, but
the result can still be displayed as one skill `B`.

## Current Scope Decisions

- Do not model multi-element selection in the calculator UI or calculation layer.
- Do not model enemy element resistance for now.
- Treat element strength as an already-resolved numeric input. The user/calculation scenario is responsible for deciding which element-strength value should be entered.
- The calculator's purpose is to calculate numbers, dilution, and gear/build interactions under complex conditions, not to reproduce every in-game targeting or element-resolution rule.

For 115-version same-grade equipment comparisons, many three-attack, main-stat, element-strength, and attack-enhancement differences may cancel out. The calculator should still support the full formula, but quick gear comparisons should emphasize skill damage, applicable skill ranges, fake skill attack, and special-effect contribution.
