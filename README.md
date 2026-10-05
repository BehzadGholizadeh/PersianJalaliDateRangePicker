# Persian Jalali Date Range Picker — Power BI Custom Visual

A Persian (Jalali / Shamsi), right-to-left date-range picker for **Power BI Report Server** reports that use an
**SSAS Tabular Live Connection**. The calendar is shown in Jalali, and the filter is applied to the Gregorian
column `Dates_Dim[MDate]`.

```
SSAS Tabular (Dates_Dim[MDate], Dates_Dim[Date_ID])
      │  Live Connection
      ▼
Power BI Desktop for Report Server ─► PBIX ─► Power BI Report Server
      │
      ▼
Persian Jalali Date Range Picker
      ├── displays Dates_Dim[Date_ID] (Jalali)
      └── filters  Dates_Dim[MDate]  (MDate >= start AND MDate < end + 1 day)
```

The visual never connects to SQL Server or SSAS directly. It contains no connection strings, makes no network
requests, has no telemetry and loads no remote scripts or fonts. Everything it needs arrives through its two data roles.

---

## Requirements

| Component | Version |
|---|---|
| Power BI Report Server | 1.26.9682.1442 (May 2026) |
| Power BI Desktop for Report Server | 2.154.956.0 (May 2026) |
| SSAS | SQL Server 2025 Analysis Services, Tabular, Compatibility Level 1700 |
| Connection | Live Connection |
| Visual API | **5.3.0** (see "Compatibility notes") |
| Build machine | Node.js 18+ (built and tested with Node 22.22), npm 10+ |

## Build

```bash
npm install
npm test            # unit tests (31)
npm run build       # = pbiviz package + copy to the stable file name (alias: npm run package)
```

Output:

```
dist/PersianJalaliDateRangePicker.pbiviz                     <- import this one
dist/PersianJalaliDateRangePickerA7C31F0E9B2D4.1.1.0.0.pbiviz  (same file, pbiviz's default name)
```

`pbiviz` (powerbi-visuals-tools 7.2.1) is a local dev dependency, so no global install is needed. Use `npx pbiviz …`
for other commands.

Optional browser harness (needs Python 3 and `pip install playwright && playwright install chromium`):

```bash
python3 tests/harness/run_harness.py              # Asia/Tehran by default
HARNESS_TZ=UTC python3 tests/harness/run_harness.py
```

## Development

```bash
npx pbiviz start
```

This serves the visual on `https://localhost:8080` for the **Developer Visual**. Note that the Developer Visual is
a Power BI Service feature and is normally **not available in Power BI Desktop for Report Server**. For this
environment, the practical development loop is: `npm run package` → re-import the `.pbiviz` into Desktop → test.

## Installation (Power BI Desktop for Report Server)

1. Open your report in **Power BI Desktop for Report Server (May 2026)**.
2. In the **Visualizations** pane, click **…** (Get more visuals) → **Import a visual from a file**.
3. Confirm the warning about importing custom visuals, then select `dist/PersianJalaliDateRangePicker.pbiviz`.
4. The calendar icon appears in the Visualizations pane.

No AppSource, Organizational Visuals, Power BI Service or Fabric is needed. The visual is embedded inside the PBIX.
When you save and publish the PBIX to Report Server, the visual travels with it.

> If Report Server's site settings restrict custom visuals, a server administrator must allow them
> (SSMS → connect to Report Server → Server Properties → Advanced → `EnableCustomVisuals` = True).
> This setting is on by default.

## Configuration

Add the visual to the page and bind:

| Field well | Field | Required | Purpose |
|---|---|---|---|
| **Date (Dates_Dim[MDate])** | `Dates_Dim[MDate]` | **Yes** | Gregorian date used by the model relationships. **The filter is applied to this column.** |
| **Jalali Date (Dates_Dim[Date_ID])** | `Dates_Dim[Date_ID]` | Recommended | `YYYY/MM/DD` Jalali text. Used **only** for display and mapping and never filtered. |

Rules:

- Bind the **column** `MDate` itself, not a measure or an aggregation such as "Earliest MDate". If an aggregation
  is bound, the visual refuses to filter and shows a message.
- `Date_ID` must be the Jalali date matching `MDate` on the same row (as in `Dates_Dim`). `Date_ID` values may use
  `/`, `-` or `.` separators and Latin or Persian digits. The 8-digit `Date_Int` form (`13990507`) is also accepted.
- If `Date_ID` is not bound, the Jalali dates are calculated from `MDate` using the built-in fallback algorithm.

### How the calendar is derived

`Dates_Dim` is the authoritative calendar. Every row the visual receives gives one (Jalali date ↔ Gregorian date)
pair, and those pairs are used for display, month lengths (Esfand 29/30) and conversion.

The built-in arithmetic Jalali calendar (Borkowski algorithm, valid for years 1178–1633 AP) is used **only** for
days not present in the received data. Examples are months outside the model's date range, the `Date_ID` role not
being bound, or `Dates_Dim` being cross-filtered by another slicer. The fallback never overrides a mapping that
came from `Dates_Dim`. Days missing from the data stay selectable but are drawn faded.

Power BI delivers the data in windows of 10,000 rows, and the visual requests up to 20 more windows (200,000 rows).
A `Dates_Dim` of several decades (about 365 rows per year) arrives complete.

## Collapsed / expanded behaviour (v1.1)

The calendar is **not** permanently on the page. The visual has two states:

```
 ┌──────────────────────────────┐   click / Enter   ┌──────────────────────┐
 │ 📅 ۱۴۰۵/۰۶/۰۱ ← ۱۴۰۵/۰۶/۳۱  │ ────────────────► │  calendar (expanded) │
 └──────────────────────────────┘ ◄──────────────── │  اعمال / انصراف / Esc │
          collapsed                 apply / cancel  │  پاک کردن             │
                                                    └──────────────────────┘
```

| State | What is shown |
|---|---|
| **Collapsed** (default) | One compact control at the top of the visual. It shows the applied range (`۱۴۰۵/۰۶/۰۱ ← ۱۴۰۵/۰۶/۳۱`, or a single date for a one-day range), or **انتخاب بازه تاریخ** when nothing is applied. The rest of the frame is transparent. |
| **Expanded** | The calendar described below. It opens on the applied range's month with that range preselected, or on the current month if nothing is applied. |

The arrow points from the start date toward the end date. In RTL the start is on the right, so the arrow is `←`.

Transitions:

| Action | Result | Power BI filter |
|---|---|---|
| Click / `Enter` on the collapsed control | Opens the calendar with the applied range loaded as the temporary selection | unchanged |
| Selecting days | Changes only the temporary selection | unchanged (unless Auto Apply) |
| **اعمال** | Validates and normalizes the range (start ≤ end), applies it and collapses. Unchanged range: just collapses. | set (no duplicates) |
| **انصراف** / `Esc` | Discards the temporary selection and collapses to the previous range | unchanged |
| **پاک کردن** | Removes this visual's filter and collapses to **انتخاب بازه تاریخ** | removed |
| Auto Apply on | The second click applies and collapses | set |
| **امروز** | Same as before; the calendar stays open | unchanged (unless Auto Apply) |

If applying or clearing fails, the calendar stays open and shows the error.

### Sizing the visual in the report

A Power BI visual can only draw inside its own frame. It cannot overlay a dropdown on top of neighbouring visuals,
so collapsing does not shrink the frame. Two ways to lay it out:

1. **Recommended: give the visual calendar room** (for example 320×260 or more). The collapsed control sits at the
   top and the rest of the frame stays transparent. Visuals placed beneath it remain visible but are covered while
   the calendar is open.
2. **Bar-sized visual** (for example 300×40). When the frame is smaller than 240×190 and the user opens the picker,
   the visual asks Power BI to show it in **focus mode** (`switchFocusModeState`), which enlarges it to the full
   canvas. Applying or cancelling returns to the report. If the user leaves focus mode with the host's
   "Back to report" button, it counts as **انصراف**. You can turn this off with Format → Buttons → *Open in focus mode
   when visual is small*; the calendar then opens inside the small frame, scrollable but cramped.
   **Focus mode on Power BI Report Server is NOT TESTED.** If it doesn't work there, use option 1.

## Usage

| Action | How |
|---|---|
| Select a range | Click the first day, then the last day. Clicking in reverse order is fine: the range is always normalized so start ≤ end. Clicking twice on the same day selects one day. A third click starts a new range. |
| Preview | While choosing the second day, hovering shows the prospective range. A selection that hasn't been applied shows the **اعمال‌نشده** (not applied) badge and a dot on **اعمال**. In that state the report still uses the previously applied range, which is marked with small dots under its days. |
| Apply — **اعمال** | Sends the filter to the report and collapses. If only a start day is selected, that single day is applied. Applying an unchanged range sends nothing and just collapses. |
| Cancel — **انصراف** / `Esc` | Discards the un-applied selection and collapses to the applied range. |
| Clear — **پاک کردن** | Removes **only this visual's filter**, resets the picker and collapses. Page, report and other visuals' filters are untouched. Safe to click repeatedly. |
| Today — **امروز** | Jumps to the current Jalali month (based on the viewer's computer clock). If nothing is selected, it sets the start to today. If only a start is selected, it sets the end to today (normalized). If a full range is selected, it starts a new range at today. It applies the result only if Auto Apply is on. |
| Navigate | ‹ › for previous/next month and « » for previous/next year. 1404/12 → 1405/01 and back wrap correctly. |
| Auto Apply — **اعمال خودکار** | Format → Buttons → Auto Apply. When it is on, the filter is applied as soon as a range is complete (second click), and the calendar then collapses. The default is off. |

### Keyboard

| Key | Action |
|---|---|
| `Tab` | Collapsed: the range control is a single Tab stop. `Enter`/`Space` opens the calendar and moves focus into the grid. Expanded: Tab moves between the buttons and the day grid, and only one day in the grid is a Tab stop. |
| Arrow keys | Move between days. In RTL mode, ← is the next day and → is the previous day. ↑/↓ move by one week. Moving past the month edge changes the month. |
| `PageUp` / `PageDown` | Previous/next month. With `Shift`, previous/next year. |
| `Enter` / `Space` | Select the focused day or press the focused button. |
| `Esc` | Cancel the un-applied selection and collapse. Focus returns to the range control. |

Every day button has an accessible name such as "انتخاب ۱۴۰۵/۰۶/۱۰". The navigation buttons are named
ماه قبل / ماه بعد / سال قبل / سال بعد, and the month label is announced politely when it changes. High-contrast
mode uses only the host's palette colors.

## Formatting options

| Card | Options |
|---|---|
| General | Background, Transparency, Border, Border color, Border radius, Accent color |
| Header | Show title, Title text (default "انتخاب بازه زمانی"), Font size |
| Date display | Show start date, Show end date, Font size, Separator (default `/`) |
| Calendar | Font size, Cell size, Month header size, Weekday header size |
| Buttons | Show Apply / Clear / Today / Cancel, Auto Apply, Open in focus mode when visual is small (default on) |
| Localization | Persian (default) / English. English switches to LTR, Latin digits and transliterated month names. The calendar stays Jalali. |
| Advanced | Filter date encoding (see below) |

The layout is responsive. At the target sizes (300×200, 400×300, 600×400 and 800×500) the visual shrinks spacing and
fonts, hides the title below 240 px of height, and never scrolls horizontally.

## Filter semantics

For a selected Jalali range `[start, end]` (both inclusive), the visual creates one official `powerbi-models`
**AdvancedFilter** (`http://powerbi.com/product/schema#advanced`) on the target `{ table: "Dates_Dim", column: "MDate" }`:

```
MDate >= start 00:00   AND   MDate < (end + 1 day) 00:00
```

This form includes the whole end day even if `MDate` is a DateTime. The filter is stored in the visual's own
`general.filter` property through `IVisualHost.applyJsonFilter(…, FilterAction.merge)` and removed with
`FilterAction.remove`. That is why Clear can never touch other filters, and why the applied range survives saving,
page navigation and reopening the report. When the visual loads, it reads its stored filter back and shows the range.
If the stored filter can't be interpreted safely, the visual shows a neutral state and a short notice.

The table and column names come from the bound field's `queryName` (`Dates_Dim.MDate`), not its display name.
Renaming the field inside the visual therefore doesn't break filtering.

### Filter date encoding (Format → Advanced)

Internally, every date is an integer day number, so selection, display and range math can't be shifted by the
viewer's time zone (Iran is UTC+03:30). The only time-zone-sensitive point is how the two boundary values are
handed to Power BI:

| Setting | Boundary value sent |
|---|---|
| **Local midnight** (default) | JavaScript `Date` at the browser's local midnight. This matches how Power BI hands `Date` values to visuals. |
| UTC midnight | JavaScript `Date` at 00:00 UTC |
| Naive ISO text | `"2026-08-23T00:00:00"` (no offset) |

**Verify this once in your environment** (see the acceptance test, step 6). If the first or last day of the range
is missing, or an extra day appears at either edge, switch this setting and re-test. No code change is needed.

## Acceptance test (to run in your environment)

1. `npm run package` and import `dist/PersianJalaliDateRangePicker.pbiviz` into Desktop for Report Server.
2. Connect with **Live Connection** to the SSAS 2025 Tabular model (CL 1700).
3. Add the visual, then bind `Dates_Dim[MDate]` → Date and `Dates_Dim[Date_ID]` → Jalali Date.
4. Add a Card (Sales), a chart (Sales by `Dates_Dim[MDate]`) and a table (`Dates_Dim[Date_ID]`, `Dates_Dim[MDate]`,
   Sales). Also add an unrelated slicer, for example on product.
5. Select ۱۴۰۵/۰۶/۰۱ → ۱۴۰۵/۰۶/۳۱ and click **اعمال**.
6. **Boundary check:** the table must show exactly `1405/06/01` (2026‑08‑23) through `1405/06/31` (2026‑09‑22), with
   both edge days present and no extra day. If not, change *Filter date encoding* (above) and repeat.
7. Check that the card, chart and table changed and the unrelated slicer's selection is intact.
8. Run the collapsed/expanded scenarios T1–T5 (see "Test results"), then test **پاک کردن**, **انصراف**/Esc, **امروز** and Auto Apply. Check that a same-day range and Esfand 30 of a leap year work.
9. Save the PBIX, publish it to Power BI Report Server, open it in **Edge** and **Chrome**, and repeat steps 5–8.
   Also check page navigation, browser refresh (F12 console: no errors) and the Network tab (no requests from the visual).

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| "لطفاً ستون تاریخ را به ویژوال اضافه کنید." | Nothing is bound to **Date**. Bind `Dates_Dim[MDate]`. |
| "ستون تاریخ باید یک ستون … باشد" | An aggregate (Earliest/Latest/Count) is bound. In the field well's dropdown, choose **Don't summarize**, or re-add the column. |
| Range is off by one day at an edge | Change Format → Advanced → **Filter date encoding** (see above). |
| Other visuals don't react | Check that `Dates_Dim[MDate]` is the column used in the model's relationships to the fact tables, and that *Edit interactions* doesn't have filtering turned off for those visuals. |
| Days look faded | Those days aren't in the data the visual received. Either they are outside `Dates_Dim`, or another slicer is cross-filtering `Dates_Dim` (adjust Edit interactions). They can still be selected. |
| Jalali dates differ from your official calendar | Bind `Dates_Dim[Date_ID]`. The model's mapping then overrides the built-in algorithm. |
| Import fails with an API version error | The host is older than API 5.3.0. See the compatibility notes below. |
| "فیلتر موجود روی ستون تاریخ قابل نمایش نیست." | The stored filter has an unexpected shape, for example from an older version of the visual. Click **پاک کردن** and apply again. |
| Persian text looks plain | The visual uses local fonts only (Vazirmatn → Segoe UI → Tahoma). Install Vazirmatn on client machines for nicer rendering. Nothing is downloaded. |

## Compatibility notes

- **API 5.3.0 was chosen deliberately.** Microsoft doesn't publish an exact "maximum visual API" for each Desktop for
  Report Server build, and I could not verify it for 2.154.956.0. API 5.3.0 (2023) is well below what a May 2026
  build is expected to support, and still includes everything the visual needs: `applyJsonFilter`/`jsonFilters`,
  the modern Formatting Model (5.1+), `fetchMoreData`, `eventService` and high-contrast palette support.
  The visual does not use any Power BI Service-only API.
- `powerbi-visuals-tools` 7.2.1 builds the package (its minimum supported API is 4.7.0).
- The `FilterAction` / `FormattingComponent` enums are compile-time constants, and the bundle has been checked to
  contain no runtime references to the host's `powerbi` global.

## Project structure

```
src/
  visual.ts       Power BI integration, DOM, state machine (temporary vs applied range)
  calendar.ts     Data-driven calendar model (Dates_Dim authoritative) + 42-cell month grid
  jalali.ts       Fallback Jalali<->Gregorian conversion via Julian Day Numbers
  date-utils.ts   toPersianDigits(), parsing of Date_ID / MDate values, formatting
  range.ts        Pure range logic (start <= end invariant, Today, normalization)
  filter.ts       AdvancedFilter construction and reading back the existing filter
  formatting.ts   Settings + Formatting Model
  i18n.ts         Persian / English strings
  types.ts
style/visual.less
tests/            Unit tests (node:test) + browser harness (tests/harness)
capabilities.json, pbiviz.json, package.json, tsconfig.json, tsconfig.test.json
scripts/copy-package.js
```

`any` is not used in the visual's source. One `null as unknown as IFilter` cast exists in `visual.ts`, because the
API typings don't model the documented `applyJsonFilter(null, …, FilterAction.remove)` call. A second cast exists
because the API types `jsonFilters` as an opaque `powerbi.IFilter`.

## Test results (build environment)

| Suite | Result |
|---|---|
| TypeScript strict compile | Pass |
| ESLint (pbiviz recommended config) | Pass |
| `pbiviz package` | Pass. The `.pbiviz` was produced. |
| Unit tests (`npm test`) | 31/31, run under the UTC, Asia/Tehran, America/Los_Angeles and Pacific/Kiritimati time zones |
| Browser harness: the **packaged** bundle in headless Chromium 141 with a **mock** host | 96/96, run under the Asia/Tehran, UTC and Asia/Dubai time zones. Includes your acceptance tests T1–T5 for the collapsed/expanded behaviour, focus-mode simulation and collapsed layout from 180×32 to 600×400. |
| Bundle audit | No fetch/XHR/WebSocket/beacon, no console logging, no SQL/server strings, `privileges: []` |

The harness is **not** Power BI. It verifies the visual's own behavior and the exact filter payloads it passes to
`applyJsonFilter`. For realistic data it cross-checks `Date_ID` against Chromium's built-in Persian calendar.

### NOT TESTED

The following were **not tested**, because Power BI Desktop for Report Server, SSAS and PBIRS were not available
in the build environment:

- Import into Power BI Desktop for Report Server 2.154.956.0 — **NOT TESTED**
- SSAS 2025 Tabular Live Connection filtering and cross-filtering of other visuals — **NOT TESTED**
- How the host serializes the filter's date boundaries (filter date encoding) — **NOT TESTED**
- `jsonFilters` round-trip (restoring state after save, page navigation and reopening) on the real host — **NOT TESTED**
- Publishing to PBIRS 1.26.9682.1442, and use in Edge/Chrome via PBIRS — **NOT TESTED**
- Collapsed/expanded behaviour (v1.1) inside Power BI Desktop / PBIRS — **NOT TESTED IN POWER BI DESKTOP/PBIRS**
- Focus mode (`switchFocusModeState`) on Desktop for Report Server and PBIRS — **NOT TESTED**

## Known limitations

- The month and weekday names are built in. `Dates_Dim[Month_Name]` / `Day_Name` are not read; the names match the
  standard Persian names.
- The fallback algorithm covers 1178–1633 AP. Outside that range, only days present in `Dates_Dim` are reliable.
- The ranges used are continuous date ranges. Multi-range or relative ranges ("last 30 days") are not supported.
- **Today** uses the viewer's computer clock, not the server's.
- Sync slicers, bookmarks-specific APIs, tooltips and the context menu are not implemented. Bookmarks still capture
  the filter, because it is stored in the visual's `general.filter` property, but this is not tested on PBIRS.
- If the table name contains a `.`, target resolution would split it incorrectly. `Dates_Dim` is not affected.

## Changelog

**1.1.0.0**
- The calendar is no longer always visible. A collapsed range control (or **انتخاب بازه تاریخ**) opens the calendar;
  **اعمال**, **انصراف**/Esc and **پاک کردن** collapse it again. Auto Apply also collapses after applying.
- The Cancel button label is now **انصراف** (it was لغو). Cancel is always enabled while the calendar is open.
- Apply is enabled whenever a range exists. An unchanged range just collapses, without sending a duplicate filter.
- Optional focus mode when the visual frame is too small for the calendar (new setting under Buttons).
- Unchanged: the Jalali calendar, the Dates_Dim mapping, range validation and normalization, and the filter on
  `Dates_Dim[MDate]` (`jalali.ts`, `calendar.ts`, `date-utils.ts`, `range.ts` and `filter.ts` were not modified).
- The version is bumped, so importing the new `.pbiviz` replaces 1.0.0.0 in a report.

**1.0.0.0** — initial version.
