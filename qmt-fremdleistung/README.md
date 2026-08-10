# QMT Fremdleistung Grossticket — UC_enb_266

BMW quality-management ticket board (German Kanban) for Fremdleistung Grosstickets. Power Apps
Canvas app over SharePoint Online, with two Power Automate flows carrying the ticket queries.

The engagement rebuilt how the board loads its data. The app previously pulled every ticket into
memory at start-up and filtered client-side, which put anything past the platform row limit out of
reach of filters and search — silently. Loading is now server-side and per bucket.

## Contents

| File | For | What it is |
|---|---|---|
| [`QMT_TECHNICAL_REFERENCE.md`](QMT_TECHNICAL_REFERENCE.md) | Developers | Architecture, the load pipeline, both flow contracts, every area that was changed, the patterns to follow and the traps that cost time. Start here. |
| [`delivery-documentation.html`](delivery-documentation.html) | Customer | Handover document: what was delivered and why, measured results, test checklist, open decisions. Open in a browser. |

## Before you change anything

Three things in the technical reference will save you a day each. They are covered in full there;
this is only the warning that they exist.

1. Flow `.Run()` arguments use raw schema names (`text`, `text_1`, …), not the input titles. Wrong
   names compile cleanly and are silently dropped.
2. The flow response key is case-sensitive and differs between the two flows — `Result` and
   `result`. The wrong case returns blank rather than an error.
3. Solution `UC_enb_266` carries the canvas app. Importing it replaces the app and reverts any
   Studio edits made since the export.
