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
`G.phase` is really `'build'` (EGG_BUILD = 60 s, defender gets all 19 `DEFENSES`, F ends early via
`edone`) then `'attack'` (EGG_ATK = 120 s, attackers get `SPECIALS` with cooldowns forced on; the
defender can only shoot ki and grab). Attackers can't enter the 12.4 m ring. State lives in `G.egg`
(`def, round, total, pts, ph, endT, over, left`). Protection powers are casts `d_use` / `d_place`
(`runDefense`, `placeDefense`), validated on the host by `eggDefOK` against `G.egg.left`. The egg
is host-authoritative: only the host's egg is dynamic and can break (`breakEgg` returns on clients
unless `G.eggNetBreak`); the host sends `eg` (pose, cracks, angel lives) at 10 Hz and clients' kinematic
copies follow (`eggNetState`/`eggFrame`). Messages: `eph` (to attack), `eover` (round result + points;
`w` = 'atk' | 'def' | 'none'), `eend` (final standings), `edone`. Points: egg broken = every attacker +1,
egg held = defender + number of attackers. Field defenses that hit players (tesla, mortar, fans,
mirror) use `eggFoe()`: the local player only if they're attacking (victim-side, like all damage).
Practice alone in Egg mode: build for 60 s, then you attack your own defense.

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
start, max 20, +5 from loot), 8 vacuum warhead, 9 kamikaze clones; ultimates B Sunfall and N Nuclear Strike.
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

**Swordsman** (module "SWORDSMAN", after the Bomber): `s_*` ids in `SWORD`; click = 3-hit combo
(`swordSwing`, `SW.combo`), 19 powers on 1-0 / Q R F G T Z X C V, ultimate World Severance on B.
Charged powers (Gate of Babylon, World Splitter) use `SW.charge` (key down/up). Melee hits are
victim-side cones/spheres (`swordHit`, `swordArea`); `cutProjectiles` deletes bombs/shots/rods.
Per-character state in `C.sw` (blade buff, orbiting blades, mirror copies); the sword in hand is
`setHandSword` (others see it via 'st' flag bit 16). Hidden follow-ups: s_orbitx, s_prisonx, s_surfx,
s_counterx, s_ldash. Perfect Counter hooks `hurt()` (`P.parryT` -> `parried`). Blade Surfing reuses
the carpet (`P.carpet.type === 'sword'`). **World Severance** no longer cuts the ground (the user wants
an indestructible ground): the strike calls `severScar` (WORLD module), a glowing ribbon along the cut that
cools over 9 s, destroys props on the line and throws everything away from it. `severWorld`, `G.splits`,
`inVoid` and `voidSteer` remain but nothing creates splits now. Once per player per round
(`G.severUsed`, host-checked); a new round regenerates the terrain.

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

## Protection (Egg mode defender)

Module "PROTECTION", just before "casting, locally and for other players". Ten barrier types
(`BAR_TYPES`: wood, stone, steel, energy, frost, fire, spike, thorn, bounce, crystal) are `DEFENSES['bar_*']`
with `barrier:k`; `placeDefense` hands them to `placeBarrier`, which **merges** into any barrier within
1.2 m (`mergeTrait`: tier +1, HP and size grow, traits stack, name like "Burning Spiked Stone Wall").
`BARS = {list, destroyed, fields}`, reset in `startMatch`. Barriers are static cannon boxes (tag
'barrier'), block bombs (`bSolid`) and summons (`eMove`), and take damage through
`summonsTakeArea/Cone` -> `barriersTakeArea/Cone` -> `barHurt` (steel layers ×0.75 each). At 0 HP the
egg's authority (host online) sends hidden cast `g_break` (clients' `g_break` is always rejected).
`barrierFrame` runs the traits (energy regen + swallows bombs, frost freezes bombs and slows, crystal
repairs neighbours and heals egg cracks, fire/spike/thorn/bounce on touch) and HP bars.
`GUARD` is the defender's 20-skill kit in the attack phase (`guardActive()`; `skillTable` returns it
first, keys via `guardKeyDown`, effects in `runGuard`): repair beam, mend egg, emergency wall, reinforce
(steel), dome recharge, interceptor, turret overdrive (`G.overdriveT`), freeze, repulsor, egg teleport,
holy shield (`G.eggInvulnT`), barrier fusion, golem, drones, tar pit / lightning rod / shockwave trap
(`guardField`), smoke, rebuild (`BARS.destroyed`), last stand. castOK only accepts GUARD casts from
`G.egg.def`. Test: scratch `guard.mjs`.

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
