# Development

## Checks and formatting

Use Python 3.10 or newer and Bash. Install the pinned tools in a virtual environment:

```bash
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements-dev.txt
export PATH="$PWD/.venv/bin:$PATH"
python3 scripts/check.py
```

Run `python3 scripts/check.py --fix` to format Python and shell and normalize text whitespace. The same checks run on pushes and pull requests. XML is checked for well-formedness; game schemas, XPath matches and gameplay require separate X4 validation. Blender scripts are parsed and linted without importing Blender.

Use UTF-8, LF, a final newline, spaces and no trailing whitespace. Indent code with four spaces and workflow YAML with two. Use descriptive names, uppercase shell variables, constant-first equality comparisons and explicit boolean checks. Ruff's E712 rule is disabled to retain explicit boolean comparisons. Keep shell free of prose comments. Keep only short, non-obvious constraints in code; put explanations here. XML continuation attributes may align with their opening attribute. Preserve XPath selectors, savegame identifiers and embedded game expressions when applying formatting.

## Installation and publishing helpers

`install.sh` and `publish.sh` both source `lib/find_x4.sh`. The library searches usual Steam roots and additional library folders. `X4_PATH`, `X_TOOLS_PATH`
and `PROTON_PATH` override discovery. Proton Experimental is preferred when found; otherwise the helper uses the last matching Proton directory it encounters.

Installation replaces the extension directory with a copy of `extension/`. Refresh it after edits; X4 enumerates real extension directories, so a symlink does not substitute for installation. Restart the game after installing or removing.

Publishing stages a separate copy inside the game's extensions directory. The repository keeps its readable extension id; `steam/workshop-id` holds the numeric Workshop id. The helper changes only the staged manifest, runs the interactive WorkshopTool and restores the manual installation after success. On Linux it runs WorkshopTool through Proton and maps paths through drive Z. A failed upload can leave the staged copy behind; rerun `./install.sh` to restore it.

## Release metadata

`content.xml` uses an integer version multiplied by 100 and an ISO release date. The date matches the corresponding released entry in `CHANGELOG.md`. Development changes belong under `Unreleased`; they do not advance the manifest's release version or date. An unreleased scaffold may retain its initial creation date until its first release. Keep existing extension ids stable.

## Implementation constraints

### extension/md/drjele_black_market_leads.xml

The whole mod reads one vanilla global: `md.$ShadyGuyMap`, written by `md/npc_shadyguy.xml`, mapping each black marketeer NPC to the station he sits on. It is savegame state, maintained by vanilla on creation and on host-station destruction, so no scan of our own is needed. `md.NPC_ShadyGuy.GameStarted.$TrackedStationGroup` holds the same stations and is read cross-script by the DLCs, but the map carries the NPC as well and is therefore the better handle.

Relevant properties: `station.shadyguy` (entity or null), `npc.isshadyguy`, `npc.tradesvisible` (true once the player has unlocked that dealer), `object.isknown`, `object.scannedlevel`.

The mod is entirely event driven; there is no periodic scan. `md.$ShadyGuyMap` holds roughly one entry per sector, so a poll over it would be a few hundred property lookups every few seconds for nothing.

There is exactly one walk of the map, in `SectorPass`. It fires on `event_object_changed_sector object="player.entity"` and on `md.Setup.Start`, the same pair vanilla uses in `Signal_Leaks.Manager.PlayerChangesSpace`. The load trigger matters: without it, installing the mod and loading a save leaves the watch group empty until the player happens to change sector. Teleporting counts as a sector change, because the player entity really does move. It rebuilds the `Watcher.$Watched` group with the stations in the player's new sector whose marketeer is still locked — normally one, occasionally two. Everything after that hangs off events on that handful of objects: `MarketeerApproached` on `event_object_changed_attention group="$Watched"` when a station crosses to `attention.visible`, and `MarketeerKnown` on `event_object_known_to_player group="$Watched"`. The group and the cues that listen on it must live in one cue tree. As siblings the group does not exist yet when the events are set up, and the game says so twice per listener — `Property lookup failed` then `Evaluated value 'null' is not of type group`. Hence the `Watcher` parent, which mirrors `Signal_Leaks.Manager` and `NPC_ShadyGuy.GameStarted`. The two listener cues deliberately have no `namespace="this"`, so the bare `$Watched` in their conditions resolves in the parent namespace, exactly as `Manager.ObjectChangedAttention` does; their action blocks run atomically, so sharing `$Station` between them is safe. `SectorPass` and `PlantCheck` do carry `namespace="this"`, because both hold values across a delay and their instances overlap.

The XSD documents `groupeventsource` as "the group is required to exist but may be empty; adding/removing group members is possible even after the event is set up", which is what makes rebuilding the group under a live listener safe.

`attention.visible` is the right trigger for two reasons: it is the moment the player is actually close enough to scan, and it is the same moment vanilla's own `Manager.ObjectChangedAttention` seeds the station, so the mod arrives right after vanilla has had its turn.

No retry machinery. Every reason `PlantLead` bails out is either permanent for this visit (no docking permission, faction excluded, attempt cap) or self-correcting through the same events (attention dropped, cleanup queued, no free slots) — flying back up to the station raises the attention event again. The cooldown in `State.$Attempts` is what stops a re-approach from planting a second time.

Only stations with `isknown` are ever reported or acted on. That restriction is the point of the mod and is deliberately not a setting: a marketeer in an unvisited sector must stay as invisible as it is in vanilla. Stations are put in the watch group before the `isknown` check, so one discovered while the player is already in the sector still fires through `MarketeerKnown`.

Vanilla unlock path, for reference. `md/signal_leaks.xml` cue `Manager.GenerateSignalLeaks` rolls leak counts in library `CalculateLeakCounts` and, for the first mission leak only, swaps in `$ShadyGuyMissionTable` when `$LeakObject.controlentity.{controlpost.shadyguy}` exists. That table has a single entry, `GM_BringItems__Trigger`, page 30135, text offsets 1000 and 1100 — the one-hour illegal-item delivery whose reward text is `{30135,106}` "Access to unsanctioned trade offers". On success `md/gm_bringitems.xml` runs `set_entity_traits tradesvisible="true"` and `unlock_achievement BLACK_MARKET`, and only then does `md/npc_instantiation.xml` cue `PlaceShadyGuy` move the character into a bar.

`$MissionLeakCount` cannot roll zero: both `set_value` calls in `CalculateLeakCounts` use `min="1"`. A station yields nothing when it has no free leak slots, already carries `$MaxLeaks = 8` leaks, is owned by a faction in `$ExcludedMissionFactions`, or is covered by `$SuppressSignalLeakGeneration` or `md.GenericMissions.Manager.$ExcludedOfferObjects`. The tick mirrors every one of those guards before it signals anything.

The docking check is the important one. `Manager.GM_BringItems__Trigger` requires `$Station.hasrelation.dock.{faction.player}`; without it the cue sets `$ReportCue.$SetupFailed` and cancels, `Mission_Report_Listener` resets `Mission_Selector`, and because `Mission_Selector` has already removed the only entry from the mission table the reset falls through to `MissionLeak_Cleanup`, which destroys the leak it just created. So on a station the player cannot dock at, the lead appears and vanishes and no amount of scanning helps. The mod never plants there, and the logbook entry and the list say why.

`$CleanupTable` is checked, not cleared. An entry means vanilla intends to remove that station's leaks and `GenerateSignalLeaks` would take its early-abort branch, wasting the attempt; `Manager.PlayerChangesSpace` clears the entry itself once the station is in the player's new sector, so skipping and retrying next tick is correct.

`$ForceMode` selects the hook. `'station'`, the default, signals `md.Signal_Leaks.Manager.GenerateSignalLeaks` with the station — the entire vanilla pass, which is why it is the verified option; the cost is a couple of unrelated data leaks. `'leak'` signals `md.Signal_Leaks.Manager.PlaceMissionLeakOnSurface` with a slot and a mission table holding only `GM_BringItems__Trigger`, placing exactly one lead and no noise. It is not the default because it uses a cue from another script as a computed table key, which vanilla never does — it only ever builds that table from inside `Manager`'s own namespace. Test it before promoting it.

The leak census and the slot filter replicate `CalculateLeakCounts` and `GetLeakSlots` rather than referencing them. `include_actions ref="md.Signal_Leaks.GetLeakSlots"` would also overwrite a local `$LeakLocations`, and vanilla never calls either library across scripts.

`State.$Attempts` and `State.$Announced` are tables keyed by station object, never variables written onto the component: X4 refuses `component.{...}.$var`, and a failed property lookup does not skip the enclosing `do_if`. `PruneState` iterates both in reverse because it mutates them while walking. Entries are dropped when the station is gone, when its marketeer has become `tradesvisible`, or after two hours without a sighting.

`PlantCheck` captures `event.param` in a first `<actions>` block, waits `30s`, and only then runs the guards. The capture has to happen before the delay — vanilla's `NPC_ShadyGuy.TrackedStationDestroyed` uses the same shape for the same reason. The wait lets the vanilla pass for that station finish; `PlaceMissionLeakOnSurface` sleeps 5s and then retries up to ten times at 1s. The delay is a literal because `<delay>` does not take a runtime expression.

The guards live in `PlantLead`, a `purpose="run_actions"` library, purely so they can bail out with `<return/>`. The XSD is explicit that `return` is "supported in AI scripts, as well as MD libraries with purpose=run_actions", and there is not a single cue-level `<return/>` anywhere in the shipped scripts — twelve nested `do_if` levels was the alternative.

`Configuration` is `instantiate="true" namespace="static"`, so it writes plain `$var` and never `Configuration.$var` — self-qualifying inside such a cue sends the write to the running instance while other cues read the static one, silently.

### extension/md/drjele_black_market_leads_menu.xml

Everything here needs SirNukes Mod Support APIs, which is why it is a separate file: none of these cues can fire when the API is absent, and the mod's own behaviour is unaffected.

`md.Simple_Menu_API.Reloaded` and `md.Simple_Menu_Options.Reloaded` are different cues. The menu registers against the first, the settings file against the second; crossing them registers nothing at all and reports no error.

`Register_Options_Menu` gives the list its own line under Extension Options and calls `FillOptionsMenu` on open. `Create_Menu` must not be called from that cue — the frame already exists. The standalone menu, opened from the chat command, the interact menu and the hotkey, calls `Create_Menu` first. Both then run the `BuildLeadRows` library, which is the only place the table is built.

`sort_list` has no "sort by element N" mode, so the rows are ordered by sorting a list of unique `sector|station|object` strings and looking the rows up in a table keyed by that string.

The hotkey is registered but is not expected to work here. `md/hotkey_api.xml` receives keys only through the Named Pipes API, fed by an external Python server, and the SirNukes readme states that pipes are set up for Windows only. The game here is the native Linux build. Extension Options, `/leads` and the station right-click entry are the openers that work.

`/leadsdebug` signals `md.NPC_ShadyGuy.GameStarted.ShadyGuy_DEBUG`, vanilla's own dump of the tracked station group and the whole marketeer map to the debug log. It is the fastest way to pick a test target.

### extension/md/drjele_black_market_leads_options.xml

Every callback does nothing but write a global that the main script reads through the usual three-level fallback, so a missing API, a missing global and a missing config table all degrade to the shipped default.

An option's `$id` owns its stored value, its widget type and its range for good: SirNukes saves the value under the id and feeds it straight back as the widget's start value, so a range change under an existing id makes validation fail and closes the entire Extension Options menu, taking every other mod's settings with it. Hence `drjele_black_market_force_cooldown_min` carries its unit in the id.

`Register_Option` invokes the callback once at registration with the stored value unless `$skip_initial_callback` is set. That is why no option here is an action button: an option that opened the menu would open it on every load.

Option values live in `uidata.xml` at profile level, not in the savegame — `Simple_Menu_Options.Load_Userdata` reads them through `md.Userdata.Read` with owner `sn_mod_support_apis`. They are therefore shared by every save on the profile.

## Verification

Not yet run. Install, restart X4 — a brand-new mdscript is not picked up by `/refreshmd` — then iterate with `/rmd`. The active log on this machine is `~/snap/steam/common/.config/EgoSoft/X4/23682333/debuglog.txt`; `~/Documents/Egosoft/X4/23682333/` is a stale copy.

1. `grep "drjele_black_market\|DrJele black market" debuglog.txt` must be free of `[=ERROR=]`.
2. Turn on the debug option, run `/leadsdebug`, and pick a known, dockable, non-player/Xenon/Kha'ak station whose marketeer is still locked.
3. Fly there. Expect the ticker line and a Tips logbook entry that opens the map on the station; about 45 seconds later a `planted a lead on ...` debug line; about 10 seconds after that a new signal leak on the hull.
4. Scan it. The offer must be the one-hour illegal-item delivery. Completing it must flip the row in the list to `Unlocked` and drop the station from `State.$Attempts`.
5. Negative tests: a station without docking permission shows `Locked - no docking permission` and is never planted on; a marketeer in an undiscovered sector appears nowhere.
6. Leave and return inside the cooldown: exactly one lead, and the station's total leak count never passes `$MaxLeaksPerStation`.
7. Save, reload, reopen the list: state survives and no duplicate logbook entries are written.
8. Disable SirNukes and reload: logbook, ticker and lead planting must all still work.

## Open questions

- Whether `table[{md.Signal_Leaks.Manager.GM_BringItems__Trigger} = ...]` resolves a cross-script cue as a computed table key. This is the whole of `$ForceMode = 'leak'`; the default does not depend on it.
- Whether the standalone `Create_Menu` opens cleanly from an interact menu callback while flying. The Extension Options submenu is the fallback.
- Whether `Manager.PlaceMissionLeakOnSurface` behaves when signalled from outside `Manager`. It is `namespace="this"` and resolves `Manager.$Leaks` up its own static chain, which should make it independent of the signaller, but that is inference from the source rather than observation.
- Whether `Make_Button`'s `$text` accepts a plain string or needs a TextProperty table.
- `extension/preview.jpg` is still the placeholder copied from `x4-unique-ship-limits` and must be replaced before publishing.

## Runtime traps found in game

Three things the schema validated happily and the game rejected on the first load.

`'@' cannot be combined with '?'`. `@md.Signal_Leaks.Manager.$CleanupTable.{$Station}?` is a parse error, and it takes the whole enclosing library down with it. Existence of a table key without either operator is `@<table>.keys.indexof.{$key}`.

`Evaluated value 'null' is not of type group`, twice per listener. Event conditions are set up when the script loads, before a sibling cue's actions have created the group. Groups and the cues that listen on them belong in the same cue tree.

`No texture found for icon name 'shadyguy'`. `libraries/icons.xml` defines both `shadyguy` and `npc_shadyguy`, but only the latter resolves as a UI texture; it is also the one `menu_map.lua` uses in the map legend.

`Duplicate cue name <name> in MD script`, on loading a savegame that already knows an older version of the script. Cue names are unique per script, and a savegame stores the cue tree it was written with, so **moving an existing cue under a different parent collides with the copy the save still holds** — the whole subtree is then rejected and the mod goes silent. Re-parenting therefore means renaming: `SectorArrived`, `StationBecameKnown`, `StationApproached` and `Evaluate` became `SectorPass`, `MarketeerKnown`, `MarketeerApproached` and `PlantCheck` when they moved under `Watcher`. Once released, a cue in this script can be renamed but never re-parented under its old name.
