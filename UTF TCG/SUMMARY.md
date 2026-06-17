# UTF Tactics TCG — Project Summary

## What It Is

A single-file turn-based tactical grid game that takes the heroes, spells, and items from **UTF Arena** (a multiplayer Canvas MOBA) and places them into a **3×5 turn-based grid** — like a TCG/tactics mashup.

- One HTML file (`tcg.html`), ~2100 lines
- No external JS libraries or frameworks
- Served via a minimal Express server (`server.js`)
- Mobile-first, monospace aesthetic matching UTF Arena (neon green/red/gold on black)

---

## Architecture

### Screens (3 phases)
```
Draft → Placement → Battle
```

**Draft** — pick 4 heroes, assign levels (1×Lv5, 2×Lv3, 1×Lv1), assign up to 3 items total  
**Placement** — drag heroes onto your half of the board (rows 3–4)  
**Battle** — turn-based, speed-ordered, player vs AI

### Battle State (`B` object)
```js
B = {
  grid: Array(5).fill(null).map(()=>Array(3).fill(null)), // 3x5 grid
  units: [],          // all living units
  turnOrder: [],      // sorted by speed at round start
  turnIdx: 0,
  round: 1,
  phase: 'move',      // move | spell | attack | enemy
  phaseStep: '',      // select_move_dest | select_spell_q | select_spell_e | select_atk_target
  moveHighlights: [], spellHighlights: [], atkHighlights: [],
  bombs: [],          // delayed bomb objects (Arson Q)
  shieldBursts: [],   // Ironclad E deferred AoE
  gameOver: false,
}
```

### Turn Flow
```
nextTurn()
  ├─ player turn: buttons enabled → player clicks → callback → endTurn()
  └─ enemy turn: doEnemyTurn()
       ├─ move (toward nearest enemy)
       ├─ pick ONE spell (if ready, heal only if HP < 50%)
       │    └─ lock cdLeft immediately → playSpellAnim → castSpell → doAtkPhase
       ├─ attack (playAtkAnim → doAttack → endTurn)
       └─ endTurn() → nextTurn()
```

**Key invariant:** spell CD is locked (`sp.cdLeft = sp.cdRounds`) *before* the animation starts, so no double-cast is possible even if the animation callback is delayed.

### Animation Engine (Canvas RAF)
```
CVS.add(particle) → _loop() → _update(dt) → _draw()
                                    │
                            dead particles → onDone callbacks (after filter)
```

Particle types: `proj` (flying glyph), `burst` (expanding glyph), `ring` (expanding arc), `line` (dashed line), `float` (rising damage number)

All game-flow callbacks go through `_once(cb)` wrapper — fires exactly once. `playSpellAnim` and `playAtkAnim` both have 1.5s/1s watchdog timers as absolute fallback.

### Hero Stats (per unit at battle start)
```
hp, maxHp, AD, AP, armor, mr, move, range,
lifesteal, pen, burnAura, onHitSlow, healPower, antiHeal,
permShield, critChance, haste, moveBonus,
stunned, slowed, silenced, shieldHp,
marked, packMark, thornsBuff, adBuff, shadeBuff, moveBuff, pierceBuff,
acted, atkDone, isActive, isPlayer, isSummon,
Q: {cdLeft, cdRounds, name, type, dmg, ...},
E: {cdLeft, cdRounds, name, type, dmg, ...},
```

Stats scale with level: Lv5 = 2.2×, Lv3 = 1.6×, Lv1 = 1.0× (HP/AD/AP/armor/MR all scale).

---

## Hero Roster (26 heroes, 6 roles)

| Role | Heroes |
|------|--------|
| FIGHTER | Vanguard, Jirina, Nemesis, Bruiser, Arson |
| TANK | Ironclad, Hana, Jailer, Goliath |
| SLAYER | Lynx, Wanderer, Kratoma, Quiller, Fusilier, Pyromancer |
| MAGE | Mage, Summoner, Tamer |
| SPLITPUSHER | Zephyr, Reaper, Volstrov |
| SUPPORT | Healer, Cleric, Eggchanter, Oracle, Doctor |

---

## Spell Types (mapped in `castSpell`)

| Type | Mechanic |
|------|----------|
| `projectile` | Flies to target cell, deals AP damage |
| `projectile_silence` | Projectile + silence 1 round |
| `line_shot` | Ranged shot + slow |
| `aoe_stun` | AoE around caster, damage + stun 1r |
| `aoe_dmg` | AoE damage to adjacent enemies |
| `aoe_slow` | AoE slow to adjacent enemies |
| `aoe_push` | AoE knockback (push away 1 cell) |
| `dash_strike` | Move 1 cell + damage on landing |
| `leap` | Dash 2 cells + damage |
| `blink` | Teleport 2 cells |
| `blink_shield` | Teleport + shield |
| `dash_slow` | Dash + slow zone |
| `dash_reset` | Dash + shield + reset Q cooldown |
| `charge_line` | Charge through row, damage all |
| `column_shot` | AoE hit entire column |
| `hook` | Pull enemy to adjacent cell |
| `pull_stun` | Pull + stun 1r |
| `mark` | Mark self — next attack empowered |
| `shot_mark` | Ranged shot that marks target |
| `pack_mark` | Mark for wolf pack bonus |
| `multi_hit` | 3 rapid hits on one target |
| `multi_target` | Hit up to 3 different enemies |
| `wolf_strike` | Hit 2 nearest enemies simultaneously |
| `shield_counter` | Shield; if survives → +30% AD next round |
| `shield_burst` | AoE burst when shield expires |
| `shield_aoe` | Shield all allies |
| `thorns` | Reflect 30% incoming damage |
| `ad_buff` | +30% AD on next attack |
| `move_buff` | +1 move for N rounds |
| `shade_mode` | Extended range + damage bonus |
| `pierce_buff` | Next 2 attacks pierce full column |
| `heal_single` | Heal one ally |
| `heal_adjacent` | Heal adjacent allies |
| `heal_aoe` | Heal all allies |
| `heal_silence` | Heal self + silence adjacent enemies |
| `beam_heal` | Damage enemy + heal ally |
| `delayed_bomb` | Plant bomb, explodes next round |
| `smoke` | AoE slow zone |
| `summon` | Spawn Ghoul (scales with caster level) |
| `summon_healers` | Spawn 2 Chick heal-bots |
| `egg` | Ranged slow + heal adjacent allies |
| `knockback_shot` | Ranged shot + push target back 1 cell |
| `cone_slow` | Slow in front cone |
| `aoe_target` | AoE centered on target cell (not caster) |
| `line_fire` | Hit entire column at target column |

---

## Files

```
UTF TCG/
├── tcg.html       Main game (all HTML, CSS, JS — single file)
├── server.js      Minimal Express server for Render.com
├── package.json   Node deps (express only)
├── README.md      Player-facing docs
└── SUMMARY.md     This file
```

---

## Known Limitations / Future Work

- AI always picks Q before E (no spell priority scoring)
- No game replay or match history
- No multiplayer (single-player vs AI only)
- Canvas animations disabled on `file://` protocol (Chrome security) — use `npm start` or deploy
- Summons (Ghoul, Chick) don't participate in turn-order UI display
- No sound
