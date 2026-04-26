---
layout: post
title: "Agent 5: Permit to Work Validator"
date: 2026-04-26
---

*View the project on GitHub: [permit-to-work-validator](https://github.com/aleatthayter/permit-to-work-validator)*

## The Challenge

Permit to work systems exist because certain types of work on a mine site carry a level of risk that requires formal controls before anyone starts. Hot work in an area with residual hydrocarbons. Entry into a confined space with limited oxygen or toxic gas risk. Work near high-voltage equipment. The permit system is meant to ensure that every required control is in place before work begins: the area is inspected, isolations are confirmed, emergency equipment is on-site, a standby person is briefed and positioned.

In practice, permits are completed under time pressure. Supervisors are managing multiple crews. A shutdown window is closing. The maintenance planner wants the work started. In that environment, a condition gets missed. Not through negligence, but through the kind of ordinary human error that time pressure produces. The fire watch is arranged but not formally documented on the permit. The atmospheric test is done but the result is not recorded against the specific condition it satisfies. The permit looks complete at a glance but has a gap.

The consequences are not theoretical. Incomplete permits are a recurring factor in serious and fatal incidents at mining and industrial sites globally. The permit system only works if it is followed completely, every time, without exception.

That is precisely the problem that a rules-based AI validator is well suited to address.

## How It Works

This agent validates permit-to-work submissions against a defined checklist of required conditions for the permit type. It supports hot work and confined space entry permits, which together cover two of the highest-risk work categories on a mining site.

The process works as follows:

- A permit submission is provided as a structured JSON file, containing the permit type, location, work description, and evidence submitted for each required condition
- The agent calls Claude with the full list of required conditions alongside the submitted evidence for each one, and uses Pydantic structured output to assess whether each condition is satisfied
- The approval decision is made deterministically in code, not by the model

That last point is the critical design decision. The model assesses evidence: it reasons about whether "Area inspected at 06:45, no flammable materials within 10m confirmed by Shift Supervisor" is sufficient to satisfy the condition "area cleared of flammable and combustible materials within 10 metres." But the model does not approve the permit. A single line of code does that:

```python
approved = all(c.satisfied for c in assessment.conditions)
```

If any condition is unsatisfied, the permit is blocked. The model cannot shortcut this logic. It cannot approve a permit by bypassing the approval step, because the approval step is not something it controls. What the model does control is whether each condition is assessed as satisfied or not. That assessment is where its judgement is applied, and where the quality of evidence provided matters. The architectural point is that the approval decision is an honest reflection of the model's per-condition assessments, not something the model can override by taking a different path through the logic.

The agent produces a validation report exported to Excel with a summary decision and a per-condition breakdown showing what evidence was submitted, how it was assessed, and whether it passed or failed.

Here is a simple view of how the agent works:

<img src="/assets/diagrams/agent-5-architecture.svg" alt="Agent 5 architecture diagram" style="width:100%;max-width:760px;display:block;margin:1.5rem auto;">

## Why It Matters

The business case for this agent is unusually direct because the risk it addresses has both a safety dimension and a financial one.

- Incident prevention: A missed condition on a confined space entry permit can result in a fatality. A missed fire watch condition on a hot work permit can result in a fire that injures workers and destroys equipment. The value of preventing a single serious incident at a mining operation runs to tens of millions of dollars when liability, regulatory response, production loss, and workforce impact are accounted for. An agent that catches incomplete permits before work starts sits at the beginning of that chain.
- Regulatory compliance: Mining safety regulators in Australia and internationally require that permit systems are followed correctly. An incomplete permit that is discovered after an incident demonstrates a systemic failure, not just an individual one. Having a validation layer that produces an audit trail for every permit reviewed creates a defensible record of process compliance.
- Consistent standards across sites and shifts: The quality of permit checking currently depends on the experience and attention of the individual supervisor approving it. A night shift with a newer supervisor carries different risk than a day shift with a senior one. This agent applies the same standard to every permit regardless of who is signing it or when.
- Speed without shortcuts: The agent does not slow the permitting process down. It runs in seconds. The benefit is that it forces completeness before work can proceed, rather than allowing gaps to be addressed later or overlooked entirely.

The financial framing is important here because it makes the case in terms that operations managers and CFOs respond to. The cost of running this agent against every permit issued at a site for a year is negligible. The cost of a single serious incident it might have prevented is not.

## The Agent in Action

### Hot work permit: blocked

The first sample permit is a hot work permit for a weld repair in a lube room. Four of the five required conditions have evidence submitted. The fire watch condition, which requires a named person briefed and on duty for the duration of work and 30 minutes after completion, has no evidence submitted.

<pre>
  HW-2026-0142  |  HOT WORK
  Location:  Ball Mill 2 — Lube Room
  Validated: 2026-04-28 06:55:12

  [PASS]  Area cleared of flammable and combustible materials within 10 metres
  [PASS]  Fire extinguisher on-site and fire watch person trained in its use
  [PASS]  Work area inspected and permit signed by an authorised supervisor
  [FAIL]  Fire watch person assigned, briefed, and on duty for duration of
          work and 30 minutes after completion
          Evidence: None submitted
          Reason:   No evidence has been provided that a fire watch person
                    has been assigned or briefed. This is a required condition
                    and cannot be waived.
  [PASS]  Planned start time and duration of hot work specified

  DECISION: BLOCKED
  1 condition(s) not satisfied:
    - Fire watch person assigned, briefed, and on duty for duration of
      work and 30 minutes after completion
</pre>

The permit cannot proceed until the fire watch condition is satisfied and resubmitted. The agent does not suggest workarounds or flag this as a minor issue. A required condition is either satisfied or it is not.

### Confined space entry permit: approved

The second sample is a confined space entry permit for an inspection inside a thickener tank. All six required conditions have clear, specific evidence submitted: calibrated atmospheric test results with instrument reference, LOTO reference number, a named standby person with confirmation they will not enter, rescue equipment inspected and rigged, emergency response reviewed with the full team.

<pre>
  CS-2026-0089  |  CONFINED SPACE
  Location:  Thickener Tank T-03 — Primary Circuit
  Validated: 2026-04-28 07:22:41

  [PASS]  Atmospheric testing completed for oxygen, flammable gases,
          and toxic contaminants
  [PASS]  Atmospheric test results within acceptable limits
  [PASS]  All energy sources isolated and locked out
  [PASS]  Standby person assigned, briefed, and positioned at the entry point
  [PASS]  Rescue equipment on-site including retrieval harness and lines
  [PASS]  Emergency response plan reviewed with all members of the entry team

  DECISION: APPROVED
  All required conditions satisfied.
</pre>

The contrast between the two permits is deliberate. The confined space permit passes because the evidence is specific and complete. The hot work permit is blocked because one condition has no evidence at all. The agent treats both consistently and without discretion.

*This is a proof of concept. A production implementation would integrate with the site's permit management system, pull required conditions dynamically from the controlled document index rather than a static configuration, support additional permit types, and include role-based access so only authorised supervisors can submit permits for validation. The intent here is to demonstrate that a hard-rule validation layer is achievable, and to show why the architectural decision to keep the approval logic in code rather than in the model matters.*
