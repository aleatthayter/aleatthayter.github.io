---
layout: post
title: "Agent 1: Master Data Remediator"
date: 2026-04-22
---

*View the project on GitHub: [master-data-remediator](https://github.com/aleatthayter/master-data-remediator)*

## The Challenge

Mining and energy companies manage large, complex asset bases (processing plants, pipelines, rotating equipment, electrical systems) and that asset data lives across multiple systems simultaneously. The problem is those systems rarely agree with each other.

Some of the most common issues:

- Equipment descriptions and tag names differ between the ERP system (such as SAP) and engineering design tools (such as AVEVA or SmartPlant)
- Engineering drawings reflect an original design that was modified during construction, but the changes were never captured in the asset register
- 3D models reference equipment that has been decommissioned, or omit equipment that was added during a shutdown
- Manual data entry across systems over many years introduces inconsistencies in naming conventions, abbreviations and classifications
- No single system is treated as the source of truth, so discrepancies compound over time

The result is that the people making maintenance, procurement and capital decisions are working from data they cannot fully trust.

## How It Works

This agent automates the process of identifying and reconciling master data discrepancies across systems. Here is what it does:

- Ingests data from multiple source systems: SAP FLOC data, AVEVA engineering metadata, a drawing register CSV, PDF engineering drawings, and CAD files
- Compares each source against the others, tag by tag, and identifies where the data does not match
- Uses an AI model to reason about each discrepancy and suggest the most likely correct value, along with an explanation of why
- Generates a remediation report flagged as pending approval, so a human engineer can review every suggested fix before anything is changed

The agent does not write anything back to source systems automatically. Every suggested fix requires a human to review and approve it first.

The agent is built in Python using LangChain as the orchestration layer and Claude (Anthropic) as the underlying model. Pydantic data models define the output schema for each suggested fix, covering the tag, the field in question, the values found across each source system, the suggested correction, and the reasoning behind it. Using structured output means every remediation report is consistently formatted and reliable enough to feed into downstream approval workflows without manual cleanup.

Engineering drawings are handled directly. PNG and JPEG images are processed using Claude's vision capability, which extracts equipment tags and descriptions from scanned P&IDs and equipment schedules. PDF drawings are rendered page by page and passed through the same vision pipeline. DXF CAD files are parsed programmatically using ezdxf, reading the block INSERT attributes that engineering CAD libraries use to store equipment tag numbers and descriptions. The result is that the agent can pull tag data from the actual drawing files sitting in a directory alongside the CSV exports, without requiring someone to manually transcribe drawing data into a register first.

Here is a simple view of how the agent works:

<img src="/assets/diagrams/agent-1-architecture.svg" alt="Agent 1 architecture diagram" style="width:100%;max-width:760px;display:block;margin:1.5rem auto;">

## Why It Matters

Better master data is not just a data quality issue. For a mining or energy operator it has direct consequences across the business:

- Maintenance planning: Accurate equipment data means maintenance schedules, spare parts lists and work orders are built on reliable information, reducing unplanned downtime and reactive maintenance costs
- Safety: Engineers and technicians working from incorrect equipment descriptions or outdated drawings face real safety risks. Consistent master data reduces the chance of working on the wrong asset or with the wrong specifications
- Capital allocation: Investment decisions, whether to replace, upgrade or decommission an asset, depend on knowing what you actually have. Poor master data leads to over- or under-investment
- Procurement: Purchasing spare parts or consumables against incorrect equipment records results in wrong parts being ordered, delays to maintenance and unnecessary inventory costs
- Regulatory compliance: Many jurisdictions require operators to maintain accurate asset registers for safety and environmental reporting. Data discrepancies create compliance risk
- Shutdown and turnaround planning: Shutdowns are expensive and time-critical. Work packs built on inaccurate master data lead to delays, rework and cost overruns
- Digital transformation readiness: Any move toward predictive maintenance, digital twins or advanced analytics depends on having a clean, trusted data foundation. Master data remediation is the prerequisite, not the afterthought

## The Agent in Action

### Before: The Raw Data

Here is what the data looked like across the five source systems before the agent ran. You can see immediately that the same physical equipment is described differently depending on which system you look at, and that some equipment only appears in certain sources.

| Tag | SAP | AVEVA | Drawing Register | PDF Drawing | CAD File |
|-----|-----|-------|-----------------|-------------|----------|
| PP-001 | Pump Primary Feed | Primary Feed Pump | Primary Feed Pump | Primary Feedstock Pump | - |
| MV-042 | Motor Drive Unit | Electric Motor Drive | Motor Drive Unit | Electric Motor Drive | MV Drive Unit |
| HX-103 | Heat Exchanger Main | Main Heat Exchanger | Heat Exchanger Main | - | - |
| PP-002 | Feed Pump Secondary | Secondary Feed Pump | Secondary Feed Pump | - | Feed Pump No.2 |
| CV-011 | - | - | - | Feed Conveyor Drive | Feed Conveyor Drive |
| TK-005 | - | - | - | Reagent Storage Tank | Reagent Tank |

No single system is wrong per se, but no two fully agree. The last two rows are equipment that exists in the drawing files but has never made it into SAP or AVEVA at all. For a maintenance planner or procurement officer searching across systems, this creates confusion, duplicated effort and risk of error.

### Agent Output

Running the agent against this data produced a remediation report with a suggested fix for each discrepancy. Here is what the agent recommended:

| Tag | Suggested Fix | Reasoning |
|-----|---------------|-----------|
| PP-001 | Primary Feed Pump | Majority of sources agree on this form. "Primary Feedstock Pump" from the PDF drawing is a variant, but the shorter form is consistent with asset register convention |
| MV-042 | Electric Motor Drive | Three of four populated sources use a descriptive form. "MV Drive Unit" from the CAD file is an abbreviation that would not be meaningful without context |
| HX-103 | Heat Exchanger Main | Equipment type leads the description, consistent with asset register standards across the site |
| PP-002 | Secondary Feed Pump | Consistent with PP-001 naming pattern. "Feed Pump No.2" from the CAD file uses a numbering convention not used elsewhere in the register |
| CV-011 | Feed Conveyor Drive | Exists only in drawing sources. Flagged for addition to SAP and AVEVA with this description |
| TK-005 | Reagent Storage Tank | Exists only in drawing sources. The PDF form is preferred over the CAD abbreviation |

Each row is flagged as pending approval in the report and exported to Excel. An engineer reviews each suggestion and either approves it or overrides it with their own judgement before anything is updated in a source system.

*This is a proof of concept. A production implementation would require integration with live systems, role-based approval workflows, audit logging, and change management processes tailored to the operator's environment. The intent here is to demonstrate that this kind of automated reconciliation is achievable, and to show where the value lies.*
