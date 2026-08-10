# Rail Tracking System (RTS) — Technical Reference

BMW rail freight transport management: need → order → confirmation → delivery → loading, with
dunning (Mahnung) and cancellation (Storno). A Power Apps Canvas app, 26 cloud flows, a Dataverse
model in a second environment, and two web masks for external partners.

**As of** 2026-08-10 · **Sources** canvas snapshot `rts_flows_3104/rts-app/` and solution export
`f8/vchild/` (2026-07-27), plus the later single-flow exports for the reminder (07-31) and bulk
(07-29) flows · **Audience** whoever develops this next.

Everything below was read from those files. Where a statement could not be verified against them
it says so.

---

## 1. The two-environment split — read this first

This is the single most important structural fact about RTS, and the one that has caused the most
wasted time.

| | Environment | Contains |
|---|---|---|
| **App env** | `c06da98d-80be-eed0-b9df-c8dfbc600157` (`orgd717881f.crm4`) | The canvas app, and the `UC_otd_3104` / `RTSFlowPackage` flows |
| **Data env** | `b83cf532…` (`orgf7d737ab.crm4`) | The `RTS` solution and all 14 `rts_` tables, including `rts_weeklyplan` |

Flows run in the app environment and reach the data environment through the
`rts_DataverseEnvironment` environment variable. The Dataverse tables are external data sources to
the app — reading the app does not require data-env access.

> **The trap:** several flows exist as **two independent copies** — an app-env copy prefixed
> `3104_flow_` and a data-env copy prefixed `RTS -`. Each has its own On/Off state. Fixing the
> wrong copy changes nothing while looking like a successful edit. F02's active copy turned out to
> be the data-env one. **Before editing any flow, establish which copy is actually running.**

Connection details:

| Thing | Value |
|---|---|
| App ID | `d5c280e7-95da-4ade-8732-f5d177d85538` |
| Solution ID | `c104bb5f-cf96-4c49-be08-e4b8881b118d` |
| Cluster | EU / Prod (`authoring.neu-il104.gateway.prod.island.powerapps.com`) |
| Flow solutions | `UC_otd_3104`, `RTSFlowPackage` |
| Canvas snapshot | `~/rts_flows_3104/rts-app/` |

`pac` environment selection **drifts** — verify with `pac org who` immediately before every
import. An import has gone to the wrong environment before.

---

## 2. The status workflow

`rts_transportstatus` is the spine of the entire system. Every flow either reads a status, writes
one, or both.

**The option values were assigned in creation order and do not follow the label order.** There is
no way to infer one from the other; use this map.

| Value | Status | Value | Status |
|---|---|---|---|
| `914890004` | 00 Offener Bedarf | `914890003` | 14 Bestellbestätigung versendet |
| `914890011` | 01 Bedarf in Prüfung | `914890005` | 21 Lieferung erwartet |
| `914890018` | 02 Bedarf geprüft | `914890006` | 22 Lieferung erfolgt |
| `914890000` | 03 Bedarf versendet | `914890007` | 31 Beladung erfolgt |
| `914890017` | 10 Bestellung geprüft | `914890016` | 32 Beladung erfolgt & kritische Menge unterschritten |
| `914890001` | 11 Bestellung versendet | `914890013` | 41 Lieferung (beladen) erwartet |
| `914890002` | 12 Bestätigung erfolgt | `914890012` | 42 Lieferung (beladen) erfolgt |
| `914890019` | 12.1 Bestätigung durch keine Antwort | `914890010` | 90 Mahnung in Prüfung |
| `914890020` | 12.2 Bestätigung editiert zurückgemeldet | `914890014` | 91 Mahnung versendet |
| `914890021` | 13 Bestätigung geprüft | `914890015` | 92 Zug ausgefallen |
| | | `914890008` | 98 Storno angefordert |
| | | `914890009` | 99 Storniert |

`12.3 – Bestätigung abgelehnt – In Prüfung` exists in the app's `colStatusToView` (`App.pa.yaml`
line 222) from the decline feature. **Its option value is not in the extracted map above** — read
it from the data-env `customizations.xml` before using it in a flow.

### Flow of a transport

```
00 Offener Bedarf
 └─ 01 in Prüfung ─ 02 geprüft ─ 03 versendet          (demand)
     └─ 10 Bestellung geprüft ─ 11 versendet           (order)
         └─ 12 Bestätigung erfolgt                     (supplier confirms)
            ├─ 12.1 keine Antwort   (no reply, auto)
            ├─ 12.2 editiert zurückgemeldet
            └─ 12.3 abgelehnt – in Prüfung
             └─ 13 geprüft ─ 14 Bestellbestätigung versendet
                 └─ 21 Lieferung erwartet ─ 22 erfolgt
                     ├─ 31 Beladung erfolgt                   (full)
                     └─ 32 Beladung, kritische Menge unterschritten
                         └─ 41 Lieferung (beladen) erwartet ─ 42 erfolgt

dunning:  90 Mahnung in Prüfung ─ 91 versendet   (Backuptransport = Ja)
                                └─ 92 Zug ausgefallen (Backuptransport = Nein)
storno:   98 angefordert ─ 99 storniert
```

The 31/32 branch is decided by loaded wagons against the Route's
`Relative Critical Wagon Amount`.

### Status → view grouping

`App.OnStart` builds `colStatusToView`, which is what the Overview screen's tabs filter on. Views
1–8: `1` = 00 · `2` = 01 · `3` = 02/03/10/11 · `4` = 12/12.1/12.2/12.3/13/14 · `5` = 21 · `6` = 22
· `7` = 90/91/92 · `8` = 31/32/41/42.

**Adding a status means touching both this map and the flow that writes it.** A status missing
from `colStatusToView` disappears from the Overview entirely, with no error.

---

## 3. The canvas app

Seven screens plus `App`. Sizes matter here — see §6.1.

| Screen | Size | Role |
|---|---|---|
| `01_StartScreen` | 26 KB | Landing, 4 tiles, DE/EN toggle, role-gated menu |
| `02_OverviewTransportScreen` | 508 KB | Weekly Plans list; filters, pagination, tabs (Alle/Storno/Mahnung), bulk actions |
| `03_SingleTransportScreen` | **1.89 MB** | The single-transport wizard. Drives a record through the status chain via `Patch` |
| `04_ManagmentScreen` | 133 KB | KPI/ratings dashboard — **placeholders, not wired to data** |
| `05_PlanningScreen` | 212 KB | Read-only 3-week rolling grid |
| `06_WeekTransportScreen` | 223 KB | Musterwoche week grid |
| `07_DecisionScreen` | 75 KB | Werk/Jahr/KW selection popup and the "Erstellen" build button |

Roles come from `Shareholders.Role` into `varIsCoE` (Center of Competence) and `varIsDispatchFDP`
(Factory Dispatch Point), set in `App.OnStart`. They gate the menu and which status steps are
editable. A `DeveloperEmails` list bypasses the gates.

Bilingual DE (default) / EN through the `Translations` table (TextID → Text by Abbreviation).

### Data model

The central fact table is **Weekly Plans** (`rts_weeklyplan`, 126 columns) — one record per planned
transport, with quantities tracked per stage: Required → Requested → Ordered → Confirmed → Loaded →
Delivered (+ Destination) + Defective, plus dates, deadlines and comments per stage.

Fourteen `rts_` tables in total. Weekly Plans relates to Routes, Suppliers, Calendars (×2:
TransportDate and OrderDate), Shareholders (×7 responsibility roles), Reasons for Dunning Notices
(×4) and Currency; Dunning Notices point back at it. Routes (the lane) relate to Quelles (Factory
Dispatch Point = source), Destinations and Suppliers, and carry the
`Relative Critical Wagon Amount` that drives the loading branch. Reference Weeks are the recurring
template that generates Weekly Plans.

The full dictionary is in `RTS_Data_Dictionary.html` in this folder.

**Wagon column names are inconsistent and easy to get wrong:** the base fields are
`rts_wagonprofil` (**no trailing `e`**) and `rts_wagontype`; the confirmed variants are
`rts_wagonprofileconfirmed` / `rts_wagontypeconfirmed` (**with** the `e`); delivery variants are
`rts_wagonprofiledelivery` / `rts_wagontypedelivery`. An F14 bug came from reading
`rts_wagonprofile@…` on the base field — always null, then a Parse JSON crash.

### Flow calls from the app

Verified in the 2026-07-27 snapshot — `02` and `03` each call three flows:

| Flow | Called from | Argument |
|---|---|---|
| `3104_flow_RTS-BulkStatusUpdate` | 02, bulk action | `ids`, `target` |
| `3104_flow_RTS-F98` | 02, bulk Storno | `JSON(ForAll(ColBulk As B, {orderId: Text(B.ID)…}))` |
| `3104_flow_RTS-F99` | 02 and 03 | bulk payload |
| `3104_flow_RTS-F92` | 02 and 03 | bulk payload / `varSelectedItem.ID` |
| `3104_flow_RTS-F91MahnungZugausfallNotification` | 03, several call sites | `varSelectedItem.ID` |
| `3104_flow_RTS-12.2DeclineMailanBahn-DLundBedarfseinsteuerer` | 03 | `varSelectedItem.ID` |

**Correction to earlier notes:** F91 *is* wired in the app as of this snapshot. A note from
2026-07-21 said it was not yet called; that has since been done.

---

## 4. The flows

26 flows in the 2026-07-27 export, plus two added after it. Grouped by how they start, because
that is what determines how you test them.

### 4.1 Scheduled

| Flow | Schedule (W. Europe) | Reads → writes |
|---|---|---|
| `RTS-F02VorläufigeBedarfsversendung` | **Weekly**, Wednesday 15:00 | 02 → 03 |
| `RTS-F10Bestellunggeprüft` | Daily 13:00 | 10 → 11 |
| `RTS-F121KeineRückmeldung` | Daily 12:30 | 10 → 12.1 |
| `RTS-F14VersendungBestellbestätigung` | Daily 11:00 | 13 → 14 |
| `RTS-F21NeueLieferungenleererwartet` | Daily | 14 → 21 |
| `RTS-F22BeladungfürWiederbeladungerwartet` | Every 5 min | 22 |
| `RTS-F41LieferungSenke` | Every 5 min | 31/32 → 41 |
| `RTS-UpdateStatus12Polling` | Every 5 min | 11 → 12.2/13 |
| `RTS-ErinnerungWagonbeladung` | Daily 07:00 | 22 — loading reminder to Factory Dispatch Points |
| `SecurityRoleAssignment` | Every 1 min | — |

The polling flow is a fallback for the confirmation flow's async status-update failure window. Its
match criteria were aligned with the confirmation flow on 2026-07-14 (`rts_orderedwagons` =
`rts_confirmedwagons`, and raw option values rather than formatted labels) so the two status
setters can no longer disagree. Whether to retire it was never decided.

### 4.2 Triggered by the app (PowerApps V2)

| Flow | Trigger parameter | Purpose |
|---|---|---|
| `RTS-F01NeueBedarfeinPrüfungNotification` | none | Fires when a record enters 01 |
| `RTS-F90NeueAufträgeinMahnungsprüfung` | none | Dunning intake, writes 90 |
| `RTS-F91MahnungZugausfallNotification` | `text` (TransportID) | Per-supplier dunning, CoC in CC |
| `RTS-122DeclineMailanBahn-DLundBedarfsein` | `text` (TransportID) | Mails Bahn-DL and Bedarfseinsteuerer; **displays** the Confirmed Deadline, does not patch it |
| `RTS-F92` / `RTS-F98` / `RTS-F99` | `text` (Input) | Zug ausgefallen / Storno angefordert / Storniert |
| `RTS-BulkStatusUpdate` | `ids`, `target` | Bulk status change from the Overview |
| `RTS-NeueLieferungleerNotification` | none | |

> **The parameter names are `text`, `ids`, `target` — not the titles.** `describe_api` prints
> `text: String — TransportID`; `text` is the name, `TransportID` only the title. Passing
> `{TransportID: x}` compiles cleanly and is **silently dropped**. Single-parameter flows are
> called positionally in this app — `'…F91…'.Run(varSelectedItem.ID)` — which sidesteps the issue,
> but any flow gaining a second parameter must switch to a record and get this right.

Related trap: `Coalesce(ctrl.Text, "")` passes `Blank()` when the control is empty, because
`Coalesce` skips empty strings too. That breaks a **required** PowerApps V2 trigger input. Either
make the input optional or use a non-empty fallback such as `"—"`.

### 4.3 HTTP-triggered — the external web masks

| Flow | Serves |
|---|---|
| `Authentification` | Token check for the masks |
| `AufträgebestätigenanLogistiksenden` | Supplier order confirmation (rtsapp) |
| `AufträgebestätigenanLogistiksenden-Senke` | Destination confirmation (rtsapp2) |
| `F21R_Lieferungleerbestätigen` | Empty-delivery confirmation, writes 22 |
| `F22R_Beladungbestätigen` | Loading confirmation, writes 31/32/92 |
| `RTS-GetReasons`, `RTS-GetDunningReasons` | Dropdown data for the masks |
| `LinkOpenFlow` | Link resolution |

**Response-flow naming convention:** a flow that receives a partner's answer to notification
`F<NN>` is named `F<NN>.R_…`. Keep it — it is the only thing that pairs a mask with its
notification.

> **HTTP trigger URLs survive re-import** as long as the `workflowid` is unchanged. Re-importing
> with the same ID preserves the URL and signature, so the deployed masks keep working. Change the
> ID and every mask breaks silently.

---

## 5. Email

All eleven email-sending flows share one canonical HTML shell: navy→blue (`#1e3a8a` → `#3b82f6`),
Segoe UI, all-German, no emojis, "Ihr RTS-Logistik-Team" signature.

The shell is the way it is because of Outlook desktop, and each constraint was a bug first:

- **Table layout, not divs.** Outlook's Word engine ignores div backgrounds and `max-width`. The
  shell is an outer `<table width=100%>` around a centred `<table width="600">`; header and footer
  are `<td bgcolor>` cells; buttons are `<td bgcolor>` table-buttons wrapping an inline-styled `<a>`.
- **Inline styles with solid-colour fallbacks.** A CSS `linear-gradient` header rendered white in
  many clients, making the white title invisible. Email-critical styling must be inline.
- **`table-layout: fixed`** plus `word-break` on `th`/`td`, and short German headers in the
  `Select_Display` projection (Auftrag/Lieferant/KW/Wagen/Datum/Profil/Typ/Abgang/Ziel). Nine
  columns of English headers overflowed the card and `overflow: hidden` clipped them.
- **F02 only** runs a wider 1080 px card with proportional column widths and
  `overflow-wrap: break-word; white-space: normal`, so long place names wrap at spaces rather than
  mid-word. The others remain at 600 px.
- **Never put a GUID in a rendered table.** A 36-character unbreakable string forces the table
  wider than the card. Keep `ID` in the main `Select` for the shareholder lookup, and project a
  separate `Select_Display` without it for the table.

Emails go to **all** of a supplier's contacts: `Filter_array_-_Contacts` drops blank `rts_email`,
`Select_-_Contact_Emails` projects them, and `request/to` is `@join(…, ';')`. F10 is the exception
— it has no shareholder lookup and mails hardcoded P3 addresses.

F14 additionally mails each **factory**, resolved by name rather than by ID: transport →
`rts_calculateddeparturename` → `rts_factorydispatchpoints` where `rts_dispatchpointname` or
`rts_factorydipatchnameshort` matches → shareholders on that FDP → drop blanks → send. An earlier
attempt used `$expand=rts_Route(_rts_factorydispatchpoint_value)`, which was **invalid** and made
the whole List return nothing, breaking the flow entirely.

---

## 6. Gotchas

### 6.1 The 512 KB compile limit

`compile_canvas` rejects any file over 512 KB, and it is the only push path. `03_SingleTransportScreen`
is **1.89 MB**, so the MCP cannot push or validate any edit to it — changes must be made by hand in
the Studio formula bar. `02_OverviewTransportScreen` at 508 KB is four kilobytes under the limit;
assume any addition to it will break the ability to push it.

The limit is not configurable. The only real fix is splitting the screen, which is how `06` and
`07` came to exist.

### 6.2 Never hardcode a UTC offset for day boundaries

F14 filtered on `rts_confirmeddeadline ge <yesterday>T23:00:00Z`, which assumes W. Europe = UTC+1.
In summer local midnight is stored at 22:00 UTC, so deadline-day records were excluded and the
List returned empty. Use the DST-aware boundary:

```
ge @{convertToUtc(concat(formatDateTime(convertFromUtc(utcNow(),'W. Europe Standard Time'),
    'yyyy-MM-dd'),'T00:00:00'),'W. Europe Standard Time')}
```

The same class of bug appears when rendering dates: `formatDateTime()` on an app-written date
shows the previous day, because the app stores user-local values. Use
`convertFromUtc(col, 'W. Europe Standard Time', fmt)`.

### 6.3 Parse JSON dies on empty choice columns

An empty choice column makes Dataverse **omit** the `…@OData.Community.Display.V1.FormattedValue`
annotation. A `Select` reading it yields `null`, and a downstream Parse JSON typed `"string"` fails
with "Invalid type. Expected String but got Null" and kills the run. Fix both ends: wrap in
`coalesce(item()?['…FormattedValue'], '')` **and** type the schema `["string","null"]`.

### 6.4 Patch a Dataverse lookup with a bind, never the display text

`rts_Reasonshortdelivery@odata.bind` needs `/entityset(GUID)`. Writing the payload text fails. The
working pattern lists the target row first, then binds:

```
@if(empty(…value), null,
    concat('/rts_reasonfordunningnotices(', first(…)?['rts_reasonfordunningnoticeid'], ')'))
```

Matching by display name did not work; the reliable key is the shared **2-character code**:
`$filter=(startswith(cr983_id,'<code>')) or (startswith(rts_reasonname,'<code>'))`, taking the code
as `substring(concat(<text>,'  '), 0, 2)` — padded so the substring can never error.

### 6.5 Solution import upserts everything in the zip

An unmanaged `pac solution import` upserts **every** component in the archive. Any flow left at its
old version in the zip **overwrites the live one**. Building an import zip by copying an old export
and replacing one flow silently reverts all the others — this reverted the F10 fix three times.

**Always build the import zip from a fresh export taken immediately before the import**, overlay
every currently-patched flow onto it, and verify every previously-patched flow afterwards — not
only the one you touched.

### 6.6 `zip` does not replace entries in place

`zip archive.zip Workflows/x.json` against a pac-exported zip **silently fails to replace** the
entry. The import then succeeds and changes nothing. Repackage properly:

```
unzip base.zip -d build/ && cp patched.json build/Workflows/ && cd build && zip -r ../out.zip .
```

Also: `pac` export/import frequently drops with "The response ended prematurely" while having
**succeeded** server-side. If a retry reports "another Import/PublishAll running", an async import
is in flight — stop importing and poll by re-exporting.

### 6.7 Importing a changed flow deactivates it

A `pac solution import` of a solution containing a flow whose **definition** changed always lands
it in Draft/Off — the `<StateCode>` in `customizations.xml` is ignored. Add **`--activate-plugins`**
(`-ap`) to the import to actually turn it on. Off = StateCode 0 / StatusCode 1; On = 1 / 2.

Deleting an unmanaged solution does **not** delete its flows.

### 6.8 Calendars lookups need a Year filter

`LookUp`/`Filter(Calendars, 'Week Number' = X && 'Day of Week' = Y)` with no `Year` matches that
week in **every** year, producing garbled dates and transports stamped across 2026–2029. Always add
`&& Year = <selected year>`. Wrong-year records already created need manual cleanup.

### 6.9 Input controls latch their value

`NumberInput`/`TextInput` in a gallery freeze once bound — `Reset` cannot reach gallery cells. The
Planning screen's cells were changed to **Labels**, which are reactive, to fix "transports don't
shift on Prev/Next while dates do".

Related: `ShowColumns` over a Dataverse source fails ("CalculatedTransportDate isn't recognized").
Use a `ForAll` record projection instead — which is also how `colPlanWP` was trimmed from 126
columns to 7. Match cells on the scalar `CalculatedTransportDate`, not on `'Transport Date'.Date`:
lookup navigation returns blank on collected records.

### 6.10 Canvas authoring hygiene

The app rides in the solution, so **publish the app before any solution import** or the canvas
edits are wiped. A Studio session refresh can drop the most recent compile batch — sync, verify,
re-apply, and have the app saved in Studio between batches. Studio must be open with an active
co-authoring session or `sync_canvas` returns zero files and zero data sources.

A Classic ComboBox with `SearchFields` over a `Filter(Choices())` throws "no text column" errors
and renders nothing — set `IsSearchable: false`.

---

## 7. Related deliverables

| Thing | Where |
|---|---|
| Data dictionary and Bedienungsanleitung | `RTS_Data_Dictionary.html` in this folder — the customer handover document |
| Weekly Transport Planner | Generative page in the Management Cockpit app (data env), page `302772f8-…`, source at `~/new-genpage` |
| External confirmation masks | `rtsapp` / `rtsapp2`, GitHub `nikolajunserrichter/rtsapp` |

---

## 8. Open items

| Item | State |
|---|---|
| `04_ManagmentScreen` | Charts and galleries are placeholders (`Items: ["",""]`) — never wired to data |
| Status 12.3 option value | Present in the app's view map; its numeric value is not in the extracted status map — read it from the data-env `customizations.xml` |
| F21 change | Flagged as the next piece of work on `F21 Neue Lieferungen (leer) erwartet`; the specific change was never decided |
| F22 response flow | The `F<NN>.R_` pair for F22 was the next one planned after F21 |
| Holidays in the Musterwoche | Transports on holiday dates are still **created**, only the deadline shifts. May need skip or move — customer decision |
| Deadline time | `DeadlineTime<weekday>` is not applied; date only |
| Multi-supplier destination | Model A chosen (a reference-week row per route) over Model B (auto-expand) |
| Deadline status | Uses `02 - Bedarf geprüft` though records are created at `00` |
| F14 factory name match | Assumes `rts_calculateddeparturename` equals the FDP's `rts_dispatchpointname` or `rts_factorydipatchnameshort`. Confirmed working in test; `rts_calculateddeparturename` is a computed column whose formula is not in the solution export |
| Polling flow | Kept as a fallback for the confirmation flow; retirement never decided |
| Cross-project connection reference | The flows carry a connection reference belonging to another project; it travels on export |
