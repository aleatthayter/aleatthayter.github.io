---
layout: post
title: "Agent 2: Shutdown Optimiser"
date: 2026-04-25
---

*View the project on GitHub: [shutdown-optimiser](https://github.com/aleatthayter/shutdown-optimiser)*

## ⚠️ The Challenge

Mining operators need their assets running continuously. Every hour a processing plant or conveyor system sits idle is production lost and revenue foregone. Yet planned shutdowns are unavoidable. Equipment degrades, statutory inspections must happen, and preventive maintenance on schedule is far less costly than an unplanned failure mid-production.

The challenge is not whether to shut down, but how to make the most of the time available when operations are offline. A typical fixed-duration shutdown might run for 48 hours and carry 60 or more work orders from SAP, each with a different priority, different crew, and different access requirements. Deciding what gets done, in what order, and whether it all fits in the available window is a complex planning problem that currently falls to a team of experienced engineers working through spreadsheets for weeks ahead of each event.

The consequences of getting it wrong are significant. A shutdown that overruns by even a few hours can cost a mining operation hundreds of thousands of dollars in lost production. And when field workers receive unclear or incomplete instructions, work slows, rework happens, and the window tightens further.

## ⚙️ How It Works

This agent takes a set of SAP work orders and produces an optimised shutdown work package ready for scheduling in Primavera P6. Here is what it does:

- Ingests work orders from SAP, each with a priority level, location, estimated duration, crew type, and access requirements (scaffolding, electrical isolations, confined space entry, etc.)
- Groups tasks that share access requirements and physical area, so scaffolding is erected once and used for all nearby work, and isolations are applied once across all tasks that need them
- Sequences work within each group by priority, ensuring the most critical tasks are completed before the window closes
- Rewrites terse SAP descriptions into clear, specific, actionable instructions that field workers can follow without needing to interpret abbreviations or seek clarification
- Calculates planned start and end times for each task against the shutdown window, and flags any task at risk of not completing within the available hours
- Outputs the full optimised plan to Excel, structured for import into P6

The agent does not make the final scheduling decision. The output is a recommended plan that a planner reviews before committing to execution.

### 🛠️ Under the Hood

The agent is built in Python using [LangChain](https://python.langchain.com) as the orchestration layer, with Claude (Anthropic) as the underlying model. The most important technical decision in the current version is the use of Pydantic data models to enforce the output schema.

Rather than asking the model to "return JSON" and hoping the format is consistent, the agent defines an explicit contract for what the output must look like — every field, every type. LangChain's `with_structured_output()` enforces this contract at runtime. If a field is missing or the wrong type, the call fails rather than silently producing a malformed plan. This matters because the output feeds directly into Excel and ultimately into scheduling tools like Primavera P6 — a plan with a missing start time or a text string where a number is expected causes downstream failures that are harder to debug than the original error.

The practical effect is that the agent's output is reliable enough to process programmatically, not just read by a human. That is a prerequisite for any future integration with live SAP exports or P6 APIs.

Here is a simple view of how the agent works:

<pre>
+----------------------------------------------------------+
|                     DATA SOURCES                         |
|                                                          |
|         SAP Work Orders (Priority, Area, Duration)       |
|                         |                                |
|                         v                                |
|               GROUP BY AREA & ACCESS NEEDS               |
|                         |                                |
|                         v                                |
|            SEQUENCE BY PRIORITY WITHIN GROUPS            |
|                         |                                |
|                         v                                |
|            AI REWRITES FIELD INSTRUCTIONS                |
|                         |                                |
|                         v                                |
|         FLAG OVERRUN RISK & ASSIGN START TIMES           |
|                         |                                |
|                         v                                |
|            EXCEL OUTPUT READY FOR P6 IMPORT              |
+----------------------------------------------------------+
</pre>

## 💡 Why It Matters

Getting shutdown planning right has direct operational and financial consequences:

- Overrun prevention: A plan that accounts for access dependencies and realistic task sequencing from the start significantly reduces the risk of the shutdown window being exceeded. Overruns are among the most expensive events in mining operations.
- Reduced planning overhead: Teams currently spend weeks ahead of each shutdown manually ordering and reviewing work orders. This agent compresses much of that work into minutes, freeing planners to focus on exceptions and final decisions rather than first-pass sorting.
- Better field execution: Clear, specific work instructions mean field crews spend less time seeking clarification, interpreting SAP shorthand, or making assumptions. Work gets done faster and more safely.
- More work per window: Grouping tasks that share access requirements means more work orders are completed in the same shutdown window. Every additional task completed reduces the backlog and defers the need for the next shutdown.
- Consistent planning quality: The quality of a shutdown plan today often depends on the experience of the planner running it. This agent applies consistent logic to every work order, regardless of who is planning the event.

## 🔬 The Agent in Action

### 📋 Before: The Raw Data

Here is a sample of what the work orders looked like coming out of SAP. Descriptions are abbreviated, there is no sequencing logic, and access requirements are listed independently with no grouping applied.

| WO ID | SAP Description | Priority | Area | Duration (hrs) | Access Required |
|-------|----------------|----------|------|----------------|-----------------|
| WO-3841 | PM - PP-001 brg chk & lub | 1 | Pump Station A | 2 | None |
| WO-3843 | Insp - MCC-1 switchboard panels | 1 | MCC Room 1 | 1 | Isolation |
| WO-3842 | PM - CV-01 idler replacement SEC 4-6 | 2 | Conveyor CV-01 | 4 | Scaffolding |
| WO-3846 | Insp - CV-01 structure above SEC 4-6 | 3 | Conveyor CV-01 | 2 | Scaffolding |
| WO-3847 | Replace - PP-001 motor coupling | 1 | Pump Station A | 3 | None |
| WO-3852 | Test - ESD system MCC-1 | 1 | MCC Room 1 | 2 | Isolation |

No priority or access grouping has been applied. A field worker receiving WO-3842 and WO-3846 as separate jobs would erect and dismantle scaffolding on CV-01 twice.

### ✅ Agent Output

Running the agent against this data produced an optimised plan. Here is a sample of what it recommended:

| Seq | WO ID | Area | Improved Description | Start | End | Grouping Rationale |
|-----|-------|------|----------------------|-------|-----|--------------------|
| 1 | WO-3843 | MCC Room 1 | Inspect all panels in MCC-1 switchboard. Check breaker condition, termination integrity, and labelling against as-built drawings. | T+0h | T+1h | High priority isolation work first. Isolation applied once for both MCC-1 tasks. |
| 2 | WO-3852 | MCC Room 1 | Test the MCC-1 emergency shutdown system. Verify all trip signals initiate correctly and confirm reset procedure with control room. | T+1h | T+3h | Shares MCC-1 isolation with WO-3843. Electrical crew already on-site. |
| 3 | WO-3841 | Pump Station A | Check PP-001 main bearing for wear and temperature anomalies. Apply lubricant to spec and record readings on maintenance sheet. | T+0h | T+2h | High priority, no access setup required. Mechanical crew deployed immediately. |
| 4 | WO-3847 | Pump Station A | Replace PP-001 motor coupling. Align to within tolerance per equipment datasheet. Inspect coupling guard before reinstatement. | T+2h | T+5h | Mechanical crew already on-site from WO-3841. No additional access setup required. |
| 5 | WO-3842 | Conveyor CV-01 | Replace worn idlers on CV-01 between sections 4 and 6. Check belt tracking on reinstatement. Remove all debris before belt restart. | T+0h | T+4h | Scaffolding erected once for both CV-01 tasks. Both completed before scaffolding is dismantled. |
| 6 | WO-3846 | Conveyor CV-01 | Inspect CV-01 structural steelwork above sections 4 to 6 for corrosion, cracking, and loose connections. Report findings before scaffolding is removed. | T+4h | T+6h | Uses same scaffolding as WO-3842. Structural crew completes inspection before dismantling. |

The two CV-01 scaffolding tasks are now grouped consecutively, MCC-1 isolation work is batched, and Pump Station A tasks are sequenced so the mechanical crew completes both jobs in one visit. Each description has been rewritten from SAP shorthand into a full field instruction.

This is a proof of concept. A production implementation would connect directly to live SAP work order exports, integrate with P6 scheduling APIs, and incorporate additional constraints such as crew availability, permit lead times, and interdependencies between work orders. The intent here is to demonstrate that this kind of automated optimisation is achievable and to show where the value lies.
