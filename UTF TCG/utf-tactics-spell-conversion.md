# UTF Arena → UTF Tactics — detailní konverze spellů

Referenční dokument: hrdina → spell → konkrétní grid implementace → na co si dát pozor.
Pokrývá všech 26 hrdinů z `classes.js`, oba spelly (Q/E) u každého.

## Konvence použité v tomto dokumentu

**Tile scale:** 1 dlaždice ≈ 65 px z originálu (odvozeno z `engageRange`/`preferredCombatDist`).

**CD v kolech:** `ceil(baseCooldown / 2)`, zaokrouhleno nahoru — zachovává relativní poměry
mezi rychlými/pomalými spelly. Jen orientační start, ladit při hraní.

**Velikost AoE — dva tiery** (protože deska je jen 3 sloupce široká, jemné rozdíly v px
80–250 se na ní ztratí):
- **BURST** (radius 1) — vlastní/cílová dlaždice + všech 8 sousedů. Pokrývá celou šířku
  desky + 1 řadu navíc každým směrem. Pro原 80–150px AoE (drtivá většina bojových spellů).
- **TEAM-WIDE** (radius 2–3) — efektivně celá vlastní polovina hřiště. Pro 200px+ heal/shield ulty.

**Dash vzdálenost — tři tiery:**
- **micro** (1 dlaždice) — 50–75px v originále (Lynx E, Reaper E, Goliath E, Volstrov E)
- **short** (2 dlaždice) — 150–180px (Vanguard Q, Bruiser E, Hana E, Goliath Q)
- **long** (3 dlaždice) — 250px (Quiller E, Wanderer E)

**Lane shot:** Protože dráha letu projektilu (`pSpeed × life`) v originále skoro vždy
přesahuje rozměr desky, neřešte dosah — netargetovaný projektil prostě zasáhne prvního
nepřítele ve zvoleném sloupci/řádku odkudkoliv na desce.

**Self-centered vs. targeted AoE — důležité rozlišení**, které se v `type: 'aoe'` ztrácí:
některé AoE jsou vystřelené na vybranou dlaždici (Mage E, Oracle Q — hráč/AI cílí tile),
jiné jsou centrované na castovatele (Vanguard E, Ironclad E, Jailer E, Pyromancer E).
U každého spellu dole je to explicitně uvedeno.

**Globální rozhodnutí: dash nahrazuje move fázi.** Spelly typu `dash` s damage komponentou
(ne čisté repositiony) by měly **spotřebovat move fázi daného kola**, ne být bonus pohyb
navíc — jinak tanky a fightery s dashem mají efektivně dva pohyby za kolo a rozbije se vám
balance, který už máte odladěný přes `playStyle.backoffAggression` apod.

---

## FIGHTER

### Vanguard (V)

**Q — Charge** (orig: dash 170px, dmg 75+0.2 AD, 60% slow 1.5s, CD 6.0s)
- Archetyp: Dash-strike
- Implementace: pohyb 2 dlaždice v přímé linii (nahrazuje move fázi) → damage + Slowed
  status všem nepřátelům na dlaždicích, kterými projel, i na cílové. CD ≈ 3 kola.
- Pozor: musíte rozhodnout, jestli hráč nejdřív vybere směr/cíl a pak se spočítá dráha,
  nebo jestli "dash" je jen alternativní move akce viditelná přímo ve výběru cílové dlaždice.
  Pokud cesta naráží na obsazenou dlaždici (jiná jednotka), dash by se měl zastavit před ní —
  jinak vám charge prochází skrz nepřátele jako duch.

**E — Sweep** (orig: aoe radius 140, dmg 90+0.25 AD)
- Archetyp: Pulse (self-centered, žádné cílení)
- Implementace: BURST radius 1 kolem vlastní dlaždice, čistý damage, žádný CC. CD ≈ 4 kola.
- Pozor: nejjednodušší spell v celé hře — dobrý kandidát na první implementaci pulse patternu
  end-to-end, protože nemá žádný status efekt co trackovat.

### Jirina (❋)

**Q — Pressure Wave** (orig: aoe_knockback radius 145, dmg 65+0.6 AP)
- Archetyp: Pulse + knockback (self-centered)
- Implementace: BURST radius 1 kolem sebe, damage + knockback 1 dlaždici od castovatele.
  CD ≈ 3 kola.
- Pozor: knockback ze self-centered pulse může postrčit více nepřátel najednou různými
  směry (každý pryč od castovatele) — ujistěte se, že resolve pořadí je deterministické
  (např. podle vzdálenosti), jinak se vám můžou dvě jednotky snažit skončit na stejné dlaždici.

**E — Heal aoe** (orig: heal_aoe radius 200, amount 60+0.45 AP)
- Archetyp: Heal pulse (self-centered, TEAM-WIDE)
- Implementace: heal sobě + všem spojencům v radiusu 2 od Jiriny. CD ≈ 5 kol.
- Pozor: zkontrolujte `antiHeal` (Grievous Wounds z items.js) na **cíli** heal efektu, ne na
  castovateli — heal_aoe healí spojence, takže grievous debuff musí číst z toho, kdo heal přijímá.

### Nemesis (N)

**Q — Vendetta** (orig: vendetta, dmg 55+0.5 AD, mark 3s, +30% AD bonus na marked, kill = reset CD)
- Archetyp: Lane shot + mark/execute (speciální mechanika)
- Implementace: vybere nepřítele v lane → damage + status "Marked" (3 kola). Útoky na
  marked cíl dávají +30% AD bonus damage. Pokud marked cíl zemře (jakýmkoliv zdrojem damage,
  ne jen od Nemesis), okamžitě `cdLeft = 0` na Q. CD ≈ 4 kola.
- Pozor: "kill resetuje CD" musí sledovat smrt marked jednotky **kdykoliv** v průběhu kola,
  ne jen po Nemesisině vlastním útoku — pokud ji zabije spojenec nebo damage-over-time efekt,
  reset by se měl spustit taky. Implementujte jako event listener na smrt jednotky s mark
  statusem, ne jako check jen v rámci Nemesisova vlastního tahu.

**E — Parry** (orig: parry, shield 55 HP/2s, broken→+18%MS+25%AD, expired untouched→-33% CD)
- Archetyp: Conditional shield (speciální)
- Implementace: shield na 1 kolo (zaokrouhleno z 2s). Pokud shield během toho kola pohltí
  útok (zlomí se) → bonus MS+AD na 1 kolo. Pokud kolo přežije nedotčený → `cdLeft` na E se
  sníží o 33%. CD ≈ 6 kol.
- Pozor: musíte přesně definovat moment kontroly — "přežil kolo nedotčený" se ověřuje na
  začátku Nemesisova příštího tahu, ne uprostřed nepřátelovy fáze. Dvě podmínky (broken vs.
  expired) jsou navzájem vylučující, takže stačí jeden boolean flag nastavený při prvním
  zásahu do shieldu.

### Bruiser (B)

**Q — Throwing weapon** (orig: projectile piercing, dmg 60+0.4 AD, 25% slow 1s, pierces)
- Archetyp: Lane shot, piercing
- Implementace: zasáhne **všechny** nepřátele ve zvoleném sloupci/řádku (ne jen prvního),
  damage + Slowed všem. CD ≈ 3 kola.
- Pozor: piercing lane shot je jediný "hit-all-in-lane" spell mezi single-target projektily —
  ujistěte se, že se neimplementuje stejnou funkcí jako normální lane shot s flagem `first:true`,
  jinak snadno zapomenete piercing flag a Bruiser bude slabší než má být.

**E — Leap** (orig: dash 150px, dmg 45+0.35 AD, radius 110 na dopadu)
- Archetyp: Dash-strike
- Implementace: pohyb 2 dlaždice (nahrazuje move), BURST radius 1 damage na dopadu. CD ≈ 5 kol.
- Pozor: standardní dash-strike, žádné speciální gotchas — dobrá referenční implementace
  pro ostatní dash-strike spelly.

### Arson (A)

**Q — Sticky bomb** (orig: sticky_bomb, dmg 70+0.65 AP, fuse 2s, AoE+stun 0.7s na explozi)
- Archetyp: Targeted projectile → delayed AoE (hybrid, speciální)
- Implementace: vybere nepřítele v lane → bomba se "přilepí" na jeho jednotku (status efekt,
  ne dlaždice). Po uplynutí fuse (≈1 kolo) exploduje BURST radius 1 **kolem aktuální pozice**
  toho nepřítele (sleduje ho, i když se mezitím pohnul) → damage + stun všem okolo. CD ≈ 4 kola.
- Pozor: musíte explicitně rozhodnout, co se stane, pokud nositel bomby zemře dřív, než
  vybuchne — buď bomba exploduje na místě smrti (zachová AoE hodnotu), nebo se prostě ztratí
  beze efektu. Doporučuju první variantu, je to víc fér vůči Arsonovi (neztrácí celé kouzlo
  kvůli tomu, že tým zabil cíl dřív). Potřebujete frontu zpožděných efektů podobnou vašemu
  existujícímu `bombs[]` poli — sticky bomb je přesně ten případ, na který je `bombs[]`
  pravděpodobně už navržené.

**E — Smoke bomb** (orig: smoke_bomb zone, radius 140, duration 2s/tick 0.25s, heal allies +
  debuff/slow enemies + final dmg burst)
- Archetyp: **Zone** — persistentní efekt na dlaždicích, ne na jednotce (nová kategorie!)
- Implementace: oblak zůstává na BURST radius 1 okolo cílové dlaždice po dobu ~2 kola.
  Na **začátku každého kola**, kdy je oblak aktivní: spojenci v oblasti se healí + dostávají
  def buff, nepřátelé v oblasti se debuffují + slow. Na konci trvání: jednorázový damage burst
  všem nepřátelům uvnitř. CD ≈ 7 kol.
- Pozor: tohle je jediný spell ve hře, který potřebuje efekt vázaný na **dlaždice**, ne na
  jednotku ani na castovatele — podobně jako `shieldBursts[]`, budete potřebovat nové pole
  v `B` objektu (např. `zones[]`) co trackuje aktivní oblasti, jejich remaining duration a
  efekt, a kontroluje se na začátku každého kola pro všechny jednotky stojící v dané oblasti
  (včetně těch, co se do ní přesunuly mezi koly).

---

## TANK

### Ironclad (I)

**Q — Shield + explode** (orig: shield_explode, shield 100 HP/4s, explode radius 144 dmg
  25+0.25AD+8%bonus max HP při zlomení/expiraci)
- Archetyp: Conditional shield → AoE (self-centered)
- Implementace: shield na 2 kola. Když praskne NEBO vyprší, okamžitě BURST radius 1 exploze
  kolem Ironcladovy aktuální pozice. CD ≈ 5 kol.
- Pozor: na rozdíl od Nemesis E (kde broken/expired dávají RŮZNÉ bonusy), tady oba případy
  spouští **stejnou** AoE explozi — implementačně jednodušší, ale neopomeňte, že exploze se
  musí spustit i v případě, že Ironclad mezitím zemřel (shield zlomený smrtelným hitem) —
  posmrtná exploze je legitimní a hráči ji čekají z originálu.

**E — Ground slam** (orig: aoe radius 135, dmg 70+0.25AD, stun 1.0s)
- Archetyp: Pulse + stun (self-centered)
- Implementace: BURST radius 1 kolem sebe, damage + Stunned status 1 kolo. CD ≈ 6 kol.
- Pozor: standardní pulse+CC, žádné speciální problémy.

### Hana (✿)

**Q — Bonus dmg stance** (orig: hana_q, 5s buff: bonus max HP dmg na útoky + 25% AS)
- Archetyp: Self-buff (stance, žádný damage instantně)
- Implementace: status "Empowered" na 3 kola — autoattacky během něj dávají bonus damage
  jako % cílova max HP + bonus attack speed efekt (vzhledem k 1 autoattack/kolo se AS bonus
  musí převést na něco jiného — viz sekce "Attack speed" v cross-cutting tématech). CD ≈ 6 kol.
- Pozor: `customMeleeAoE: 'ring'` u Hany znamená, že její základní útok v originále zasahuje
  kruh kolem cíle, ne kužel — pokud budete na grid převádět i autoattacky (ne jen spelly),
  Hanin základní útok by měl zasáhnout i sousední dlaždice okolo cíle, ne jen cíl samotný.

**E — Dash + def** (orig: dash_def 180px, dmg 55+0.5AP, slow 15%/1.7s na dopadu)
- Archetyp: Dash-strike
- Implementace: pohyb 2 dlaždice (nahrazuje move), BURST radius 1 damage+slow na dopadu.
  CD ≈ 4 kola.
- Pozor: standardní dash-strike, bez gotchas.

### Jailer (J)

**Q — Hook** (orig: projectile pull, dmg 55+0.5AP, pull target k caster, +9% max HP dmg)
- Archetyp: Lane shot + pull (speciální)
- Implementace: vybere nepřítele v lane → damage + posune ho na dlaždici **přímo sousedící**
  s Jailerem (ne jen "blíž", ale konkrétně adjacent). CD ≈ 5 kol.
- Pozor: pokud je cesta mezi cílem a Jailerem obsazená jinou jednotkou, kam se pull zastaví?
  Doporučuju: zastaví se na první volné dlaždici od Jailera směrem k cíli (takže pokud je
  cesta blokovaná, cíl skončí dál než adjacentně) — jednotné řešení sdílené s ostatními
  pull/push spelly (viz Oracle Q).

**E — Ground slam** (orig: aoe radius 120, dmg 70+0.4AP, slow 55%/2.0s)
- Archetyp: Pulse + slow (self-centered)
- Implementace: BURST radius 1 kolem sebe, damage + Slowed 1 kolo. CD ≈ 4 kola.
- Pozor: bez gotchas, standardní pulse.

### Goliath (G)

**Q — Unstoppable charge** (orig: dash 180px, dmg 25+0.35AD+3.75% current HP, radius 120 v cestě)
- Archetyp: Dash-strike (s "unstoppable" flagem)
- Implementace: pohyb 2 dlaždice (nahrazuje move), damage všem na cestě i na dopadu. CD ≈ 4 kola.
- Pozor: "unstoppable" v originále znamená, že charge nelze přerušit CC efekty (stun/slow) —
  pokud máte v tahovce status efekty co by normálně zabránily pohybu, Goliath Q by je měl
  ignorovat. Je to malá, ale důležitá výjimka z obecných pravidel pohybu.

**E — Heal + silence** (orig: dash_heal_silence, dmg 50, heal 64, distance 50, silence 1.5s na dopadu)
- Archetyp: Dash-strike (micro) + self-heal + silence
- Implementace: pohyb 1 dlaždice (micro dash), heal sobě + BURST radius 1 silence (ne damage
  primárně — `baseDamage:50` je tu vedlejší efekt) všem okolo. CD ≈ 6 kol.
- Pozor: silence efekt musí blokovat **spelly**, ne autoattacky — ujistěte se, že vaše
  reprezentace statusů rozlišuje Silenced (blokuje Q/E) od Stunned (blokuje vše včetně pohybu).

---

## ASSASSIN

### Lynx (L)

**Q — Triple dagger** (orig: projectile count 3, spread 0.45, dmg 45+0.2AD)
- Archetyp: Volley/cone → na 3 sloupce širokém poli se nejlíp převede jako "zasáhne až 3
  nepřátele v cílovém řádku najednou" (každá dýka = jeden cíl, pokud jsou volné dlaždice
  obsazené více nepřáteli)
- Implementace: zasáhne všechny nepřátele v cílovém řádku (max 3, podle obsazení). CD ≈ 3 kola.
- Pozor: pokud je v cílovém řádku jen 1 nepřítel, dostávají všechny 3 "dýky" stejný cíl
  (originál je cone, takže by se to dalo interpretovat i jako 3x damage na jednoho) — rozhodněte
  se a buďte konzistentní s Fusilier Q (5-shot), který má stejný problém.

**E — Dash + blade burst** (orig: dash 50px, dmg 75+0.35AD, radius 100, +10%MS 1s)
- Archetyp: Dash-strike (micro)
- Implementace: pohyb 1 dlaždice, BURST radius 1 damage + MS buff 1 kolo. CD ≈ 4 kola.
- Pozor: standardní, bez gotchas.

### Zephyr (Z)

**Q — Speed buff** (orig: buff_ms, +20% MS, 3.0s)
- Archetyp: Self-buff, žádný damage/cíl
- Implementace: status "Hastened" na 2 kola — v tahovce nejjednodušší konverze: zvyšte počet
  dlaždic, o které se Zephyr může pohnout daný tah (např. +1 dlaždice navíc k normálnímu pohybu).
  CD ≈ 5 kol.
- Pozor: `baseDamage:0, scaleAP:0.001` je očividně jen "technický" non-nulový scaling, aby
  spell systém nepadal na nulovém poweru — neimplementujte žádný damage komponent u tohoto spellu.

**E — Air burst** (orig: aoe_knockback radius 90, dmg 70+0.8AP+0.2AD)
- Archetyp: Pulse + knockback (self-centered)
- Implementace: BURST radius 1, damage + knockback. CD ≈ 3 kola.
- Pozor: bez gotchas.

### Reaper (R)

**Q — Charges** (orig: reaper_q, 3 charges, dmg 20+0.5AP + bonus range + slow na zásah)
- Archetyp: Buff s countery (speciální — překlad je tu přímější než u time-based spellů!)
- Implementace: status "Charged x3" — příštích 3 autoattacky dávají bonus damage + slow cíli,
  countdown se snižuje s každým útokem, ne s časem. Pokud charges nevyčerpá do 2 kol, vyprší.
  CD ≈ 5 kol.
- Pozor: tohle je jeden z mála spellů, kde je tahová verze **jednodušší** než originál — counter
  místo timeru. Akorát ošetřete edge case: co se stane, pokud Reaper za jedno kolo dá víc než
  1 autoattack (např. kombinace s nějakým item efektem)? Měl by spotřebovat tolik charges,
  kolik útoků provedl.

**E — Dash + reset** (orig: reaper_e, dash 75px, shield 60+40%MS 1.5s, resetuje Q CD)
- Archetyp: Dash-strike (micro) + shield + CD reset (speciální)
- Implementace: pohyb 1 dlaždice, shield + MS buff 1 kolo, **okamžitě** `Q.cdLeft = 0`. CD ≈ 7 kol.
- Pozor: CD reset by se měl aplikovat i v případě, že Q je už ready (no-op, žádná chyba) —
  hlavně netrigerujte žádný "double cast" bug, protože Q a E se castují v různých spell-slotech.

### Wanderer (W)

**Q — Spin to win** (orig: spin_to_win, 2s duration/0.25s tick, dmg 25+0.3AD per tick, radius 80,
  "move while spinning")
- Archetyp: Channel — **doporučuju lump-sum burst** (jednorázový damage = součet všech ticků,
  vyřeší se celý v rámci jednoho spell-phase tahu), ne multi-turn lock. Originál explicitně
  povoluje pohyb během spinu, takže rootovat Wanderera na 2 kola by byl odklon od jeho identity.
- Implementace: BURST radius 1 kolem sebe, jednorázový damage = ~8 ticků agregovaných do
  jednoho čísla (8 × (25+0.3AD) zhruba, naladit podle balance). CD ≈ 5 kol.
- Pozor: pokud později chcete "drsnější" verzi (Wanderer fakt stojí na místě 2 kola a tiká),
  je to legitimní alternativa — ale pak nezapomeňte přidat status, který blokuje move fázi,
  a respektovat, že originál to nedělá (designová odchylka, ne bug).

**E — Omnislash** (orig: dash 250px na hit → blink 5x na nedaleké nepřátele, dmg 40+0.4AD per hit)
- Archetyp: Multi-hit chain (speciální)
- Implementace: pohyb 3 dlaždice (long dash, nahrazuje move) → pokud zasáhne nepřítele, "blikne"
  na až 5 dalších nepřátel v okolí (radius 2 od aktuální pozice), zasáhne každého jednou,
  postupně. CD ≈ 8 kol.
- Pozor: na malé desce (15 dlaždic) může mít Wanderer reálně k dispozici míň než 5 cílů —
  implementujte jako "zasáhne tolik nepřátel, kolik je v dosahu, max 5", ne pevně 5. Také:
  pokud item efekty jako lifesteal nebo on-hit slow procují **per hit**, Wanderer s tímto
  spellem proccuje až 6x za jedno seslání (1 dash hit + 5 blink hitů) — ověřte, že to je
  záměrný balance, ne náhodný power creep.

### Nemesis (N)

— viz sekce FIGHTER výše (Nemesis je v kódu zařazen jako FIGHTER role, ale v UI hero-pickeru
je v ASSASSIN řádku).

---

## RANGED

### Quiller (Q)

**Q — Long range bolt** (orig: projectile, dmg 40+0.4AD, pSpeed 1200)
- Archetyp: Lane shot
- Implementace: zasáhne prvního nepřítele ve zvoleném sloupci/řádku, damage only. CD ≈ 4 kola.
- Pozor: bez gotchas — nejjednodušší lane shot ve hře, dobrý template pro ostatní.

**E — Long dash** (orig: pure dash 250px, žádný damage)
- Archetyp: Pure reposition
- Implementace: pohyb 3 dlaždice (long), bez damage komponenty. Vzhledem k tomu, že je to čistě
  úniková akce a Quiller má `retreatSpeedMod: 1.35`, zvažte, jestli tento dash NEMUSÍ nahrazovat
  move fázi (na rozdíl od dash-strike spellů) — designově to sedí jako bonus mobilita navíc.
  CD ≈ 7 kol.
- Pozor: tohle je přesně ten případ z úvodní diskuze, kde dash NEnahrazuje move — buďte
  konzistentní a stejné pravidlo aplikujte na Volstrov E (taky pure reposition + shield).

### Kratoma (K)

**Q — Projectile + summon pet** (orig: projectile_summon, dmg 40+0.55AD, slow, spawnuje
  Pheasant pet při dopadu, pet žije 6s)
- Archetyp: Lane shot + on-hit summon (speciální)
- Implementace: zasáhne prvního nepřítele v lane, damage + slow. Při zásahu spawne pet token
  (HP 50, AD 30) na volnou dlaždici sousedící s Kratomou. Pet žije ~3 kola nebo do smrti,
  útočí na nejbližšího nepřítele ve svém vlastním turn-order slotu. CD ≈ 7 kol.
- Pozor: pet se spawnuje **podmíněně** (jen při zásahu, ne při každém castu) — pokud projektil
  mine (žádný cíl v lane), žádný pet nevznikne. Potřebujete obecný "minion" entity typ v
  `B.units`, který je odlišný od hrdinů (menší HP/AD, žádné spelly, vlastní AI: "attack nearest").

**E — AD/AS buff + shield** (orig: buff_ad_as, 4.0s duration, +25% AD/AS, shield 40+0.3AD)
- Archetyp: Self-buff + shield
- Implementace: status "Empowered" 2 kola (bonus damage na autoattacky jako proxy za AS bonus,
  viz cross-cutting sekce), + shield 1 kolo. CD ≈ 6 kol.
- Pozor: kombinace buff + shield ve stejném spellu — ujistěte se, že shield expiruje nezávisle
  na buffu (mají různé časování v originále, i když castujete oba najednou).

### Fusilier (F)

**Q — 5-shot volley** (orig: projectile count 5, spread 0.25, dmg 25+0.2AD, "lethal at point-blank")
- Archetyp: Volley/cone, lethal at close range
- Implementace: zasáhne nepřátele v cílovém řádku, ale počet "zásahů" (a tedy celkový damage)
  se zvyšuje, pokud je cíl adjacentní (point-blank). Doporučuju: 1 zásah na dálku, 3-5 zásahů
  pokud je cíl v sousední dlaždici. CD ≈ 3 kola.
- Pozor: tohle je jediný spell, kde "point-blank" mechanika z originálu (shotgun-style, víc
  pelletů dopadne zblízka) potřebuje explicitní vzdálenostní check — bez něj se ztratí
  Fusilierova identita "blízký ranged".

**E — Cone knockback** (orig: cone_knockback radius 80, cone 90°, dmg 68+0.35AD)
- Archetyp: Pulse + knockback (self-centered, směrový kužel)
- Implementace: BURST radius 1 ve zvoleném směru (ne celý kruh kolem sebe jako ostatní pulse
  spelly — Fusilier E je směrový), damage + knockback. CD ≈ 6 kol.
- Pozor: je to jeden z mála pulse spellů, který NENÍ omnidirectional — potřebuje výběr směru
  (vpřed/vzad/do strany) jako součást cast UI, ne jen "potvrď cast".

### Volstrov (@)

**Q — Range stance** (orig: volstrov_q, 3s duration, +80 range, pierce, +50%AS, -50%MS self)
- Archetyp: Self-buff (stance s trade-off)
- Implementace: status "Overcharged" 2 kola — autoattacky během něj zasáhnou celou lane
  (piercing) a Volstrov se nemůže pohnout dál než 1 dlaždice za kolo (proxy za -50% MS). CD ≈ 7 kol.
- Pozor: tohle je jediný buff spell s **negativním** side-effektem na pohyb — nezapomeňte ho
  implementovat, jinak Volstrov ztratí svůj risk/reward (je to "zůstaň stát a maximalizuj DPS"
  hrdina podle `playStyle` komentáře v kódu).

**E — Dash + shield + CD reduction** (orig: volstrov_e, dash 60px, shield 50+0.3AP, halves Q CD)
- Archetyp: Pure reposition (micro) + shield + partial CD reset
- Implementace: pohyb 1 dlaždice (nemusí nahrazovat move, viz Quiller E), shield 1 kolo,
  `Q.cdLeft = Math.floor(Q.cdLeft / 2)`. CD ≈ 5 kol.
- Pozor: na rozdíl od Reaper E (plný reset), tady je to jen poloviční snížení — pokud je Q
  cdLeft už 0 nebo 1, `floor(x/2)` může dát 0, což je v pořádku (no-op).

---

## MAGE

### Mage (M)

**Q — Magic orb** (orig: projectile, dmg 95+0.65AP, krátký CD)
- Archetyp: Lane shot, vysoký damage/nízký CD
- Implementace: standardní lane shot. CD ≈ 2 kola (nejnižší CD ve hře — 3.9s originál).
- Pozor: bez gotchas, ale pozor na balance — je to jednoznačně nejrychleji opakovatelný
  damage spell ve hře, takže v tazích bude Mage Q dostupné téměř každé kolo.

**E — Targeted AoE** (orig: aoe radius 140, dmg 95+0.7AP, 20% slow 0.5s, "explosion at target
  location")
- Archetyp: **Targeted burst** (NE self-centered — hráč/AI vybírá libovolnou dlaždici v dosahu)
- Implementace: vybere cílovou dlaždici (kdekoliv v dosahu castu, ne jen lane), BURST radius 1
  kolem ní, damage + slow. CD ≈ 4 kola.
- Pozor: tohle je rozdíl proti Vanguard E/Ironclad E — Mage E potřebuje vlastní targeting UI
  krok (vyber dlaždici), ne jen "potvrď self-cast". Nezaměňte tyto dvě kategorie ve společném
  spell-cast handleru.

### Summoner (S)

**Q — Shadow bolt** (orig: projectile, dmg 55+0.8AP, 60% slow 1s, 1s silence)
- Archetyp: Lane shot + slow + silence
- Implementace: standardní lane shot s dvěma statusy najednou (Slowed + Silenced). CD ≈ 3 kola.
- Pozor: bez gotchas.

**E — Summon ghouls** (orig: summon count 2, dmg 45+0.65AP, žijí 8s, 15% max HP/sec self-drain)
- Archetyp: **Aggro summon** — minioni útočí na nepřátele (na rozdíl od Eggchanter E, kde
  minioni follow allies)
- Implementace: spawne 2 minion tokeny na volné sousední dlaždice. Žijí ~4 kola NEBO dokud
  neztratí HP na drain (drain ~15% max HP/kolo navíc k případnému damage od nepřátel). Každý
  ghoul má vlastní slot v turn order, AI = "attack nearest enemy". CD ≈ 6 kol.
- Pozor: self-drain znamená, že ghouli umřou sami i bez zásahu nepřítele — implementujte jako
  tick na začátku ghoulova vlastního tahu, ne jako globální timer, ať se to nerozsynchronizuje
  s turn order.

### Pyromancer (P)

**Q — Flamethrower** (orig: flamethrower channel, 3s/0.1s tick, dmg 250+1.07AP celkem rozloženo,
  cone 40°, range 160, "move freely while channeling")
- Archetyp: Channel — **lump-sum burst doporučeno** (originál explicitně dovoluje pohyb,
  takže multi-turn root by byl designová odchylka)
- Implementace: zvolí směr/cíl v kuželu (na 3-wide desce ≈ "dlaždice přímo před sebou + 1
  sousední"), jednorázový damage = agregovaná hodnota ze ~30 ticků. CD ≈ 4 kola.
- Pozor: `baseDamage:250` je extrémně vysoké číslo oproti ostatním spellům, protože v originále
  je rozložené přes 3 sekundy/30 ticků — nezapomeňte při lump-sum konverzi použít efektivní
  celkovou hodnotu (ne raw 250 jako kdyby to byl jednorázový hit, to by Pyromancera enormně
  přebalancovalo).

**E — Fire explosion** (orig: aoe_knockback radius 140, dmg 60+0.55AP)
- Archetyp: Pulse + knockback (self-centered)
- Implementace: BURST radius 1, damage + knockback. CD ≈ 5 kol.
- Pozor: bez gotchas.

### Tamer (T)

**Q — Magic sphere + mark** (orig: tamer_q, dmg 76+0.45AP, "Wolf prioritizes marked target")
- Archetyp: Lane shot + mark (vázané na pet mechaniku)
- Implementace: standardní lane shot + status "Marked". Pokud implementujete Wolfa jako
  perzistentní pet jednotku (viz E níže), jeho AI by měla preferovat marked cíl. Pokud Wolfa
  nemáte, mark efekt prostě nemá v tahovce smysl a klidně ho vynechte. CD ≈ 4 kola.
- Pozor: tohle je závislé na designovém rozhodnutí u E — implementujte E nejdřív, ať víte,
  jestli Wolf v tahovce vůbec existuje jako samostatná entita.

**E — Heal/revive Wolf** (orig: tamer_e, heal 195+0.6AP pokud žije; pokud mrtvý, 3s revive
  ritual na 50% HP, přerušitelný stunem)
- Archetyp: **Persistent pet** management (speciální — Wolf NENÍ summonovaný spellem!)
- Implementace: **Důležité:** Wolf je v originále zjevně permanentní pet existující od
  začátku zápasu (komentář v kódu: "Tamer má nejmenší HP, vlk kryje frontlinu"), ne něco, co
  vytváří Q nebo E. V tahovce budete potřebovat samostatný pet slot pro Tamera už od placement
  fáze. E pak buď healí Wolfa (pokud žije), nebo spustí "revive" stav na 2 kola (přerušitelný
  Stunned statusem), po kterém se Wolf vrátí na 50% HP. CD ≈ 6 kol.
- Pozor: tohle je jediný hrdina ve hře, který potřebuje vlastní pet entitu existující mimo
  normální spell-cast flow — promyslete si placement fázi (zabírá Wolf vlastní dlaždici od
  začátku? Má vlastní turn order slot?) dřív, než začnete implementovat ostatní pet/summon
  spelly, ať máte jednotnou architekturu pro "non-hero units".

---

## SUPPORT

### Healer (H)

**Q — Light beam** (orig: projectile, dmg 65+0.6AP, 50% slow 1.5s)
- Archetyp: Lane shot + slow
- Implementace: standardní lane shot. CD ≈ 3 kola.
- Pozor: bez gotchas.

**E — Healing wave** (orig: heal_aoe radius 200, amount 120+0.8AP)
- Archetyp: Heal pulse (self-centered, TEAM-WIDE)
- Implementace: heal radius 2 kolem Healera, všem spojencům včetně sebe. CD ≈ 4 kola.
- Pozor: stejně jako u Jiriny E — zkontrolujte `antiHeal`/Grievous Wounds na **přijímajícím**
  spojenci, ne na Healerovi.

### Cleric (C)

**Q — Healing pulse** (orig: heal_aoe radius 120, amount 64+0.65AP, `selfHealPenalty: 0.7`)
- Archetyp: Heal pulse (self-centered, BURST)
- Implementace: heal radius 1 kolem Clerica. **Self-heal penalty:** pokud je Cleric sám
  v healed oblasti (vždy je, je to self-centered), jeho vlastní heal amount se vynásobí
  0.7 — spojenci healí na 100 %, Cleric sám na 70 %. CD ≈ 3 kola.
- Pozor: tohle je jediný heal spell s rozdílným heal amountem pro castovatele vs. spojence —
  implementujte jako per-target multiplier check (`target === caster ? 0.7 : 1.0`), nesnažte
  se to řešit globálním nastavením heal funkce.

**E — Triple bolt cone** (orig: projectile count 3, spread 0.25, dmg 45+0.65AP, 1s silence)
- Archetyp: Volley/cone + silence
- Implementace: zasáhne až 3 nepřátele v cílovém řádku, damage + Silenced. CD ≈ 4 kola.
- Pozor: stejný edge case jako Lynx Q/Fusilier Q (co když je v řádku jen 1 cíl) — buďte
  konzistentní ve volbě řešení napříč všemi volley spelly.

### Eggchanter (E)

**Q — Egg throw** (orig: projectile_egg, dmg 30+0.83AP, slow, na dopadu se vejce vylíhne
  na "Hen" co healí + damaguje okolí)
- Archetyp: Targeted projectile → drobná zóna (hybrid, podobné Arson Q, ale menší rozsah)
- Implementace: zasáhne nepřítele v lane, damage + slow. Na místě dopadu vznikne malá
  healing/damage zóna (BURST radius 1) co tiká 1-2 kola — healí spojence, damaguje nepřátele
  v oblasti. CD ≈ 4 kola.
- Pozor: čísla `amount: 5.3` a `healInterval: 1.0` v originále vypadají jako malé per-tick
  hodnoty násobené nějakým externím level multiplierem, který v datech není plně vidět —
  netranslatujte 5.3 jako finální heal hodnotu, dolaďte při playtestu podle celkového dojmu,
  ne podle syrového čísla z JSON.

**E — Summon 3 chicks** (orig: summon_healers, healInterval 2, amount 5.3+0.46AP)
- Archetyp: **Follower summon** — na rozdíl od Summoner E (ghouli útočí na nepřátele), tihle
  "Chicks" sledují spojence a healí je + damagují okolní nepřátele
- Implementace: spawne 3 minion tokeny, které se každé kolo přesunou na dlaždici sousedící
  s nejbližším spojencem (ne s nejbližším nepřítelem!) a healí ho + damagují cokoliv nepřátel
  okolo. Žijí omezený počet kol nebo do smrti. CD ≈ 9 kol (nejvyšší CD ve hře).
- Pozor: AI pro tento typ summonu je **opačná** než u Summoner E — pokud implementujete
  obecnou "minion AI" třídu, potřebujete flag `followAllies: true/false`, ne univerzální
  "attack nearest enemy" pro všechny summony.

### Oracle (O)

**Q — Pull orb** (orig: projectile_pull, dmg 90+0.3AD, AoE radius 120 na dopadu, pull + stun 1.2s)
- Archetyp: Targeted/lane projektil → AoE pull (hybrid)
- Implementace: vybere cíl v lane, projektil letí, na dopadu BURST radius 1 kolem cílové
  dlaždice — všichni nepřátelé v té oblasti se přitáhnou o 1 dlaždici blíž k bodu dopadu +
  Stunned 1 kolo. CD ≈ 7 kol.
- Pozor: tohle je jediný spell, který pullá **více** jednotek najednou (na rozdíl od Jailer Q,
  co pullá jen jeden cíl) — použijte stejné řešení kolizí jako u Jailer Q (zastaví se na první
  volné dlaždici), ale aplikujte ho pro každý postižený cíl nezávisle a v deterministickém
  pořadí (např. podle vzdálenosti od bodu dopadu, nejbližší první).

**E — Shield aoe** (orig: shield_aoe radius 250, amount 64+0.5AD, duration 5.0s)
- Archetyp: Shield pulse (self-centered, TEAM-WIDE)
- Implementace: shield radius 2-3 kolem Oracle, všem spojencům včetně sebe, na 3 kola. CD ≈ 7 kol.
- Pozor: nejdelší shield duration ve hře (5s → 3 kola) — ujistěte se, že shield hodnota
  nevyprchává předčasně kvůli zaokrouhlování CD/duration konverze.

### Doctor (D)

**Q — Heal beam** (orig: heal_beam toggle, range 200, continuous tick, "Auto Uber after 5s")
- Archetyp: Channel (sustained heal) — **lump-sum nebo multi-turn stance**, vyberte si
- Implementace, varianta A (lump-sum): jednorázový heal vybranému spojenci v dosahu 2-3
  dlaždic, hodnota = agregace z více ticků. CD ≈ 4 kola.
- Implementace, varianta B (stance): status "Channeling [target]" — pokud Doctor zůstane
  v dosahu 2-3 dlaždic od stejného spojence a nepřeruší se (nehne se mimo dosah, není stunnutý),
  heal pokračuje každé kolo automaticky bez nutnosti re-cast.
- Pozor: "Auto Uber after 5s" v originále zní jako referenční easter egg (TF2 mechanika) na
  nějaký bonus efekt po delším kontinuálním channelu — v datech nemáte jasně definováno, co
  přesně "Uber" dělá, takže si tuto část buď domyslete vlastní (např. po 2-3 kolech nepřerušeného
  channelu dostane cíl krátkou neviditelnost/imunitu), nebo ji vynechte a flagujte jako
  "feature cut" v poznámkách k balance.

**E — Cone slash + shield** (orig: cone_slow_shield, dmg 60+0.2AD, cone 90°, 60% slow 1.5s,
  shield 72/2.5s)
- Archetyp: Volley/cone (směrový) + self-shield
- Implementace: BURST radius 1 ve zvoleném směru (podobně jako Fusilier E), damage + slow
  nepřátelům + shield sobě. CD ≈ 6 kol.
- Pozor: kombinuje offensive cone s defensive self-shield v jednom seslání — žádný technický
  problém, jen nezapomeňte implementovat oba efekty (je snadné při testování ověřit jen damage
  část a zapomenout na shield).

---

## Průřezová témata (engine-level)

**Nové typy entit potřebné v `B.units`.** Minimálně tři kategorie navíc k hrdinům:
- *Aggro minion* (Summoner ghouls, Kratoma pheasant) — AI: attack nearest enemy.
- *Follower minion* (Eggchanter chicks) — AI: move to nearest ally, heal + damage nearby enemies.
- *Persistent pet* (Tamer Wolf) — existuje od placement fáze, ne ze spellu; vlastní slot
  v turn orderu po celou hru, ne jen po dobu trvání.

Doporučuju jednotnou strukturu `{ kind: 'minion'|'pet', ownerId, ai: 'aggro'|'follow', hp,
ad, ... }` spíš než tři oddělené datové typy — usnadní vám to turn order a render logiku.

**Zone efekty mimo unit/self-pulse.** Arson E (smoke bomb) a okrajově Eggchanter Q (hen) jsou
jediné spelly, co vytváří efekt vázaný na **dlaždice**, ne na jednotku. Potřebujete pole
podobné vašemu existujícímu `bombs[]`/`shieldBursts[]`, např. `zones[]`, kontrolované na
začátku každého kola pro všechny jednotky stojící v dané oblasti — včetně těch, co se do ní
přesunuly mezi koly (na rozdíl od jednorázové AoE, zóna "chytá" i pozdější příchozí).

**Attack speed (`aaScale`, `attackDelay`) se ztrácí při 1 autoattack/kolo.** Hrdinové jako
Quiller (`aaScale:0.75`) nebo Fusilier (`aaScale:0.7`) jsou v originále identitně postavení
na rychlosti útoku, kterou tahovka nativně nepodporuje. Doporučení: hrdinům s `attackDelay < 0.9`
dejte buď (a) druhý autoattack za kolo, nebo (b) flat damage bonus na základní útok úměrný
rozdílu oproti průměru (`attackDelay ≈ 1.0-1.4` u většiny ostatních). Bez tohoto úpravu budou
všichni "auto-attack carry" hrdinové (Quiller, Fusilier, Kratoma, Hana po Q buffu) mechanicky
nerozeznatelní od mage/tanků s nízkým `aaScale`.

**`cdLeft` lock-before-animation invariant platí i pro multi-hit spelly.** Váš existující
princip (zamknout CD před spuštěním animace, ne po jejím dokončení) je obzvlášť důležitý
u Wanderer E (5 postupných hitů) a Wanderer Q (8 ticků agregovaných do lump-sum) — pokud by
animace běžela přes víc "kroků" a CD se zamykal až po posledním z nich, hrozí race condition
při rychlém opakovaném inputu.

**Self-centered vs. targeted AoE — shrnutí pro implementaci.** Self-centered (Vanguard E,
Ironclad E, Jailer E, Pyromancer E, Jirina Q, Zephyr E, Doctor E, Fusilier E) nepotřebují žádný
targeting krok — jen confirm cast. Targeted (Mage E, Oracle Q, Arson Q, Eggchanter Q) potřebují
výběr cílové dlaždice/jednotky před resolve. Doporučuju mít ve spell datech explicitní flag
`targeting: 'self' | 'tile' | 'enemy_unit' | 'lane'`, ne odvozovat to z `type` stringu — `type:
'aoe'` se totiž používá pro obě varianty (Vanguard E je self, Mage E je tile-targeted) a to je
přesně ten typ chyby, která se objeví až při testování konkrétního hrdiny.
