# QMT Fremdleistung Grossticket — Technical Reference

How the app works after the performance/completeness rebuild: the load pipeline, the three flows,
every area that was touched, the patterns to follow, and the traps that cost time.

**As of** 2026-08-12 · **Source** coauthoring sync from Studio 2026-08-12, flow solution
`UC_enb_266_Perf` v1.0.0.40 · **Audience** whoever develops this app next.

App formulas are quoted from the snapshot. **Flow internals are quoted from the as-built record**,
not from a local file — no flow export exists on this machine. The live flow in
`UC_enb_266_Perf` is the authority; verify before relying on a detail.

---

## 1. Environment and artefacts

| Thing | Value |
|---|---|
| Environment | `4602cbc3-50b5-e450-88fa-4e6255521247` (`org231f32af.crm4`) |
| App ID | `a1725c6c-622f-4c49-aa1a-b4918c045d55` |
| Tenant | `ce849bab-cc1c-465b-b62e-18f07c9ac198` |
| Login | `Santiago.Ruiz@partner.bmw.de` |
| App solution | `UC_enb_266` (unmanaged) — **carries the canvas app**, and `266_flow_ChecklistPatch` |
| Flow solution | `UC_enb_266_Perf` (unmanaged) — the two board service flows only |
| SharePoint site | `/teams/QMTFremdleistung_Grossticket` |
| Lists | `Tickets` (~2.086), `Tickets HIS` (~1.201), plus reference lists |
| Connection reference | `enb266_sharedsharepointonline_ceca8` ("266_con_SharePoint") at author time |

Canvas MCP connect needs `login_hint=Santiago.Ruiz@partner.bmw.de`. `pac` needs `DOTNET_ROOT`,
and its environment selection **drifts** — the QMT profile is index `[5]`. Run `pac org who`
immediately before any import; an import into the wrong environment has happened before.

**Do not re-import `UC_enb_266`.** It contains the canvas app, so importing it replaces the app
and reverts any Studio or MCP edits made since the export.

---

## 2. Architecture

Before the rebuild, `App.OnStart` mass-loaded every ticket into `col_AllTicketLoad` through
CustomID-range chunked paging, and the board filtered that client-side copy. Anything past the
2.000-row delegation ceiling was invisible to filters and search, silently.

Now the app holds no ticket cache at rest. Every board load is one server-side query.

```
App.OnStart              roles, language, menu, theme — no ticket data
        │
OverviewScreen.OnVisible builds colTops (one row per status, Top: 20)
        │                Select(OS_btn_LoadTickets)
        ▼
OS_btn_LoadTickets.OnSelect          ← the single entry point for ALL loading
        │
        ├─ build varFilterQuery      scope clause + 13 filter clauses → one OData string
        │
        ├─ Concurrent(
        │     266_flow_SearchTickets   → varSearchRaw        (JSON array of tickets)
        │     266_flow_GetBucketCounts → varBucketCountsRaw  (JSON array of counts)
        │  )
        │
        ├─ ERROR check → Notify()
        ├─ ClearCollect(col_AllTicketLoad, ForAll(ParseJSON(varSearchRaw) …))
        ├─ ClearCollect(colBucketCounts, …)
        └─ ClearCollect(colActiveFilters, …)   filter chips
        │
        ▼
Galleries filter col_AllTicketLoad by TicketStatus.Value — no further server calls
```

Everything that changes what is on screen ends in `Select(OS_btn_LoadTickets)`. There are nine
call sites: `OnVisible`, the active/history tab, the Meine/Alle toggle, the search box, each
filter control, chip removal, Alle löschen, and after a save or delete.

**Consequence:** if a new control must affect the board, it does **not** get its own query. It
appends a clause inside `OS_btn_LoadTickets.OnSelect` and then calls `Select(OS_btn_LoadTickets)`.

---

## 3. State

### Collections

| Collection | Built in | Contents |
|---|---|---|
| `col_AllTicketLoad` | `OS_btn_LoadTickets` | The loaded tickets, mapped from flow JSON. The only ticket source for every gallery. |
| `colBucketCounts` | `OS_btn_LoadTickets` | `{List, Status, Count}` — server-side total per bucket under the current filter. |
| `colTops` | `OnVisible`, tab button | `{Status, Top}` — per-bucket page size. Drives "Mehr laden". |
| `colActiveFilters` | `OS_btn_LoadTickets` | Filter chips: `{Id, Label, Val}`, filtered to non-blank. |
| `colBatchTickets` | ticket-open handlers | IDs only, fetched server-side for the opened batch. |
| `ColAllowedDepartments` | `App.OnStart` | `Benutzer.Bereich` of the current user — the hard scope boundary in every mode. Sourced from `varCurrentUserRecord`, so role and Bereiche can never come from different rows (§4.1). |
| `ColCurrentLanguage` | `App.OnStart`, lang toggle | `Sprachen` rows for the active language. All UI text resolves through it. |
| `List` | `OnVisible`, accordion | Which buckets are expanded. |

### Variables

| Variable | Meaning |
|---|---|
| `varFilterQuery` | The composed OData `$filter`. Built fresh on every load. |
| `varSearchRaw` / `varBucketCountsRaw` | Raw JSON strings from the two flows. |
| `varselectedBucketType` | `1` = active (`Tickets`), `2` = history (`Tickets HIS`). |
| `varTicketView` | `1` = Kanban, `2` = Liste. Set in `App.OnStart`, so it survives navigation. |
| `varAssignedTo` / `varAppliedAssignedTo` | Meine/Alle as selected vs. as currently loaded. The pair is what makes the panel revert on abandon. |
| `varCurrentUserRecord` | `Benutzer` row — `.Rolle.Value` and `.DienstleisterID.Id` drive the scope clause. |
| `varSearchText` | Debounced search box value. |
| `varCurrentLanguage` | `"de"` / `"en"`. |
| `varSelectedItem` | The ticket opened in the detail panel. |
| `varT0`, `varFlowsMs`, `varTotalMs` | Timing instrumentation. Still present; strip only when no further measuring is planned. |

---

## 4. The load pipeline

`OverviewScreen` → `OS_btn_LoadTickets.OnSelect` (snapshot line ~8999). Read this formula first;
everything else on the screen is downstream of it.

### 4.1 Scope clause

Two independent parts, **AND-ed**, set before any filter clause.

```powerfx
Set(varIsMeine, varAssignedTo = LookUp(ColCurrentLanguage, TextID = "MyTextID").Text);

// Who: ownership (Meine) or Dienstleister (Alle, external only)
Set(varScopeOwner,
    If(varIsMeine,
        "(AuthorId eq {ME} or ZugewiesenAnId eq {ME} or VertreterId eq {ME} or BMWAnsprechpartnerCLQId eq {ME})",
        If(varCurrentUserRecord.Rolle.Value = "Externe Steuerung",
            "DienstleisterId eq " & varCurrentUserRecord.DienstleisterID.Id,
            "")));

// Where: the user's Bereiche — applied in EVERY mode
Set(varScopeBereich,
    If(CountRows(ColAllowedDepartments) = 0,
        "AbteilungShort eq '__KEIN_BEREICH__'",
        "(" & Concat(ColAllowedDepartments,
              "AbteilungShort eq '" & Substitute(ThisRecord.Value, "'", "''") & "'", " or ") & ")"));

Set(varFilterQuery,
    If(varScopeOwner = "",   varScopeBereich,
       varScopeBereich = "", varScopeOwner,
       varScopeOwner & " and " & varScopeBereich));
```

| Mode | Result |
|---|---|
| Meine (any role) | four ownership fields **and** the Bereich chain |
| Alle, `Externe Steuerung` | `DienstleisterId eq <id>` **and** the Bereich chain |
| Alle, internal | the Bereich chain alone |
| No Bereiche resolvable | `AbteilungShort eq '__KEIN_BEREICH__'` — empty board |

`{ME}` is a literal token. The app never resolves the current user's SharePoint ID — the flow
substitutes it (§5.3). That removes a round trip and works because the flows run as the invoker.

Four roles exist (`BMW Admin`, `BMW Steuerung`, `Externe Steuerung`, `User`); the role now branches
in **one place only**, the Alle side of `varScopeOwner`. Meine is role-independent.

**Four rules here are load-bearing, and each was learned by getting it wrong on 2026-08-12.**

**Bereich is AND-ed, never an alternative.** The clause used to be either/or: Meine returned the
ownership fields with *no* department restriction, and `Externe Steuerung` in Alle returned
`DienstleisterId eq <id>` alone — so an external saw every ticket of their Dienstleister across all
Bereiche. Both are now bounded.

*Consequence:* a ticket outside a user's Bereiche is invisible to them even when they created it or
it is assigned to them. That is the chosen rule; it is also the report you will get. Before changing
the clause, check whether the ticket's `AbteilungShort` or the user's `Benutzer.Bereich` is the thing
that is actually wrong.

**The empty case fails closed.** `CountRows(ColAllowedDepartments) = 0` used to yield `""` — no
clause, no restriction, every ticket in the tenant. Any user whose `Benutzer` row could not be
resolved silently saw everything.

**Role and Bereiche must come from the same row.** `App.OnStart` looked the `Benutzer` row up three
times under two keys — `EntraObjectId` for the departments, `Name.Email` for the record. For a
partner UPN the first key missed and the second hit, so the role resolved correctly while
`ColAllowedDepartments` came back empty and the boundary vanished with nothing looking broken. It is
now one lookup with a fallback, and `ColAllowedDepartments` derives from `varCurrentUserRecord`.

**Do not add a guard that skips the chain.** A "skip the scope chain when a Bereich is picked in the
filter" optimisation existed for a few hours to save 257 URI characters. It fired when it should not
have, and `varFilterQuery` came back as `DienstleisterId eq 1` with the boundary silently gone. If
URI length becomes a problem again, shorten the query template in the flow (§9.10) — never the scope.

**Open business decisions.** Meine counts four of the five person fields: `BMWSteuerung` is excluded,
`Author` is included. Neither was explicitly decided. Does Meine mean *my work* or *anything I am
named on*?

### 4.2 Filter clauses### 4.2 Filter clauses

Seventeen clauses, each `If(!IsBlank(...), Set(varFilterQuery, ... & " and " & ...))`.

Controls were renamed on 2026-08-12 to `Filter_<type>_<Feld>`, so the formula now reads as what it
filters. The old `OS_cmb_Filter_<n>` names are gone.

| Control | Clause emitted | Sel. |
|---|---|---|
| `Filter_cmb_Dienstleister` | `DienstleisterId eq <id>` | multi |
| `Filter_cmb_Bereich` | `startswith(AbteilungShort,'<code>')` | multi |
| `Filter_cmb_Abteilung` | `Abteilung eq '<name>'` | multi |
| search box → `varSearchText` | `substringof(...,TicketName) or substringof(...,IPQUmfangs_x002d_ID) or substringof(...,Abteilung) or startswith(ProjektPhase,...)` plus `ID eq <n>` when numeric after stripping `QMT-FL-`, plus `Ticketgroe_x00df_eId eq <id>` for matching Ticketgröße rows | — |
| `Filter_cmb_Prioritaet` | `Prioritaet eq '<value>'` | multi |
| `Filter_rdo_KontrollStatus` | `KontrollStatus eq '<value>'` | single |
| `Filter_rdo_ProjektPhase` | `ProjektPhase eq '<value>'` | single |
| `Filter_cmb_Ticketgroeße` | `Ticketgroe_x00df_eId eq <id>` via `Filter(Ticketgröße, Ticketumfang = …)` | single |
| `Filter_cmb_ZugewiesenAn` | `ZugewiesenAn/EMail eq '<mail>'` | multi |
| `Filter_cmb_BMWQMT` | `Vertreter/EMail eq '<mail>'` | multi |
| `Filter_cmb_BMWCLQ` | `BMWAnsprechpartnerCLQ/EMail eq '<mail>'` | multi |
| `Filter_cmb_ErstelltVon` | `Author/EMail eq '<mail>'` | multi |
| `Filter_dte_ErstelltAmVon` / `…Bis` | `Created ge …` / `Created lt …` | date |
| `Filter_dte_StartdatumVon` / `…Bis` | `StartDatum ge …` / `StartDatum lt …` | date |
| `Filter_dte_EnddatumVon` / `…Bis` | `EndDatum ge …` / `EndDatum lt …` | date |

**All six date clauses use the same form**, and it is not optional:

```powerfx
& "Created ge datetime'" & Text(<picker>.SelectedDate, DateTimeFormat.UTC) & "'"
& "Created lt datetime'" & Text(DateAdd(<picker>.SelectedDate, 1, TimeUnit.Days), DateTimeFormat.UTC) & "'"
```

`Created`, `StartDatum` and `EndDatum` are stored UTC. The original code formatted the *local* date
and appended `T00:00:00Z` / `T23:59:59Z`, which asserts local midnight is UTC midnight — two hours
off in summer, one in winter. "bis 11.08." then included tickets created on 12.08. before 02:00 and
"ab 11.08." dropped those created before 02:00 on the 11th. Wrong rows, never an error.
`DateTimeFormat.UTC` converts local→UTC itself; the half-open `lt <Tag+1>` replaces `le 23:59:59`
and closes the final-second gap.

Filter on `StartDatum` / `EndDatum` (per Zeitraum), never on `Startzeitpunkt` / `Deadline`
(per batch) — §7.7.

Person columns filter on `/EMail` and require the column to be in the flow's `$expand`. This works —
an earlier note claiming it could not be done was wrong; the real cause was dropped flow arguments (§9.1).

> **Known redundancy:** the `ProjektPhase` block appears **twice**, emitting
> `ProjektPhase eq 'x' and ProjektPhase eq 'x'`. Harmless, no wrong results. Delete one when next in the file.

### 4.3 Invocation

```powerfx
Concurrent(
    Set(varSearchRaw, '266_flow_SearchTickets'.Run({
        text:   If(varselectedBucketType = 1, "Tickets", "Tickets HIS"),
        text_1: varFilterQuery,
        text_2: "20",
        text_3: "Created desc",
        text_4: Concat(colTops, Status & ":" & Top, ",")
    }).Result),
    Set(varBucketCountsRaw, '266_flow_GetBucketCounts'.Run({
        text:   varFilterQuery,
        text_1: If(varselectedBucketType = 1, "Tickets", "Tickets HIS")
    }).result)
)
```

Note the response keys differ in case: `.Result` and `.result`. That is not a typo — see §9.2.

### 4.4 Error handling and mapping

```powerfx
If(StartsWith(varSearchRaw, "ERROR"),
   Notify("Suche fehlgeschlagen: " & Left(varSearchRaw, 250), NotificationType.Error, 10000));
ClearCollect(col_AllTicketLoad,
   ForAll(ParseJSON(If(StartsWith(varSearchRaw, "ERROR"), "[]", varSearchRaw)) As t, { … }))
```

A flow failure yields an empty board **plus** a message, rather than an empty board alone.

---

## 5. Flow contracts

Three flows. The two board flows are PowerApps V2-triggered, in `UC_enb_266_Perf`, running
`runtimeSource: invoker` — they execute as the signed-in app user, so SharePoint permissions apply
per user and no elevated identity exists. The third, `266_flow_ChecklistPatch`, sits in
`UC_enb_266` — see §5.4.

### 5.1 `266_flow_SearchTickets`

| Schema name | Title | Meaning |
|---|---|---|
| `text` | ListTitle | `Tickets` or `Tickets HIS` |
| `text_1` | FilterQuery | The OData `$filter` body, minus the status clause |
| `text_2` | Top | Default rows per status |
| `text_3` | OrderBy | `Created desc` |
| `text_4` | StatusesCsv | `Status:Top,Status:Top,…` — per-status override |

Response body key: **`Result`** (capital R). Return schema `![Result:s]`.

The flow derives its status list from `text_4` (defaulting to the active or history set from
`text`) and issues **six parallel** REST calls, one per status, each with
`$filter=<FilterQuery> and TicketStatus eq '<status>'`. `Compose_Union` merges them into one array.
Per-status querying is what keeps every Kanban column populated — a single global `$top` would
starve the later buckets.

Three things about the query are load-bearing and were each paid for in debugging time (§9.10–9.12):

- **`$top=2000` in the URI, trimmed to the requested count in `Compose_Union`** via
  `take(arr, N)`. Querying with the app's `$top=20` directly returns an *empty first page* whenever
  the filter forces a scan — SharePoint answers `200 OK` with `[]` and a `__next` link the flow
  never follows. The app still receives its 20 per bucket; the wide window only exists between the
  flow and SharePoint.
- **`$select=*` plus the expanded target fields**, not a list of ~23 scalar column names. The
  connector's `uri` parameter tops out near 1024 characters and the long form blew past it.
  `$select` cannot be dropped altogether: `$expand` requires every expanded field to be named in it.
- **Every fan-in `runAfter` lists all four terminal statuses.** `Compose_Union` accepting only
  `Succeeded, Failed` meant a `TimedOut` call stranded the whole run with no response at all.

On any non-success the response is a diagnostic string instead of the ticket array:

```
ERROR st=S/S/T/S/S/S U=S urilen=884 filterlen=140 MSG=<SharePoint's message>
```

`st` is one letter per call (**S**ucceeded / **F**ailed / **T**imedOut / **S**kipped / **N**one),
`U` the union's own status, then the composed URI length, the filter length, and SharePoint's
message. The app surfaces it through the existing `StartsWith(varSearchRaw, "ERROR")` path. Keep
this instrumentation — three plausible-but-wrong hypotheses died on its numbers.

### 5.2 `266_flow_GetBucketCounts`

| Schema name | Title | Meaning |
|---|---|---|
| `text` | FilterQuery | Same filter as the board |
| `text_1` | ListTitle | `Tickets` or `Tickets HIS` |

Response body key: **`result`** (lowercase). Shape: `[{List, Status, Count}]`.

Counts are **row counts** — `length(body('Select_N'))`. They were distinct-`CustomID` counts until
2026-08-07; that made a batch collapse to 1 on the right side of the header while the left side
counted rows, so the two halves of `20/538` contradicted each other for ItO batches. Eight
expressions were changed (6 active buckets in `Set_Compose_Active`, 2 in `Set_Compose_Hist`).
If a count ever looks wrong again, check first that no `union(` has crept back in.

`$top=5000` with no `__next` paging: a bucket above 5.000 would undercount. Not reachable today.

### 5.3 `{ME}` substitution

Both flows begin with `HTTP_CurrentUser` (`GET _api/web/currentuser?$select=Id`) →
`Compose_MyId` → `Compose_ScopeSuffix`, which does
`replace(FilterQuery, '{ME}', outputs('Compose_MyId'))`. Because the flows run as the invoker,
`currentuser` is the app user. The app writes the literal `{ME}` and never learns the ID.

### 5.4 `266_flow_ChecklistPatch`

| Schema name | Title | Meaning |
|---|---|---|
| `text` | CLJSON | `JSON(colItOChecklist, JSONFormat.IncludeBinaryData)` |
| `text_1` | SPID | `JSON(colCreatedTicketIDs, …)` |

Response body key: **`status`**; the app checks `varFlowResult.status = "Success"`.

Lives in `UC_enb_266`, not in the perf solution, and is **deliberately switched off in dev** — so
creating an ItO ticket there raises an error after the tickets themselves are written. Ticket
creation happens *before* the call (§7.7), so a Zeitraum test in dev is still valid; only the
Checkliste rows are missing.

---

## 6. Ticket mapping

`ParseJSON` → typed record. The shapes below are what SharePoint's verbose REST actually returns.

```powerfx
{
    ID: Value(t.ID), TicketName: Text(t.TicketName),
    Abteilung: Text(t.Abteilung), AbteilungShort: Text(t.AbteilungShort),
    BatchID: Text(t.BatchID), Anmerkungen: …, Description: …,
    ChecklistFulfilled: Value(…), TotalChecklist: Value(…),
    AnzahlSubTickets: Value(…), EKWNummer: Value(…), EskalationsZ_x00e4_hler: Value(…),

    Created:   If(IsBlank(Text(t.Created)), Blank(), DateTimeValue(Text(t.Created))),
    Deadline: …, StartDatum: …, EndDatum: …, Startzeitpunkt: …, Vorgespraech: …,

    TicketStatus: {Value: Text(t.TicketStatus)},     // choice → plain string
    KontrollStatus: …, ProjektPhase: …, Prioritaet: …, Phase: …,

    Ticketgroe_x00df_e: {Value: ""},                  // ← STILL STUBBED
    Dienstleister: {Value: Text(t.Dienstleister.Name)},   // lookup → .Name via $expand
    Komponente: …, LeadDerivate: …, Lieferaten: …,

    ZugewiesenAn: {Email: Text(t.ZugewiesenAn.EMail), DisplayName: Text(t.ZugewiesenAn.Title)},
    Vertreter: …, BMWAnsprechpartnerCLQ: …, BMWSteuerung: …
}
```

Rules worth knowing before touching this:

- **Choice fields** come back as plain strings, not objects — wrap them yourself.
- **Person fields** are `{EMail, Title}` when filled and `{__deferred}` when empty. `EMail` has a
  capital M. An empty person maps to blank, which is correct.
- **Lookup fields** need `$expand` plus the target subfield. Four are mapped; **`Ticketgröße`
  remains `{Value: ""}`** because its target field was never confirmed. It is fully filterable and
  searchable (both go through the lookup ID), but it renders blank wherever the card shows it.
- Mapping cost is roughly **11 ms per ticket**. Adding fields is not free; see §8.4.

---

## 7. Areas of the app that were changed

### 7.1 Rendering — buckets

Bucket galleries take `Filter(col_AllTicketLoad, TicketStatus.Value = ThisItem.Value && …)`. The
trailing condition is currently a **tautology** (`X in [...] || !(X in [...])`) with the real
predicate commented out — a leftover from an attempt to show only the `T1` row of a batch. Effect
today: **every row is its own card**, so a 3-phase ItO batch renders 3 cards, each labelled `T1-T3`.

Header counter (`OS_lbl_…`, snapshot line ~1501):

```powerfx
CountRows(Filter(col_AllTicketLoad, TicketStatus.Value = ThisItem.Value))   // loaded
& "/" &
Coalesce(LookUp(colBucketCounts,
    List = If(varselectedBucketType = 1, "Tickets", "Tickets HIS") && Status = ThisItem.Value,
    Count), 0)                                                              // total, server-side
```

Both sides count rows. Keep it that way — that symmetry is the 2026-08-07 fix.

"Mehr laden" patches `colTops` for that one status by `+50` and re-runs the load:

```powerfx
Patch(colTops, LookUp(colTops, Status = ThisItem.Value),
      {Top: LookUp(colTops, Status = ThisItem.Value).Top + 50})
```

Initial `Top` is `20` per status, set in two places — `OnVisible` and the tab button.

### 7.2 Rendering — list view

`con_TicketView_List`, visible when `varTicketView = 2`. Columns: `QMT-FL-<ID>`, Erstellt am
(`dd.mm.yyyy`), Projektphase, Abteilung, Enddatum, Lieferant, Komponente. Headers resolve through
`ColCurrentLanguage`. It has its own "Mehr laden" and its own batch-open handler, both duplicating
the Kanban logic — **any change to one must be made in both.**

### 7.3 Ticket open and batch fetch

Both views run the same guard on select:

```powerfx
If( (varSelectedItem.TicketStatus.Value = "Ticket-Pool" || … = "In Vorbereitung")
    && varSelectedItem.ProjektPhase.Value = "ItO"
    && !IsBlank(varSelectedItem.BatchID),
    Set(varBatchRaw, '266_flow_SearchTickets'.Run({
        text: …, text_1: "BatchID eq '" & Substitute(varSelectedItem.BatchID, "'", "''") & "'",
        text_2: "10", text_4: varSelectedItem.TicketStatus.Value }).Result);
    ClearCollect(colBatchTickets, ForAll(ParseJSON(varBatchRaw) As t, {ID: Value(t.ID)})),
    Clear(colBatchTickets))
```

`colBatchTickets` holds **IDs only** — deliberately. Hand-built person records cannot be
reconciled by `Coalesce` against a combobox selection (the "Missing column 'Claims'" error), so
the save re-reads each record from SharePoint instead of trusting a client copy.

The `!IsBlank(BatchID)` guard is load-bearing: without it, a legacy ticket with a blank `BatchID`
matches every other blank-`BatchID` ticket, and the delete button would have removed all of them.

### 7.4 Write paths

- **`OS_btn_NewProject_20`** — save. Iterates `colBatchTickets`, and for each takes previous values
  from `sp: LookUp(Tickets, ID = Loop.ID)` rather than from the client cache. Circuit breaker above
  4 records. No local cache update afterwards — it calls `Select(OS_btn_LoadTickets)` instead,
  costing ~1,3 s but guaranteeing the board matches SharePoint.

  The panel holds **one** date pair, and the save loops the whole batch — deliberately, because
  status, assignee and priority are batch-level. Dates are the exception: they belong to the row the
  user opened.

  ```powerfx
  StartDatum: If(Loop.ID = varSelectedItem.ID, dte_Startdate_1.SelectedDate, sp.StartDatum),
  EndDatum:   If(Loop.ID = varSelectedItem.ID, dte_Deadline_1.SelectedDate, sp.EndDatum),
  ```

  Read as: *for the ticket that was opened take the picker value, for every other row in the batch
  write back what it already had.* Single tickets need no special case — `colToPatch` falls back to
  `Table({ID: varSelectedItem.ID})`, so the condition is true and the picker value is written.

  Before this, the condition was `CountRows(colBatchTickets) > 1`, which asked *"is this a batch?"*
  and wrote the existing value back for **every** row including the opened one — protecting the
  siblings by abandoning editing altogether. That was a stopgap after the original version wrote one
  date pair into all four rows and collapsed a batch's Zeiträume onto whichever phase happened to be
  open.

  `StartDatum` / `EndDatum` are the edited fields for **both** ItO and OtD; `Startzeitpunkt` and
  `Deadline` stay create-only (§7.7). For OtD that means the two pairs drift apart after the first
  edit — nothing in the app reads `Deadline`, but a SharePoint view or report would show the
  original date.

- **`OS_btn_NewProject_21`** — delete. Same guard, same circuit breaker, same reload.

### 7.5 Toggles

- **View** — native control on `varTicketView`; the old `cmp_MD_Toggle_Round_1` component was removed.
- **Language** — `OS_con_LangToggle_1` on the board itself, rebuilding `ColCurrentLanguage` and
  carrying the Meine/Alle scope across the switch (the scope is stored as the localised label, so
  it has to be re-derived).

### 7.6 FormScreen

| Control | Change |
|---|---|
| `FS_cmb_Dienstleister`, `FS_cmb_Component`, `FS_cmb_LeadDerivat`, `FS_cmb_FolgeDerivat`, `FS_cmb_Abteilung`, `FS_cmb_Interresenvertretung_6` | `SearchFields` was `["ComplianceAssetId"]` (a SharePoint system GUID) — typing returned nothing. Now `["Name"]`. |
| `FS_cmb_Interresenvertretung_8` | Still `DisplayFields: ["ComplianceAssetId"]`. `IsSearchable = false`, and the stored value comes from `Self.Selected.Checkliste` in `OnChange`, so this is display-only. Open (§10). |
| `FS_cmb_Lieferant` | Delegation fix; `DisplayFields: ["Name","Nummer"]`. |
| `FS_cmb_Interresenvertretung_7`, `HiddenGal_SubTicket_Dropdown_source` | Delegation fixes using the `With()` pattern (§8.2). |
| 4 dead references to `OS_cmp_MD_Navigation_Multilevel` | Inside `Size = If(…, 7, 7)` → replaced with `= 7`, behaviour-neutral, to unblock compilation. |

**Studio regenerates `SearchFields` whenever a combobox `Items` formula is edited**, and picks a
system column when it cannot infer one. Re-check these six after any `Items` change.

### 7.7 Ticket creation and the ItO Zeitraum split

`SS_tmr_Fade_1.OnTimerEnd` on **SuccesScreen** is the whole create path. The legacy version — one
`Patch` plus a CustomID-chunked reload — is commented out above it.

Order matters and is load-bearing:

```
Clear(colCreatedTicketIDs); Clear(colCreatedTickets); Set(varBatchID, GUID());
  ↓  rebuild colTicketPeriods from the T-pickers          ← must come first
Set(varCreateCount, CountRows(colTicketPeriods));
Set(varStartCustomID, …)                                   ← next CustomID fetched ONCE
ForAll(Sequence(varCreateCount), Patch(Tickets, …))        ← one row per Zeitraum
266_flow_ChecklistPatch (ItO only)
Collect(col_AllTicketLoad, colCreatedTickets)              ← append, do not reload
```

The rebuild has to sit **before** `Set(varCreateCount, …)`. Counting first and rebuilding after
means the loop indexes a different collection than it counted: too few rows and tickets are created
with blank `TP` (blank `StartDatum`/`EndDatum`/`Phase`), too many and the last Zeitraum is silently
dropped, and an empty stale collection creates nothing at all.

**Why the rebuild exists.** `colTicketPeriods` is otherwise built only inside
`FS_dte_Startdate.OnChange` / `FS_dte_EndDate.OnChange`, and none of the eight T-pickers has an
`OnChange` — so a hand-moved boundary would never reach SharePoint. Those handlers also read the
pickers in the same pass that just set the variables the pickers derive from, so they can capture
stale values. Rebuilding at create time fixes both.

**The split** (both Gesamtzeitraum handlers, byte-identical — keep them that way):

| `varMonthDiff` | Zeiträume |
|---|---|
| < 3 | 1 |
| 3 – 5 | 2 |
| 6 – 9 | 3 |
| ≥ 10 | 4 |

`varMonthDiff` is **completed** months:

```powerfx
DateDiff(start, end, TimeUnit.Months) - If(Day(end) < Day(start), 1, 0)
```

Raw `DateDiff(…, Months)` is `(Jahr₂−Jahr₁)*12 + (Monat₂−Monat₁)` and ignores the day entirely, so
15.08 → 01.11 came back as 3 and a range under three months was split in two.

Every period is `RoundDown(varTotalDays / varNumPeriods)` days long **except the last**, which is
pinned to the Gesamtzeitraum end so no day is invented or lost. Periods are contiguous:
`Tn+1.Start = Tn.End + 1 Tag`.

**Editing.** A boundary is one date wearing two hats, so only one side is typeable: the **End**
pickers are editable (except the last period's, which is the Gesamtende), and every **Start**
derives from the preceding End through `DefaultDate`. There is no cascade code — the derivation is
the cascade. `OnChange` on the three editable Ends only clamps against the neighbours, using
`Set` + `Reset` because `SelectedDate` cannot be assigned. Row visibility and the `DefaultDate`
guards all key off `varNumPeriods`, never `varMonthDiff`; mixing the two produced phantom T3/T4 rows
with an inverted range.

Both Gesamtzeitraum handlers reset all eight pickers unconditionally — a `DatePicker` stops
following `DefaultDate` once touched, so without that a manual boundary survives into a split it no
longer belongs to. `OS_btn_Filter_1_Clear_32` offers the same reset on demand.

### 7.8 Supplier search — prefix, delegated

`FS_cmb_Lieferant.Items` filters the `Lieferanten` data source directly:

```powerfx
If(
    Len(Self.SearchText) < 2,
    FirstN(Lieferanten, 0),
    Filter(
        Lieferanten,
        StartsWith(Name, Self.SearchText) || StartsWith(Nummer, Self.SearchText)
    )
)
```

`Filter` + `StartsWith` are delegable, so SharePoint does the matching and the search covers the
**whole list at any size** — the 2.000 limit caps rows returned, not rows searched. `Name` and
`Nummer` are indexed, and `StartsWith` is index-assisted (`BeginsWith` in CAML), so this keeps
working past the 5.000-item list view threshold.

**A caching version existed briefly on 2026-08-11 and was removed on 2026-08-12.** It held
`colLieferanten` as a session cache and searched it in memory with `q in Name || q in Nummer`, which
bought mid-string matching. `Lieferanten` had already reached 2.000 rows, so `ClearCollect` was
silently truncating to the oldest 2.000 by ID: newer suppliers could neither be selected nor created
— the "Neuer Eintrag" duplicate check queries the data source, so it correctly refused them — and
nothing on screen explained the dead end.

**Do not reintroduce mid-string search over a cache.** The only forms that stay complete are
server-side `substringof` in a flow, which itself dies at the 5.000-item threshold because contains
cannot use an index, or a normalised indexed `SearchKey` column queried with `StartsWith` per token.
The business was asked and confirmed prefix search is sufficient.

---

## 8. How to make common changes

### 8.1 Add a filter

1. Add the control to the filter panel.
2. Add a clause in `OS_btn_LoadTickets.OnSelect`, following the multi-select pattern:

```powerfx
If(!IsBlank(Concat(<ctrl>.SelectedItems, "<Column> eq '" & Substitute(<Field>, "'", "''") & "'", " or ")),
   Set(varFilterQuery,
       If(varFilterQuery = "", "", varFilterQuery & " and ")
       & "(" & Concat(<ctrl>.SelectedItems, "<Column> eq '" & Substitute(<Field>, "'", "''") & "'", " or ") & ")"))
```

Three things are doing work here and all three are required: the **`IsBlank` output guard** (an
unmatched selection would otherwise emit `()` and SharePoint returns 400), the **`Substitute` apostrophe
escaping**, and the **`varFilterQuery = ""` test** so the first clause does not start with ` and `.

3. Add the control's `OnChange` → `Select(OS_btn_LoadTickets)`.
4. Add a chip row to `colActiveFilters`.
5. If the column is a person or lookup, confirm it is in the flow's `$select`/`$expand`.
6. Index the column in SharePoint list settings.

### 8.2 Query a source list without truncating it

Isolate the delegable part in `With()` so one non-delegable clause cannot truncate the whole query:

```powerfx
With({ cand: Filter(ChecklisteReferenz, Title = FS_cmb_Interresenvertretung_7.Selected.Value) },
     Filter(cand, Not(Checkliste in ShowColumns(Filter(ColSubTickets, …), Subticket))))
```

The tell for a truncated query is a count of **exactly 500** (or 2.000, depending on the row limit).

### 8.3 Add an active status

The status list is hardcoded in **three places** and `SearchTickets` has a fixed set of branches.
A seventh status added in fewer than all of them is dropped silently, with no error:

1. `OverviewScreen.OnVisible` — the `colTops` array.
2. The active/history tab button `OS_btn_NewProject_13` — the same array again.
3. The bucket accordion / `List` collection.
4. `266_flow_SearchTickets` — the per-status branch set.

### 8.4 Add a field to the card

Flow `$select` (and `$expand` if it is a lookup or person) → app mapping in `col_AllTicketLoad` →
the card template. Every added field costs mapping time on every row of every load.

### 8.5 Change a flow

Export the current solution first, patch the JSON, re-import — and read §9 before doing so. If the
change touches **trigger inputs**, the app must remove and re-add the flow in Studio afterwards to
refresh its cached schema.

---

## 9. Gotchas

These each cost real time. They are ordered by how expensive they were to diagnose.

### 9.1 `.Run()` arguments use raw schema names, not titles

PowerApps V2 inputs are `text`, `text_1`, `text_2`… `describe_api` prints
`[optional] text_1: String — FilterQuery`; `text_1` is the **name**, `FilterQuery` is only the
**title**. Passing `{FilterQuery: x}` **compiles cleanly and is silently dropped** — the flow then
runs on its defaults and appears to ignore the filter. This is what made ProjektPhase, Priorität
and search look broken. Diagnosed by having the flow return its composed URI.

### 9.2 The response key is case-sensitive and differs per flow

`SearchTickets` returns **`Result`**, `GetBucketCounts` returns **`result`**. Power Fx resolves an
unknown property to **blank rather than erroring**, so the wrong case yields an empty collection
that looks exactly like "the flow returned nothing". Symptom: `varSearchRaw` is empty — not `[]`,
not `ERROR:`. Confirm with `describe_api` (`Return schema: ![Result:s]`) before assuming.

### 9.3 A hand-authored solution can import and change nothing

If `customizations.xml` has no `<Workflow>` entry, the flow JSON is unreferenced. The import still
prints "Published All Customizations". **Never trust the import message** — re-fetch `clientdata`
and assert the change is present.

### 9.4 Re-import deactivates the flows

After importing `UC_enb_266_Perf`, flows can come back **Draft** and must be switched on by hand
(there is no `pac flow` command). The board returns nothing until they are. The user has asked
explicitly that flows not be repeatedly switched off — prefer portal edits over re-imports now
that the flows are final.

### 9.5 Invoker connection rewriting

Once a flow is added to the canvas app, Power Apps rewrites its connection to
`runtimeSource: "invoker"`, connection-references key `shared_sharepointonline_1`, logical name
`new_sharedsharepointonline_7c1e7`. When re-editing actions, **keep**
`host.connectionName = shared_sharepointonline_1`. Hardcoding the original
`shared_sharepointonline` breaks it with `InvokerConnectionOverrideFailed`
("could not find connection … in APIM header").

That same error at runtime usually means the app's cached connection-reference set is stale, not
that the flow is broken — remove and re-add the flow in Studio.

**A second shape of the same trap, 2026-08-13.** `266_flow_GetBucketCounts` ended up declaring
**two** connection references to the same connector — `shared_sharepointonline-1` pointing at
`enb266_266_con_SharePointHTTP` (8 actions) and `shared_sharepointonline-2` pointing at the older
`enb266_sharedsharepointonline_ceca8` (**1 action**, `HTTP_Count_5`, the Kontrolle bucket). The app
passes only the connections it knows about in the APIM header, so the orphaned reference could not
be resolved and every run reported

```
Failed to parse invoker connections from trigger 'manual' outputs.
Could not find any valid connection for connection reference name 'shared_sharepointonline' in APIM header.
```

while otherwise working, because the eight correctly-bound actions still ran. Fixed in v1.0.0.41 by
repointing that one action and deleting the orphaned reference.

**How to check:** export the solution and confirm that every action's `host.connectionName` appears
in that flow's `connectionReferences`. A flow should declare exactly one reference per connector.
Note the two flows use different key styles — `shared_sharepointonline_1` (underscore) on
SearchTickets, `shared_sharepointonline-1` (hyphen) on GetBucketCounts. Harmless while each is
internally consistent; mismatched keys *within* one flow are the first thing to check if this error
returns.

Note: `new_sharedsharepointonline_7c1e7` is actually **485_con_SharePoint**, belonging to the R2Q
project. Harmless under invoker, but it is a cross-project dependency that travels on export.

### 9.6 Designer rejects cross-scope references

A top-level action must not reference an action **inside a Condition** (e.g. `outputs('Compose_MyId')`
where `Compose_MyId` sits in an If-branch). The runtime resolves it fine, but the designer refuses
to open with "Definition contains invalid parameters". Route the value through a global variable,
or keep the action at top level.

### 9.7 Canvas authoring

- `compile_canvas` resets runtime collections — galleries are blank until OnStart/OnVisible re-runs.
- `compile_canvas` reports "FAILED" on pre-existing delegation **warnings** but still commits when
  there are 0 errors.
- A Studio session refresh can drop the most recent compile batch. Sync, verify, re-apply, and ask
  for a Save in Studio between batches.

### 9.8 Dates

`Created` is written by the app in user-local time. Formatting it in a flow with `formatDateTime()`
can render the previous day; use `convertFromUtc(col, 'W. Europe Standard Time', fmt)`. The date
filter was tested and works; its residual failure mode is a 1–2 h sliver at each boundary, which
would only surface for a ticket created between 00:00 and 02:00 local. If it ever does, the fix is
`Text(date, DateTimeFormat.UTC)` plus `lt` next-midnight.

### 9.9 A flow that answers with nothing at all

`runAfter` on a fan-in action must list **all four** terminal statuses —
`Succeeded, Failed, Skipped, TimedOut`. `Compose_Union` accepted only the first two, so a single
`TimedOut` call skipped it, skipped `Respond`, and Power Apps received
`502 BadGateway / NoResponse` — a raw gateway error with no message, from a flow that was working
correctly for five of six calls. This is what made combined filters look like a network problem.

### 9.10 The connector's `uri` parameter ends at ~1024 characters

Beyond it the SharePoint connector returns `400 "Uri parameter missing"` — from
`sharepointonline-*.azconn-*`, header `x-ms-plex-failed: 400` — while the run history plainly shows
the Uri. Measured: **884 and 930 succeeded, 1124 failed.**

`GetBucketCounts` is immune (its URI is `$filter` + `$top=5000&$select=CustomID`), which is exactly
why bucket counts stayed correct while the board went empty — the single most misleading symptom in
this app's history.

Levers, in order of value: `$select=*` instead of naming ~23 scalar columns (−277), the Bereich
scope chain when a Bereich is selected (−257), the duplicated `ProjektPhase` clause (−30).

### 9.11 An empty first page is a valid SharePoint answer

With `$orderby` and a filter that cannot be served from an index, SharePoint scans in list order and
may return `200 OK` with `[]` plus a `__next` link. A client that reads `d.results` and stops sees
nothing. The flow queries `$top=2000` and trims with `take(arr, N)` per status instead, so the app
still gets its 20 per bucket. `$top=20` sent straight to SharePoint is what produced "counts say 53,
board shows 0".

### 9.12 `$expand` requires `$select` to name the target fields

Dropping `$select` to shorten the URI fails with *"The query to field 'Dienstleister' is not valid.
The `$select` query string must specify the target fields and the `$expand` query string must
contains Dienstleister."* `$select=*` plus the nested subfields is the short form that works.

---

## 10. Open items

| Item | State | Needs |
|---|---|---|
| **Three board states** | Not built | Loading / error+retry / no-results+reset. Today all three render as an empty board — the ambiguity that made §9.9–9.12 take a full day to isolate. `varLoad` and the `Load` container already exist; add `varLoadError` and two containers, and reuse `OS_btn_FIlter_ClearAll_2` for the reset |
| **Per-phase gallery in the detail panel** | Stopgap only | §7.4's guard prevents the data loss; the panel still cannot edit a phase's own dates. Widen `colBatchTickets` (it maps IDs only; the flow already returns `Phase`, `StartDatum`, `EndDatum`), then an editable gallery, per-row save, and gap/overlap validation against the frozen `Startzeitpunkt`/`Deadline` |
| `Ticketgröße` lookup unmapped | `{Value: ""}` in the mapping | Its target field in SharePoint list settings, then `$expand` + mapping |
| `StartDatum` / `EndDatum` not indexed | 4 new filters run on them | SharePoint list settings |
| `ResetTextID` missing in `Sprachen` | Button renders unlabelled | One row, `de` + `en` |
| "Projekt" column in the list view | Not built | Business decision: Ticketname, Lead-Derivat or IPQ-ID. If IPQ-ID, one flow change |
| Does "Meine" include Ersteller? | Currently yes | Business decision |
| Priorität missing "Urgent" | Cached schema | Studio → Data → Tickets → Refresh. No formula change |
| `FS_cmb_Interresenvertretung_8` display | Shows `ComplianceAssetId` | Choose `Checkliste` (DE) or `ChecklisteEnglish` (EN). It cannot follow the language toggle — `DisplayFields` may not hold a conditional; that would need an `Anzeige` projection on the source gallery |
| View-toggle cosmetics | Unchanged after two syncs | `OS_con_View_List.Fill` → `Color.Transparent`; gradient `Height` → `Parent.Height+1`; tooltips still read "Neues Ticket" |
| Duplicate `ProjektPhase` clause | Benign | Delete one |
| SharePoint indexes | 7 added | Confirm Priorität, KontrollStatus, ProjektPhase, Abteilung and the person columns |
| Instrumentation | Present | Strip `varSearchRaw`/`varT0`/`varFlowsMs`/`varTotalMs` when measuring is finished — keep the `ERROR:` path |

**Latent, no action yet:** `FS_cmb_BMWSteuerung` truncates above 500 `Benutzer` rows (at 387);
`GetBucketCounts` `$top=5000` is unpaged; the status list lives in three places; legacy ItO batches
have `AnzahlSubTickets = 0` and fall back to counting loaded rows.

**Explicitly dropped — do not re-raise:** `Gallery6_1.ShowScrollbar`; German `HintText` literals on
FormScreen (~24 placeholders, pre-existing, out of scope); removing the cross-project connection
reference (risk without runtime gain).

---

## 11. Measurements

| Scenario | Rows | Flows | Total | Client-side |
|---|---|---|---|---|
| Alle, no filter | 120 | 1.738 ms | 3.623 ms | 1.885 ms |
| Alle, one filter | 120 | 1.287 ms | 3.060 ms | 1.773 ms |
| Meine | 4 | 1.257 ms | 1.794 ms | 537 ms |

Model: **~500 ms fixed + ~11 ms per row** client-side. The flow call itself is flat — 200, 500 and
2.000 tickets all returned in ~1.077 ms, so it is latency-bound, not volume-bound. About 1,25 s of
every load is Power Automate invocation overhead and cannot be tuned away.

Implication for future work: **fetching more rows is nearly free; fetching more often is not, and
mapping more fields is not.** Reducing rows below 20/bucket was rejected — completeness is the point.

---

## 12. Change log

| Date | Change |
|---|---|
| 2026-08-03 | `{ME}` token design; `HTTP_CurrentUser` in both flows; `GetBucketCounts` gains `FilterQuery`; per-row append replaced by one `union()` per status |
| 2026-08-05 | Row limit → 2000; 3 delegation fixes; 7 SharePoint indexes; apostrophe escaping (10 clauses); search widened; flow error surfacing; `DelayOutput`; filter badge; counts query one list not two; Meine/Alle revert-on-abandon; **P2.1 batch save server-side** |
| 2026-08-06 | Batch delete + `!IsBlank(BatchID)` guard; E2 dead `ClearCollect(List,…)` removed; history-tab regression fixed (`colTops` rebuilt in the tab); `Result` key case restored; designer cross-scope errors fixed |
| 2026-08-07 | Bucket counts → row counts (v1.0.0.30); Ticketgröße filter/search → `Ticketumfang`; language toggle on the board; view toggle → native control; 6 FormScreen `SearchFields` fixed |
| 2026-08-11 | ItO Zeitraum split reworked: bands 1/2/3/4, `varMonthDiff` as completed months, editable End boundaries with derived Starts, clamps, `colTicketPeriods` rebuilt at create time (§7.7). Batch-save guard on `StartDatum`/`EndDatum` (§7.4). Lieferanten contains-search on a session cache (§7.8). **QMT-4 resolved after ten flow versions** — three independent faults (§9.9–9.11) behind one symptom |
| 2026-08-12 | `Created` filter corrected to `DateTimeFormat.UTC` + half-open range; four new `StartDatum`/`EndDatum` filters in the same form; all filter controls renamed to `Filter_<type>_<Feld>`; duplicate `ProjektPhase` clause removed; chip removal consolidated into one button; Bereich scope chain skipped when a Bereich is selected |
| 2026-08-12 | **Scope clause reworked (§4.1)** — Bereich is now AND-ed onto ownership/Dienstleister in every mode instead of being an alternative to it; the empty-Bereiche case fails closed; the URI-saving guard that skipped the chain was removed after it silently dropped the boundary.  consolidated to one  lookup with a fallback, so role and Bereiche always come from the same row |
| 2026-08-12 | Supplier search reverted to delegated `StartsWith` (§7.8) — the cache was truncating at 2.000 rows; business confirmed prefix search is sufficient. Board states: `varLoadError` captured in `OS_btn_LoadTickets`, surfaced in the header label in place of the count. Clear-all chip beside the filter chips; `OS_btn_FIlter_ClearAll_2` no longer races its own reload |
