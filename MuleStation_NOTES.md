# MuleStation EQ2 Custom UI: Field Notes & Gotchas

Hard-won, mostly-undocumented findings from getting a custom standalone window to
load on the live EQ2 client (build DGCBuild=19163L, 2026/6/4). Keep this with the
project.

---

## 1. THE BIG ONE: file encoding is load-critical for custom windows

A custom standalone window (one the client builds via `createWindow`, i.e. anything
you summon with `/show_window`) **must be saved as UTF-8 *with a BOM* AND with *CRLF*
(`\r\n`) line endings.** This is an AND, not an OR; we isolated it with a 4-cell
truth table:

| BOM | line endings | result          |
|-----|--------------|-----------------|
| yes | CRLF         | **manufactures**|
| yes | LF           | fails           |
| no  | CRLF         | fails           |
| no  | LF           | fails           |

If either is wrong, the XML still parses, but the window **silently fails to
to manufacture**: nothing appears, no chat error. The *only* trace is a line in
`alertlog.txt`:

```
Eq2GuiModule::createWindow(): failed to manufacture window <Name>
```

Nuance: plain **override** files (e.g. `eq2ui_inventory_bag.xml`, which modify an
existing Default window rather than create a new one) tolerate LF/no-BOM. The strict
requirement is on *new* manufactured windows. When in doubt, save BOM+CRLF anyway.

Many text editors default to LF/no-BOM. Notepad++ → Encoding → "UTF-8 BOM", and
Edit → EOL Conversion → "Windows (CR LF)".

---

## 2. Skin activation: `eq2_recent.ini` overrides `eq2.ini`

Two files can set `cl_ui_skinname`:
- `eq2.ini`: the manual override you create.
- `eq2_recent.ini`: written by the client on exit; **this one wins at startup.**

So if you ever `/loadui` and pick Default, the client writes `cl_ui_skinname Default`
into `eq2_recent.ini`, and from then on every launch loads Default no matter what
`eq2.ini` says. Symptom: "my custom UI stopped loading and restarting doesn't help."
Fix: re-select your skin in `/loadui`, or hand-edit `cl_ui_skinname` in
`eq2_recent.ini` (while the client is closed).

---

## 3. A skin folder needs `eq2ui_skininfo.xml` to appear in `/loadui`

Without it, the folder loads fine *if pointed at by the ini*, but never shows up in
the Load UI picker, so you can't re-select it, and you're stuck restarting to reload
(see #2 for the trap that creates). The file supplies the Author/Version/Description
shown in the picker. Minimal form:

```xml
<Page Name="SkinInfoPage" ScrollExtent="800,600" Size="800,600" Visible="false">
    <Data author="..." contact="..." description="..." Name="SkinInfo"
          text=":f:MuleStation" version="..."/>
</Page>
```

Once present, you can reload edits fast: `/loadui` → Default → `/loadui` → your skin,
no full client restart needed.

---

## 4. Custom windows do NOT auto-load: register + summon

There is no `custom_*.xml` auto-scan convention (a comment in an early draft claimed
otherwise; it was wrong). To load a custom window:

1. Put the window file in the skin folder.
2. Register it: add `<include>yourfile.xml</include>` inside `eq2ui_custom.xml`,
   whose root is `<Page Name="Custom" ...>`.
3. Summon it: `/show_window Custom.<WindowName>`.

`eq2ui_custom.xml` example:
```xml
<Page IgnoreTab="false" ismodule="true" Name="Custom" ... Visible="false">
  <include>custom_mulestation.xml</include>
  <include>custom_toggle_bag1.xml</include>
</Page>
```

---

## 5. The window's root element is `<Page>`, not `<Window>`

A standalone window is `<Page Name="MuleStation" ...>` and is addressed by joining the
registry page name to it: `/show_window Custom.MuleStation`. (Assuming `<Window>` was
an early wrong turn.)

---

## 6. Running a slash command from a button: the "command emitter" pattern

There's no direct "run command on click." Instead, a button's `OnPress` flips a hidden
child `<Page>` visible; that page's `OnShow` holds the **raw** command (no leading
slash) and then re-hides itself so it can fire again:

```xml
<Page Name="CmdToggle" Visible="false" OnShow="
    togglebags
    Visible=false" />
...
<Button ... OnPress="Parent.Parent.CmdToggle.Visible=true" />
```

Mind the relative path (`Parent.Parent...`); it must resolve from the button up to
wherever the emitter page lives.

---

## 7. `alertlog.txt` is your instrument

Manufacture failures only surface here, at UI-load time (before `/show_window`). When
testing variants, delete `alertlog.txt` first, reload, then read it; the failing
window names tell you exactly what broke. This is how we bisected the encoding rule.

---

## 8. Confirmed native styling vocabulary (from Daybreak's own bag XML)

Safe to use; pulled from `eq2ui_inventory_bag.xml`:
- `<Text ...>label</Text>` with `TextColor="#RRGGBB"`: colored static labels work.
- Fonts: `/TextStyles.Large.LargeStyle`, `/Fonts.FontZapf15`.
- `ShadowStyle="/ShadowStylesNew.Outline.style"`: outline shadow for titles.
- Window background: a child `<Page>` with `RStyleDefault="/rectlist.win_plain"`.
- Buttons: `Style="/CommonElements.SmallPushButton.data.style"`, label via the
  `Text="..."` *attribute* (not inner content). `Color="#RRGGBB"` tints a button.

MuleStation palette (zone identities):
- PERSONAL = teal  `#9ED8C6`
- BANK     = purple `#C9B6E8`
- SHARED   = amber  `#E8C49A`
- Title    = gold   `#F0D080`  (this is Daybreak's own title gold)

---

## 9. Command verification status

- `togglebags`: CONFIRMED.
- `bag_open N` / `bag_close N`: CONFIRMED (tested in-game).
- `sort_bags tier d | name a | level d`: family CONFIRMED; exact arg behavior to retest.
- `finditem`: exists; argument behavior to verify.
- `depositall`: **UNVERIFIED**; placeholder. If the Deposit button no-ops, find the
  real deposit-all command and fix the one `OnShow` string.

---

## 10. Command reality check (current client, verified)

**Confirmed working (safe to wire to buttons):**
- `togglebags`: open/close all personal bags.
- `finditem`: searches ALL inventory: bags, personal bank, shared, house vault.
- `dump_items_to_log`: writes full inventory to the log file. Foundation for the
  offline organizer.
- `empty_overflow`: pushes overflow-window items back into bags.
- `bag_open 1`: opens personal bag 1. NOTE: the argument is effectively ignored on
  this client; it opens bag 1 regardless of the number, and negative args do nothing.

**Does NOT work on this client (don't rely on it):**
- `sort_bags <method> <a|d>`: documented (tier/name/level, a/d), but params don't
  take here: same confirmation dialog, no reorder, even from chat. Demoted.
- `bag_open -3` (personal bank view) / `bag_open -4` (shared bank view): legacy
  2008 syntax, dead on current client.

**Not possible by command at all:**
- Opening a specific bank / shared bag. The bank window (`eq2ui_inventory_bank.xml`)
  has only 3 buttons (window chrome); bag slots are engine-bound `DynamicData`/
  `ActionData` widgets, opened by built-in double-click, not a fireable command.
- Moving an item between containers (the classic two-decade wall). Drag-and-drop only;
  the lone exception is swapping a whole bag onto an empty bag slot.

## 11. The movement lead (bounded, and EULA-sensitive)

The exe extraction shows these protocol commands DO exist in the client:
`/bank_deposit`, `/bank_withdraw`, `/move_item`, `/house_deposit`, `/place_house_item`,
`/pickup`. They are "UI-fired": normally invoked by the client WITH context (item id,
source slot, destination) that a drag operation supplies. Fired blind from a button
they will almost certainly no-op. Determining their argument format to drive movement
is deep reverse-engineering AND lands on the scripted-item-movement EULA line. Park as
a careful future experiment, not a foundation.

## 12. The real organizer is OFFLINE

Rule-based organization belongs outside the client and stays clearly EULA-safe:
`dump_items_to_log` -> parse the log externally -> generate macros / move-lists / a
layout plan. Read your own log, write your own config; never automate the live client.

---

## 13. Big finds from the full exe extraction (1,369 commands)

Commands the wikis never list, pulled from the client binary:

- `output_inventory_to_file` -- THE location win. Writes `<Server>_<Char>_Inventory.txt`
  to the EQ2 root, **container-structured**: `[Bag] name` with `[Bag Slot N]` items,
  plus explicit `[Bank]`, `[Shared Bank]`, `[House Vault]`, `[Overflow]`,
  `[Equipped Items]`, `[Mounts]`, `[Familiars]` sections. It has NO tier/level/qty.
  Pair it with `dump_items_to_log` (which HAS tier/level/qty but no location) and
  join on item name = full picture. Note: name-join is approximate for items split
  across multiple stacks.
- `togglebankbags` -- opens/closes ALL bank bags at once (revives the bank panel).
- `togglehouseinventory` -- opens the house vault.
- `toggleinventory` / `showinventory` -- inventory open/close.
- `destroy_inventory_item` -- destroys an item. Exists; do NOT wire to a button.
- `cl_disable_bag_confirmation_dialog true` -- kills the "Are you sure you want to
  sort?" popup. Set once; also lets you tell if sort is actually reordering.

## 14. In-game browser -> local web app (embedding)

- `browserexternal <url>` -- opens the URL in your SYSTEM browser (full rendering).
- `browser <url>` -- opens the in-game (embedded CEF) browser. Old engine; may not
  render a modern app well. Good for simple pages.
- `cl_browser_homepage`, `cl_enable_browser` -- browser config.
- `file:///` local paths ARE supported by the client (`cl_help_path file:///help/`
  in eq2_default.ini proves it). So a button can open a local HTML file.
- EULA-safe: opening a local read-only page touches nothing in the client.
- CONFIRM in-game whether browser/browserexternal take the path as a direct argument,
  then set the Open Manager button's OnShow path to wherever the web app lives.

## 15. The inventory web app (milestone, deferred)

Data model = output_inventory_to_file (WHERE: layer -> bag -> slot)
            + dump_items_to_log (WHAT: tier, level, qty)
            + eq2_storelog.txt   (VALUE: recent broker sale prices)
Joined on item name into one grid: name, location, tier, level, qty, ~price.
All read-only local files; pure client-side HTML/JS; no game automation.

## 16. GOTCHA: special chars in an OnShow command break manufacture

A raw command in an `OnShow` containing `:` `//` `.` (e.g. a `file:///...` URL) is parsed
by EQ2's expression engine at MANUFACTURE time, not run time -- and unparseable chars
make the whole window fail to manufacture (alertlog: "failed to manufacture window X").
CONFIRMED in-game: single-quoting manufactures fine. FIX: wrap the literal argument in SINGLE QUOTES so it's treated as an opaque string:
    OnShow="browserexternal 'file:///C:/path/app.html'  Visible=false"
(Same trick jeffjl uses: window_close Inventory 'Bag_clone_17_-1'.)

## 17. Browser commands confirmed in-game

- /browserexternal file:///<path>  -> opens in the SYSTEM default app/browser (works).
- /browser file:///<path>          -> opens the in-game CEF browser to the path (works;
  renders blank on plain-text files, will paint real HTML).
Both take the path as a direct argument.

## 18. Item location / split stacks: shipped vs. deferred

SHIPPED (the low-hanging fruit; data was already parsed, just not surfaced):
- Per-LAYER occurrence counts on the location tags (e.g. "Shared x2"). The inventory
  file records every slot occurrence with its bag + layer; build() already counts them
  per layer. Tag shows xN only when N>1.
- Per-BAG breakdown on hover (native title on the location cell), grouped by layer, e.g.
  "Personal: meager harvesting bag  /  Shared: pristine tanned leather backpack x2".
- Retained per-stack SIZES from the dump (StackCount per line, previously summed away).
  Stacks column shows the count; hover title shows "Stack sizes: 48, 18".
- Verified against real Wuoshi_Biornn data: 3 items span >1 layer, 4 have 2+ in one
  layer, 9 have multiple dump stacks. All counts/sizes/breakdowns correct.

DEFERRED FUTURE MILESTONES:

(a) Pin a specific stack SIZE to a specific LOCATION ("the 48 is in personal, the 18 in
    shared"). Needs correlating the dump's BagSlot / VectorPos indices to the inventory
    file's bag enumeration. EQ2's vector indexing is undocumented and differs across
    equipped / bank / shared containers, so the mapping is guesswork that breaks quietly.
    Today we give exact stack COUNTS per layer (free) and the multiset of sizes (free),
    but NOT which size sits where. If attempted: zip dump-stacks to inventory-occurrences
    by sorted slot, BEST-EFFORT only, with a visible caveat. Risk: silent mis-attribution.

(b) Cross-character, same-server ROLLUP ("Adamantine: 30 on BobWeHaddaBaby + 17 shared bank +
    12 on Bubbs = 59 on Wuoshi"). This is a NEW view mode, requiring a pretty significant change:
    - The SHARED BANK is server-wide, so it already appears (redundantly) in EVERY
      same-server character's export. A naive sum double-counts it once per character.
    - Needs SHARED-BANK DEDUP: identify that two characters' shared bags are the same
      physical bank (match by bag name + contents) and count those stacks ONCE per server.
    - Needs a server-level aggregation keyed by server, summing item name across all
      loaded same-server roster entries, plus UI to switch between per-character and
      per-server views.
    - Rows today are per-active-character; a server view changes the app's shape. Scope
      this as its own milestone so the dedup gets proper attention.

(c) Store-log / pricing "trade arm" (Wuoshi_Biornn_eq2_storelog.txt), still deferred;
    architecture stays open (merge on item name later). Unchanged from earlier.

(d) Persistent storage of personal item inventory, along with changes in inventory for further rreporting later? (Maybe useful for trade, or to set alerts whne getting low on specific item?)
