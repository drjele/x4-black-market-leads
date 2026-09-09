# Black Market Leads for X4: Foundations

<p align="center"><img src="extension/preview.jpg" width="512"></p>

Black marketeers are placed on about one station per sector when the game starts, and then hidden. They are not in the bar, not in the station's people list, and nothing on the map points at them — the game only moves the character into the bar once you have already unlocked him. The single way in is a signal leak on the station hull, and nothing tells you which station is worth scanning.

This mod does not hand you the galaxy. It tells you which of the stations **you already know** has a marketeer aboard, and makes sure that scanning that station produces the unlock lead instead of three decoys.

## Install

```bash
./install.sh
```

The helper copies `extension/` into the game's `extensions/<extension-id>`
directory, using the id in `extension/content.xml`. It searches the usual Steam layouts and additional library folders. To choose an installation:

```bash
X4_PATH="/path/to/X4 Foundations" ./install.sh
```

Restart X4 after installing or updating. To remove the manual installation:

```bash
./install.sh --uninstall
```

Requires X4: Foundations 9.00. Works on an existing savegame; no new game needed.

## What it does

- **Points at the station.** The first time a station you already know is revealed to carry a black marketeer, you get one logbook entry under **Tips**, and optionally a ticker line. Clicking the entry opens the map on that station.
- **Makes the scan pay off.** When you fly up to such a station and its marketeer is still locked, the mod asks the vanilla signal leak manager to run a pass on it, so the unlock lead gets placed instead of maybe getting placed. If mission leaks are already on the hull the mod does nothing — the lead is among them.
- **Leaves only the lead worth scanning.** The game seeds a station with several mission leaks at once and only the first one is the black market delivery, so you scan three decoys to find it. With **Only the black market lead** on — the default — the mod clears the mission leaks off a station that has a locked marketeer and puts back a single one that can only be the black market delivery. Data leaks are untouched, and a mission you have already accepted has no leak left, so only pending generic offers on that one station are lost.
- **Costs nothing while you play.** No periodic scan. One pass over the game's marketeer table when you change sector, then plain events on the one or two stations in that sector that actually matter.

## What it deliberately does not do

Only stations the game already considers known to you are ever reported, listed or touched. A marketeer in a sector you have never visited stays exactly as invisible as it is in vanilla. You still fly there, you still scan the hull, you still run the delivery mission. That restriction is not a setting.

## The thing nobody tells you

**A black marketeer can only be unlocked on a station you are allowed to dock at.**

The vanilla unlock mission checks `hasrelation.dock.{faction.player}`. If that fails it reports a setup failure; the leak's mission table has already been emptied by then, so the retry falls through to cleanup and the leak that was just created is destroyed a moment later. That is the real reason some stations never reward scanning, however long you spend on them.

The mod never wastes a lead on such a station, and the logbook entry says so.

## How it works

Everything hangs off `md.$ShadyGuyMap`, the game's own table of black marketeer to station. Nothing is patched and nothing is spawned.

When you change sector, the mod walks that table once and collects the stations in your new sector whose marketeer is still locked — normally one. Those go into a small watch group. When one of them crosses to `attention.visible`, which is both the moment you are close enough to scan and the moment the game seeds its own leaks, the mod waits thirty seconds for the vanilla pass to finish and then looks at the station.

With **Only the black market lead** on, the mod then destroys the mission leaks on the hull and asks `md.Signal_Leaks.Manager.PlaceMissionLeakOnSurface` for one more, handing it a mission table that contains nothing but the black market delivery. Whatever mission leak you find on that station afterwards is the right one.

With it off, the mod only steps in when the station has no mission leak at all, and then signals `md.Signal_Leaks.Manager.GenerateSignalLeaks` — the first mission leak of any pass on a marketeer station is always the unlock delivery, so the pass produces the lead. Either way a per-station cooldown, an attempt limit and a cap on total leaks keep the hull from filling up.

## Where to change things

With SirNukes Mod Support APIs installed: **Options → Extension Options → Black Market Leads**.

| Setting                          | Default | What it does                                                                           |
|----------------------------------|---------|----------------------------------------------------------------------------------------|
| Plant a lead worth scanning      | on      | Seeds the unlock lead on a known station whose marketeer is locked                     |
| Only the black market lead       | on      | Clears the decoy mission leaks off such a station and leaves a single black market one |
| Retry the same station after     | 30 min  | Cooldown before a second attempt on the same station                                   |
| Logbook entry for a new lead     | on      | One entry under Tips per station, clickable to the map                                 |
| Ticker line for a new lead       | on      | Also shows a short ticker message                                                      |
| Write decisions to the debug log | off     | Needs the game started with `-debug scripts`                                           |

## Configuring without the settings menu

Without SirNukes the mod still runs — logbook entries, ticker lines and lead planting all work off the constants at the top of `extension/md/drjele_black_market_leads.xml`, re-read on every savegame load, so editing one and reloading is enough. Any other mod or cheat menu can override them at runtime through `global.$DrJeleBlackMarketForceLeaks`,
`$DrJeleBlackMarketOnlyLead`, `$DrJeleBlackMarketForceCooldown`, `$DrJeleBlackMarketLogbook`, `$DrJeleBlackMarketNotify` and `$DrJeleBlackMarketDebugChance`.

Note that SirNukes option values live in `uidata.xml` next to your savegames, not in the savegame itself, so they are shared by every save on the profile.

## Debugging

Add this to the game's launch options — Steam, right click X4, **Properties → General → Launch Options**:

```
-debug scripts -debug error -logfile debuglog.txt
```

The log lands next to your savegames: `$HOME/.config/EgoSoft/X4/<userid>/debuglog.txt` on Linux, `Documents\Egosoft\X4\<userid>\debuglog.txt` on Windows. If Steam is installed as a snap it runs the game with a redirected home, which puts both under `~/snap/steam/common/`.

The mod is silent by default. Turn on **Write decisions to the debug log**, or set `$DebugChance` to `100` in the configuration cue of `extension/md/drjele_black_market_leads.xml`, and every decision is written out:

```
DrJele black market leads: 1 watched station(s) in Argon Prime
DrJele black market leads: planted a lead on Trading Station Argon Prime, attempt 1, mode station
DrJele black market leads: Wharf Argon Prime already carries 3 leak(s), 1 of them mission leaks - nothing to add
```

## Status

**Verified in game on 9.00**, on a 49-day save:

| Check         | Result                                                                                                             |
|---------------|--------------------------------------------------------------------------------------------------------------------|
| Sector pass   | fires on arrival and on savegame load; reported 0, 2, 3 and 5 stations across four sectors                         |
| Reporting     | logbook entry and ticker line appear on entering the sector                                                        |
| Watch group   | attention events on it drive every later step; no polling                                                          |
| Decoys        | vanilla seeds a station to eight leaks with four mission leaks in one pass, and the mod clears them                |
| Single lead   | after the clear, the black market delivery was found on the first mission leak scanned                             |
| Version patch | the `sinceversion="2"` reset cleared stale bookkeeping on load, and the affected stations were reprocessed at once |
| Script errors | none                                                                                                               |

Not exercised yet: the `Plant a lead worth scanning` path with **Only the black market lead** turned off, and completing the delivery through to `tradesvisible`.

There is deliberately no in-game list. One was built — a table of known marketeer stations, reachable from Extension Options, a chat command, a hotkey and the station's right-click menu — and then removed, because in play the logbook entry already says everything the list did.

All three scripts validate against `md/md.xsd` extracted from the 9.00 archives, with no errors. The mechanism itself is read straight out of the shipped files rather than inferred:

| Claim                                                             | Where it comes from                                                                                                         |
|-------------------------------------------------------------------|-----------------------------------------------------------------------------------------------------------------------------|
| marketeers live in `md.$ShadyGuyMap`, one per sector              | `md/npc_shadyguy.xml`, cues `GameStarted.Init`, `EvaluateSectors`, `AddShadyGuys`                                           |
| the character is absent until unlocked                            | `md/npc_instantiation.xml`, cue `PlaceShadyGuy`, gated on `$ShadyGuy.tradesvisible`                                         |
| the trade options are greyed out until unlocked                   | `md/npc_itemtrader.xml`, cue `DefaultComm`, tooltip `{1002,12075}`                                                          |
| the first mission leak on such a station is always the unlock one | `md/signal_leaks.xml`, `Manager.GenerateSignalLeaks`, the `$i == 1` branch selecting `$ShadyGuyMissionTable`                |
| the reward is the unlock itself                                   | `md/gm_bringitems.xml`, `set_entity_traits tradesvisible="true"` plus `unlock_achievement BLACK_MARKET`                     |
| no docking permission means the lead is destroyed again           | `md/signal_leaks.xml`, `Manager.GM_BringItems__Trigger` → `$SetupFailed` → `Mission_Selector` reset → `MissionLeak_Cleanup` |
| a station can never roll zero mission leaks                       | `md/signal_leaks.xml`, library `CalculateLeakCounts`, `min="1"` on both rolls, `$MaxLeaks = 8`                              |
| the watch group may be rebuilt under a live listener              | `libraries/common.xsd`, `groupeventsource`: "adding/removing group members is possible even after the event is set up"      |
| `<return/>` only works in a `run_actions` library                 | `libraries/common.xsd`, element `return`; there is no cue-level `<return/>` anywhere in the shipped scripts                 |

What still needs a run in game is listed under **Open questions** in [DEVELOPMENT.md](DEVELOPMENT.md), together with the test procedure.

## Publishing to the Steam Workshop

Install **X Tools** (Steam app 282160) and keep Steam running and logged in with an account that owns X4. On Linux, install Proton as well; on Windows, run the helper from Git Bash, MSYS or Cygwin.

```bash
./publish.sh publish
./publish.sh update "what changed"
```

Use `publish` once, then `update` with a change note. `X4_PATH`,
`X_TOOLS_PATH` and `PROTON_PATH` override automatic discovery. The staging location must contain an `extensions` directory.

The first upload records the numeric id in `steam/workshop-id`; retain that file for future updates. The readable id in the repository's `content.xml`
stays unchanged. After publishing, open the printed Workshop URL, complete any required Steam agreement and choose the item's visibility. Avoid keeping both the manual installation and a subscription to the same mod enabled.

Update the manifest version and release date together with `CHANGELOG.md`
when releasing. See [Development](DEVELOPMENT.md) for staging, platform and release conventions.

## Development

See [DEVELOPMENT.md](DEVELOPMENT.md) for setup, code style, validation and release conventions.

## Legal

MIT, see [`LICENSE`](LICENSE). Non-commercial fan project; X4: Foundations belongs to Egosoft GmbH and this project is not affiliated with or endorsed by Egosoft.
