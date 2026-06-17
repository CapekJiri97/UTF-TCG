# UTF Tactics TCG

Turn-based tactical grid game built from the heroes, spells and items of [UTF Arena](https://github.com/your-org/utf-arena).

## Play

```
npm install
npm start
# open http://localhost:3000
```

Or deploy to Render.com (see below).

## How to Play

### Draft
- Pick **4 heroes** from the roster
- Assign levels: 1x Level 5, 2x Level 3, 1x Level 1
- Assign up to **3 items total** (max 2 per hero, no duplicates per hero)

### Placement
- Place your 4 heroes in your half of the board (rows 3–4)

### Battle — turn order
Each unit acts once per round in speed order. On your turn:

| Button | Action |
|--------|--------|
| `+` MOVE | Move up to your move range (BFS pathing, no passing through units) |
| `Q` / `E` | Cast spell — select target cell from highlighted options |
| `!` ATTACK | Manual attack — select an enemy in range |
| `▶` END | End turn (auto-attacks nearest enemy if you haven't attacked) |

### Grid
- **3 columns × 5 rows**
- Row 2 (middle) = capture zone — holding it scores capture points
- Melee range = 1, Ranged range = 2

### Win condition
Eliminate all enemy non-summon units.

---

## Heroes (23)

| Glyph | Name | Role | Type |
|-------|------|------|------|
| V | Vanguard | Fighter | Physical |
| ❋ | Jirina | Fighter | Magical |
| N | Nemesis | Fighter | Physical |
| B | Bruiser | Fighter | Physical |
| A | Arson | Fighter | Magical |
| I | Ironclad | Tank | Physical |
| ✿ | Hana | Tank | Magical |
| J | Jailer | Tank | Magical |
| G | Goliath | Tank | Physical |
| L | Lynx | Slayer | Physical |
| W | Wanderer | Slayer | Physical |
| K | Kratoma | Slayer | Physical |
| Q | Quiller | Slayer | Physical |
| F | Fusilier | Slayer | Physical |
| M | Mage | Mage | Magical |
| S | Summoner | Mage | Magical |
| P | Pyromancer | Slayer | Magical |
| T | Tamer | Mage | Magical |
| Z | Zephyr | Splitpusher | Magical |
| R | Reaper | Splitpusher | Magical |
| @ | Volstrov | Splitpusher | Magical |
| H | Healer | Support | Magical |
| C | Cleric | Support | Magical |
| E | Eggchanter | Support | Magical |
| O | Oracle | Support | Physical |
| D | Doctor | Support | Physical |

## Items (14)

| Icon | Name | Effect |
|------|------|--------|
| [AD] | Power | +15% AD/AP +5 |
| [HP] | Vitality | +20% HP +20 |
| [AR] | Plate | +20% Armor +3 |
| [MR] | Veil | +20% MR +3 |
| [CD] | Haste | -1 spell CD (min 1) |
| [MV] | Boots | +1 move range |
| [LS] | Vamp | +12% lifesteal |
| [PEN] | Pen | +20% armor/MR pen |
| [BRN] | Burn | +8% maxHP AoE burn on attack |
| [SLO] | Frost | On-hit slow 1 round |
| [H+] | HealPwr | +20% to all heals |
| [GW] | GW | -30% enemy healing |
| [SH] | Barrier | +15 permanent shield |
| [CR] | Crit | +20% crit chance |

## Status Effects

| Tag | Meaning |
|-----|---------|
| [STN] | Stunned — skip turn |
| [SLO] | Slowed — no move this turn |
| [SIL] | Silenced — no spells this turn |
| [S:N] | Shield with N HP remaining |
| [MRK] | Marked — next hit deals +60% damage |
| [PKM] | Pack Mark — bonus damage from wolf |
| [THN] | Thorns — reflect 30% of incoming damage |
| [PWR] | AD buff — next attack +30% damage |
| [SHD] | Shade mode — extended range, bonus damage |
| [SPD] | Speed buff — +1 move |
| [PRC] | Pierce — attacks hit entire column |

---

## Deploy to Render.com

1. Push the `UTF TCG/` folder to a GitHub repository
2. On [Render.com](https://render.com): **New → Web Service**
3. Connect your GitHub repo
4. Settings:
   - **Build Command:** `npm install`
   - **Start Command:** `node server.js`
   - **Environment:** Node
5. Deploy — Canvas animations work fully on `https://` (no `file://` restrictions)

---

## Tech Stack

- Single HTML file (`tcg.html`) — no framework, no build step
- Canvas 2D for spell/attack animations (RAF particle engine, UTF Arena style)
- Pure CSS grid: 3×5 board
- BFS pathfinding for movement
- Chebyshev distance for range checks
- Enemy AI: move → spell → attack chain, fully animated
