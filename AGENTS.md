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

### Equipment Base Chassis（装备基础底子）

All equipment of the same grade shares a common stat baseline. When comparing
equipment of the same grade, the chassis values cancel out. When mixing grades
or comparing weapon vs. no weapon, the chassis difference matters.

```text
base_chassis = {
  grade: "115_epic",
  main_stat: 7000,              // 力智 → main_stat zone (additive)
  attack: 4000,                 // 三攻 → attack zone (additive)
  element_strength: 500,        // 属强 → element zone (additive)
  attack_enhance: 2200,         // 底子攻强 → attack_enhance zone (additive)
  skill_attack_base: 1.00,      // 底子技攻系数（与额外技攻相乘）
}
```

The chassis is applied exactly once regardless of how many equipment units are
configured. It represents the character's equipped baseline before any set
bonuses or unique effects.

### Equipment Unit（装备单元）

A composable equipment piece that inherits a chassis and provides additional
modifiers, branches, and variant conditions.

```text
equipment_unit = {
  name: "龙战八荒",
  type: "armor_set" | "weapon" | "accessory",
  inherits: "115_epic_chassis",

  // 套装/武器额外技攻（mutually multiplicative with chassis skill_attack_base）
  extra_skill_attack: 0.4396,   // Example: +43.96%

  // 分支变体——同一分支下的 variant 互斥
  branches: [
    {
      name: "燃烧世界之火",
      variants: {
        orb:    { modifiers: [{ zone: "skill_damage", value: 0.4396 }, ...] },
        normal: { modifiers: [{ zone: "skill_damage", value: 0.4165 }, ...] }
      }
    },
    // ...
  ],

  // 装备特效（走独立 effect branch）
  effects: [
    { name: "燃烧世界之火·烈焰", category: "burst", single_percent: 182.5, cooldown: 10, efficiency: 1.0 },
    { name: "燃烧世界之火·余烬", category: "sustained", single_percent: 139, cooldown: 3, efficiency: 0.95 },
    { name: "燃烧世界之火·火花", category: "per_hit", single_percent: 129.5, cooldown: 0.5, manual_count: 8 }
  ]
}
```

- `type` determines which slot the unit occupies.
- `branches[].variants` are mutually exclusive — only one variant per branch
  can be active at a time.
- Equipment effects route through the equipment-effect branch, not the skill
  branch.

### Active Configuration（当前装备组合）

The user's selected gear combination. This is the runtime state that determines
which modifiers are active.

```text
active_config = {
  chassis: "115_epic",
  armor:  { unit: "龙战八荒", branch_variants: { "燃烧世界之火": "orb", "龙帝降临": "normal" } },
  weapon: { unit: null },
  // Future: accessory, magic_stone, earring...
}
```

Calculation: collect chassis modifiers + all active unit modifiers + active
branch variant modifiers + active effect entries. Route into damage zones and
compute both branches.

### Exclusion & Composition Rules（互斥与组合规则）

1. **Slot mutual exclusion**: Only one unit per `type` slot (e.g., one armor
   set, one weapon). Selecting a new unit in the same slot replaces the
   previous one.
2. **Branch variant mutual exclusion**: Within one branch, variants are mutually
   exclusive. E.g., "增幅12" and "增幅11" cannot both be active.
3. **Chassis applied once**: The base chassis contributes its stats exactly once
   regardless of how many equipment units share that grade.
4. **Weapon vs. no weapon**: When no weapon is selected, only the armor
   chassis/set contributes. When a weapon is added, its chassis and extra stats
   are additive where the damage zone rule requires.

### Solution Model（方案模型）

A solution is the user-facing unit of comparison. It equals one equipment unit ×
one set of active branch variants, acting as a container for all modifiers that
the calculator evaluates.

```text
solution = {
  display_name: "龙战八荒 · 燃烧世界之火 · 如意珠",
  unit: "龙战八荒",
  variant_selections: { orbMode: "orb", golden: "12" },
  modifiers: [...]     // All modifiers active under this variant combo
}
```

- The ranking tab compares 36 solutions (12 sets × ~3 branches each).
- A solution's data only spans `skill_damage`, `special_coefficient`, and
  `equipment_effect` zones. Attack, main-stat, element-strength, and
  attack-enhancement zones come from the chassis + character + condition
  preset areas.

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
- `base_effect_value_j`: base value for an equipment-effect entry, calculated as `single_percent × trigger_count × efficiency × special_coefficient`, with trigger count and cooldown accounted for when calculating DPS.
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

## Calculation Mode（计算模式）

The calculator supports two mutually exclusive modes, controlled by `ifExpert`:

```text
ifExpert = 1  →  专家模式（Expert Mode）
ifExpert = 0  →  粗放模式（Coarse Mode）
```

### Expert Mode（专家模式，ifExpert = 1）

Skill data is built from individual skill entries with precise parameters. The
total skill pool is the sum of independently calculated skill components:

```text
ALLSKILLS = Σ( single_percent_i × trigger_count_i × efficiency_i )
skill_i = single_percent_i × trigger_count_i × efficiency_i
```

- Skill data entry panel is **enabled**.
- Global percentage input is **disabled** (ALLSKILLS is derived, not entered).
- Suitable for class-specific deep analysis and precise damage research.

### Coarse Mode（粗放模式，ifExpert = 0）

Skill data is driven by a single preset total (`ALLSKILLS`) and a set of share
weights. Individual skill values are derived top-down:

```text
ALLSKILLS = preset_input     // 用户或系统预设的全局技能百分比总和
skill_i = ALLSKILLS × share_i
```

- Skill data entry panel is **disabled** (data is protected, read-only).
- Global percentage input is **enabled**.
- Purpose: normalize skill profiles across classes, providing a rough strength
  analysis focused on equipment effects rather than class-specific skill data.

## Skill Entry Model

Each skill entry uses the same underlying parameter model as equipment effects.
Skills and effects share the same base parameters because both are "triggerable
actions with animation and damage distribution over time."

### Skill Parameters（专家模式）

```text
skill_entry = {
  name: "崩山击",
  single_percent: 4500,         // 单次等效百分比（含等级/被动/VP 强化/施放数）
  cooldown: 8,                  // CD（秒）
  efficiency: 1.0,              // 效率（默认 100%）
  level: 35,                    // 技能等级（匹配 level-range skill attack）
  tags: ["转职技能", "出血"],    // 标签（匹配 skill-specific bonuses）
  vp: 0,                        // VP 系统：0=不选, 1=模式1, 2=模式2
  vpEnhance: 0,                 // VP 强化：0=不强化, 1=+55%伤害(3技能), 2=+38%伤害且CD-15%(3技能)

  // 衍生
  // trigger_count = int(time_window / cooldown) + 1
  // (vpEnhance=2 时 cooldown 实际值为 cooldown × 0.85)
}
```

- `vp` replaces the old talisman/rune system. Each skill can independently
  select VP mode 0, 1, or 2.
- `vpEnhance` grants a bonus to up to 3 selected skills:
  - `1` → `+55%` damage (applied as a skill-specific multiplier on `single_percent`).
  - `2` → `+38%` damage + cooldown reduced by `15%`.
  - `0` → no enhancement.
user-facing UI typically shows aggregated skill groups (e.g., "觉醒技能" +
"80 级技能" + "其他技能"), while the engine can expand to individual entries
for complex analysis.

### Hierarchical Display

Skills may have display-level parents and calculation-level components. For
example, display skill `B` may contain components `B1, B2, B3`. A modifier
targeting `B` cascades to its components by default. Output displays the
merged result `B = B1 + B2 + B3`.

### Final Skill Damage

```text
total_skill_damage = sum(skill_i × coefficient_i)
```

Common coefficients can be factored out only when they apply universally.
Skill-specific bonuses must stay inside the sum.

### Normalized Share vs Actual Values

```text
sum(actual_damage_i × relative_multiplier_i)
≡ baseline_total_damage × sum(share_i × relative_multiplier_i)
```

where `share_i = actual_damage_i / baseline_total_damage`. The normalized-share
mode only changes output scale and produces the same relative gain. A new damage
source not represented in the original shares must be entered in the same
normalized unit or as an actual value with a known baseline.

## Coarse Mode Data（粗放模式数据）

When `ifExpert = 0`, skill data follows a preset template:

```text
coarse_skills = {
  ALLSKILLS: 85000,             // 用户或系统预设的全局技能百分比总和
  shares: [
    { name: "觉醒技能",       share: 0.18 },
    { name: "80 级技能",      share: 0.12 },
    { name: "75 级技能",      share: 0.08 },
    { name: "其他技能",       share: 0.62 }
  ]
}
```

- `ALLSKILLS` is a single numeric input — the total skill percent across all
  entries, representing the character's overall skill output before equipment
  modifiers.
- `shares` define how ALLSKILLS is distributed into individual skill entries.
  The sum of all shares must equal 1.0.
- Each skill entry's base value is derived as `skill_i = ALLSKILLS × share_i`.
- The preset shares are editable in admin mode for advanced tuning.

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

Each equipment unit may have multiple special effects. Every effect is calculated
independently through the effect branch, then summed.

### Effect Parameters

```text
equipment_effect = {
  name: "燃烧世界之火·烈焰",
  category: "sustained" | "burst" | "per_hit",

  // 基础参数
  single_percent: 182.5,        // 单次百分比
  cooldown: 10,                 // CD（秒）
  efficiency: 1.0,              // 效率（默认 100%，仅 sustained 可调低）

  // 爆发型专属
  damage_time: null | 5,        // 伤害分布时长（秒），如 100万 分 5s 打完

  // 按次型专属
  manual_count: null | 30,      // 手动输入次数，公式值仅作上限
}
```

### Trigger Count（触发次数）

默认公式（所有技能在初始化时处于可用状态）：

```text
trigger_count = int(time_window / cooldown) + 1
```

例外：
- **per_hit 型**：`trigger_count = manual_count`，公式值 `int(time_window / cooldown) + 1` 作为输入上限。

### Effect Total Percent（特效总百分比）

```text
effect_total_percent = single_percent × trigger_count × efficiency
```

### Three Categories

| 类别               | 单次伤害 | CD             | 特点                                                |
| ------------------ | -------- | -------------- | --------------------------------------------------- |
| `sustained` 持续型 | 较低     | 较短           | CD ≤ 3s 时效率可折损（`efficiency < 1.0`）          |
| `burst` 爆发型     | 很高     | 很长           | 有 `damage_time` 分布时长；策略编排不受放置位置影响 |
| `per_hit` 按次型   | —        | 极短（≤ 0.5s） | 次数手动输入，不需要 `efficiency`                   |

### Full Calculation

```text
per_effect_damage_j =
  // ----- 基础聚合 -----
  single_percent_j × trigger_count_j × efficiency_j
  × special_coefficient          // 特效转化系数（special_coefficient 区，通常为 10）

  // ----- 乘区放大 -----
  × common_effect_coefficient    // 三攻 × 力智 × 属强 × 攻强 × 全局技攻

common_effect_coefficient =
  effect_attack_multiplier
  × effect_main_stat_multiplier
  × effect_element_multiplier
  × attack_enhance_multiplier
  × global_skill_damage_product
```

### Restrictions

- Special effects do **not** receive level-range skill attack or fake skill attack.
- Special effects do **not** receive wedding-room stats (currently).
- Multiple special effects dilute each other because they sum in the same data area, then share common multipliers.

## Cooldown System（冷却缩减系统）

DNF's cooldown reduction system is multiplicative across all sources. Each CD
reduction source contributes independently via multiplication, with a hard cap
of 70% total reduction.

### Cooldown Multiplier Formula

```text
final_CD_multiplier = (1 - CD_a) × (1 - CD_b) × (1 - CD_c) × ...
final_cooldown = base_cooldown × final_CD_multiplier
total_CD_reduction = 1 - final_CD_multiplier
```

Example: three CD sources at 5%, 15%, 18.5%:

```text
multiplier = 0.95 × 0.85 × 0.815 = 0.6581125
reduction  = 1 - 0.6581125 = 34.19%
40s skill  → 40 × 0.6581125 = 26.3245s
```

### 70% Hard Cap

Total CD reduction (`1 - final_CD_multiplier`) **cannot exceed 0.70**. When a
new CD source pushes the total beyond this cap, the excess portion is voided
and the effective value of that source must be recalculated:

```text
effective_multiplier = 0.30 / (product of all prior multipliers)
effective_CD_value  = 1 - effective_multiplier
```

Example: given existing `0.95 × 0.85 × 0.815 = 0.6581125`, adding a "-55% CD" source:

```text
unbounded = 0.6581125 × 0.45 = 0.296150625 → 1 - 0.29615 = 70.38% (exceeds 70%)
effective  = 0.30 / 0.6581125 = 0.455849114 → effective CD = 1 - 0.455849 = 54.415%
```

The user-facing display should show the **nominal** value (-55%) and the
**effective** value (-54.415%) when overflow occurs.

### Damage Per Second（秒伤）

DPS provides a normalized comparison of damage output rate across skills:

```text
dps = single_percent / cooldown
```

- Available for all skill entries and equipment effects.
- When `vpEnhance = 2`, use `cooldown × 0.85` in the denominator.
- Primary use: ranking display and marginal gain analysis across different
  time windows.

### CDR Log Ratio（CD 对数比）

A ranking-oriented metric that converts cooldown reduction into an equivalent
skill-attack multiplier. Used when comparing CD-focused equipment against
non-CD equipment in leaderboard scenarios.

**Base pair**: +4% skill attack & −8% CD form a standard "skill-attack : CD" pair.

```text
base_pair:  1.04 (skill attack)  ×  0.92 (CD)
```

Both skill attack and CD compound multiplicatively. Any sequence of
skill-attack/CD pairs decomposes into n units of the base pair:

```text
total_skill_attack = 1.04^n
total_CD_mult      = 0.92^n
total_CDR          = 1 - 0.92^n
```

Given a known CDR value, solve for the equivalent skill-attack multiplier:

```text
n = ln(1 - CDR) / ln(0.92)
equivalent_skill_attack = 1.04^n = exp(ln(1.04) × ln(1 - CDR) / ln(0.92))
```

The base pair values (4% / 8%) act as a common unit system — any composite
of real skill-attack and CD lines maps to the same ratio at the limit,
making CDR-to-damage conversion consistent across all equipment.

**Usage boundary**:

| Mode                            | CD evaluation                                                              |
| ------------------------------- | -------------------------------------------------------------------------- |
| Expert (`ifExpert = 1`)         | Use real cooldowns + `trigger_count` formula — exact                       |
| Coarse ranking (`ifExpert = 0`) | Use CDR log ratio — approximate CDR as equivalent skill attack for ranking |

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
