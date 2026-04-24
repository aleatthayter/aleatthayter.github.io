---
layout: post
title: "Agent 1: Master Data Remediator"
date: 2026-04-22
---

*View the project on GitHub: [master-data-remediator](https://github.com/aleatthayter/master-data-remediator)*

## ⚠️ The Challenge

Mining and energy companies manage large, complex asset bases (processing plants, pipelines, rotating equipment, electrical systems) and that asset data lives across multiple systems simultaneously. The problem is those systems rarely agree with each other.

Some of the most common issues:

- Equipment descriptions and tag names differ between the ERP system (such as SAP) and engineering design tools (such as AVEVA or SmartPlant)
- Engineering drawings reflect an original design that was modified during construction, but the changes were never captured in the asset register
- 3D models reference equipment that has been decommissioned, or omit equipment that was added during a shutdown
- Manual data entry across systems over many years introduces inconsistencies in naming conventions, abbreviations and classifications
- No single system is treated as the source of truth, so discrepancies compound over time

The result is that the people making maintenance, procurement and capital decisions are working from data they cannot fully trust.

## ⚙️ How It Works

This agent automates the process of identifying and reconciling master data discrepancies across systems. Here is what it does:

- Ingests data from multiple source systems (in this proof of concept: SAP FLOC data, AVEVA engineering metadata, and an engineering drawing register)
- Compares each source against the others, tag by tag, and identifies where the data does not match
- Uses an AI model to reason about each discrepancy and suggest the most likely correct value, along with an explanation of why
- Generates a remediation report flagged as pending approval, so a human engineer can review every suggested fix before anything is changed

The agent does not write anything back to source systems automatically. Every suggested fix requires a human to review and approve it first.

Here is a simple view of how the agent works:

<pre>
+----------------------------------------------------------+
|                     DATA SOURCES                         |
|                                                          |
|   SAP FLOC Data    AVEVA Metadata    Engineering Drawings|
|        |                |                   |            |
|        +----------------+-------------------+            |
|                         |                                |
|                         v                                |
|                  COMPARE & FIND CONFLICTS                |
|                         |                                |
|                         v                                |
|                   AI SUGGESTS FIXES                      |
|                         |                                |
|                         v                                |
|                HUMAN REVIEWS AND APPROVES                |
|                         |                                |
|                         v                                |
|              CLEAN, CONSISTENT MASTER DATA               |
+----------------------------------------------------------+
</pre>

## 💡 Why It Matters

Better master data is not just a data quality issue. For a mining or energy operator it has direct consequences across the business:

- Maintenance planning: Accurate equipment data means maintenance schedules, spare parts lists and work orders are built on reliable information, reducing unplanned downtime and reactive maintenance costs
- Safety: Engineers and technicians working from incorrect equipment descriptions or outdated drawings face real safety risks. Consistent master data reduces the chance of working on the wrong asset or with the wrong specifications
- Capital allocation: Investment decisions, whether to replace, upgrade or decommission an asset, depend on knowing what you actually have. Poor master data leads to over- or under-investment
- Procurement: Purchasing spare parts or consumables against incorrect equipment records results in wrong parts being ordered, delays to maintenance and unnecessary inventory costs
- Regulatory compliance: Many jurisdictions require operators to maintain accurate asset registers for safety and environmental reporting. Data discrepancies create compliance risk
- Shutdown and turnaround planning: Shutdowns are expensive and time-critical. Work packs built on inaccurate master data lead to delays, rework and cost overruns
- Digital transformation readiness: Any move toward predictive maintenance, digital twins or advanced analytics depends on having a clean, trusted data foundation. Master data remediation is the prerequisite, not the afterthought

## 🔬 The Agent in Action

### 📋 Before: The Raw Data

Here is what the data looked like across the three source systems before the agent ran. You can see immediately that the same physical equipment is described differently depending on which system you look at.

| Tag | SAP Description | AVEVA Description | Drawing Description |
|-----|----------------|-------------------|---------------------|
| PP-001 | Pump Primary Feed | Primary Feed Pump | Primary Feed Pump |
| MV-042 | Motor Drive Unit | Electric Motor Drive | Motor Drive Unit |
| HX-103 | Heat Exchanger Main | Main Heat Exchanger | Heat Exchanger Main |
| PP-002 | Feed Pump Secondary | Secondary Feed Pump | Secondary Feed Pump |

No single system is wrong per se, but no two systems fully agree either. For a maintenance planner or procurement officer searching across systems, this creates confusion, duplicated effort and risk of error.

### ✅ Agent Output

Running the agent against this data produced a remediation report with a suggested fix for each discrepancy. Here is what the agent recommended:

| Tag | SAP Value | AVEVA Value | Suggested Fix | Reasoning |
|-----|-----------|-------------|---------------|-----------|
| PP-001 | Pump Primary Feed | Primary Feed Pump | Primary Feed Pump | Follows standard naming convention, consistent with engineering documentation standards |
| MV-042 | Motor Drive Unit | Electric Motor Drive | Electric Motor Drive | More specific and descriptive, clearly identifying the equipment type and function |
| HX-103 | Heat Exchanger Main | Main Heat Exchanger | Heat Exchanger Main | Equipment type should lead the description for consistency with asset register standards |
| PP-002 | Feed Pump Secondary | Secondary Feed Pump | Secondary Feed Pump | Follows standard naming convention and is consistent with PP-001 naming pattern |

Each row is flagged as pending approval in the report. An engineer reviews each suggestion and either approves it or overrides it with their own judgement before anything is updated in a source system.

This is a proof of concept. A production implementation would require integration with live systems, role-based approval workflows, audit logging, and change management processes tailored to the operator's environment. The intent here is to demonstrate that this kind of automated reconciliation is achievable, and to show where the value lies.
