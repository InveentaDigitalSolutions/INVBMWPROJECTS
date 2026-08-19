# BMW Projects

Technical documentation for BMW engagements. One folder per project.

> **Confidential — customer material.** This repository contains customer environment
> identifiers, data model details and internal architecture notes. Keep it private.

## Projects

| Project | Folder | Platform | Status |
|---|---|---|---|
| QMT Fremdleistung Grossticket (UC_enb_266) | [`qmt-fremdleistung/`](qmt-fremdleistung/) | Power Apps Canvas + Power Automate + SharePoint | Delivered, in test |
| Rail Tracking System (UC_otd_3104) | [`rail-tracking-system/`](rail-tracking-system/) | Power Apps Canvas + Power Automate + Dataverse (two environments) + web masks | In development |
| OPS-App Sondertransporte | [`ops-sondertransporte/`](ops-sondertransporte/) | Power Apps Canvas + SharePoint | In development — LKW; Bahn and Schiff parked |
| Race to Quality (UC_enb_485) | `race-to-quality/` | Power Apps Canvas + Power Automate + SharePoint | Pending — awaiting a Studio sync |

## Conventions

- One folder per engagement, named after the use case rather than the internal offer number.
- Each folder holds a technical reference aimed at the next developer, and — where one exists —
  the customer-facing handover document.
- Technical references record **why** a thing is the way it is, not only what it does. The
  non-obvious constraints are the point; the code is already in the app.
- Documents state the date and the source snapshot they were written against. When you update a
  document, update that line.
