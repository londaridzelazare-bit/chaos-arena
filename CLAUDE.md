# Chaos Arena — project brief for Claude Code

This project was built in a long claude.ai chat and moved here so it can be deployed,
tested locally and developed further. Read this whole file before changing anything.

## What the game is

**Chaos Arena** is an online 3D team deathmatch in the browser (think Krunker, but with
absurd super-powers). Two teams (Blue vs Red), up to ~10 players, first team to 50
knockouts wins. Everyone has 100 HP, respawns after 4 s at their team's side.

- Third-person characters: WASD, mouse look (pointer lock), Space jump + double jump,
  Shift = "Naruto run", Shift×2 = dash. Click with no skill selected = energy blast (`ki`).
- 31 skills (keys shown on the Skills bar): nukes, black hole, tornado, volcano, lightning,
  giant milk cow, giant frog, knife rain, Satan, sword slash, supernova, vacuum decay,
  One Punch / Serious Punch, freeze/slow time, upside-down world, God's judgement, etc.
  Each has a cooldown in PvP (`CD` table). Skills fuse into combos via an element system
  (`ELEM`, `SPECIAL_ELEM`, `comboAfterCast`, `infuse`).
- Procedural heightfield terrain that never changes shape (blasts only scorch it), destructible trees/rocks/ruins/logs/stumps,
  endless decorative countryside outside a 140×140 m play area.
- Stylized look: toon shading, black inverted-hull outlines, gradient sky, clouds.

## Files

| File | What it is |
|---|---|
| `index.html` | **The game (Chaos Arena).** One self-contained HTML file. This is what gets deployed. |
| `egg-defense.html` | The earlier single-player/hotseat "Egg Defense" version it grew out of (build a fort, then attack the egg). Kept for reference only. |
| `CLAUDE.md` | This brief. |

There is no build step and no package.json. Everything is inline JS in `index.html`.
External libraries load from cdnjs: three.js r128 (UMD) and cannon.js 0.6.2 (UMD).
PeerJS 1.5.4 is inlined in a `<script>` block near the top.

## Architecture (inside index.html)

- Rendering: three.js r128, `MeshToonMaterial` via `M3(color)`; outlines via `addOutline()`
  and `outlineSweep()`; sky shader `skyMat`; `updateVisuals()`.
- Physics: cannon.js 0.6.2. Terrain is a `CANNON.Heightfield` (`hfBody`, collision group 8).
  Characters and skeletons do **not** collide with the heightfield; they walk on the surface
  via `groundH(x,z)` + `snapWalker()` (this fixed characters snagging on triangle edges).
- Terrain: `genTerrain(seed)`, `groundH`, `syncTerrain`. `craterAt` now only scorches (`scorchAt`, colour only).
  Destructible props live in `props` (`damageProps`, `destroyProp`, `eraseProps`).
- Game state lives in the global `G`. Players live in `chars` (keyed by player id).
  `attacker()` returns the current *caster* key (`G.casterKey || G.localKey` in PvP).
  `activeChar()` / `localP()` return the local player.
- Skills: `runSkill(id, p)` is the single entry point (used for local casts and for replaying
  other players' casts). `localCast()` runs it locally, sets the cooldown and sends a `cast`
  message. `replayCast()` runs another player's cast with `G.casterKey` and `G.yaw` swapped.
- Damage: `hurt(P, dmg)` only ever damages the **local** player (each client is authoritative
  for its own HP). Called from `blastAt`, projectile hits (`stunPlayer`), punches, sword,
  tornado, black hole, etc. Death -> `die()` -> `death` message -> host scores it.
- Summons (tornado, skeletons, Satan, volcano lava, milk cow, judgement) chase the nearest
  enemy via `eggTarget(x, z, ownerTeam)` -> `nearestEnemy()`. Friendly fire is ON for blasts.

## Game modes

`G.mode` is `'egg'` (shown to players as **Alpha Mode**, the default) or `'chaos'` (team deathmatch). Pain mode and
Swarm mode were removed from the title screen on 2026-10-06 (`normMode` maps anything else to 'egg'); their code is
still there (`isPain()` is now always false, Swarm is unreachable). In Alpha Mode the Chaos (native) and Summoner
classes are hidden (`classCycle`); `startMatch` moves you to the first allowed class.
Picked on the title screen (`ME.mode`, localStorage key `ca-mode2`); the host's choice travels as `NET.mode` in
`lobby`/`start` messages and can be switched in the lobby. `startMatch(seed, list, score, mode, cdOn, eg)`.

**Giant Boot** (`bootKick`, next to `keepInBounds`): the local player touching the map edge, or an attacker touching
the egg ring in Alpha Mode, gets kicked back 30 m by a boot that swings in from behind (`P.bootSpin` flips the rig).

**Egg mode** (module "EGG MODE (online)", right before the Pain module): the old hotseat Egg Defense,
online. Each round the host picks the next defender (`NET.eggOrder`, everyone defends once, late
joiners are appended). The defender is team blue and spawns by the egg; everyone else is red.
`G.phase` is really `'build'` (NO time limit since 2026-10-06: `E.endT` stays Infinity; the defender presses
Ready (F / button) -> `edone`) then `'attack'` (no time limit either: it ends when the egg breaks or every attacker
presses Give up -> `egu` -> host `eggGaveUp` -> `eggRoundOver('def','gaveup')`; attackers get the classes; the
defender can only shoot ki and grab). Attackers can't enter the 12.4 m ring. State lives in `G.egg`
(`def, round, total, pts, ph, endT, over, left`). Protection powers are casts `d_use` / `d_place`
(`runDefense`, `placeDefense`), validated on the host by `eggDefOK` against `G.egg.left`. The egg
is host-authoritative: only the host's egg is dynamic and can break (`breakEgg` returns on clients
unless `G.eggNetBreak`); the host sends `eg` (pose, cracks, angel lives) at 10 Hz and clients' kinematic
copies follow (`eggNetState`/`eggFrame`). Messages: `eph` (to attack), `eover` (round result + points;
`w` = 'atk' | 'def' | 'none'), `eend` (final standings), `edone`. Points: egg broken = every attacker +1,
egg held = defender + number of attackers. Field defenses that hit players (tesla, mortar, fans,
mirror) use `eggFoe()`: the local player only if they're attacking (victim-side, like all damage).
Practice alone in Egg mode: build, Ready; Y switches between defending and attacking; Give up restarts the round.
The egg breaking plays `eggBreakFX` (shake + light through cracks, burst, top half flies, yolk puddle, a chick runs off)
with the slow-motion egg camera (`G.eggCam`, `G.slowmo`).

**Pain class** (module "PAIN MODE" in index.html, right before `runSkill`; rebuilt 2026-10-06 to the user's list):
ids `p_*`, dispatched by `runPain()`. Click (hold) = **Telekinesis** (`TK`: `tkDown`/`tkUp`/`tkRight`; hidden casts
`p_grab` {tg} and `p_throw` {c}, c = -1 drops). Grab targets: 'P'+player id (victim side, `tkForces`), 'R'+prop index,
'K'+core index, 'O'+x,y,z (nearest body there on each screen). Right-click held while holding = throw charge (no limit).
1/2 Gravity From Below/Above (`kind:'aimcharge'`: hold shows the marker and charges, release casts at it; radius `gravR`,
force `shinraKick`), R/T Shinra Tensei front/360 (charge), F Levitation (`painLevitate`), Q Pull Everything (charge,
`pullEverything(..., m)`), X Planetary Orbit (14 rocks + nearby debris; sets `G.targeting='p_orbit'` so the marker shows;
Click or X fires `{fire:1}`), C Earth Titan (`kind:'hold'`, `BOULD`: `p_boulder` at the marker starts the rise,
release sends `p_boulderx` {c} -> rock size, it flies 38 m up and follows the marker (`pb` in the state message), Click
sends `p_boulderx` {drop:1}), V Gravity Vortex (`painVortex`). Ultimates (generic `ult:true`, see Bomber): B Almighty
Push (rises 28 m, a dome of warped space comes down and sweeps outward to 60 m), N Catastrophic Chibaku.
Chibaku cores: every push/pull/lift/slam/vortex/boulder kicks them (`kickCores`), Telekinesis can grab and throw them,
and two cores that touch merge (`coresMerge`). Shift x2 warp dash and Space x2 gravity jump remain (hidden from the bar).
Removed: Chakra Rod, Gravity Push, plain Chibaku Tensei (`PAIN.rods`/`updateRods` stay for other code).
While charging anything, walking sideways/backward turns the character the way it walks (`controlPlayer` facing).
Shared systems: GravityForce (`radialPush`, `conePush`, `knock`), gravity fields (`PAIN.fields`, applied every physics
step by `applyPainForces()`), PhysicsObjectAttractor (`liftProp`, `regrowProps`, `makeCore`/`attractorField`/
`stickToCore`/`collapseCore`), timed effects in `PAIN.fx`, and `SpaceWarp` (screen-space lens; `warpPulse`).
- Damage/knockback stay victim-side (`victimFor`, `painHurt`, no friendly fire). `castOK()` (host) validates casts.
Pain changes 2026-10-06 (second pass, user's list): Telekinesis = RIGHT-click grabs/drops (`tkGrabToggle`), hold
LEFT-click charges the throw (`tkDown`/`tkUp`, meter in `tkThrownFrame`). Gravity from below/above: the key only selects
the ring; hold Click charges (`chargeStart(id,'mouse')` from `actionDown`), release casts; the ring stays selected.
Earth Titan renamed Bansho Ten’in; its impact leaves real rubble (`rubbleRock`/`rubbleBurst`, objs kind 'rubble', newest
90 kept) that every power moves. Levitation (F) and the new Gravity Shield (X, max 5 s, `painShield`: bounces bombs,
shots, rods, thrown things in random directions) are `kind:'holdkey'`: key down casts, key up casts `<id>x`
(`PAIN.holdKeys`). Shinra Tensei (R) carries the camera direction (`x.d`, `aimDir3`) and pushes in 3D. Planetary Orbit
was folded into the Gravity Vortex (V): rocks torn from the ground + everything around orbit you (`PAIN.vortex`, up to
20 s), V again or Click hurls it all at the crosshair. Every class's Skills bar puts ultimates last.
Pain third pass (2026-10-06): Gravity Jump 8.5 m/s (was 17); Gravity From Below never launches its caster. Gravity ring
charges show a terrain-hugging ring (`groundRing`, drawn through hills) with no size limit and the camera backs off
(`G.camZoomTgt`). Bansho Ten’in (C) is repeatable: every press tears a rock up (hold = bigger) and parks it in the sky
(`BOULD[key].rocks`, max `BOULD_MAX` 40); a ground ring at the crosshair shows the landing area; Click rains every rock
in the sky down over it (sunflower spread, `boulderSlot`). Levitation's area follows you and grows while F is held
(`L.R`, visible ring + faint wall); levitating bodies are tagged `_levT` and the Gravity Vortex pulls them in from any
distance (`_vxT` marks the vortex's bodies so Levitation lets go). The vortex no longer makes dust.
X is now Gravity Link (`p_link`/`p_linkx`, `painLink`, `PAIN.links`): X locks onto a loose body, a prop (lifted with
`liftProp`), a Chibaku core or one of your Bansho rocks in the sky (`linkPick` ids K/O/R/B, `linkResolve`); it hovers,
the far end follows your crosshair (local), X again or Click hurls it straight along the link (`painLinkHurl`,
`linkArrive`: Bansho rocks use `boulderImpact`, cores slam down, bodies hit like a boulder). The Gravity Shield code
(`painShield`) is still there but off the bar.
G Falling Circuit (`p_circ`, follow-ups `p_circgo` run / `p_circx` break; `PAIN.circ[key]`): G marks points in the air
(`circuitAim`), aiming back near the first point closes the loop (`circuitKey` sends exactly pts[0]); starting catches
everything within `CIRC_CATCH` of the first point (bodies, props via liftProp, the local victim) and a field drives each
item point to point, x1.25 speed per point; open circuits throw them off the last point, G/Click breaks a closed loop
(`circuitLaunch`, damage via TK.thrown / painHurt). Ultimates are now four: B Almighty Push, N Catastrophic Chibaku,
(World Wring, `painWring`, was removed from the bar at the user's request; its code is unused) and Z Zero Gravity
(`painZeroG`, `PAIN.zeroG`, ZEROG_T 30 s; every loose body outside the defense plus the ZEROG_PROPS 100 nearest props float
(`Z.bodies`, no gravity, so thrown things keep flying; rescanned each second);  the caster gets P.carpet type 'zerog' = free 3D flight (WASD along the view, Space
up), everyone else (not defenders) gets type 'float'; dummies float; objects and the defense are untouched; star field
`zeroGStars`). Click priority in `tkDown`: wring release, circuit run/break, link hurl, Bansho drop, telekinesis.
Camera (2026-10-06): PITCH_MAX 1.38 (look almost straight up; the boom shortens as you look up), obstacle pull-in eases
(2e-6/s instead of a one-frame snap), the ground lift eases (`CAM.lift`) with a hard floor at ground + .25.
Performance (2026-10-06): all `fx` particles draw through shared InstancedMeshes (`fxInstAdd`, one per geometry +
lit/unlit; the particle's own mesh is detached), `puff`/`sparkle` share materials (`pmat`), Babylon gates share ring
geometry/materials (`babRing`, `babMat`), every Bansho rock is one merged mesh (`boulderGeo`), and `propCull` hides prop
outlines past `OUTLINE_FAR` (55 m; outlines were half of all draw calls). Measured: ~25-30% fewer draw calls.

## Networking (peer-to-peer, no server)

- PeerJS over WebRTC using the free public PeerJS broker. The **host's browser** is the hub.
- Host peer id = `chaosarena-v1-<5-letter code>`; joiners connect to it (`hostGame()`,
  `joinGame(code)`). Host relays every message to everyone else (`netBroadcast`).
- Messages (`hostHandle` / `clientHandle`):
  - `hello` {name, team, look} → host adds player, sends `lobby`
  - `lobby` {players}
  - `start` {seed, players, score} → everyone runs `startMatch(seed, players)`
    (same seed ⇒ same map/terrain on every screen)
  - `join` / `left` for mid-match joins/leaves
  - `st` player state, 15 Hz: position, facing, velocity, flags (naruto/grounded/dash/stun), hp, alive
  - `cast` {id, s: skill, p: target, y: yaw, o: caster pos} → replayed on every screen
  - `death` {id, by} → host increments score, broadcasts `kf` (kill feed) + `score`, and `end` at 50
- Cosmetic chaos (debris, smoke, flying logs, randomness inside skills) is simulated locally
  on each screen and is allowed to differ.

## Classes (all modes)

`G.cls` is the local player's class: 'native' (the mode's own kit: Chaos `SPECIALS` or `PAIN_SKILLS`) or
'bomber'. Tab, or the row of class tabs on top of the Skills bar (`addClassButton` builds `.classTabs`; tabs
switch on pointerdown because the bar redraws right after), swaps it any time; while tabs show,
`body:has(.classTabs)` lifts `#hpBar` and `#micB` so they don't cover them;
it's remembered in localStorage `ca-cls`. `skillTable()` follows the class; `modeTable()` is always the
mode's kit (used for `G.specials`). Cooldowns live per power id in `G.cd` and the host's `castLog`, so
swapping never resets them. The host accepts a cast if the id is in the mode kit or any class table
(`BOMBER[m.s] || modeTable()[m.s]`). To add a class: a table like `BOMBER` (name, e, key, kind:
click/aim/throw/target/now, cd, range, r, lim, hide), add its id to `CLASS_IDS` and `classInfo`, route
its keys in the keydown handler and its click in `actionDown`, dispatch it in `runSkill`.

**Bomber** (module "BOMBER", before "casting, locally and for other players"): 10 powers + 2 ultimates, `b_*` ids
(rebuilt 2026-10-06 to the user's list). Click = bazooka; 1 B2 bomber, 2 ballistic missile, 3 airstrike (8 jets from 8
directions), 4 cluster bomb, 5 landmines (max 10, fixed 1 s cooldown), 6 guided missile, 7 grenade (ammo `G.nades`, 10 at
start, max 20, +5 from loot), 8 tank drop (`bTank`, `tankMesh`: lands on parachutes between you and the target, a static
box body while it fights, fires 3 shells at whoever is nearest the target, then sinks), 9 rocket squad (`bSquad`,
`soldierMesh`: 5 soldiers for 10 s, each shoots the nearest enemy or the target, advances if far); ultimates B Carpet
Bombing (`bCarpet`: 20 heavy bombers in a row, 6 bombs each over a 60 m band; `heavyBomberMesh` is a cached, merged
template) and N Nuclear Strike (a Fat Man `nukeMesh` dropped from a heavy bomber crossing over the mark).
Ballistic missile: `ICBM_SHOTS` = 5 shots per trip into the sky view, then targeting ends (`castAt`, `SKY.shots`).
Grenade: hold 7 (`nadeStart`/`bomberKeyUp`), speed grows with the hold up to `NADE_VMAX`; dots along the arc, a ring
and the distance where it lands (`nadeAimFrame`, `nadePath`); the velocity travels with the cast (`x.v`).
Flying bombs are plain objects in `BX.bombs` (`addBomb`/`updateBombs`: gravity, drag, `bounce` for grenades, hits on
ground/dome/enemies/egg/props via `bSolid`, animals and loose things via `wildSegHit`), scripted things in `BX.fx`
({update(dt) -> false when done}), plus `BX.mines` and `BX.guided`. Explosions go through `bBoom`. Spreads are seeded
from the cast point (`ci.rng`). **Sky aim** (`kind:'sky'`: ballistic missile, nuke): while targeting, `skyCamera` lifts the
camera over the target (camera.up = facing) and mouse movement goes to `skyLook` instead of `look`. **Guided missile**:
the caster's screen steers it toward `aimPoint` and sends its position as `gm` in the 15 Hz `st` message (`guidedState` /
`guidedNet`); the hit is the hidden cast `b_guidex`. Clicking (bazooka) or pressing 6 again detonates it.
Old powers (nuke, doom nuke, fire bomb, bettys, boomerang, matryoshka, cloud, reverse, carpet, cracker, mortar, bowling,
magnet, cargo, big red button) were removed; `flyCarpet` stays (sword surf, eagle, steed, lich use it).

**Ultimates (generic)**: any power with `ult:true` charges for `ULT_CHARGE` (30 s) from the round start (`G.ultT0`, set by
`ultReset()` in `startMatch`; recharge potions add `G.ultBonus`) and works once per round (`G.ultUsed`). `cdFor` returns 0
for them; `castOK` enforces charge + once per round on the host (`G.hostUlt`). The Skills bar shows the charge seconds,
then glows (`.ultready`), then USED. Powers with `fixed:true` keep their cooldown even when cooldowns are off.

**Swordsman** (module "SWORDSMAN", after the Bomber; rebuilt 2026-10-06 to the user's list): `s_*` ids in `SWORD`.
Second pass (2026-10-06, supersedes the older notes below where they differ): 4 is now "Radial Slash" (id still
`s_titan`). 7 Mihawk's Plunge was replaced by Blade Dash (`s_bdash`, `sBladeDash`: 10 swords posed on the rig
(`BDASH_POSES`), the rig spins around its middle and rolls 30 m forward; `TELEPORTS.s_bdash`). Gate of Babylon: up to
100 swords (`babylonN`, ~5 s), then every 5 s of holding upgrades the swords instead (`babylonTier(c)`: giant, black
`swordBlackGeo`, 4D `babylonSword` with additive ghost copies) and at tier 4 the secret `babylonSecret`: 240 gates on a
hemisphere around the egg/dome (or the aim point) open and all fire at once, then a `goldBurst` finale. World Severance is
aimed now (`kind:'aim'`, line preview `SW.sevLine` from the marker along your facing): a flaming sword plunges in and
ploughs a burning trench (`fireStrip`, `craterAt`) to `severLen`; the old map split (`severWorld`, `G.splits`) is unused.
Excalibur's wave runs to the defense circle (`excalLen`: AREA_R or the dome, Alpha Mode) or the map edge and ends in
`goldBurst` + bBoom. Splitter: 6 switches upright/flat while aiming; `cutIndicator` draws a terrain line, an upright
blade + pole or a flat sheet + rim, and a label, all depthTest-off.
Click = six combos that run in order and loop (`SW_COMBOS`: Crescent, Whirlwind, Rising Dragon, Piercing Fang, Cross
Cut, Storm Flurry; `SW.set`/`SW.step`, cast `s_slash` {k: set*10 + step}; arm poses 0-5 in `swordAnim`). 1 Heaven's
Blade (sinks, then shatters), 2 Rain of Swords (magic cloud, merged sword meshes `swordOne`), 3 Gate of Babylon (hold,
no limit: `babylonN`, local gate preview while charging, camera pulls back via `G.camZoomUntil`/`G.camZoomK`),
4 Titan Slash (flat giant sword, one full turn, trail via drawRange), 5 Chain Cut (hold to count to 100, `chainCount`;
blinks between up to 12 targets), 6 Splitter (`kind:'cut'`: `cutIndicator` shows the line, 6 again toggles standing/flat
`SW.cutMode`; `sCut` splits props, objects and dummies into two physics halves with clipping planes (`splitMesh`,
`piecesFrame` keeps each plane on its piece; `renderer.localClippingEnabled`), kills animals/Swarm, 150 to players on
the line), 7 Mihawk's Plunge (`yoruMesh`, `C.fly`, `P.sitT` sitting pose), 8 Astral Sword (`C.sw.astral`, guards and
attacks for 25 s, cuts projectiles near you), 9 Sword Trap (cages up to 6 enemies inside the circle, `SW.traps`).
Ultimates: B World Severance (the giant sword sweeps along the line and `severWorld` really cuts the map: halves slide
apart around a bottomless void, `G.splits`/`inVoid`/`voidSteer`; falling in kills), N Excalibur (`sExcal`: light pillar,
then a 130 m golden wave). Removed: radial, flame/lightning swords, quickdraw, anime finisher, moon cleaver, orbiting
blades, prison, mirrors, surfing, black hole blade, execution line, perfect counter, storm, world splitter.

**Summoner** (module "SUMMONER", after the Swordsman): `u_*` ids in `SUMMON`; click = Command
(`u_cmd`: every summon you own uses its signature move at the crosshair, `cmdPoint`), 20 animals on
1-0 / Q R F G T Z X C V / B. Each summon is an entity in `SUM.list` (`makeSummon`, its own `tick`
AI and `anim`), simulated on every screen; helpers `eTarget`, `eMove` (respects the void), `eArea` /
`eCone` (victim-side), `grab` (holds the local player via `P.held` in `controlPlayer`, or a dummy via
`SUM.held`), `fling`, `lashOut` (tongues/trunk), `dartTo` (BX bombs). Max 2 big summons per player.
Peacock hypnosis = `P.hypno` (steers `controlPlayer`); octopus ink = `G.blindT` (#blindFx overlay) and
`G.hiddenUntil`; chameleon also hides you; hidden = 'st' flag bit 32 (remote root hidden, finder arrow
skips them) until a bat's echolocation (`G.revealT`, marks in `SUM.marks`). The turtle is a kinematic
deck with `platformVel` (`groundCheck` -> `P.platformV` carries riders); the eagle is a typed carpet
('eagle', 20 m/s, 9 m up; let go = hidden cast `u_eaglex`).

**Necromancer** (module "NECROMANCER"): `n_*` ids in `NECRO_SK`, click = homing Hellfire Skull, ultimate
Nine Circles on B (once per round, digs real terraces). Corpses in `NECRO.corpses` (player deaths via
`necroOwnerDied`; practice dummies after 100 damage via `dummyDamage`). Curses: `NECRO.possessed`,
`NECRO.debts` (`necroOnCast` in localCast/replayCast). Minions reuse the Summoner entities (`necroMinion`).
BX bombs support homing (`home`, `turn`).

**Summons can die**: `e.hp`/`SUMMON_HP`, hooks in `bBoom`, `swordArea`, `swordHit`, `blastAt`, `radialPush`,
`conePush` (`summonsTakeArea/Cone`); the owner's screen decides death and sends hidden `u_kill`. Enemy
summons are in `bEnemies` (ids start with 'S'), so projectiles hit them and summons fight each other.
Known: duoall reported one host rejection of `n_furnace` aimed at a player 10 m away; not yet investigated.

**Voice chat** (module "VOICE CHAT"): P or the always-visible #micB button toggles the mic (`toggleMic`).
While on, you PeerJS-call every player in `NET.players` with your mic stream (`voiceCallAll`, retried every
2 s for late joiners); everyone answers every call (`voiceAttach` on each Peer), so you hear others with your
mic off. Mute = disable the track. Talking indicators use analysers on the game's AudioContext (`voiceCtx`):
#talkers list and a 🔊 sprite over the speaker. Players stuck on the MQTT backup relay can't get voice.
P used to be Chaos's World-ending nuke; that moved to '-'.

**Pain class**: the Pain powers are also a class ('pain', first after the default in the Tab cycle) in every
mode. `painKit()` = Pain class, or the default kit in Pain mode; input, dash, air jump, help and charge
release use it. `painFrame` and `applyPainForces` run in every mode. Map rules (indestructible terrain, sky)
stay tied to Pain mode (`isPain()`). In Pain mode the class is skipped (`classCycle`).

Egg mode follows the cooldown setting like the other modes (it used to force cooldowns on).

The Skills bar fills row by row and is sorted by key: click, 1-9, 0, then letters A-Z.

## Swarm mode (single-player)

Module "SWARM MODE" (before "casting, locally and for other players"). Mode id 'swarm', practice only
(host button disabled; `normMode` still maps online modes). Native kit = Chaos `SPECIALS`, all class tabs.
`swarmSpawn(n)` (panel buttons 5/10/20/50/100, Alt+1-5; Alt+0 clear; Alt+G invincible) queues enemies on
a ring 20-32 m out; types in `SWARM_TYPES` (grunt / runner / brute). Each enemy: one cannon sphere
(group 16, ignores other enemies, material PM.char, tag 'dummy'), a record `e.rec` pushed into
`PAIN.dummies` + `bodyObj` so every power that hits practice dummies hits them (`dummyHit` routes to
`swarmHurt`). Also `summonsTakeArea/Cone` -> `swarmTakeArea/Cone`, `swarmBodyHurt` next to every
`if (b.player) { hurt(...)` force-field damage, hard velocity changes (>9 m/s jolt) and fast flight.
Same-frame damage through two channels counts once (`fa`/`fh`). AI: flow field (Dijkstra on the 70x70
2 m grid from the player, `swarmBlockMap` marks props/barriers/heavy objects), straight line when clear
and within 9 m, separation via cell buckets, windup -> strike (`swarmStrike`, `painHurt`). Rendering is
all InstancedMesh (`SWARM.im`: body/head/arm/leg + outlines, eyes, mouth, billboard health bars, death
voxels) — instance colours must be created before `count` is lowered. Damage numbers are a pooled sprite
set with cached textures. Test: scratch `swarm.mjs` (100 enemies, every class, combos, rain, severance).

## World: fog, rain, pixel look, nature

Module "WORLD". `FOG` near/far (40/235). Rain (`RAIN`, ` key or 🌧️ button, local setting `ca-rain`):
line streaks around the camera, splash rings, rain noise loop, `rainLight` scales whatever light/fog the
game last set, `updateVisuals` tints sky/fog/clouds by `RAIN.k`. Pixel look (`PIXEL`, localStorage `ca-pixel2`, on by
default; title button reloads): renderer without antialias at pixel ratio 1/`pixelScale()` (3-6) + CSS `image-rendering:
pixelated`, `pixelGrain` (blocky texel noise, GRAIN_D cells per metre, GRAIN_A strength) and a posterized palette
(14 levels per channel) on toon materials, voxel particles (`UNIT` boxes), `perfTune` off. Nature: trees are single merged
meshes (`treeGeo` pine/oak/birch/dead, `NAT_MAT` vertex colours), `moreNature` adds logs, stumps, boulders and
instanced `DECOR` (bushes, branches, pebbles, mushrooms; `decorDamage/Erase`, rustle). Fire
(`igniteAt` from fire blasts and `groundFire`) burns trees/logs/stumps/plants and spreads (`BURN`).

**Map size**: `ISL` = 140 (280 x 280 m; it was 70). `MAPK` = area factor (4) scales prop, decor, animal and bird counts.
Terrain grid `TN = ISL + 1`; biome colours in `terrainColor`, wide hills further out in `genTerrain`. Team spawns at
x = ±0.48·ISL.

## WILD (module "WILD", before "COMBO BUILDER")

- `wildBuild(place)` runs first in `buildRound` (before forests, so there is room): arches, broken towers, villages (huts
  round a well, fences, crates, loot), stone circles, statues, watchtowers, carts, broken walls. All are props made by
  `wildBlock` (one static box, mesh group merged by `mergeToon` into one vertex-coloured mesh, `lift` for lintels).
- Loose physics things (`wildLoose`, kind 'item' so E picks them up): crates (break), hay, red explosive barrels
  (`xbarrelBoom`, chain reactions; `bBoom` with no owner hurts everyone). Hits arrive through `wildBlastHit` (from
  `blastAt`, bodies flagged `.wild`), `wildSegHit` (bombs) and velocity jolts.
- Loot barrels (`WILD.loot`, kind 'loot'): E (`nearLoot`/`openLoot`) or any hit opens one; the potion (hp +50, recharge:
  all cooldowns 0 + ultimates 10 s sooner, or +5 grenades) is taken by walking into it -> hidden cast `w_loot` {i, g}
  (castOK accepts it early); barrels return after 45 s (`gen` increments, contents from `lootRoll(seed, i, gen)`).
- Animals (`ANIMALS`: rabbit, deer, fox; `spawnAnimal` in `wildStart()` at the end of `startMatch`): cannon spheres that
  skip terrain contacts (snapped to `groundH`), tag 'dummy', records in `PAIN.dummies` + `bodyObj` with `.animal`, so
  `dummyHit`, `swarmBodyHurt`, `summonsTakeArea` (-> `animalsTakeArea`) and jolts hurt them. `bEnemies` and the pull
  lock-on skip them. They graze/walk/flee (players, explosions via `wildScare` from `boomVisual`), are hidden and
  frozen beyond 110 m, die into a flopping corpse, respawn after 30 s.
- Birds (`BIRDS`, two InstancedMeshes in the scene, not the round): flocks circling 20-42 m up; `birdsTakeArea` from
  `blastAt` drops them; they come back after 40 s.

## Performance with the big map (2026-10-06)

- `HashBroadphase` replaces SAP: static bodies in an 8 m spatial hash, movers test only their cells (+ huge statics and
  other movers). `world.addBody/removeBody` mark it dirty. Sparse `ObjectCollisionMatrix` instead of the dense one
  (that was N²/2 entries cleared every step). Physics went 13.8 -> ~1.4 ms per step.
- `propCull` (every 8 frames) hides props by distance (100/150/215 m by size). `destroyProp`/`liftProp` unhide.
- Headless Chrome on the user's machine: ~70 FPS (old 140 m map: ~200).

## Camera (orbit, third-person RPG style)

In the frame loop's camera block (`CAM`): yaw/pitch are applied directly, so the camera always circles the
character (no position lerp that cut across on fast turns). Pivot height is damped (`CAM.py`), big moves
(> 6 m: respawn, teleport) cut. `cameraBoom` now only *returns* the free boom length; `CAM.len` eases toward
it (pulls in fast, lets out slowly), so obstacles and zoom never snap. `PITCH_MAX` is .62 (looking further
up put the camera under the ground) and looking up shortens the boom. Mouse: `lookHold` ignores movement for
160 ms after the lock is gained, the window regains focus or the tab comes back (Chrome sends stale jumps),
events are clamped to ±400 px, and the free-cursor fallback only turns at the left/right edges (resting the
cursor at the top used to tilt the camera up forever). Test: scratch `cam.mjs`.

## Combo builder (Chaos kit)

Module "COMBO BUILDER". The user wants combos built *before* casting: press a power (armed, `G.targeting`),
press more powers and they join `CB.build` instead of replacing it (`comboPress`, called at the top of the
Chaos branch of `useSpecial`, so keys and Skills-bar clicks both work). Compatible = brings an element
(`SPECIAL_ELEM`) the combo doesn't have, max `COMBO_MAX` (4); pressing a member removes it; right-click
cancels (comboFrame clears `CB.build` when `G.targeting` is gone). Instant powers fire at once when nothing
is armed and join a combo when something is. Click -> `castAt` -> `comboFire`: every power is cast at the
marker (carriers first), then `comboFuse` adds every element to every new carrier for 3 s (late carriers
too). `#comboBar` shows [icon] + [icon] + ?, the combo name and "Can combine"; `.combo-ok` pulses on the
Skills bar. Test: scratch `combo.mjs` (real key presses).

## Defender kit (Alpha Mode) — rebuilt 2026-10-06

Roles (2026-10-06): up to `MAX_DEFS` (2) players defend at once (`G.egg.defs`, `isDef(id)`, `defIds()`; `G.egg.def` is
just the first defender, for names). Anyone switches any time with Y or the button (`eggSwitchRole` -> host `eggRole`,
message `erole`; team blue = defending, red = attacking). Building ends when every defender pressed Ready (`eggReady`,
`edone` -> host, `erdy` to everyone; `eggCheckReady` also starts the fight when nobody defends). Things the fort makes
belong to `FORT_OWNER` ('fort', team blue), so every defender is safe from them; "not the defender's" checks use
`!isDef(owner)`. Defender points go to every defender. The egg ignores gravity powers while the angel, devil or
sorceress lives (`eggGuarded`, `gravityProof(body)` in the Pain loops that touch bodies directly).

Everything the defender summons can be hurt and shows a health bar (2026-10-06): `fortUnits()` lists drones (hp 60),
spirits (`SPIRIT_HP`: angel, devil, dragon, sorceress), golems, swarm minions and devices; `fortUnitsTakeArea` (called
from `summonsTakeArea` and `summonsTakeCone`) damages drones/spirits, `fortUnitHurt` kills them; `fortBarsFrame` keeps a
bar over each; bombs collide with them in `updateBombs`. Anything aimed (marker/ring casts, `teleRetarget` in
`localCast`) into the teleport portal is re-aimed at the caster (`x.tp`, `teleSwallowFX`), and `G.teleBackUntil` lets
the caster's own power hurt them for a few seconds (`victimFor`). Bomber's rocket squad parachutes from a plane
(`paratrooper`) and in Alpha Mode only targets the egg and the defender's units (`fortTargetNear`). Alpha Mode classes:
Pain, Bomber, Swordsman (no Chaos, Summoner, Necromancer).

The old build-phase protection bar (DEFENSES) and the attack-phase GUARD kit were replaced by ONE kit the defender uses
in both phases (`GUARD`, `f_*` ids; module "the defender's kit" inside PROTECTION, before WORLD). `guardActive()` is
true for the defender in build and attack; the Skills bar, keys (`guardKeyDown`/`guardKeyUp`), aiming
(`updateTargeting`) and clicks (`actionDown` -> `fortRocket`, the rocket launcher) all follow it. Casts are checked by
`fortOK` (host; counts per round in `G.egg.left2`, `inside` placements, ultimates attack-only + once), replayed in both
phases. Powers: 1 dome layers (`G.domeLayers`, up to 4; `G.dome` is the outermost alive one; `hurtDome` promotes the
next), 2 lasers / 3 missile batteries / 4 tesla coils (`fortDevice`: static DEFENSES bodies with hp in
`FORT.devices`; batteries also hunt BX bombs in `fortSams`; coils cancel Pain casts near them via `fortBlocksPain` in
`runPain`), 5 lava ring (`LAVA`, kills), 6 drones, 7 angel / 8 devil / 18 dragon / 19 sorceress (`FORT.spirits`,
`spiritsFrame`; the sorceress makes `fortShielded()` true inside the area: forEachObject, liftProp, knock and kickCores
skip it), 9-12 rings (`fortRing`: brick, steel, energy, reflective; a new type goes one ring further out, the same type
again merges into every segment = taller and stronger; segments are BARS records), 13 reflective roof / 14 teleport
roof (`FORT.roofs`, `fortReflect` in `updateBombs`: the mirror roof covers the area and bounces; the teleport roof is now
a portal `TELE_R` wide just `TELE_UP` over the egg: bombs/thrown things that fall in come out of `telePortalFX` 12 m over
their owner and drop on them as the defender's (practice: owner 'fort'); an attacker who lands on it is thrown out of
the area; each still adds to `FORT.store`), 15 teleport-out gun (burst
scales with the stored count), 16 cosmic stars (`fortDurability()` divides damage to egg, walls, devices), 17 chained
golems (outside the lava, 12 m chain), 20 repair. Ultimates: B swarm (hold to charge 5-100, `FORT.minions`; getting hit
while charging loses it and waits 10 s, `fortHurtHook`), N God's hand (`fortGod`: sweeps once around, throws people to
the map edge).
Arena: `AREA_R` 25 (twice the old 12.4 ring; attackers are booted out, the defender is booted back in), river
`RIVER` 25.6-30.2 carved into the terrain by `genTerrain` in Alpha Mode (falling in drowns, `fortFrame`), props and
animals are kept out of the arena. The egg has health (`G.eggHP`, host-authoritative, sent as `h` in `eg`; `eggHit`,
bar over the egg); in Alpha Mode `breakEgg` becomes a 40 damage hit unless `G.eggForce`, blasts in kill range call
`eggHit`, and the egg is firm (blasts and pushes skip its body). The old Protection barrier types (`BAR_TYPES`, BARS,
`barrierFrame`) are reused by the rings; BAR_TYPES gained `brick` and `mirror`.
Fixes 2026-10-06 (user: "some powers are not working"): `G.spCd` (the anti-double-click timer) now also counts down in
the build phase; before, the first power used while building blocked every later one. Ultimates work in both phases.
One dome cast raises all four layers one after another (`fortDome`/`domeLayer`, recast recharges). Rings are aimed
(`ringAimHit` reads the floor through your own walls, `ringTarget`, `ringIndicator` gold = combine / white = new ring
further out / red = not allowed). Aimed placements are one per press, then the click is the rocket launcher again.
While the defender kit is active, unmapped keys never reach a class kit. Repair is an 8 s process.
**Practice in Alpha Mode**: nobody attacks you (the user did not want bots: attacks only come from real players online).
After building you stay the defender; Y (or the bar button) switches sides at any time in practice (`practiceSwitch`:
attack your own fort from 44 m out, `practiceStandOutside`, or go back to defending). Limited defender powers show
used/limit on the Skills bar (`G.egg.left2`). Test the defender with real key presses and clicks (scratch
`t13.mjs`, `t15.mjs`, `t17.mjs`): direct `localCast` calls hid the build-phase timer bug.

## Network checks (castOK)

Casts carry `o` (where the caster stood *before* the power moved them) and `ts` (the caster's own
clock). The host judges cooldowns and charge times on `ts` (bursts from a laggy link are fine; a clock
running more than 1.5 s ahead of real arrival times falls back to arrival timing), and allows the
position check to stretch with speed, update age and teleporting powers (`TELEPORTS`). `localCast`
sends even if the local effect throws. Test with scratch `duoall.mjs` (every power both ways) and
`lagnet.mjs` (bursty link).

## Physics (all modes)

Earth gravity everywhere (9.81 m/s², the low-gravity "Moon Day" map was removed). Masses are real
kilograms: player 80, brick wall 4000, cow 650, turret 400, chicken 2.5, egg 20, nuke 400. Jump is
4.7 m/s (~1.1 m), one air jump of 4.2 m/s (Pain: the Gravity Jump is the air jump), reset only by
landing at least 0.25 s after take-off (`P.jumpT`). Because most powers push by changing velocity directly, those
pushes go through `heft(body, e)` = min(1, (80/mass)^e): explosions (`blastAt`) e=.4, dash and fireball
e=.75, Chaos Shinra/sword/punch and every Pain push (wrapped in `forEachObject`) e=.5. Forces written
as `accel * b.mass` (fields, wind, fans, magnets) stay mass-independent on purpose. `controlPlayer`
keeps its snappy velocity lerp but drops the part of your input that pushes into a loose object over
300 kg (`P.blockN`, side contact normals collected in `groundCheck`), so a runner can't shove a wall
over (an acceleration cap did this before and made running feel sluggish); a dash stops with a bump when it reaches anything over 300 kg (`dashImpact`, by bounding box).
Throw speed falls with weight; anything over 300 kg can't be picked up. When adding a body, give it a
real mass in kg. `cameraBoom` pulls the camera in front of trees, rocks, ruins and also loose objects
wider than 0.9 m (walls, defenses, golems; ray vs body AABB), so it never sits behind a wall.

## Status / known issues (please check these first)

1. **Verified, not a bug (2026-10-01):** the knocked-out player's score/kill feed. Tested
   with two visible game instances over the real PeerJS broker: client knocked out, host
   knocked out, and repeat knockouts all updated `kf` + `score` on both screens within
   ~0.5 s. The earlier 0–0 reading came from the mock BroadcastChannel test. Note that
   paused/throttled tabs stop the game loop, so a dead player won't respawn (and can't be
   knocked out again) until their tab is visible.
2. **Confirmed 2026-10-01: no TURN relay.** PeerJS's free TURN servers (`eu-0/us-0.turn.peerjs.com`)
   no longer resolve, and the free no-signup relays (openrelay.metered.ca, freestun.net,
   anyfirewall, numb.viagenie) are all dead. Players whose network needs a relay (mobile data,
   school/office Wi-Fi, VPNs, symmetric NAT) can't connect. Fix: add credentials from a TURN
   provider account to `TURN_SERVERS` in index.html (not done: publishing relay credentials in the
   public page was declined). **Backup relay (2026-10-01):** when the direct link is blocked the
   joiner falls back to public MQTT brokers over wss (`relayConn`, `RELAY_BROKERS`: HiveMQ,
   Mosquitto, EMQX; mqtt.js loaded lazily from jsdelivr with SRI). The joiner's random 128-bit
   topic reaches the host via PeerJS connection metadata; the host listens on every broker. Adds
   ~40 ms each way; tested: fallback in ~7 s, movement/casts/scoring/leave all work. The user's
   home network is cone NAT (direct P2P works). Join/host report the real reason on failure.
3. Friendly fire is on for explosions; may want team-safe damage by tracking the owner of
   each blast.
4. Cooldowns (`CD`) and damage numbers are first guesses — tune after playtesting.
5. Performance (profiled 2026-10-01): the big fight lag was cannon's heightfield pillar cache
   being wiped by every crater (`syncTerrain` now clears only nearby pillars, full reset only
   when the field's min/max height changes) plus a debug `console.error` in
   `ConvexPolyhedron.computeNormals` (overridden without the check). Render resolution is capped
   at 1.5x and `perfTune()` lowers it when GPU-bound. Particles are capped (`FX_MAX`) and don't
   cast shadows; knives have no outlines. The local player + camera are interpolated between the
   60 Hz physics steps so 120/144 Hz screens look smooth. An FPS counter is always shown (`#fps`).
   The counter also shows which GPU draws the game (`GPU`); if the browser renders WebGL in
   software (hardware acceleration off: ~9-30 FPS even on gaming PCs) a `#gpuWarn` banner explains the fix.
   Debris is capped at ~60 objects and physics sub-steps are capped at 4 per frame.

## Deploying (first task)

**Live:** https://londaridzelazare-bit.github.io/chaos-arena/ — GitHub Pages from `main` /
(root) of `github.com/londaridzelazare-bit/chaos-arena`. To redeploy, commit and
`git push` to `main`; Pages rebuilds in about a minute at the same URL.
**Bump `GAME_VERSION` in index.html on every deploy.** Pages sends `Cache-Control: max-age=600`, so
browsers kept showing the old game; on the title screen the page fetches itself with `no-store` and
reloads into `?v=<new version>` when the version differs (once per version, never mid-match).

The user wants a public link to send to friends. Deploy `index.html` as a static site:
- Preferred: whatever the user has used before (check their GitHub repos / Netlify sites).
- Otherwise GitHub Pages: create a public repo `chaos-arena`, push this folder, enable Pages
  on `main`, and give the user the URL. Netlify also works (`npx netlify-cli deploy --prod --dir .`).
- For updates, redeploy to the same site so the link never changes.

## How to test locally

- Open `index.html` directly in a browser and click **Practice alone** (no network needed).
- Online: serve the folder (`npx serve .` or `python3 -m http.server`), open two browser
  windows, host in one and join with the code in the other. Both windows must be visible,
  since background tabs pause the game loop.

## Working with this user

- They iterate fast with short, excited requests ("add X", "make it cooler"), often with typos.
  Interpret generously, build it, test it, and ship it.
- After each change: redeploy, then tell them in plain language what changed and how to try it.
- Skill names like Chibaku Tensei, Shinra Tensei, Bijuu Bomb, One Punch and End of Evangelion
  come from existing anime; fine for playing with friends, but they'd need renaming for a
  commercial release.
