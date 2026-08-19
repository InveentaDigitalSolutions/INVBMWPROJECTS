# OPS-App — Sondertransporte

BMW special-transport management for vehicle logistics. A Power Apps Canvas app over SharePoint
Online: a Steuerer raises a transport need, costs are computed against maintained rates, and an EKW
name and a **Genehmigungsnotiz** — the approval note — are generated for purchasing.

The engagement covers nine change requests on the EKW name and the note, agreed with the customer
on 2026-08-13. The substantial one is structural: the note's storage format changed from twelve
pipe-joined values to labelled lines, which makes it unparseable and therefore forces it to become
write-once.

> The use case number is not recorded here — add it when someone confirms it.

## Contents

| File | For | What it is |
|---|---|---|
| [`OPS_TECHNICAL_REFERENCE.md`](OPS_TECHNICAL_REFERENCE.md) | Developers | The note pipeline, why the note must lock, every change against the customer's CR list, what is still open, and the traps. Start here. |

## Before you change anything

1. **Every change is three screens.** `TransportLKW`, `TransportBahn` and `TransportSchiff` each
   own a full copy of every control — the SharePoint column limit forced the lists apart. The
   `Switch(App.ActiveScreen, …)` branches inside a copy are dead code on the two screens they do
   not belong to.
2. **The parse branches and the lock must ship in one save.** The composition already emits the
   pipe-free format while eleven controls still read values back by counting pipes. Reopening a
   generated note today dumps the whole text into the Transportmittel field. Fixing either half
   alone leaves that, or leaves dead code.
3. **Do not lock on `'EKW Status' = "generiert"`.** The status later becomes `abgeschlossen`, which
   would release the note exactly when the EKW number has been assigned. Key on the note itself.
4. **Bahn and Schiff are parked** and keep the old pipe format with an unlocked generator, so
   behaviour differs between transport types on purpose.
5. **Studio must be open** for `sync_canvas` to return anything. A connect against a closed app
   succeeds and then syncs zero files.
