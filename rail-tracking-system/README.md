# Rail Tracking System (RTS) — UC_otd_3104

BMW rail freight transport management. A Power Apps Canvas app drives wagon transports through a
24-state workflow — need → order → confirmation → delivery → loading — with dunning (Mahnung) and
cancellation (Storno) branches. Around it sit 26 cloud flows, a Dataverse model in a **second**
environment, and two web masks through which suppliers and destinations confirm without needing a
BMW licence.

## Contents

| File | For | What it is |
|---|---|---|
| [`RTS_TECHNICAL_REFERENCE.md`](RTS_TECHNICAL_REFERENCE.md) | Developers | The two-environment split, the status map, the canvas screens, all 26 flows with their schedules, the email shell, and the gotchas. Start here. |
| [`RTS_Data_Dictionary.html`](RTS_Data_Dictionary.html) | Customer | Data dictionary and Bedienungsanleitung, prepared for turnover. Open in a browser. |

## Before you change anything

1. **Establish which copy of the flow is live.** Several flows exist twice — an app-env copy
   (`3104_flow_`) and a data-env copy (`RTS -`), each with its own On/Off state. Fixing the wrong
   one changes nothing and looks like success.
2. **`03_SingleTransportScreen` is 1.89 MB and cannot be pushed by the MCP** — the compile limit is
   512 KB. Edits to that screen are Studio-formula-bar only. `02` is 4 KB under the limit.
3. **A solution import upserts every flow in the zip.** Build the import from a fresh export taken
   immediately before, or previously-fixed flows silently revert.
4. **Status option values do not follow the label order.** `03` is `914890000`, `14` is
   `914890003`. Use the map in the reference; never infer.
