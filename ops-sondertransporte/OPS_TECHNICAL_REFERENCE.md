# OPS-App (Sondertransporte) — Technical Reference

How the EKW name and the Genehmigungsnotiz are generated, why the note format change forces the
note to become write-once, and what is still open.

**As of** 2026-08-19 · **Source** coauthoring sync from Studio 2026-08-19 10:02 · **Audience**
whoever develops this app next.

Two changes handed over on 2026-08-19 after that sync — the blank-Regelspediteur variant of
`_regel` (§6.3) and the QMT-style ordering work — are **recorded as specified, not as verified**.
Re-sync and confirm before relying on them.

---

## 1. Environment and artefacts

| Thing | Value |
|---|---|
| Environment | `c06da98d-80be-eed0-b9df-c8dfbc600157` (shared with RTS) |
| App ID | `d7356740-8ab3-4bee-89ca-dc3ba981d2b0` |
| Solution | `415abd2f-d8e8-4611-8232-4e9adcf3b301` |
| Login | `Santiago.Ruiz@partner.bmw.de` |
| Lists | `Sondertransporte`, `Sondertransporte Bahn`, `Sondertransporte Schiff`, `Spediteure`, `S-TRAM-Raten`, `CHW_Compounds_Häfen_Werke`, `DIB_Version`, `Ratenversion` |
| Customer contact | Weber Christian, TV-612 |

Canvas MCP `connect` needs `login_hint=Santiago.Ruiz@partner.bmw.de`. **The app must be open in
Power Apps Studio for `sync_canvas` to return anything** — a connect against a closed app succeeds
and then syncs zero files, which reads like a broken tool rather than a closed tab. If more than
one Power Apps Studio tab is open, the connector resolves by app ID, so both apps can be synced in
turn without closing anything.

---

## 2. What the app does

Sondertransporte (special transports) for vehicle logistics. A Steuerer raises a transport need,
costs are computed against maintained rates, and an EKW (Einkaufswagen) name and a
**Genehmigungsnotiz** — the approval note — are generated for the purchasing process.

Sixteen screens. Three matter here, one per transport type:

| Role | TransportLKW | TransportBahn | TransportSchiff |
|---|---|---|---|
| Composed name | `txt_final` | `Name_txt_final_Bahn_Name` | `Name_txt_final_Bahn_Name_1` |
| Composed note | `NOTIZ_txt_Final` | `NOTIZ_txt_Final_Bahn_Notiz` | `NOTIZ_txt_Final_Bahn_Notiz_1` |
| Generator open button | `TL_btn_Split_3` | `TL_btn_Split_4` | `TL_btn_Split_5` |

**Every change is three screens.** The popup exists once per transport type because the SharePoint
column limit forced the lists apart. There is no shared component — each screen owns its own copy
of every control, and the `Switch(App.ActiveScreen, …)` branches inside those copies are dead code
on the two screens they do not belong to.

---

## 3. The note pipeline

Three layers of controls, by suffix:

- **`_param`** — the input. Either what the user picks, or a value derived from the transport
  record.
- **`_res`** — the normalised form that goes into the note: `Substitute(Upper(Trim(…)), " ", "")`
  for the hashtag parts, plain text elsewhere.
- **`NOTIZ_txt_Final`** — the composition. One formula, `Default`, which either lists the missing
  fields or emits the finished note.
- **`Notiz_lbl_*_res`** — three labels that render the costs as German currency. They exist
  because `Text(…, "[$-de-DE]#,##0.00")` was unreliable here; the labels format with `en-US` and
  swap the separators with a triple `Substitute`.

`NOTIZ_txt_Final.Default` is the whole contract. It is quoted in §6.

---

## 4. The pipe format, and why the note must become write-once

The note used to be twelve values joined by `|`. Every field found its own value again by counting
pipes:

```powerfx
If(!IsBlank(varEKWNotiz), Index(Split(varEKWNotiz, "|"), 7).Value, <derive>)
```

The 13.08 meeting replaced that format with labelled lines. **The moment the pipes went away the
note stopped being parseable.** Most of the twelve values recompute from the transport record and
do not care. Five do not: `Thema`, `Hintergrund`, `Planstand`, `Konsequenzen` and the new
`Regresspotential` are user input, and they are lost when the popup is reopened.

Therefore the note has to be generated once and then locked. This is not a preference — it is
forced by the format change, and it matches what the customer asked for independently:

> *"wenn ich den speichere, dass alles passt, alles korrekt ist und dann will ich in dem Auftrag
> nichts mehr bearbeiten."*

**The lock is absolute.** Agreed with the Fachbereich before the 13.08 meeting. There is no
correction path: a note generated with a wrong Thema stays wrong, and the only recovery is
recreating the transport order. Do not build a hidden admin override — it defeats the requirement
and nobody has asked for it.

### 4.1 This is currently half-done, and it is visible in production

The composition already emits the pipe-free format. **The parse branches are still in place** —
twelve occurrences across eleven controls:

```
NOTIZ_txt_transmittel_param   NOTIZ_txt_kategorie_param     NOTIZ_txt_Regelspediteur_param
NOTIZ_txt_Subkategorie_param  NOTIZ_txt_Thema_res           NOTIZ_txt_Hintergrund_res
NOTIZ_txt_Gesamtkosten_res    NOTIZ_txt_Mehrkosten_res      NOTIZ_txt_Gesamtmehrkosten_res
NOTIZ_cmb_Planstand_param     NOTIZ_cmb_Konsequenzen_param
```

Reopening a generated EKW today runs `Split` over a string with no pipes. The result is a
single-element table, so **the Transportmittel field receives the entire note text and the other
ten come back blank.** The lock is not in place either, so nothing prevents a user from getting
there.

The two halves must ship in one save. Dropping the parse branches without the lock leaves the same
mess; adding the lock without dropping them leaves dead code that will confuse the next reader.

---

## 5. What the 13.08 meeting decided

| CR | Decision | State |
|---|---|---|
| 1 | EKW name separator `\|` → `_` | Done, all three screens |
| 2 | Hashtags stay; extra dropdown deferred to a new request | Deferred |
| 3 | Show the **Regelspediteur**, not the Performance-Spediteur, in brackets. **Display only** | Value done; the field is still editable — §9.2 |
| 4 | Insert the label `Thema:`; the rest of the black text unchanged | Done |
| 5 | Textbausteine for the Hintergrund — not a bot | Deferred to a new request |
| 6 | New Ja/Nein field, labelled **`Regresspotential`** (never `Regress`), pre-filled from OPSEP, creator may change it | Done, incl. write-back — §7 |
| 7a | Number formatting and currency | Done via `Notiz_lbl_*_res` |
| 7b | Regelspediteur becomes a Pflichtfeld, plus a `Kein Regelspediteur vorhanden` option | Superseded — §6.3 |
| 8 | Remove the **word** `Ausplanung` from all three Planstand options | Done, incl. a missing `I` typo |
| 9 | Generate once, then lock | **Open** — §4.1 |
| new | After saving, close popup and form, return to the overview | Done — §7 |
| new | Hinweis text per Kategorie/Subkategorie | Parked |

Two corrections worth recording, because the pre-meeting reading of the slide was wrong both times:

**CR 8 was inverted.** The slide looked like "remove the option *Im Planstand Ausplanung
enthalten*". The customer wanted the opposite — keep all three options, delete the word
`Ausplanung` from each, because *Ausplanung* is a Bewertungsmeilenstein at BMW and the wording has
to stay general.

**CR 5 is not a bot.** It is a Textbaustein picker with a free-text fallback.

---

## 6. The composition today

`NOTIZ_txt_Final.Default`. The `With` block computes four values, then the formula either lists
what is missing or emits the note.

### 6.1 `_hint` — flattening the Hintergrund

The Hintergrund is the `Kommentar` column, typed by the requester into `TL_txt_Kommentare`, which
is `Mode: TextMode.MultiLine`. Requesters write paragraphs, and the line breaks were reaching the
approval note verbatim.

```powerfx
_hint: Trim(
    Substitute(
        Substitute(NOTIZ_txt_Hintergrund_res.Text, Char(13), " "),
        Char(10), " "
    )
)
```

`Trim` in Power Fx removes extra spaces **inside** the string as well as at the ends, so the two
spaces left by a `CRLF` collapse to one. No separate de-duplication step is needed.

The blank check for the Hintergrund is `Len(_hint) = 0` rather than `IsBlank` on the raw text —
otherwise a Kommentar of nothing but blank lines passes the gate and prints an empty Hintergrund.

The input carries the rule as its placeholder:

```powerfx
HintText: ="Hintergrund als Fließtext beschreiben: ganze Sätze, keine Absätze, keine Hashtags."
```

That is guidance, not enforcement: `HintText` disappears on the first keystroke and never shows at
all when editing an existing record, because `Default: =varLKWItemSelected.Kommentar` fills the
box. If the stored data has to be clean too, normalise in the patch instead.

### 6.2 The costs

Three labels, not `Text()` calls in the composition:

```powerfx
Substitute(Substitute(Substitute(
    Text(Value(NOTIZ_txt_Gesamtkosten_res.Text), "[$-en-US]#,##0.00"),
    ",", ";"), ".", ","), ";", ".") & " €"
```

### 6.3 `_regel` — the Regelspediteur is optional

CR 7b asked for a mandatory field with a `Kein Regelspediteur vorhanden` escape option in the
dropdown. That was specified (a pseudo-record in `colRegelspediteure2` carrying `Rate: 0`, plus a
filter to keep it out of the alternatives gallery) but **superseded**: the field was made optional
in the note instead, which is far less machinery.

The synced version substitutes a fallback text:

```powerfx
_regel: If(
    IsBlank(NOTIZ_cmb_Regelspediteur_res.Text),
    "Kein Regelspediteur vorhanden",
    NOTIZ_cmb_Regelspediteur_res.Text
)
```

**Superseded again on 2026-08-19, not yet verified in a sync** — a blank Regelspediteur should
print nothing at all. `_regel` carries the whole line including its `Char(13)`, so the line
disappears rather than leaving empty brackets:

```powerfx
_regel: If(
    IsBlank(NOTIZ_cmb_Regelspediteur_res.Text),
    "",
    " (" & NOTIZ_cmb_Regelspediteur_res.Text & ") " & Char(13)
)
```

and the composition line reduces to `_regel &`.

Because the field is optional, `IsBlank(NOTIZ_cmb_Regelspediteur_res.Text)` was removed from the
gate and `• Regelspediteur` from the missing-fields list.

**This softens a customer requirement.** CR 7b was explicit about the Pflichtfeld. The form still
requires a Regelspediteur on its own edit path; only the note stopped enforcing it. Worth
confirming rather than leaving implicit.

---

## 7. The save handler

`OS_btn_NewProject_67.OnSelect` on `TransportLKW` — unique to that screen, so Bahn and Schiff are
untouched by any of this.

It patches name, note and `EKW Status`, re-reads the record, sets the notification, and then —
since 2026-08-19 — writes the Regresspotential back and returns the user to the overview.

**The Regresspotential write-back** goes on the LKW branch only, because
`NOTIZ_txt_Regress_param` exists on that screen alone:

```powerfx
Regress: NOTIZ_txt_Regress_param.Selected.Value = "Ja"
```

`Regress` is a **boolean** (SharePoint Yes/No). The `= "Ja"` comparison resolves before the value
reaches the column; assigning `.Selected.Value` directly pushes the string `"Ja"` at a Yes/No
column and fails. The OPSEP form does the same thing with `TL_rdo_Regress`.

**Consequence.** `Regress = Ja` has a partner column, `RegressSped`, which the OPSEP form treats as
mandatory in step 3:

```powerfx
If(TL_rdo_Regress.Selected.Value = "Ja", IsBlank(TL_cmb_Regress.Selected.'Sped Name'))
```

If the EKW creator flips Regresspotential to Ja, `RegressSped` stays empty and the **next** person
to edit that transport in the form cannot advance past step 3 until they pick a Regress-Spediteur.
The EKW creator never sees this. Accepted for now; the alternatives are relaxing the form check or
not writing back at all.

**The cleanup is now conditional.** It used to run unconditionally, so pressing Übernehmen with
fields missing showed the warning *and* closed the generator, discarding the input. It is now
wrapped in the same success test the notification uses, and followed by:

```powerfx
Concurrent(
    Set(newMode, false), Set(editMode, false), Set(viewMode, false),
    Set(varBudgetBuddy, false), Set(varBudgetPopUp, false), Set(varBackgroundPopUp, false)
);
Navigate(varLastScreen, ScreenTransition.Fade)
```

**`varLastScreen`, never a hardcoded overview.** The generator is reachable from `OverviewLKW`,
`OverviewEKW` and `OverviewEntscheidung`, and all three set the variable on the way in. The four
mode flags are the same ones the screen's own Zurück button clears; without them the screen reopens
in a half-open state.

---

## 8. Outstanding

**One work package, two halves, one save** — drop the eleven controls' parse branches (§4.1) and
add the lock. Everything else on LKW is done.

The lock goes on `TL_btn_Split_3.DisplayMode`, which is currently unset:

```powerfx
=If(
    IsBlank(varLKWItemSelected.'Generierter EKW Notiz') ||
    IsBlank(varLKWItemSelected.'Generierter EKW Name'),
    DisplayMode.Edit,
    DisplayMode.Disabled
)
```

Pair it with a `Tooltip` so the disabled state reads as a rule rather than a bug.

Three small ones, all on `TransportLKW`:

| | Fix |
|---|---|
| A dead `Reset(NOTIZ_cmb_performancespediteur_param)` survives in the cancel path; the control was renamed to `NOTIZ_txt_Regelspediteur_param`. Bahn and Schiff still have the old control — do not touch those. | Point it at the new name |
| `NOTIZ_cmb_Planstand_param` has no `SelectMultiple`, and the Classic ComboBox default is `true`, so contradictory Planstand options can both be chosen and both print, joined by `"; "` | `SelectMultiple: =false` |
| `NOTIZ_txt_Regelspediteur_param` is a `Classic/TextInput` with no `DisplayMode`, so it defaults to `Edit` — CR 3 says display only, and typing in it changes the note heading | `DisplayMode: =DisplayMode.View` |

---

## 9. Gotchas

### 9.1 `EKW Status` has two values, and the obvious lock condition is wrong

The generator sets `generiert`. The completion button on the same screen sets `abgeschlossen` when
the Einkäufer enters the EKW-Nummer. A lock written as `'EKW Status'.Value = "generiert"` therefore
**releases the note exactly when the order is finished and the EKW number has been assigned** — the
most protected point in the process, not the least.

Key the lock on the note itself. It survives any later status value, it covers legacy records in
the pipe format that carry no status at all, and it is the same condition the completion button
already uses.

### 9.2 CR 3 is not met by putting the right value in the field

The Regelspediteur shows the correct name, but the control is still writable (§8). "Display only"
needs the `DisplayMode`, not just the right `Default`.

### 9.3 The Pflichtfeld is set, but unequally

Two paths check the Regelspediteur on the Backuptransport form, and they disagree:

```powerfx
// create path
If(IsEmpty(colRegelspediteure2), 0, IsBlank(bb_cmb_Regelspediteur_2.Selected)) ||
// edit path
IsBlank(bb_cmb_Regelspediteur_2.Selected) ||
```

The create path lets the field through when the rate list knows no Regelspediteur for the relation.
The edit path does not. **An order created through that gap can never be saved again** — the
warning fires every time and the user has no field with which to resolve it. Give the edit path the
same escape, and prefer `false` over the `0` the create path uses.

### 9.4 Two findings in the cost chain, both unconfirmed

Neither was discussed with the customer. Both are invisible today and both become visible once the
rates are maintained.

**`HAQuelle` reads the Senke.** The two parameters are identical — the source's Hafenaufschlag is
never read and the destination's counts twice:

```powerfx
NOTIZ_txt_HAQuelle_param = LookUp(CHW…, Lagerort = varLKWItemSelected.'Senke:VDS-ID'.Value, Hafenaufschlag)
NOTIZ_txt_HASenke_param  = LookUp(CHW…, Lagerort = varLKWItemSelected.'Senke:VDS-ID'.Value, Hafenaufschlag)
```

**A unit mismatch in Mehrkosten pro Fahrzeug.** The first two terms are per vehicle; the third
multiplies the surcharge by the total vehicle count, so `Mehrkosten/Fzg × Anzahl Fahrzeuge ≠
Gesamtmehrkosten`. It does not show on either slide example because both have a zero
Hafenaufschlag.

Raise both as questions. They are inside what CR 7 touches.

### 9.5 Old notes stay in the pipe format

They are never re-parsed once the popup is locked, so nothing breaks. Do not convert them.

---

## 10. Parked scope

Parked 2026-08-18, LKW only for now.

**Bahn and Schiff.** The same packages, plus a `BB_Regelspediteur` column that exists in neither
list — that is SharePoint work with an owner and a lead time before any app work starts. Nothing in
the LKW work depends on them; each screen owns its controls. **They keep the old pipe format and an
unlocked generator**, so behaviour visibly differs between transport types until someone picks them
up. Say so rather than letting it be discovered.

**Kategorie hints.** A Hinweis column on the Kategorie and Subkategorie lists, displayed beside the
dropdown, so a requester filling the OPSEP knows which Kategorie to pick. Agreed with the customer,
who owes the definitions. Note this lives on the OPSEP form, not the note popup. The mechanism
could be built with an empty column to get off the customer's critical path.

---

## 11. Change log

| Date | Change |
|---|---|
| 2026-08-13 | EKW name separator `\|` → `_` on all three screens (CR 1). `BB_Regelspediteur` persisted on LKW — column plus four patch sites — and read in `OverviewEntscheidung` |
| 2026-08-14 | Planstand wording, `Ausplanung` removed from all three options plus a missing `I` (CR 8). Currency labels (CR 7a). Regresspotential radio (CR 6). Note composition rewritten to labelled lines, no pipes (CR 4). Regelspediteur replaces the Performance-Spediteur in the heading (CR 3, value only) |
| 2026-08-18 | Bahn, Schiff and the Kategorie hints parked |
| 2026-08-19 | `_hint` flattens the Hintergrund and the gate moves to `Len(_hint) = 0` (§6.1); `TL_txt_Kommentare` hint text. `_regel` makes the Regelspediteur optional (§6.3), superseding the CR 7b dropdown option. Regresspotential written back as a boolean (§7). Cleanup gated on success and `Navigate(varLastScreen)` added (§7). Hashtag line restructured by the customer-facing developer |
