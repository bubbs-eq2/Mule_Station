# MuleStation

An inventory-management UI for EverQuest II (Live, designed off and tested on TLE Wuoshi. Haven't tested on other server types yet), plus a standalone offline web app that turns the game's inventory exports into a sortable, searchable manifest.

This README is the developer reference. The user-facing install guide is
`MuleStation/readme.txt` (which ships inside the release zip). The deep,
hard-won EQ2 findings live in `MuleStation_NOTES.md`; this file points to it rather
than repeating it.

---

## Overview and architecture

MuleStation has two halves that meet at two plain-text files:

1. **The skin** (EQ2 XML windows). A custom panel of one-click buttons, a transparent
   "Bag Mat" staging backdrop, and two bag-window overrides that add a resize nub. The
   panel's Export button writes your inventory to disk; its launcher buttons open the
   web app.
2. **The web app** (`index.html`, a single self-contained HTML/CSS/JS file). It reads
   the two exported files, merges them, and renders a manifest grid. It is pure
   client-side code: no build step, no dependencies, no network calls except the
   optional wiki-tooltip image lookups.

Nothing automates the live client. The skin only fires built-in slash commands and
opens a local read-only page; the web app only reads files you exported. This keeps
the whole thing clearly inside the EULA.

### Data flow

```
  in-game  --Export Snapshot-->  <Server>_<Char>_Inventory.txt   (WHERE: layer/bag/slot)
                                  invdump.txt                     (WHAT: tier/level/qty/stacks)
                                        |
                                        v
  web app (index.html):  parseInventory() + parseDump()  ->  build()  ->  render()
                         merge on item name into one row per item
```

The two exports are joined **on item name**. The inventory file knows where each item
sits but not its tier or quantity; the dump knows tier/level/quantity/stack sizes but
not location. Merged, you get the full picture. (Name-join is approximate for an item
split across multiple stacks; see Deferred milestones.)

---

## File map

Release files (inside `MuleStation/`, all ship in the zip):

| File | Role |
|------|------|
| `eq2ui_custom.xml` | Registry. `<include>`s the two custom windows under `Page Name="Custom"`. |
| `eq2ui_skininfo.xml` | Makes the skin appear in `/loadui` (author/version/description). |
| `custom_mulestation.xml` | The main panel: three zones plus the command-emitter pages. Holds the two hardcoded web-app paths. |
| `custom_bagmat.xml` | The transparent Bag Mat backdrop (3 large zones, 3 small zones). |
| `eq2ui_inventory_bag.xml` | Override of the default Bag window; adds the resize nub. |
| `eq2ui_inventory_shared_tradeskill_bag.xml` | Same override for the Shared Tradeskill Bag. Near-identical to the above; keep them in sync. |
| `index.html` | The offline manifest web app. Lives inside the skin folder by design. |
| `readme.txt` | User-facing install guide. |

Repo-only files (NOT in the zip):

| File | Role |
|------|------|
| `README.md` | This developer reference. |
| `MuleStation_NOTES.md` | Field notes and gotchas; the source of truth for EQ2 behaviour. |

---

## The skin

### Registration and summoning

There is no auto-scan for `custom_*.xml`. A custom window must be registered in
`eq2ui_custom.xml` with an `<include>`, then summoned by joining the registry page
name to the window name: `/show_window Custom.MuleStation`. See NOTES section 4.

### The command-emitter pattern

EQ2 has no "run this slash command on click". Instead, a button's `OnPress` makes a
hidden child `<Page>` visible; that page's `OnShow` holds the raw command (no leading
slash) and then re-hides itself so it can fire again. See NOTES section 6. Example
from `custom_mulestation.xml`:

```xml
<Page Name="CmdToggle" Visible="false" OnShow="
    togglebags
    Visible=false" />
...
<Button ... OnPress="Parent.Parent.CmdToggle.Visible=true" />
```

### The hardcoded web-app path

`CmdMgrExt` and `CmdMgrInt` open `index.html` with `browserexternal` / `browser`.
EQ2's `file://` needs an absolute path, so it cannot be relative; the path is
documented prominently in the XML with common-install alternatives as comments, and
"edit this path" is the first setup step in `readme.txt`. The URL is single-quoted on
purpose (see Lessons Learned).

### Encoding (load-critical)

The custom standalone windows (`custom_*.xml`, `eq2ui_custom.xml`,
`eq2ui_skininfo.xml`) **must** be saved UTF-8 with a BOM and CRLF line endings, or
they silently fail to manufacture. The two bag overrides tolerate other encodings but
ship BOM+CRLF too for consistency. See NOTES section 1. `index.html` is a normal web
file and stays UTF-8 / LF.

---

## The web app (`index.html`)

One file: `<style>` tokens and rules, the `<body>` markup, and a `<script>` organized
into labelled sections.

- **Parsing.** `parseInventory()` reads the location export into per-slot occurrence
  records (inferring bag layer greedily from the Bank / Shared Bank name lists).
  `parseDump()` reads the item dump into `name -> {qty, tier, level, stacks, sizes}`.
  `detect()` classifies a loaded file as inventory or dump by its contents.
- **Merge.** `build()` joins the active character's occurrences and dump details into
  `state.rows`, one row per item name, with per-layer counts and a per-bag breakdown.
- **Render.** `render()` draws the sortable table; `renderChips()`, `renderStats()`,
  and `renderSnapshot()` draw the filters, the stats strip, and the header line.
- **Roster.** Per-character data lives in `state.roster`, keyed by server+char. The
  active character is read through the `active()` accessor (single source of truth).
  `finishLoad(preferredKey)` is the shared tail for every load path (file pick, folder
  scan, paste, character removal).
- **Tooltips.** Internally named "icons" but surfaced as "Item tooltips". `probeIcon()`
  walks candidate wiki URLs and caches the hit (or a "none" marker). Off by default;
  the only outside network call.

### localStorage keys

| Key | Contents |
|-----|----------|
| `mulestation` | UI state and the character roster (inventory/dump text, sort, filters, search). |
| `mulestation_config` | Settings (auto-load, dump pattern, stale threshold, tooltips default). |
| `mulestation_icons` | Cached item-name to wiki-image-URL map (and "none" markers). |

### Environment detection

The app sniffs the user agent: EQ2's embedded CEF reports an old Chromium and is not
Firefox, which flags `isInGame`. That gates the file-path features the in-game browser
blocks (Load defaults / auto-load), steering in-game users to Load files, Scan folder,
or Paste.

---

## Deferred milestones

Tracked in `MuleStation_NOTES.md` section 18. Not built; listed so they are not
rediscovered:

- **Pin a stack size to a location** (which of the 48/18 split is in personal vs
  shared). Needs correlating the dump's bag/vector indices to the inventory file's bag
  enumeration, which is undocumented and breaks quietly. Today we give exact per-layer
  counts and the multiset of sizes, but not which size sits where.
- **Cross-character, same-server rollup.** A new view mode, summing an item across all
  same-server characters, which requires shared-bank dedup (the shared bank appears in
  every character's export).
- **Pricing / store-log "trade arm"** (merge `eq2_storelog.txt` on item name).

---

## Rebuild and re-zip

The web app needs no build. To repackage the release:

1. Keep the `MuleStation/` folder contents intact (the six XML files, `index.html`,
   `readme.txt`).
2. Confirm encoding before zipping (this is the easiest way to ship a broken skin):
   every `*.xml` must be UTF-8 + BOM + CRLF; `index.html` stays UTF-8 / LF.
   ```
   for f in MuleStation/*.xml; do head -c3 "$f" | od -An -tx1; file "$f"; done
   ```
   `efbbbf` and `CRLF` on each XML is correct.
3. Zip the folder (zip only, no rar/exe), keeping the `MuleStation/` directory at the
   root of the archive:
   ```
   zip -r MuleStation.zip MuleStation
   ```
4. For an eq2interface submission: the zip must contain `readme.txt`, only modified
   files (no unmodified defaults), and be under the size limit. A screenshot is
   uploaded separately (JPG or GIF, no punctuation in the filename). `README.md` and
   `MuleStation_NOTES.md` stay out of the zip.

---

## Lessons Learned

The EQ2 custom-UI things that cost real debugging time and aren't necessarily poorly documented, just aged (or
documented wrong) elsewhere. Full detail and the test tables are in
`MuleStation_NOTES.md`; this is the short list.

1. **Encoding silently decides whether a window loads.** A manufactured (custom)
   window must be UTF-8 **with a BOM** AND **CRLF** line endings. Either one wrong and
   the XML still parses but the window never appears, with no chat error. This is an
   AND, not an OR. Override files (that modify an existing default window) are lenient,
   but new windows are strict.

2. **`alertlog.txt` is the only trace.** A manufacture failure surfaces nowhere in
   chat. The single signal is a line in `alertlog.txt`
   (`createWindow(): failed to manufacture window <Name>`), written at UI-load time,
   before `/show_window`. Delete it, reload, read it; that is how the encoding rule
   was bisected.

3. **`eq2_recent.ini` overrides `eq2.ini`.** The client writes the last-used skin to
   `eq2_recent.ini` on exit, and that wins at startup. Pick Default once and every
   launch loads Default no matter what `eq2.ini` says. Symptom: "my UI stopped loading
   and restarting doesn't help." Fix: re-select in `/loadui`, or hand-edit
   `eq2_recent.ini` while the client is closed.

4. **A skin needs `eq2ui_skininfo.xml` to show in `/loadui`.** Without it the folder
   can load (if the ini points at it) but never appears in the picker, so you cannot
   re-select it, which feeds directly into the trap above.

5. **No `custom_*.xml` auto-scan.** Custom windows do not load by being present. You
   must register them in `eq2ui_custom.xml` and summon with `/show_window Custom.Name`.
   An early draft assumed an auto-scan convention; there isn't one.

6. **A window's root is `<Page>`, not `<Window>`.** Addressed by joining the registry
   page name: `/show_window Custom.MuleStation`.

7. **No "run command on click"; use the command-emitter pattern.** A button flips a
   hidden child page visible, whose `OnShow` runs the raw command and re-hides itself.

8. **Special characters in an `OnShow` break manufacture.** A raw command containing
   `:` `//` `.` (for example a `file:///` URL) is parsed by EQ2's expression engine at
   manufacture time, and unparseable characters make the whole window fail to
   manufacture. Fix: wrap the literal argument in **single quotes** so it is treated as
   an opaque string, e.g. `browserexternal 'file:///C:/path/app.html'`.

9. **Command reality is narrower than the wikis suggest.** Confirmed working:
   `togglebags`, `finditem`, `dump_items_to_log`, `output_inventory_to_file`,
   `empty_overflow`, `togglebankbags`, `togglehouseinventory`. Not working on the
   current client: `sort_bags` parameters (the family fires but the sort args are
   ignored), and `bag_open`'s numeric argument (it opens bag 1 regardless; negative
   "bank view" args are dead). You cannot open a specific bank/shared bag by command,
   and you cannot move an item between containers by command (drag-and-drop only). Wire
   buttons only to the confirmed list.

10. **The location win is two files joined on name.** `output_inventory_to_file` gives
    container-structured location but no tier/qty; `dump_items_to_log` gives tier/qty
    but no location. Joining them on item name yields the full manifest. The join is
    approximate for items split across multiple stacks, which is why "which stack is
    where" is a deferred milestone rather than a shipped feature.

11. **`file://` works, but the in-game browser blocks local file reads.** `browser`
    and `browserexternal` both take a `file:///` path as a direct argument and open it.
    But the in-game CEF browser cannot `fetch()` local files, so the app's auto-load
    only works when a copy of `index.html` sits in the EverQuest II folder and is
    opened in a normal external browser. In-game, load via the file picker, folder
    scan, or paste. The app detects its environment and steers users accordingly.

---

## Credits

MuleStation is original work, but a lot of people lit the path:

- **Daybreak Game Company**, for EverQuest II and its default UI (the base these
  windows modify; custom UI modding is sanctioned).
- **jeffjl**, whose bag-button mods demonstrated the `OnShow` command-emitter pattern
  and the single-quote technique that the buttons and web-app launchers rely on.
- **ger**, for the original frameless, no-dead-space bag concept behind the resize-nub
  windows and the Bag Mat (concept only, no code).
- **Deathbane27, Jaxel, Sinbad**, for the position-persistent "stay put" bag idea that
  shaped the Bag Mat's location staging.
- **abbelyn**, for the `OnShow` auto-action pattern the emitters follow.
- **uberfuzzy**, for the bag-window action-button precedent.
- Documentation that made the command research possible: the EQ2i wiki
  (eq2.fandom.com), ZAM/Fanbyte, the EQ2Emu wiki, Kendricke's UI writeups, EQ2 Traders
  Corner (Niami Denmother), eq2wire.com, and the official everquest2.com.

If a name is missing or miscredited, it is an oversight, not a slight; let me know and
I will fix it. You guys fucking rock.

## License

Free to use and modify for personal use; a credit back is appreciated for derivatives.
EverQuest II and its default UI assets belong to Daybreak Game Company.
