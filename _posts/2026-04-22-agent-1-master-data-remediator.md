---
layout: post
title: "Agent 1: Master Data Remediator"
date: 2026-04-22
---

*View the project on GitHub: [master-data-remediator](https://github.com/aleatthayter/master-data-remediator)*

---

**The Problem**

Mining and energy companies manage large, complex asset bases — processing plants, pipelines, rotating equipment, electrical systems — and that asset data lives across multiple systems simultaneously. The problem is those systems rarely agree with each other.

Some of the most common issues:

- Equipment descriptions and tag names differ between the ERP system (such as SAP) and engineering design tools (such as AVEVA or SmartPlant)
- Engineering drawings reflect an original design that was modified during construction, but the changes were never captured in the asset register
- 3D models reference equipment that has been decommissioned, or omit equipment that was added during a shutdown
- Manual data entry across systems over many years introduces inconsistencies in naming conventions, abbreviations and classifications
- No single system is treated as the source of truth, so discrepancies compound over time

The result is that the people making maintenance, procurement and capital decisions are working from data they cannot fully trust.

---

**The Solution**

This agent automates the process of identifying and reconciling master data discrepancies across systems. Here is what it does:

- Ingests data from multiple source systems — in this proof of concept, SAP FLOC data, AVEVA engineering metadata, and an engineering drawing register
- Compares each source against the others, tag by tag, and identifies where the data does not match
- Uses an AI model to reason about each discrepancy and suggest the most likely correct value, along with an explanation of why
- Generates a remediation report flagged as pending approval, so a human engineer can review every suggested fix before anything is changed

The agent does not write anything back to source systems automatically. Every suggested fix requires a human to review and approve it first.

Here is a simple view of how the agent works: