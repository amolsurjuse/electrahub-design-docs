# ElectraHub Design Docs

Design and architecture documentation for ElectraHub.

This repository is intentionally separate from the ElectraHub application source tree so service code, generated workflow artifacts, and large design documents do not live in the same project folder.

## Contents

```text
electrahub-design-docs/
├── .gitignore
├── README.md
├── analysis/
│   └── ios-active-charging-sse-events-analysis.md
├── design/
│   └── architecture/
│       ├── CSMS_Services_LLD_Alignment_Report.docx
│       ├── ElectraHub_CSMS_High_Level_Design.docx
│       ├── ElectraHub_CSMS_Low_Level_Design.docx
│       ├── Pricing_Service_High_Level_Design.docx
│       ├── Pricing_Service_Low_Level_Design.docx
│       ├── RBAC_Framework_High_Level_Design copy.docx
│       ├── RBAC_Framework_High_Level_Design.docx
│       ├── RBAC_Framework_High_Level_Design.md
│       ├── RBAC_Framework_Low_Level_Design copy.docx
│       ├── RBAC_Framework_Low_Level_Design.docx
│       ├── RBAC_Framework_Low_Level_Design.md
│       ├── WebSocket_Connection_Management_HLD.docx
│       └── WebSocket_Connection_Management_LLD.docx
└── migration/
    └── REST-to-gRPC-Migration-Plan.docx
```

## Document Areas

- CSMS platform high-level and low-level design
- pricing service high-level and low-level design
- RBAC framework high-level and low-level design
- WebSocket connection management design
- CSMS service LLD alignment
- iOS/backend incident analysis and handoff notes
- REST-to-gRPC migration planning

## Repository Structure

### `analysis`

Investigation notes and handoff reports for active ElectraHub issues.

| File | Purpose |
|---|---|
| `ios-active-charging-sse-events-analysis.md` | Analysis report for the iOS active charging screen not receiving backend SSE charging events. |

### `design/architecture`

Architecture and service design documents for core ElectraHub platform capabilities.

| File | Purpose |
|---|---|
| `CSMS_Services_LLD_Alignment_Report.docx` | Alignment notes across CSMS low-level service designs. |
| `ElectraHub_CSMS_High_Level_Design.docx` | CSMS high-level architecture and platform design. |
| `ElectraHub_CSMS_Low_Level_Design.docx` | CSMS low-level service/component design. |
| `Pricing_Service_High_Level_Design.docx` | Pricing service high-level design. |
| `Pricing_Service_Low_Level_Design.docx` | Pricing service low-level design. |
| `RBAC_Framework_High_Level_Design.docx` | RBAC framework high-level design. |
| `RBAC_Framework_High_Level_Design.md` | Markdown version of the RBAC high-level design. |
| `RBAC_Framework_Low_Level_Design.docx` | RBAC framework low-level design. |
| `RBAC_Framework_Low_Level_Design.md` | Markdown version of the RBAC low-level design. |
| `WebSocket_Connection_Management_HLD.docx` | WebSocket connection management high-level design. |
| `WebSocket_Connection_Management_LLD.docx` | WebSocket connection management low-level design. |

The `copy.docx` files are preserved as source backups until the canonical RBAC docs are reviewed and deduplicated.

### `migration`

Migration planning documents that cut across services.

| File | Purpose |
|---|---|
| `REST-to-gRPC-Migration-Plan.docx` | Migration plan for moving selected ElectraHub service contracts from REST toward gRPC. |

### Root Files

| File | Purpose |
|---|---|
| `README.md` | Repository overview and structure. |
| `.gitignore` | Ignores macOS metadata and Office lock files. |

## Source Repository

The ElectraHub application source repository remains at:

```text
/Users/amolsurjuse/development/projects
```

Generated workflow logs and old non-release artifacts were moved to:

```text
/Users/amolsurjuse/development/electrahub-project-archive
```
