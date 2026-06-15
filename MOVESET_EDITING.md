# Beginner's Guide to Editing Movesets

This guide covers the four core combat tables and how they connect. Think of them as four layers: **input → combo string → move logic → hit effect**.

```
Button press
    ↓
ActionDataTable      "Which attack set does X / Y / B / R1 use?"
    ↓
AttackSetDataTable   "What hits fire, in what order?"
    ↓
AttackDataTable      "How does each hit behave?"
    ↓
DamageDataTable      "What happens when it connects?"
```

---

## ID naming

Row IDs follow a predictable pattern:

| Pattern | Meaning |
|---------|---------|
| `CP_010` | Character (Yuji). `CN_250` = enemy/alternate cast. |
| `CP_010_ATTACKA` | **Attack set** — a whole button string (e.g. X combo). |
| `CP_010_ATTACKA_01_01` | **Single hit/phase** inside that set. |
| `…_03_SETUP_01` | Setup phase before the active hit (no damage usually). |
| `…_AIR` | Air variant of an attack set. |
| `…_SOLO` | Solo-mode variant. |
| `…_1`, `…_2`, `…_3`, `…_4` | CE tier variants (see below). |

The **same string** is usually the row key under `Rows` and the `ID` field inside that row. Keep them in sync.

---

## 1. ActionDataTable — buttons to attack sets

One row per character (`CP_010`, `CN_250`, etc.). This table answers: *"When I press a button, which attack set do I get?"*

### Normal attacks

| Field | Input |
|-------|-------|
| `Id_AttackSet_Normal_1` | X / Square |
| `Id_AttackSet_Normal_2` | Y / Triangle |
| `Id_AttackSet_Normal_3` | B / Circle |
| `Id_AttackSet_Normal_1_1` | X + Down |
| `Id_AttackSet_Normal_2_1` | Y + Down |
| `Id_AttackSet_Normal_3_1` | B + Down |

Each value is a single attack-set ID like `"CP_010_ATTACKA"`.

The `Id_AttackSet_Normal_Air_*` fields mirror the above for airborne inputs. During a combo, **Down variants take priority** — you can use an air down-move mid grounded combo if ground and air sets differ.

`_Auto` fields are alternate sets reached via auto-combo transitions (`AutoTransitionKind` on attack rows). `_Solo` fields are used in solo mode only.

### Cursed Energy (R1 specials)

CE fields are **arrays of 4 attack-set IDs**, one per CE level:

```
Index 0 → CE Level 1
Index 1 → CE Level 2
Index 2 → CE Level 3
Index 3 → CE Level 4 (Domain Expansion, when present)
```

Example from Jogo (`CP_140`):

```json
"Id_AttackSet_CursedEnergy_1": [
  "CP_140_ATTACKH_1",
  "CP_140_ATTACKH_2",
  "CP_140_ATTACKH_3",
  "CP_140_ATTACKH_4"
]
```

Characters without a domain use `"None"` in slot 4 (e.g. Yuji). There are separate fields for:

- `Id_AttackSet_CursedEnergy_1` / `_2` — R1 and R2 specials (ground)
- `Id_AttackSet_CursedEnergy_Air_*` — air versions
- `*_Auto` variants — auto-combo / chain routes into CE moves

### Other attack fields

| Field | Input |
|-------|-------|
| `Id_AttackSet_SuperCursedEnergy` | L2 + R2 super (ground) |
| `Id_AttackSet_SuperCursedEnergy_Air` | L2 + R2 super (air) |
| `Id_ExtraAttack_1` / `_2` | Extra attack slots (character-specific) |

Movement (dash, jump, step, breakfall) is on the **same row** via `Id_ActionDash`, `Id_ActionJump`, etc. — those point to different tables, not attack sets.

### Easiest edit here

Swap which move a button uses without touching the move itself:

```json
"Id_AttackSet_Normal_2": "CP_010_ATTACKB"
```

---

## 2. AttackSetDataTable — combo strings

One row per attack set (`CP_010_ATTACKA`, `CP_010_ATTACKE_1`, etc.).

### `Id_Attack` — the combo chain

An ordered list of attack-row IDs (up to 12 slots):

```json
"CP_010_ATTACKA": {
  "ID": "CP_010_ATTACKA",
  "Id_Attack": [
    "CP_010_ATTACKA_01_01",
    "CP_010_ATTACKA_02_01",
    "CP_010_ATTACKA_03_SETUP_01",
    "CP_010_ATTACKA_03_01",
    "CP_010_ATTACKA_04_01"
  ]
}
```

Pressing X starts this set; the game walks through `Id_Attack` in order as you continue the string.

### Other useful fields

| Field | Purpose |
|-------|---------|
| `bCursedEnergyAttack` | `true` for CE moves (affects CE recovery, etc.) |
| `AttackRangeType` | Short / Mid / Long classification |
| `Inherit_Inertia_Rate` | How much prior momentum carries into the set |

### Single-hit sets

Not every set is a combo. Y+Down might be one row:

```json
"CP_010_ATTACKB": {
  "Id_Attack": ["CP_010_ATTACKB_01_01"]
}
```

### Special cases

Some characters (Megumi, Nanami) pack **multiple possible moves** into one set; blueprint logic picks which one fires. Swapping those sets for unrelated moves (e.g. replacing a summon with a melee combo) often breaks — the animation/blueprint still expects the original behavior.

### Easiest edit here

Reorder or trim a combo without creating new attacks:

```json
"Id_Attack": [
  "CP_010_ATTACKA_01_01",
  "CP_010_ATTACKA_03_01"
]
```

---

## 3. AttackDataTable — per-move properties

Split across **5 shard files** (`AttackDataTable1.json` … `AttackDataTable5.json`). Each attack ID lives in exactly one shard — find the file that already contains your character's rows.

One row per hit/phase (`CP_010_ATTACKA_01_01`).

### What lives here

Move **logic**: animations, input windows, transitions, armor, homing, CE gain on use.

### Fields beginners touch most

| Field | What it does |
|-------|--------------|
| `CharacterAnimation` | Which anim key plays (links to `CharacterAnimationDataTable`) |
| `AttackTransitionType` | When the move can chain (on hit, on whiff, auto, etc.) |
| `AttackTransitionKind` | Which buttons can cancel/continue from this hit |
| `AutoTransitionKind` | Auto-combo follow-up route |
| `AutoTransitionKind_CursedEnergy` | Auto route into CE moves |
| `bSuperArmorEnabled` / `bHyperArmorEnabled` | Armor during the move |
| `AttackTiming` | Phase timing / active frames |
| `Id_ActionHoming` | Homing behavior during the attack |
| `CursedEnergy_Add` | CE meter change when using this move |

---

## 4. DamageDataTable — per-hit damage properties

Also sharded across 5 files (`DamageDataTable1.json` … `DamageDataTable5.json`), same rules as attack tables.

One row per **damage event** — usually same ID as the attack hit it belongs to.

### What lives here

Hit **effects**: damage numbers, launch type, guard break, meter gain, hitstop, knockback.

### Fields beginners touch most

| Field | What it does |
|-------|--------------|
| `Damage` | Base HP damage |
| `DamageType` | Launch / blowback / spin / etc. |
| `GuardDurability_Damage` | Guard gauge damage |
| `Down_Damage` | Damage to downed state |
| `CursedEnergyExp_Add_Attacker` | CE meter gain on hit |
| `TensionGauge_Add_Attacker` | Awakening / tension gain |
| `HitSlow_Attacker` / `HitSlow_Receiver` | Hitstop strength |
| `HitSlow_Time_*` | Hitstop duration |
| `DamageSpeed_Start` / `_End` | Launch speed |
| `DamageDirectionAdjustPitch` / `Yaw` | Launch angle |
| `bIgnoreGuard` | Unblockable |
| `bIgnoreSuperArmor` | Breaks super armor |

### Important limitation

Damage row IDs are **baked into animations** (animation notifies in the pak). You can freely edit existing damage rows, but **cloning a row with a new ID won't work** unless the animation is also edited to reference that new ID.

Multi-hit moves can reference **multiple** damage rows from one animation.

Attack and damage IDs don't always pair 1:1 — SETUP phases often have attack rows but no damage row.

---

## Common workflows (easiest → hardest)

### 1. Rebind a button to a different existing move

**Tables:** `ActionDataTable` only

Change the relevant `Id_AttackSet_*` to point at another attack-set ID.

### 2. Change combo order or length

**Tables:** `AttackSetDataTable`

Edit `Id_Attack` — reorder, remove hits, or swap in IDs from another set.

### 3. Tune damage / launch / meter on an existing hit

**Tables:** `DamageDataTable`

Find the row matching the hit ID, adjust `Damage`, `DamageType`, meter fields, etc.

### 4. Change move behavior (armor, cancels, transitions)

**Tables:** `AttackDataTable`

Edit transition enums, armor flags, homing links on the hit row.

---

## Finding your character's data

Per-character slices live in `datatables_by_char/<CHAR_ID>/` (e.g. `datatables_by_char/CP_010/`) in the (datatables repo)[https://github.com/JJKCursedClashModding/DataTables/tree/master/datatables_by_character]. Each folder has that character's rows from every table — much easier than searching the full `datatables/` files.

Character ID reference: see `wiki/reference/characters.md` (`CP_010` = Yuji, `CP_140` = Jogo, etc.).

---

## Pitfalls

1. **Shard consistency** — don't duplicate the same ID across `AttackDataTable1` and `AttackDataTable2`.
2. **ID mismatch** — row key, `ID` field, and any reference from `Id_Attack` must all match exactly.
3. **Missing attack row** — every ID in `Id_Attack` needs a row in `AttackDataTable`. Missing rows crash or soft-lock.
4. **Damage without animation support** — new damage IDs need pak/animation edits.
5. **Blueprint-locked moves** — summons, domains, and some character-specific moves have logic outside the tables. Table swaps alone may not be enough.
6. **`"None"` vs missing** — unused slots should be `"None"`, not omitted (match surrounding rows' style).
7. **Shared rows** — some characters reference another's step/breakfall IDs. Editing the source row affects both.

---

## Related tables (not core, but you'll hit them)

| Table | When you need it |
|-------|------------------|
| `CharacterAnimationDataTable` | Changing which montage plays |
| `CollisionModifyDataTable` | Hitbox active frames / size |
| `EffectMoveDataTable` | Projectile speed and homing |
| `CharacterSpecialAttackDataTable` | Super/domain metadata |
| `ActionHomingDataTable` | Tracking behavior during attacks |

---

## Quick reference

| Table | Question it answers |
|-------|---------------------|
| **ActionDataTable** | "What attack set does this button + CE level use?" |
| **AttackSetDataTable** | "Which hits fire, in what order?" |
| **AttackDataTable** | "How does each hit behave?" |
| **DamageDataTable** | "What happens when it connects?" |
