# Black Market Leads for X4: Foundations

<p align="center"><img src="extension/preview.jpg" width="512"></p>

Black marketeers are placed on about one station per sector when the game starts, and then hidden.
They are not in the bar, not in the station's people list, and nothing on the map points at them —
the game only moves the character into the bar once you have already unlocked him. The single way in
is a signal leak on the station hull, and you have no way of telling which station is worth scanning.

This mod does not hand you the galaxy. It tells you which of the stations **you already know** has a
marketeer aboard, makes sure that scanning that station actually produces the unlock lead, and keeps
a list so you do not have to.

## Install

```
./install.sh
```

Copies `extension/` into `<X4>/extensions/drjele_black_market_leads`. `./install.sh --uninstall`
removes it. The game path is found automatically, or set `X4_PATH`.

Requires X4: Foundations 9.00. Works on an existing savegame. No new game needed.

## What it does

- **Points at the station.** The first time you are in a sector where a station you already know has
  a black marketeer aboard, you get one logbook entry under **Tips**, and optionally a ticker line.
  Clicking the entry opens the map on that station.
- **Makes the scan pay off.** While you are there and the marketeer is still locked, the mod asks the
  vanilla signal leak manager to run a pass on that station, so the unlock lead gets placed instead
  of maybe getting placed. Per-station cooldown, attempt limit and a cap on how many leaks a station
  can be pushed to.
- **Remembers.** A list of the marketeers on stations you know — sector, station, owner, and what is
  currently blocking each one.

## What it deliberately does not do

Only stations the game already considers known to you are ever reported, listed or touched. A
marketeer in a sector you have never visited stays exactly as invisible as it is in vanilla. You
still fly there, you still scan the hull, you still run the delivery mission. That restriction is not
a setting.

## The thing nobody tells you

**A black marketeer can only be unlocked on a station you are allowed to dock at.** The vanilla
unlock mission checks `hasrelation.dock.{faction.player}`; if that fails it reports a setup failure,
the leak's mission table is already empty, and the leak that was just created is destroyed again a
moment later. That is the real reason some stations never reward scanning, however long you spend on
them. The mod never wastes a lead on such a station and says so in the logbook entry and in the list.

## Where to change things

With SirNukes Mod Support APIs installed: **Options → Extension Options → Black Market Leads**.

| Setting | Default | What it does |
| --- | --- | --- |
| Plant a lead worth scanning | on | Seeds the unlock lead on a known station whose marketeer is locked |
| Retry the same station after | 30 min | Cooldown before a second attempt on the same station |
| Logbook entry for a new lead | on | One entry under Tips per station, clickable to the map |
| Ticker line for a new lead | on | Also shows a short ticker message |
| List locked marketeers | on | Show the ones not yet unlocked |
| List unlocked marketeers | on | Show the ones already unlocked |
| Right-click entry on stations | on | Adds the list to a station's interaction menu |
| Write decisions to the debug log | off | Needs the game started with `-debug scripts` |

## Opening the list

Four ways, all showing the same list:

- **Options → Extension Options → Black Market Leads**
- the **`/leads`** chat command
- the right-click menu of a station that has a marketeer aboard
- a hotkey, bindable under the SirNukes hotkey options

On Linux the hotkey backend needs a named pipe server that currently only exists for Windows, so use
one of the other three there.

## Configuring without the menu

Without SirNukes the mod still runs — logbook entries, ticker lines and lead planting all work off
the constants at the top of `extension/md/drjele_black_market_leads.xml`, re-read on every savegame
load, so editing one and reloading is enough. Any other mod or cheat menu can override them at
runtime through `global.$DrJeleBlackMarketForceLeaks`, `$DrJeleBlackMarketForceCooldown`,
`$DrJeleBlackMarketLogbook`, `$DrJeleBlackMarketNotify`, `$DrJeleBlackMarketListLocked`,
`$DrJeleBlackMarketListUnlocked`, `$DrJeleBlackMarketInteract` and `$DrJeleBlackMarketDebugChance`.

Note that SirNukes option values live in `uidata.xml` next to your saves, not in the savegame, so
they are shared by every save on the profile.

## Debugging

Start the game with `-debug scripts -debug error -logfile debuglog.txt` in the Steam launch options.
Turn on **Write decisions to the debug log** and watch for `DrJele black market leads:` lines. The
`/leadsdebug` chat command triggers the game's own dump of every black marketeer and its station.

## Status

Written against X4: Foundations 9.00. Not yet verified in game — see `DEVELOPMENT.md` for the open
questions and the test procedure.

## Development

See [DEVELOPMENT.md](DEVELOPMENT.md).

## Legal

MIT, see [LICENSE](LICENSE). Not affiliated with or endorsed by Egosoft.
