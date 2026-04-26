---
layout: post
title: "Agent 6: CER Onboarder"
date: 2026-04-26
---

*View the project on GitHub: [cer-onboarder](https://github.com/aleatthayter/cer-onboarder)*

## The Challenge

Australia's uptake of rooftop solar has made the country one of the highest per-capita adopters in the world. Home batteries are following. As these Consumer Energy Resources (CER, the term now used by AEMO and the AER for solar panels, home batteries, and EV chargers) become mainstream, the process of connecting them to the grid and setting up a feed-in tariff has become a significant operational burden for energy retailers.

A customer who wants to set up a feed-in tariff needs to navigate several distinct steps: confirming their state's eligibility rules, verifying that their system meets the technical requirements for grid export under the AS4777 standard, understanding their tariff rate and how it compares to participation in a Virtual Power Plant (VPP) program, and then completing the application itself. At a traditional energy retailer, this involves multiple handoffs between different teams, each with different domain knowledge and different access to systems.

What makes this genuinely difficult is not the complexity of any individual step. It is the fact that each step depends on the last, the customer's details need to be carried forward correctly at every point, and the person handling each stage only has visibility of their own piece of the process. A customer who has already explained their system details to the eligibility team should not have to repeat them to the technical team. An application specialist who needs to know the customer's NMI should not have to ask whether the technical check has been completed.

Multi-agent orchestration is a natural architectural fit for this problem.

## How It Works

The agent runs as a multi-turn conversation and is built using LangGraph, which handles the graph structure, shared state, and routing between nodes. The architecture has a single orchestrator node and four specialist nodes: eligibility, technical, tariff, and application.

Each customer message triggers the orchestrator first. The orchestrator reads the full conversation history alongside the current state, extracts any new information the customer has provided, and decides which specialist should respond. It does this in a single structured LLM call using Pydantic, which returns both the state updates (account ID, system size, installer name, NMI, and so on) and the routing decision at the same time.

Routing is then handled deterministically in code rather than by the model:

```python
def route_from_orchestrator(state: CERState) -> str:
    return state.get("next_specialist", "eligibility")
```

Each specialist handles its domain using tool calls at runtime:

- The eligibility specialist retrieves account details from the CRM and checks state-level eligibility rules against the relevant jurisdiction
- The technical specialist checks inverter compliance against AS4777 and confirms whether the customer's meter supports export metering
- The tariff specialist retrieves the customer's feed-in rate, calculates estimated annual earnings based on system size, and introduces the VPP program
- The application specialist checks what information is still missing, collects the remaining details, presents a full summary, and submits on explicit customer confirmation

A key design decision was that each specialist directly sets its own state flag when it confirms its outcome. When the eligibility specialist's response contains a specific handoff phrase, the node code detects it and writes `eligibility_status: "eligible"` into shared state directly. This makes the routing transition reliable regardless of how the model phrases the confirmation in a given turn.

A second design decision was that when a specialist confirms its outcome, the graph automatically routes to the next specialist in the same turn rather than waiting for the customer to say something. The customer sees the eligibility confirmation and the technical specialist's first question in sequence, with no additional prompt required from them between the two. This is handled through conditional edges that loop back through the orchestrator when a specialist sets its flag:

```python
def route_after_eligibility(state: CERState) -> str:
    if state.get("eligibility_status") == "eligible":
        return "orchestrator"
    return END
```

When the orchestrator is called mid-chain with no new user message, it skips the LLM call entirely and routes from state directly, which also avoids an Anthropic API constraint that requires conversations to end with a user message.

Each specialist also only sees a filtered view of the conversation history. Internal tool calls and tool results from other specialists are stripped out before the message history is passed to the current specialist. This keeps each specialist's context clean and prevents it from being confused by another specialist's internal reasoning.

The tariff specialist introduces Australia's National Electricity Market as part of the VPP explanation. The NEM settles at 5-minute intervals, and spot prices can reach the market price cap of $17,500 per MWh during peak demand events such as summer heatwaves or winter evening peaks. A home battery enrolled in a VPP can export during those windows and earn a revenue share well above the standard feed-in rate. This is particularly relevant for customers who have both solar and a battery, which is the primary use case this agent is designed around.

Here is a simple view of how the agent works:

<img src="/assets/diagrams/agent-6-architecture.svg" alt="Agent 6 architecture diagram" style="width:100%;max-width:760px;display:block;margin:1.5rem auto;">

## Why It Matters

The practical case for an agent like this sits in the operational economics of energy retail.

The volume of CER-related enquiries at Australian energy retailers is growing as solar and battery adoption accelerates. AEMO projects that distributed CER assets will represent a significant and growing share of grid capacity through the 2030s. Each enquiry that involves eligibility checking, technical verification, and application completion currently requires either a customer to navigate a multi-step process across different teams, or a specialist customer service team trained across all four domains. Neither scales easily with volume.

The VPP program adds a specific commercial dimension. Retailers with a VPP offering have a genuine product to sell to battery owners, but the conversation about it is currently buried inside a multi-step onboarding process that many customers abandon before reaching it. An agent that surfaces the VPP explanation at exactly the right moment in a coherent, guided conversation is more likely to convert a customer who would otherwise not have heard the pitch.

There is also a consistency argument. The eligibility rules vary by state. AS4777 has specific export limits by phase type. The feed-in rate depends on the customer's current plan. A human agent working across all of these simultaneously is likely to make errors on the detail, particularly under volume pressure. The agent looks up the correct values every time.

## The Agent in Action

### Eligibility and automatic handoff to technical

A customer contacts the retailer with a solar and battery system. Once they confirm they are an owner-occupier, the agent confirms eligibility and immediately routes to the technical specialist without requiring another customer message.

<pre>
[Eligibility Specialist]
Agent: Great news, James — you are fully eligible for a feed-in tariff.

| Field       | Status                                       |
|-------------|----------------------------------------------|
| Ownership   | Owner-occupier                               |
| State       | NSW — eligible for systems up to 100kW       |
| Meter type  | Smart meter — no upgrade required            |

I'll now pass you to our Technical Requirements Specialist who will
check your system setup.

[Technical Requirements Specialist]
Agent: To check your system meets the requirements for grid export, I
need a few details. What is your inverter brand and model, your system
size in kW, and is the property on single or three-phase power?

Customer: 6.6kW system, Fronius Primo inverter, single phase
</pre>

The two responses appear in the same turn. The customer does not need to type anything to trigger the handoff.

### VPP introduction from the tariff specialist

Once the technical check passes, the tariff specialist retrieves the feed-in rate, calculates estimated earnings, and makes the case for the VPP program.

<pre>
[Tariff Specialist]
Agent: Here is your feed-in tariff summary:

| Detail                      | Value                      |
|-----------------------------|----------------------------|
| Feed-in tariff rate         | 6.5 cents per kWh          |
| Estimated annual export     | ~2,891 kWh                 |
| Estimated annual earnings   | ~$188/year                 |

Because you have a home battery, you are also eligible for the Volta
VPP program. Australia's National Electricity Market settles every 5
minutes, and spot prices can spike to $17,500/MWh during peak demand
events. Volta monitors these prices in real time and can dispatch
enrolled batteries to export during high-value windows, sharing the
additional revenue back to you.

VPP participation is estimated to add $200 to $600 per year on top
of your standard feed-in earnings, with a 12-month initial contract
and 30 days written notice to exit after that.

The application specialist will handle enrolment if you decide to
proceed.
</pre>

### Explicit confirmation before submission

Before the application is submitted, the agent presents a full summary and requires the customer to confirm.

<pre>
[Application Specialist]
Agent: Here is a summary of your application before we submit:

| Field                    | Detail                      |
|--------------------------|-----------------------------|
| Customer name            | James Okafor                |
| Account ID               | ACC-20041                   |
| System size              | 6.6 kW                      |
| Inverter                 | Fronius Primo               |
| NMI                      | 4102567891                  |
| Installer                | SolarEdge Solutions         |
| Feed-in tariff rate      | 6.5c per kWh                |
| Estimated annual earnings| ~$187.90                    |
| VPP enrolment            | Yes                         |

Type Y to confirm and submit your application.

Customer: Y

[Application Specialist]
Agent: Your application is now complete.

Application reference: CER-847293

Processing will take up to 5 business days. A confirmation email will
follow, and a Volta technician will contact you within 2 business days
to confirm smart meter configuration for export metering.
</pre>

The submission only proceeds after the customer types Y. The agent does not submit speculatively, which matters in a context where a submitted application triggers downstream processes.

*This is a proof of concept. A production implementation would connect to a live CRM, billing system, and application management platform rather than mock data files; operate over a web or app interface rather than a terminal; include identity verification before sharing account data; handle the non-compliant and meter-upgrade-required technical paths through to completion; and persist state across sessions so a customer who drops out mid-process does not have to start again. The intent here is to show that the orchestrator-plus-specialist pattern works reliably across a multi-step domain, and to demonstrate the specific design choices (state-driven routing, specialist-owned flags, filtered message history, and auto-chaining) that make a multi-agent conversation feel coherent rather than mechanical.*
