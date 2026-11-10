# Design: club-ready fork of Old World Builder

**Status:** Agreed, final (revision 4). **Implementation has not started and starts only when the owner explicitly gives the go-ahead.**
**Owner:** NGallop
**Last updated:** 2026-11-10

**Revision history:**

- **Revision 4:**
  - All questions decided (§13).
  - Back up / Restore all lists confirmed for Phase 0, kept simple.
  - "Personal tool, keep it simple" added as a design principle.
- **Revision 3:**
  - Dropbox removed.
  - GitHub setup steps added.
  - Magic Resistance and casting/dispel targets added.
  - Rules section added (summaries, plus a full-text toggle kept on the device).
  - Offline support made low priority.
- **Revision 2:**
  - Warriors of Chaos (Wolves of the Sea first) scope.
  - LLM-assisted extraction from the rules index.
  - GitHub issues bug loop.
- **Revision 1:** initial draft.

## 1. Summary

This fork replaces old-world-builder.com for use at the club. There are two pieces of work:

1. **Host the fork** at `https://ngallop.github.io/old-world-builder/` with automatic deploys, so it can be opened on a phone during games.
2. **Redesign the game view's unit cards** so that everything needed mid-game is on the card, with nothing to tap:
   - armour save, ward save and Magic Resistance
   - weapon profiles
   - casting and dispel targets for Wizards
   - the unit's rules, as short summaries or as the full rules text

## 2. Principle, goals, scope and non-goals

**Principle: a personal tool, kept simple.** This fork is mainly for the owner's own games, not a public or widely used product. Wherever there's a choice, take the simplest approach that works at the table. Features can be expanded later if other club members become interested.

**Goals**

- **G1:** The fork is reachable from a phone at a stable URL and redeploys automatically when `main` changes.
- **G2:** Every unit shows a calculated **armour save**, **ward save** and, where it applies, **regeneration** and **Magic Resistance** as part of its profile.
- **G3:** Every weapon a unit carries is shown with its profile (range, Strength, AP and special rules).
- **G4:** Wizards show their casting and dispel bonus, dispel range, and for each of their spells the type, range, casting value and the 2D6 roll needed to cast it.
- **G5:** Every rule on a unit is visible on the card: a one-line summary by default, and the full rules text at the flick of a toggle. The full text is kept on the device once loaded.
- **G6:** When the app can't work something out, the card says so instead of showing a wrong number, and you can override it.
- **G7:** Wrong values are quick to report (GitHub issues) and fix.
- **G8:** The fork stays easy to update from upstream, so dataset fixes keep flowing in.

**Scope: Warriors of Chaos first**

The first release covers **Warriors of Chaos** in all its compositions, with **Wolves of the Sea** as the priority. Both live in [warriors-of-chaos.json](public/games/the-old-world/warriors-of-chaos.json):

- 49 units, 22 of which have Wolves of the Sea-specific entries
- The army's magic items, gifts and mutations: 182 entries across `general`, `warriors-of-chaos`, `gifts-of-chaos`, `chaos-mutations` and `chaotic-traits`
- The spells of the army's lores
- Units a Wolves of the Sea list can take from other armies: Gor Herds and Ungor Herd (Beastmen mercenaries)

Other armies come later (Phase 7), once Warriors of Chaos is working well at the table. Until then, units from other armies show the new layout with `?` wherever data is missing.

**Non-goals (for now)**

- Armies other than Warriors of Chaos (Phase 7).
- Dropbox sync. It is removed, and lists move between devices as JSON files (§4.3).
- Offline loading of the app. It is low priority and will be revisited after testing at the club (Phase 6).
- Changes to list building, points or validation.
- Changes to the Print page or text export.
- Old dataset versions (`1.5/`) and legacy WHFB lists.
- Real translations of new UI strings. English placeholders go into all seven i18n files, as the repo's convention requires.

## 3. What exists today

- **Game view:** [GameView.jsx](src/pages/game-view/GameView.jsx). Each unit shows its name and points. Below that is one comma-separated line from `getAllOptions()` ([unit.jsx](src/utils/unit.jsx)) that mixes weapons, armour, options (Shields, Mark of Nurgle), command, mount and magic items. Each name has an ⓘ that opens its rules index page in a pop-up. Special rules, the stat table ([Stats.jsx](src/components/stats/Stats.jsx)), generated spells ([GeneratedSpells.jsx](src/components/generated-spells/GeneratedSpells.jsx)), notes and victory-point buttons follow.
- **Stat lines** come from the bundled [rules-index-export.json](src/components/rules-index/rules-index-export.json) through `getStats()`. That file maps names to rules index URLs (tow.whfb.app), and holds M–Ld stat lines and troop types for units.
- **No save values, weapon profiles or spell casting values exist in the app's data.** Datasets record kit only as display names. [magic-items.json](public/games/the-old-world/magic-items.json) has names, points and types but no effects. [lores-of-magic-with-spells.json](src/assets/lores-of-magic-with-spells.json) has spell names and their order in each lore, but no casting values. All of the numbers live only in the rules index pages.
- **Kit names are inconsistent free text.** Across all datasets there are 356 distinct weapon option names and 56 armour option names. For example:
  - Compound options: `"Hand weapons, Great weapons, Shields"`
  - Flavour names in either order: `"Heavy armour (Scaly skin)"`, `"Scaly skin (Heavy armour)"`
  - Bare values: `"4+"`, `"No armour"`
  - Shields appear as an armour option, a weapon option or a separate option
- **Some effects are inside special rules or options:** `Chaos Armour (6+)`, `Armoured Hide (1)`, `Regeneration (5+)`, and Mark of Tzeentch, which grants Magic Resistance (-1).
- **A rules-text fetcher already exists.** [rule-descriptions.js](src/utils/rule-descriptions.js) fetches rules index pages in the browser for the Print page and caches them in `localStorage`. That cache is:
  - wiped on every app version bump
  - capped at 150 entries
  - sharing storage space with your lists

  So it isn't durable enough for the game view.
- **Hosting:** `yarn deploy` pushes `build/` to a `gh-pages` branch, and `public/CNAME` claims `old-world-builder.com`. The app assumes it is served from the root of a domain.
- **Upstream-specific services are hard-coded:** the Sentry DSN ([index.jsx](src/index.jsx)) and Sentry Vite plugin ([vite.config.js](vite.config.js)), and a Dropbox app that only works on old-world-builder.com. Dropbox is wired into [App.jsx](src/App.jsx) and [Header.jsx](src/components/page/Header.jsx).
- **Lists are stored per website and per device** (`localStorage`). Lists can already be moved one at a time: the Export page's **Download** button saves a JSON file, and the Import page reads it back.

## 4. Part 1: Hosting

### 4.1 GitHub Pages, deployed by GitHub Actions

The site will live at **`https://ngallop.github.io/old-world-builder/`**. Hosting is free, every push to `main` deploys automatically, and you can add it to your phone's home screen like an app. A custom domain can be added later without redoing anything.

**Setup on GitHub (one-off, done by you).** These are settings on your fork, so they can't be done from code. Each takes a few clicks:

1. **Settings → Pages → Build and deployment → Source:** choose **GitHub Actions**.
2. **Settings → General → Features:** tick **Issues**. They are off by default on forks, and bug reports need them (§9).
3. **Actions tab:** click **"I understand my workflows, go ahead and enable them"**. GitHub disables workflows on new forks.

After the first deploy, the live URL is shown under **Settings → Pages**.

**Code changes.** These are needed because the site lives in a sub-folder (`/old-world-builder/`), not at the root of a domain, and because some upstream-only parts must go:

| Change | Why | Files |
| --- | --- | --- |
| Set Vite `base` to `/old-world-builder/` | Built assets load from the sub-folder | [vite.config.js](vite.config.js) |
| `BrowserRouter basename={import.meta.env.BASE_URL}` | Routes work under the sub-folder | [App.jsx](src/App.jsx) |
| Default `baseUrl` in the fetcher becomes `import.meta.env.BASE_URL`; same for the dataset download link | Army datasets are fetched from `/games/…` | [fetcher.js](src/utils/fetcher.js), [CustomDatasets.jsx](src/pages/custom-datasets/CustomDatasets.jsx) |
| `pathSegmentsToKeep = 1` in `404.html`; check the matching script in `index.html` | Refreshing a deep link such as `/game-view/abc` still works | [public/404.html](public/404.html), [index.html](index.html) |
| Update the manifest `start_url` and `scope`, and `og:url` | The home-screen icon opens the fork | [public/manifest.json](public/manifest.json), [index.html](index.html) |
| Delete `public/CNAME` | Otherwise the deploy claims old-world-builder.com | [public/CNAME](public/CNAME) |
| Remove Sentry | Otherwise your errors are reported to the upstream author's Sentry project | [index.jsx](src/index.jsx), [vite.config.js](vite.config.js) |
| Remove Dropbox (§4.2) | It can't work on the fork | [App.jsx](src/App.jsx), [Header.jsx](src/components/page/Header.jsx) |
| Add `.github/workflows/deploy.yml`: test, build, deploy to Pages | Automatic deploys on push to `main` | new file |
| A small "fork" marker in the header or title | Avoids confusion with the official site | `index.html` / header |

### 4.2 Dropbox: removed

The Dropbox login button and the auth hook come out of the UI. To keep upstream merges easy, the Dropbox code file stays in the repo, unused, and isn't deleted.

### 4.3 Moving lists: JSON export and import

This already exists for single lists:

- **Export a list:** open it, go to **Export → Download**, and the list is saved as a JSON file.
- **Import a list:** use **Import** on the home page and pick the file.

Use this once to bring your lists over from old-world-builder.com, and afterwards to move a list from PC to phone.

**Addition (agreed, Phase 0): Back up / Restore all lists** in Settings.

- **Back up** downloads every list (and its folders) as a single JSON file.
- **Restore** loads that file back and skips lists that are already present, matched by list ID.
- **Why:** without Dropbox, your lists only exist in one browser, and clearing site data would lose them. A one-file backup also makes moving everything to a new phone a single step.
- **Deliberately simple:** no merging of changed lists, no scheduled backups and no cloud storage. These can be added later if needed.

### 4.4 Offline: low priority

Once the app has loaded, the game view works without signal, apart from fetching rules text you haven't loaded before (§7). Loading the app itself without signal needs a service worker. That waits until you've tested at the club (Phase 6).

## 5. Part 2: Game view redesign

### 5.1 Unit card

Mock-ups of the new cards are below.

- **Checked against the rules index while writing this design:** the Chaos Warrior stats, and the Halberd, Shield, Chaos Armour, Mark of Tzeentch, Magic Resistance, Fireball and casting/dispel values.
- **Illustrative only:** the points.
- **Still to be written:** rule summaries shown as `…`. They are produced during extraction (§6.1).

**A regiment:**

```
10 Chaos Warriors ⓘ                                               [250 pts]

 Model          M  WS BS  S  T  W  I  A  Ld │ Sv   Ward  MR
 Chaos Warrior  4   5  3  4  4  1  4  1   8 │ 4+   6+    -1
 Champion       4   5  3  4  4  1  4  2   8 │ 4+   6+    -1
 Sv 4+ = Heavy armour 5+, Shield +1 · Ward 6+ = Chaos Armour (6+)
 MR -1 = Mark of Tzeentch

 Weapon        R       S         AP         Rules
 Halberd       Combat  5 (S+1)   -1 (-2*)   –
 Hand weapon   Combat  4 (S)     –          –
 * AP -2 against enemy models the wielder charged this turn

 Rules                                                    [Full text ▸]
 Mark of Tzeentch ⓘ    Flaming Attacks and Magic Resistance (-1). A Wizard
                       joined to this unit gets +1 to Casting rolls while
                       the unit has Unit Strength 10+.
 Chaos Armour (6+) ⓘ   6+ Ward save against any wounds (shown under Ward).
 Magic Resistance ⓘ    Enemy spells targeting this unit: -1 to the Casting roll.
 Close Order ⓘ         …
 Furious Charge ⓘ      …

 Other: Champion, Standard bearer, Musician                              ⚑
```

**A Wizard.** Only the magic block is shown. Profile, weapons and rules appear as above.

```
Sorcerer Lord ⓘ · Level 4 Wizard
 Magic   Cast +2 · Wizardly dispel +2 · Dispel range 24"
 Spell        Type            Range   CV    Cast on 2D6
 Fireball     Magic Missile   24"     8+    6+
```

### 5.2 Behaviour

**Saves and defence**

- **Sv** and **Ward** columns are added to the profile table, in the game view only. `Stats` gets optional extra columns, so the Unit and Print pages are unchanged.
- **Regen** and **MR** (Magic Resistance) columns appear only when they apply to the unit.
  - **MR** uses the highest value in the unit, because the rule isn't cumulative.
- **Saves are calculated per row.** A champion with magic armour, a character and a monstrous mount, or a chariot and its crew can each have different values. Rows that share the rider's save show `–`.
- **A breakdown line sits under the table and is always visible.** It shows how each value was reached.
- **Unknown kit is flagged, never ignored.** Something on the unit that may affect a value but isn't in the data adds a `?` to the value, and the breakdown says what wasn't counted.
- **Conditional effects stay out of the number** and are shown as footnotes (e.g. *Shield: not in combat when using a Requires Two Hands weapon*).
- **Override:** tapping a value lets you type a correction. The value is stored on the unit inside the list (a new `unit.combatOverrides` field, added to the `editUnit` whitelist in [lists.js](src/state/lists.js)) and marked `✎`. Please also report the bug (§9).

**Weapons**

- **There is one row per weapon carried:** selected weapons, default weapons, magic weapons and mount weapons. Mount weapons are labelled with who uses them.
- **Strength is shown resolved** against the model's S, with the modifier in brackets: `5 (S+1)`. AP includes its situational value (`-1 (-2*)`), with the condition as a footnote.
- **Weapons missing from the data** show their name, *profile not in data*, and the rules link.

**Magic (Wizards only)**

- **Level line:** the Wizard's Level, its bonus to Casting rolls and Wizardly dispel rolls, and its dispel range. The bonus is the Level divided by 2, rounded up. Dispel range is 18" at Levels 1–2 and 24" at Levels 3–4.
- **Casting modifiers** from items and rules are included in the bonus where they always apply, and footnoted where they're conditional (e.g. Mark of Tzeentch's +1 inside a unit of Unit Strength 10+).
- **Spells table:** one row per spell the Wizard has. That means the spells selected in the existing generated-spells picker, or all available spells if none are selected yet. Each row shows type, range, casting value and **Cast on 2D6**, which is the casting value minus the bonus.
- **Dispelling** is an opposed roll: the dispel result must *beat* the opponent's casting result. So there's no fixed target, and the card shows the bonus and range instead.

**Rules**

- **Every rule on the unit is listed once:**
  - its own special rules
  - rules from its options, marks, items and mount
  - the special rules of its weapons

  Each is shown with its ⓘ link.
- **Summaries are shown by default.** Each rule gets one line in our own words, written during extraction (§6.1), e.g. *"Re-roll To Hit rolls of 1"* or *"+1 Ld when charging"*. Summaries are bundled with the app, so they're always available, even offline.
- **The full text is a toggle away.** **Full text ▸** on a card switches that card's rules to the full rules index text. The game-view filter menu also gets a **Full rules text** switch that does the same for every card.
- **Full text is kept on your device** (§7). The first time a list's game view opens with signal, the full text of all its rules is fetched and saved. After that, it shows without signal. A small status line in the game view says *Rules saved on this device: 41 / 42*, with a **Refresh** button.
- **The rules sit next to the numbers they produced**, which makes checking a calculated value against its rule quick.

**Everything else**

- **The "options" line becomes "Other".** It contains only what isn't shown elsewhere, such as command, banners and lore. Nothing that's shown today disappears.
- **Report link:** each card has a ⚑ link that opens a pre-filled GitHub issue (§9).
- **New game-view toggles:** **Weapon profiles** and **Rule summaries** (both on by default), and **Full rules text** (off by default). The saves follow the existing **Show stats** toggle, and the magic block follows the existing **Show generated spells** toggle.
- **Phone layout:** the card should fit a 360px-wide screen without sideways scrolling where possible. Stat columns get narrower in the game view, and long rules and footnotes wrap underneath their row.

## 6. Part 3: Data and calculation

### 6.1 Where the numbers and summaries come from: LLM-assisted extraction

The app already knows the rules index page for almost every rule, weapon, item and spell name, through [rules-index-export.json](src/components/rules-index/rules-index-export.json) and [rules-map.js](src/components/rules-index/rules-map.js). The work is turning those pages into data **once, while writing the data**, not live in the app.

**The pipeline:**

1. **Collect names.** A script walks the in-scope dataset (§2), its magic items and its lores. It lists every weapon, armour, option and sub-option, special rule, mount, item and spell name with its rules index URL. Warriors of Chaos has about 390 distinct names that link to a page.
2. **Fetch once and cache locally.** The same script downloads each page at no more than one request per second. It stores the text in a **git-ignored** cache (`.cache/rules-index/`), together with each page's "Last update" date.
3. **Parse the structured pages with a script.**
   - Armour pages: `Armour Value: 4+`
   - Weapon tables: Halberd is `Combat | S+1 | -1(-2) | -` plus a note
   - Spell blocks: Fireball is `Type Magic Missile | Casting Value 8+ | Range 24"`
4. **Interpret the prose with an LLM.** Claude reads every page and writes two things for each:
   - **A summary:** one line in our own words.
   - **Effects:** in a fixed format, e.g. `{ "ward": "X" }`, `{ "magicResistance": -1 }`, `{ "castingModifier": 1, "condition": "…" }`, or `none`.
5. **Review by playing.** You use the result in games, with the full text one toggle away, and report anything wrong (§9). Each fix also adds a test.

**Every data entry records where it came from:**

```json
"mark of tzeentch": {
  "summary": "Flaming Attacks and Magic Resistance (-1). A Wizard joined to this unit gets +1 to Casting rolls while the unit has Unit Strength 10+.",
  "effects": {
    "grantsRules": ["Flaming Attacks", "Magic Resistance (-1)"],
    "castingModifier": { "value": 1, "condition": "Wizard joined to a Mark of Tzeentch unit of Unit Strength 10+" }
  },
  "source": "special-rules/mark-of-tzeentch",
  "sourceUpdated": "2024-02-14",
  "method": "interpreted"
}
```

- `method` is `parsed` or `interpreted`. If something looks wrong, check the interpreted entries first.
- **Rules text itself is never committed.** The repo is public, and upstream links to the index instead of copying it. Only our own-words summaries, the numbers and the links are committed. The full text reaches you through the device cache (§7).
- `sourceUpdated` lets the coverage script flag entries whose page has changed since they were extracted (errata).

**Why not have the app ask an LLM live, at the table?** At the table you need answers that are instant, work offline, cost nothing and are the same every time. A live LLM call would need a connection, an API key and money per lookup, and could give different answers on different days. Extracting once gives the same coverage with none of those problems, and every value can be checked.

### 6.2 Data files (bundled with the app)

The data lives in `src/assets/combat-data/`:

- **`rules.json`:** every special rule, option, mark, armour, item and mount rule. Each entry has a `summary` and `effects`, which may be `none`, and records its provenance. Effects include:
  - armour value set or improved
  - ward
  - regeneration
  - Magic Resistance
  - casting and dispel modifiers
  - rules granted
  - conditions
- **`weapons.json`:** range, Strength, AP (with any situational AP), special rules and notes, plus provenance.
- **`spells.json`:** lore, type, range and casting value, plus provenance.

**Why the data is bundled in `src/` instead of `public/games/`:**

- The game view works offline once loaded.
- No version bump is needed to bust caches.
- New files never conflict with upstream merges.

**Name matching** works on a normalised name:

- Lower-case the name, strip `*` and `{faction}`, and map plurals to singular. This reuses `normalizeRuleName()` and the `synonyms` map.
- Split compound options on `, ` and ` + `.
- For flavour names in brackets, check the text both inside and outside the brackets.
- Read parameters such as `(6+)`, `(1)` and `(-1)` **before** normalising, because `normalizeRuleName()` strips anything in brackets.
- Prefer faction-specific entries where the index has them, e.g. `chaos armour warriors of chaos` over `chaos armour daemons of chaos`.

### 6.3 Calculation

A new pure function in `src/utils/combat-profile.js`, so it can be tested without the UI:

```js
getCombatProfile(unit, { armyComposition }) => ({
  rows: [{ name, stats, save, ward, regen }],     // each: { value, breakdown[], unknown[], conditional[] }
  magicResistance: { value, breakdown[] },
  weapons: [{ name, carriedBy, range, strength, strengthResolved, ap, apSituational, rules, notes, known }],
  magic: { level, castBonus, dispelBonus, dispelRange, conditional[], spells: [{ name, type, range, castingValue, castOn2D6 }] },
  rules: [{ name, url, summary, known }],
  other: [ /* what's left for the "Other" line */ ],
})
```

**For each profile row:**

1. **Collect active kit:** equipment, armour, options and sub-options, command magic items, unit items, the mount and its options, and special rules (the unit's, its composition-specific ones, and the mount's). Reuse the filtering `getAllOptions()` already does. Expand rules granted by other rules (e.g. Mark of Tzeentch grants Magic Resistance (-1)).
2. **Base armour** is the best "set" value among the sources. With no armour, the base is **7+ for the purposes of improvements**, from the Armoured Hide rule. An unimproved 7+ is shown as `–`.
3. **Apply improvements:** shield, barding, Armoured Hide (which also improves the rider's save), magic items and so on.
4. **Clamp to the maximum armour value for the row's troop type:** 2+ for infantry and cavalry; 3+ for chariots, monsters and war machines.
5. **Ward and Regen:** use the best single source of each.
6. **Record every step** for the breakdown. Anything relevant that isn't in the data goes into `unknown[]`. Conditional modifiers go into `conditional[]` and stay out of the value.
7. **Apply the override** if one is set.

**For the unit as a whole:**

- **Magic Resistance:** the highest value among the unit's models.
- **Magic (Wizards):**
  - The Level comes from the existing `getUnitWizardryLevel()`.
  - Bonus = ⌈Level ÷ 2⌉, plus any modifiers that always apply.
  - Dispel range comes from the Level.
  - For each spell, `castOn2D6` = casting value − cast bonus.
- **Rules:** the de-duplicated list of every rule from step 1, plus weapon special rules, with summaries from `rules.json`.

### 6.4 Coverage script

`scripts/combat-data-coverage.js` lists every name in the in-scope datasets, items and lores that has no entry in `rules.json`, `weapons.json` or `spells.json`, grouped by army. It also lists entries whose rules index page has changed since extraction. The target for Phase 5 is nothing missing for Warriors of Chaos.

## 7. Full rules text on your device

**What it does:**

- **Fetched in the browser** from tow.whfb.app, the same pages the ⓘ pop-up shows. The site allows this from any website (checked: `access-control-allow-origin: *`), so it works from the fork's address.
- **Kept in IndexedDB**, the browser's larger on-device database, not in `localStorage`. This keeps it away from your lists' storage space.
- **Keyed by rules index URL**, not app version, so it survives app updates. An entry is re-fetched only when you press **Refresh** or when the page's "Last update" date is newer than the one stored.
- **Fetched automatically for the whole list** when its game view opens with signal. A few requests run at a time, and only for rules not already saved.
- **Turned into simple, safe content** before it is stored and shown: paragraphs, lists and tables only. Third-party HTML is never injected into the page.
- **When the text isn't saved yet and there's no signal**, the card shows *"Full text not saved on this device yet. Open this list once with signal."* The summary is still shown.

**Implementation:** extend the existing [rule-descriptions.js](src/utils/rule-descriptions.js) parser into a new module (e.g. `src/utils/rules-text-store.js`), leaving the Print page's behaviour unchanged.

## 8. Rules questions

Most of these were answered by reading the rules index while writing this design. The rest are answered during extraction in the same way. This table records what the calculator does, and bug reports correct it.

| # | Question | Answer | Source |
| --- | --- | --- | --- |
| R1 | Base armour values | Full plate 4+ confirmed; light and heavy armour are extracted the same way | `weapons-of-war/full-plate-armour` |
| R2 | Shield | Improves armour by 1. Not usable in combat with a Requires Two Hands weapon, so this is a footnote | `weapons-of-war/shield` |
| R3 | Barding | Answered during extraction | `weapons-of-war/barding` |
| R4 | Armoured Hide (X) | Improves armour by X, including the rider's. No armour counts as 7+ for improvements | `special-rules/armoured-hide` |
| R5 | Bare value in a dataset (`"4+"`) | Used as the base armour | dataset convention |
| R6 | Chaos Armour (X+), Warriors of Chaos | Ward save X+. Also lets Wizards wear armour, which doesn't affect saves | `special-rules/chaos-armour-warriors-of-chaos` |
| R7 | Regeneration (X+) | Separate Regen value; details answered during extraction | `special-rules/regeneration` |
| R8 | Several ward or regen sources | Answered during extraction | — |
| R9 | Magic armour alongside mundane armour | Answered during extraction from each item's page | per item |
| R10 | Maximum armour value | 2+ for infantry and cavalry; 3+ for chariots, monsters and war machines | `weapons-of-war/maximum-armour-value` |
| R11 | Cavalry: one save for rider and mount? | Answered during extraction | — |
| R12 | Character on a monster or chariot | Answered during extraction | — |
| R13 | Magic Resistance (-X) | -X to the Casting roll of enemy spells targeting the unit. Not cumulative: use the highest | `special-rules/magic-resistance` |
| R14 | Mark of Tzeentch | Flaming Attacks and Magic Resistance (-1). +1 to Casting rolls for a Wizard joined to a Mark of Tzeentch unit of Unit Strength 10+ | `special-rules/mark-of-tzeentch` |
| R15 | Casting result | 2D6 + ⌈Level ÷ 2⌉; the spell is cast if the result equals or beats the casting value | `magic/casting-roll-casting-result-and-casting-value` |
| R16 | Dispel | Wizardly dispel: 2D6 + ⌈Level ÷ 2⌉. Fated dispel: 2D6 only, once per turn. Either must **beat** the casting result. Range 18" (Levels 1–2), 24" (Levels 3–4) | `magic/dispel-roll-and-dispel-result`, `magic/wizardly-dispel`, `magic/fated-dispel` |
| — | Example magic item: Talisman of Protection | Ward save 5+ | `magic-item/talisman-of-protection` |
| — | Example spell: Fireball | Magic Missile, 24", casting value 8+ | `spell/fireball` |

## 9. Bug reports and fixes (GitHub issues)

1. **Report from the card.** The ⚑ link opens a new issue on `NGallop/old-world-builder` with the title and body already filled in:
   - unit name, army and composition
   - the unit's kit
   - the calculated values with their breakdown
   - the data entries used (with `source` and `method`)
   - the app version

   You add one line (e.g. *"Should be 3+, Chaos Steed has barding"* or *"Furious Charge summary is wrong"*) and submit it. Mid-game, use the ✎ override and report later.
2. **Fix it.** In a Claude Code session, ask it to "fix issue #N". It:
   - re-reads the source page
   - corrects the data entry or the calculation
   - adds the reported unit as a test case
   - opens a PR that closes the issue
3. **Deploy.** Merging the PR redeploys the site within a few minutes.

## 10. Delivery plan

Each phase is a separate PR into the fork's `main` and deploys automatically on merge.

| Phase | Scope | Done when |
| --- | --- | --- |
| **0. Fork goes live** | GitHub setup (you, §4.1); hosting changes; Sentry and Dropbox removed; fork marker; Back up / Restore all lists | The fork opens on your phone at the URL, an imported list opens in the game view, and refreshing a deep link works |
| **1. Extraction pipeline and saves** | Name collection, fetch and cache, and coverage scripts; `rules.json` save and Magic Resistance effects for Warriors of Chaos (Wolves of the Sea first); Sv / Ward / Regen / MR columns; breakdown line; `?` marker; override; ⚑ report link | Every Warriors of Chaos unit shows its saves (or `?`), and the tests pass |
| **2. Rules on the card** | Summaries in `rules.json`; Rules section; full text kept on the device; toggles; saved-rules status | Every rule on a Warriors of Chaos card shows a summary, and full text after one visit with signal |
| **3. Weapon profiles** | `weapons.json`; weapon table; "Other" line; Weapon profiles toggle | Every mundane weapon in the army shows a full profile |
| **4. Magic** | `spells.json` for the army's lores; Wizard level line; spells table with Cast on 2D6; casting modifiers | Every Warriors of Chaos Wizard shows correct cast and dispel figures |
| **5. Full Warriors of Chaos coverage** | Magic items, gifts, mutations and chaotic traits; mounts; Gor Herds and Ungor Herd | The coverage script reports nothing missing for Warriors of Chaos |
| **6. Offline (low priority)** | Service worker so the app loads without signal, if testing at the club shows it's needed | The app opens in airplane mode after one earlier visit |
| **7. Other armies** | Re-run the pipeline one army at a time | Same as Phase 5, per army |

## 11. Testing

- **Unit tests** go in `src/utils/combat-profile.test.js`, using Warriors of Chaos fixtures copied from the dataset:
  - Chaos Warriors with shields, halberds and Mark of Tzeentch (the mock-up above)
  - Chaos Knights with barding
  - Marauders with no armour plus a shield
  - A Chaos Lord with Chaos Armour, magic armour and a talisman
  - A character on a monstrous mount with Armoured Hide (to check the rider bonus and the caps)
  - A Level 1 and a Level 4 Sorcerer, for cast and dispel figures
  - A unit with an unknown item, which should show `?`
- **Every bug report adds a test** built from the reported unit.
- **Manual check** on a phone at 360px width, including the full-text toggle with signal off.
- `yarn test` runs in the deploy workflow before every deploy.

## 12. Staying in sync with upstream

- Add the original repo as a remote, then merge it in from time to time:
  `git remote add upstream <original repo URL>`, then `git fetch upstream && git merge upstream/main`
- **Keep fork changes mostly in new files:** `combat-data/`, `combat-profile.js`, `rules-text-store.js`, `scripts/` and new game-view components. Edits to existing files stay small: `GameView.jsx`, `Stats.jsx`, `lists.js`, `Header.jsx`, `App.jsx` and the hosting files.
- **Expect a trivial conflict on `package.json` `version`**, because upstream bumps it with dataset changes. Take upstream's value.
- **Fork changes aren't added to the in-app Changelog.** This document and the PR descriptions record them.
- **After each merge, run the coverage script.** It catches new kit names and rules index pages that have changed.

## 13. Decisions

All questions are decided. There are no open questions.

**Decided (2026-11-10)**

| # | Decision |
| --- | --- |
| Q1 | Host on GitHub Pages at `ngallop.github.io/old-world-builder`. You do the one-off GitHub setup in §4.1 |
| Q2 | Drop Dropbox. Move lists with JSON export/import |
| Q3 | Offline loading is low priority; revisit after testing at the club |
| Q4 | Bug reports go to GitHub issues |
| Q5 | Saves are columns in the profile |
| Q6 | Weapon Strength is shown resolved, e.g. `5 (S+1)` |
| Q7 | The breakdown line is always visible |
| Q8 | Nothing is removed from the card. Add Magic Resistance, casting/dispel targets, and rules shown on the card with a toggle for the full text |
| — | Warriors of Chaos first, with Wolves of the Sea as the priority; other armies once that works |
| Q-A | Rules show as summaries by default. Full text is a toggle away, per card or for all cards, and is kept on the device |
| Q-B | Add Back up / Restore all lists in Settings, kept simple (§4.3) |
| — | This is a personal tool: keep it simple, and expand only if other club members become interested |

## 14. Next step

**Implementation starts only on the owner's explicit go-ahead.**

- **Before then:** nothing is built or committed. The GitHub setup in §4.1 can be done at any time before Phase 0's first deploy.
- **After the go-ahead:** work starts with Phase 0 (§10). Each phase is delivered as its own PR for review.
