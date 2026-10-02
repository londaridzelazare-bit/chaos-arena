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
- Deformable procedural terrain (heightfield + craters), destructible trees/rocks/ruins,
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
- Terrain: `genTerrain(seed)`, `craterAt(p, r, depth)`, `groundH`, `syncTerrain`.
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

`G.mode` is `'egg'` (the default), `'chaos'` (the original 31 skills, `SPECIALS`) or `'pain'`
(gravity powers, `PAIN_SKILLS`). Picked on the title screen (`ME.mode`, localStorage key `ca-mode2`);
the host's choice travels as `NET.mode` in `lobby`/`start` messages and can be switched in the lobby.
`startMatch(seed, list, score, mode, cdOn, eg)`. `skillTable()` / `skillDef()` return the active table;
tray, keys (`PAIN_KEYS`), cooldowns all follow it.

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

**Pain mode** (module "PAIN MODE" in index.html, right before `runSkill`): ids are prefixed `p_`
and dispatched by `runPain()` from `runSkill()`. Tunables in `PAIN_CFG`. Shared systems:
GravityForce (`radialPush`, `conePush`, `knock`), gravity fields (`PAIN.fields`, applied every
physics step by `applyPainForces()` from `applyForces`), PhysicsObjectAttractor (`liftProp` turns
props into debris objects and `regrowProps` puts them back after 40 s; `makeCore`/`attractorField`/
`stickToCore`/`collapseCore` build the Chibaku masses with instanced rock chunks), timed phases in
`PAIN.fx` (`update(dt)` returns false when done), and `SpaceWarp` (screen-space lens/shockwave pass,
only renders through a texture while an entry is alive; `warpPulse` for one-off rings).
- Terrain is indestructible in Pain mode (`craterAt` returns early). Props are uprooted, never deleted.
- Damage/knockback stay victim-side like the rest of the game (`victimFor`, `painHurt`, no friendly fire).
- Pull sends its target as `tg`; the warp dash sends its direction `d` and end point `e` (host rejects `e` farther than WARP_DIST).
- `castOK()` (host) rejects casts with a spoofed id, unknown skill for the mode, broken cooldown,
  out-of-range target or a teleported origin; `st`/`death` must come from the sender's own id.
- Practice in Pain mode spawns training dummies (`PAIN.dummies`, physics objects with damage numbers).
- Abilities owned by a player stop when they're knocked out (`painOwnerDied`).
- Input kinds (`PAIN_SKILLS[id].kind`): `charge` (hold, release casts with `c` = seconds; Chakra Rod
  `chargeMul` has no cap for any of them: rod +1x per second, both Shinra +0.5x per second; Shinra damage and reach use the full multiplier, knockback `shinraKick(m)` = m^0.55, at most `SHINRA_MAX_PROPS` props torn loose), `aimhold` (Pull: hold Q shows a lock-on via
  `pickPullTarget`, release casts `{tg}`; no target = `pullEverything`), `aim` (red marker + click), `now`, `move` (dash/jump). Keyboard goes
  through `painKeyDown`/`painKeyUp`, Skills-bar clicks through `painUse`.
- Area gravity uses `bodyFor()` (moves the caster and allies too, `harm` only for enemies);
  direct hits use `victimFor()`. Chibaku cores are movable bodies (`trackCore`, `moveCore`,
  `kickCores`): From Below lifts them, From Above slams them into an early collapse, pushes/pulls
  shove them. Chibaku orbs fly from the hand first (`throwOrb`).
- Planetary Orbit (`PAIN.orbits`, second press fires `PAIN.shots`). Meteor Dive, Weightless World,
  Deva Barrier and Six Paths were removed at the user's request (2026-10-03).
- Warp Dash (Shift x2, `p_gdash`): `warpEnd` walks the line in 0.5 m steps up to WARP_DIST (32 m),
  stopping before props (`warpBlocked`), objects over 300 kg, cliffs and the island edge; the caster
  rides `P.warp` in `controlPlayer` for WARP_T (0.14 s). Visuals: lenses, a warpMaterial tunnel,
  see-through afterimages (`warpGhost`). Gravity Jump stays on Space x2 but is hidden from the
  Skills bar (`hide:true`; the tray skips hidden entries).
- Cooldowns are a match setting (`G.cdOn`, off by default; title button for practice, host button
  in the lobby, sent as `cd` in `lobby`/`start`). `cdFor(id)` gives 0 when off. The host still
  rate-limits casts (14/s per player) and checks that a claimed charge `c` is not longer than the
  time since that player's previous cast of the same ability.

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

## Physics (all modes)

Earth gravity everywhere (9.81 m/s², the low-gravity "Moon Day" map was removed). Masses are real
kilograms: player 80, brick wall 4000, cow 650, turret 400, chicken 2.5, egg 20, nuke 400. Jump is
4.7 m/s (~1.1 m), double jump 4.2 m/s. Because most powers push by changing velocity directly, those
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
