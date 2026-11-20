# Family Time – Friendly Fire 0.13.0

A BepInEx 5 / Harmony plugin for **Family Time** (Episode 3 build, Unity Mono, DX11).

It turns the axe and the shovel into tools that can hurt the creatures around you, including the pups and Mama Wolf, and builds a layer of consequences around that: injuries, healing, fear, trust, resentment, Mama's temper, night raids, orphaned pups and more. Everything is configurable, and every feature can be switched off.

- Plugin GUID: `local.familytime.friendlyfire`
- Output: `FamilyTimeFriendlyFire.dll`
- Language: C# 9, target framework `net472`

> **Heads-up:** this mod is deliberately dark. Pups can be injured or killed, Mama Wolf can be killed permanently (opt-in), and a few features change save-game state. Read the "Safety and save-game notes" section before you play.

---

## 1. Requirements

### To play

| Requirement | Notes |
|---|---|
| Family Time, Episode 3 build | The project is set up for `family_time_ep3_dx11`. The mod patches internal game classes (`AxeTool`, `ShovelTool`, `WolfMama`, `BedManager`, `HandService`, `GAT.*`, `family.GameSession`, ...), so a different game version may break some hooks. |
| BepInEx 5.x (Mono) | Must be installed in the game folder (`BepInEx\core\BepInEx.dll`, `0Harmony.dll`). |
| Unity Input System | Ships with the game (`Unity.InputSystem.dll`). Used for all hotkeys. |

### Optional mods (soft dependencies)

None of these are required. The mod finds them by reflection at runtime. If a mod is missing, the matching integration silently turns itself off.

| Mod | What the integration adds |
|---|---|
| **Uppies Please** | "Offended" pups stop asking to be picked up, reaction icons (Sad / Angry / Mama) appear above pups' heads, and Mama can walk up and scold the player using Uppies Please's own scene. |
| **Pup Traits** | Pup temperament is taken from its traits (Brave, Shy, Scared). Clingy and Scared pups run to Mama more often, and Violent pups may bite back. Without it, temperament is assigned randomly. |
| **Pups Wander** | Injured or frightened pups are not allowed to wander off. |
| **Red Menace** | Its "wolf mama wrath" (teleport behind the player, knock-out, game over) can be disabled. See section 5. |
| **Free Roam**, **Child Spawn** | No code link. They only share hotkeys, so the defaults are chosen to avoid clashes. |

### To build from source

- Windows with the **.NET SDK** (`dotnet` on your PATH).
- The game installed with BepInEx, because the project references these files from your game folder:
  - `BepInEx\core\BepInEx.dll` and `0Harmony.dll`
  - `<GameName>_Data\Managed\Assembly-CSharp.dll`
  - `UnityEngine.dll`, `UnityEngine.CoreModule.dll`, `UnityEngine.IMGUIModule.dll`, `UnityEngine.TextRenderingModule.dll`, `UnityEngine.InputLegacyModule.dll`, `Unity.InputSystem.dll`
- Internet access on the first build, so NuGet can restore the .NET Framework 4.7.2 reference assemblies. (Alternatively, install the .NET Framework 4.7.2 Developer Pack.)

---

## 2. Installation

1. Edit the game path if yours differs from the default `F:\family_time\family_time_ep3\family_time_ep3_dx11`:
   - in `build.cmd`: the `GAME_DIR` variable
   - in `FamilyTimeFriendlyFire.csproj`: the `GameDir` property
   - If your data folder is not named `FamilyTime_Data`, also fix the `ManagedDir` property and the `MANAGED` variable.
2. Run `build.cmd`. It checks that the needed files exist, then runs `dotnet build -c Release`.
3. Copy `dist\FamilyTimeFriendlyFire.dll` into `BepInEx\plugins\`.
4. Start the game once. The config is generated at `BepInEx\config\local.familytime.friendlyfire.cfg`.
5. Check `BepInEx\LogOutput.log` for `[offended]`, `[traits]`, `[redmenace]` lines, which tell you which integrations were detected.

**Uninstall:** delete the DLL from `BepInEx\plugins`. You can also delete `BepInEx\config\FamilyTimeFriendlyFire\` (per-save pup data). Note that permanent changes made in a save, such as a killed Mama Wolf or pups that have left the family, are not undone by removing the mod.

---

## 3. Controls

All keys are rebindable in the config.

| Key | Action | Config section |
|---|---|---|
| **V** | Free swing: hit whatever is in front of the camera with the equipped axe or shovel | `[FreeSwing]` |
| **G** | Shoo: the pup in front of you runs away and Mama growls | `[Shoo]` |
| **H** | Intimidate Mama: hold an axe or shovel, aim at her (up to 6 m) and press | `[Intimidate]` |
| **F10** | Manually reset the game's native Mama rage (rescues a stuck save) | `[Mama]` |

Keys that belong to other mods (listed so you can avoid conflicts): F2 Child Spawn, F7 Pups Wander, F8 Free Roam (Pups Wander's clock also uses F8, so change one of them), F9 Pup Traits debug.

---

## 4. How the mod works

### 4.1 Damage
- Every axe swing while chopping and every shovel dig casts a "beam" along the strike line. Everything inside it takes damage through the game's own health system.
- Defaults: axe **2** damage, shovel **1** damage. A pup has **10 HP**.
- Targets are filtered by `[Targets]`: pups (PLAYER faction), passive animals, hostile animals. Pups you are carrying are not hit by default.
- A per-target cooldown prevents one swing from hitting the same creature repeatedly.
- A killed creature that has no loot of its own drops meat, borrowed from items other creatures already drop (`pork_chop`, `chicken_drumstick`).

### 4.2 Pups: injury, healing, fear, resentment
- **Injury:** a pup at or below 50% health whimpers periodically.
- **Healing:** hurt pups regenerate slowly (1 HP per 45 s). It is faster when carried (12 s), near Mama, and during sleep (+4 HP). Hungry pups do not heal.
- **Fear:** a hit pup stays afraid of the player for a while (90 s per hit, max 300 s) and runs away when you come within 4 m.
- **Comfort:** a hit pup may run to Mama for comfort. Near her it heals faster, calms down and regains some trust, and Mama looks at it.
- **Resentment** (Uppies Please): a hit pup stops asking to be picked up for 180 s and shows a Sad icon.
- **Trust:** every pup has a trust value (0–100, start 50) toward the player. Hits reduce it, carrying raises it, and it slowly drifts back to the start value.
- **Temperament:** each pup is Brave (20%), Normal or Timid (20%). It changes how much trust a hit costs and how much the pup fears you. With Pup Traits installed, temperament comes from its traits.
- Trust and temperament are saved per savegame in `BepInEx\config\FamilyTimeFriendlyFire\<savename>.pups.txt`.

### 4.3 Mama Wolf
- **Temper:** the mod keeps its own counter, separate from the game's rage. Hits on Mama (and every 2nd hit on a pup) raise it. It decays by 1 every 90 s.
- **Native rage:** if Mama is still angry when you go to bed, 1 point is added to the game's native rage counter. If she has calmed down, nothing is added. Native rage drops by 1 after each night.
- **Game over is disabled by default.** The only source of game over in the vanilla game is the native "terminal rage" check, and `[Safety] NoGameOver` forces it to false. Sleep always works, and saves stuck at 3/3 rage load normally.
- **Growls:** she growls after a pup is hit (50% chance), when she is hit herself, and when you approach a hurt or frightened pup holding an axe or shovel. Each growl also calls free pups to her. Growls are rate-limited to one per 40 s.
- **Revenge:** at temper 2 or higher she can bite you (1 damage, non-lethal by default, 7 m range, 25 s cooldown). It never knocks you out and never causes game over. The health comes back after 45 s.
- **Mama's health:** she has 20 HP. Tool hits take it down, and it regenerates over time and during sleep. At zero she goes into **fury**, a 45 s rage in which she growls and bites, all pups lose trust, and she cannot die.
- **Intimidation:** press H aimed at her, or hit her 3 times within 15 s. She is then cowed for 300 s: no growling, no biting, no anger, no fury.
- **Shock:** when a pup dies, Mama gasps and growls, and the other pups flinch and huddle (or scatter), and a few of them cry out.

### 4.4 Mama's death (opt-in, dangerous)
- Off by default. Set `[MamaDeath] CanDie = true` to allow a lethal blow to remove her from the family **permanently**.
- `Mode = Remains` (default): her body stays as meat (12 pieces) where she fell, and it can be cooked into soup in the cauldron.
- `Mode = WalkAway`: she walks into the forest and is hidden.
- Afterwards she cannot be called, so following, commands and mod scenes are blocked. This state is saved.
- **Orphans:** some pups go feral, some leave the family, and the rest lose trust. Without Mama, a pup with trust below 20 goes feral at night, and a feral one with trust below 15 leaves. Boar raids become more likely.

### 4.5 Shoo, feral pups and strays
- **G** makes a pup run away and Mama growl. After 3 shoos in a row the pup leaves the family and becomes feral.
- Feral pups are **not counted as children**: they are excluded from the "N Children" counter and the "not asleep / hungry" counters, the game does not pick them for commands, they wander on their own and they do not ask to be picked up. They therefore never block bedtime or add rage.
- Stand still near a feral pup without a tool (or carry it) and its trust rises. At 50 it rejoins the family.
- If the whole family is wiped out, two stray pups appear in the forest after about 90 s and can be befriended.

### 4.6 Night raids, falls and the cauldron
- **Raids:** each evening (from 18:00) a boar can approach the camp. Base chance is 15%, plus 12% per wounded pup, capped at 80%, and +25% while Mama is absent.
- **Falls:** pup throws are weaker (speed ×0.65). On landing, a pup takes damage from impact speed and fall height. Falling down slopes hurts more.
- **Cauldron:** a pup thrown into a hot cauldron dies instantly (200 damage). Mama then goes into fury and the other pups are shocked.

---

## 5. Compatibility notes

### Red Menace
Red Menace sends Mama behind the player, knocks them out and triggers game over whenever **any** pup dies, including deaths caused by this mod. The result is that the player wakes up but cannot do anything. Friendly Fire patches that by default:

```
[Compat] RedMenaceWrath = Off      (default) wrath never happens
                          Ours     only for pups that died from this mod
                          Vanilla  leave Red Menace untouched
```

You can also disable it in Red Menace's own config (`BepInEx\config\com.redmenace.hazards.cfg`). Key names are created on first run, so check them in the file:

```
[WolfMamaWrath]  Enabled = false      (knock-out and game over)
[PanicBehavior]  Enabled = false      (pups constantly shouting "daddy")
[Piranha]        BiteDamage = 1, BiteCooldownSeconds = 4, SwimGraceSeconds = 12,
                 MaxPiranhasPerPup = 1, TotalMaxPiranhas = 6, CanDamagePlayer = false
```

### Master switches
- `[Compat] LinkMods = false` turns off every integration with other mods at once. Use it first if something odd happens in another mod.
- `[Compat] PupsWander`, `UsePupTraits`, `ViolentPupsBiteBack` toggle individual integrations.
- `[Compat] RedMenaceWrath` is described above.

### Noise reduction
- `[Mama] RoarMinIntervalSeconds = 40` limits Mama's roars.
- `[Speech] LimitGreetings = true` limits "Mama!" to once per 25 s and "Daddy!" to once per 12 s, family-wide.

---

## 6. Configuration overview

The config file is generated on first launch. All sections are listed here, and every entry has a description in the file.

| Section | Controls |
|---|---|
| `Damage`, `Hit`, `Targets` | Tool damage, beam width, per-target cooldown, who can be hit |
| `FreeSwing` | The V key swing: reach, radius, cooldown |
| `Mama`, `MamaHp`, `MamaDeath`, `Fury` | Temper, rage, health, death mode, orphans, fury |
| `Revenge`, `Guard`, `Intimidate` | Biting back, growling at the approaching player, cowing Mama |
| `Injury`, `Heal`, `Fear`, `Offended`, `Shock` | Pup reactions to damage and death |
| `Trust`, `Temperament`, `Care`, `Mom` | Trust, characters, comfort, Mama stepping in |
| `Shoo`, `Strays` | Shooing and feral or stray pups |
| `Raids`, `Falls`, `Cauldron` | Boar raids, fall damage, cauldron deaths |
| `Meat`, `Sound`, `Speech` | Meat drops, hurt sounds, greeting limits |
| `Safety` | `NoGameOver` (default true), `NightlyRageRelief` (default 1) |
| `Compat` | Mod integrations and Red Menace handling |
| `Debug` | `Verbose` (frequent events, default off), `DebugLog` (default on), `ScreenMessages` |

---

## 7. Safety and save-game notes

- **`MamaDeath.CanDie`** is off by default. Turning it on can permanently change your save: Mama is removed, and pups may leave or go feral. Back up your saves first.
- **`NoGameOver`** is on by default. Turn it off only if you really want the vanilla rage/game-over rules back.
- Trust data is stored outside the save, in `BepInEx\config\FamilyTimeFriendlyFire\`. Deleting that folder resets trust and temperament.
- If the log gets long, make sure `Debug.Verbose` is `false`. If you do not need hit and sound details, set `Debug.DebugLog` to `false` as well.
- The source contains no network access and no process launching. Its only file write is the per-save `.pups.txt` file described above.

---

## 8. Changelog – 0.13.0

1. **Red Menace no longer kills the player.** Its wolf-mama wrath is disabled by default (`[Compat] RedMenaceWrath = Off`), with options for the mod's own kills only or the vanilla behaviour.
2. **Mama roars less.** A minimum interval between roars, plus a family-wide limit on the pups' "Mama!" and "Daddy!" voice lines.
3. **Intimidate Mama.** H with an axe or shovel, or 3 hits within 15 s, makes her quiet for 300 s: no growling, no biting, no anger, no fury.

---

## 9. Package contents

| File | Purpose |
|---|---|
| `FamilyTimeFriendlyFire.cs` | The entire mod (about 3800 lines) |
| `FamilyTimeFriendlyFire.csproj` | Project file: net472, references to game and BepInEx DLLs |
| `build.cmd` | Path checks, then `dotnet build -c Release`, output to `dist\` |
| `README.md` | This file |
