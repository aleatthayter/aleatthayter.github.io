---
layout: post
title: "Agent 4: Billing Query Resolver"
date: 2026-04-26
---

*View the project on GitHub: [billing-query-resolver](https://github.com/aleatthayter/billing-query-resolver)*

## The Challenge

Energy retailers are high-volume, high-frustration support environments. Customers contact them when something has gone wrong with their bill, when they cannot afford to pay, when they have been disconnected, or when they simply do not understand why they owe what they owe. Call centres are expensive, wait times are long, and most queries are repetitive enough that the same small set of questions accounts for the majority of contacts: what do I owe, why is this higher than last month, what rate am I on, did my payment go through.

At the same time, the energy retail environment is tightly regulated. Customers experiencing financial hardship are entitled by law to a formal support program under the National Energy Customer Framework. Disconnections are subject to strict procedural requirements. Complaints must be handled through a defined process. An AI agent operating in this context cannot simply resolve queries autonomously. It needs to know when to hand over to a human, and it needs to do so reliably, not occasionally.

The combination of high query volume, repetitive content, and non-negotiable escalation rules makes this a natural fit for an AI support agent.

## How It Works

This agent runs as a multi-turn conversation in the terminal, playing the role of a support agent for Volta Energy, a fictional Australian energy retailer. Here is what it does:

- Maintains full conversation context across turns, so the customer does not need to repeat themselves and the agent can answer follow-up questions naturally
- Calls tools at runtime to retrieve account information, billing data, payment history, and tariff details, rather than having those values baked into the prompt
- Follows a strict set of rules about when to escalate: any mention of financial hardship, a disconnection notice dispute, or a request to lodge a formal complaint immediately triggers a transfer to a human agent
- When escalating, generates a summary of the conversation and passes it to the human agent so they can pick up with full context, rather than making the customer repeat themselves again

The agent does not guess at account data or make up tariff rates. When it needs information, it calls a tool. When the query is outside what it can resolve autonomously, it escalates. Those two constraints, grounded data and defined escalation boundaries, are what make the agent safe to use in a regulated customer context.

The agent is built in Python using LangChain as the orchestration layer and Claude (Anthropic) as the underlying model. Tool use is the key design pattern here. Rather than embedding account and tariff data in the system prompt, the agent is given four tools it can choose to call: one for account and billing data, one for payment history, one for tariff details, and one for escalation. At each turn, the model decides which tool it needs, calls it, reads the result, and then responds. This is meaningfully different from the pipeline pattern used in the earlier agents, where inputs were processed in a fixed sequence. Here the agent is reasoning about what it needs and acting accordingly, which is what makes it feel like a conversation rather than a lookup.

The agent's persona was designed around a specific constraint: it should not perform empathy it cannot follow through on. An agent that responds to a hardship disclosure with warmth and then attempts to resolve the situation autonomously is more damaging than one that acknowledges, escalates immediately, and tells the customer exactly what happens next. That constraint shaped the three escalation categories (hardship, disconnection disputes, and formal complaints) and the agent's tone more broadly. Outside those categories, the agent is direct and factual rather than warm. The intent is to feel efficient and trustworthy, not to simulate a human support interaction.

Here is a simple view of how the agent works:

<img src="/assets/diagrams/agent-4-architecture.svg" alt="Agent 4 architecture diagram" style="width:100%;max-width:760px;display:block;margin:1.5rem auto;">

## Why It Matters

The value of a well-designed support agent in this context is not just cost reduction. It is about raising the floor on the customer experience:

- Consistent application of entitlements: A human agent under pressure during a busy shift might not always raise the hardship program with a customer who mentions they are struggling. A correctly configured AI agent will escalate every single time, without exception. Regulatory compliance is not dependent on the individual agent's recall.
- Faster resolution for routine queries: A customer asking why their bill is higher than last month can get a factual, grounded answer in seconds. The agent retrieves their actual usage and rate, compares it to prior periods, and explains the difference. No hold time, no transfer.
- Context-preserving escalation: When the agent does hand over to a human, it passes a summary of the conversation. The human agent does not ask the customer to start again. This is a small thing that makes an enormous difference to a customer who is already frustrated.
- Defined boundaries: The agent knows what it can and cannot do. It does not attempt to offer payment extensions beyond its authority, provide legal advice, or handle a formal complaint without human involvement. Knowing the limits of what should be automated is as important as knowing what can be.

## The Agent in Action

### Conversation 1: Billing enquiry

A customer contacts support about an unexpectedly high bill. The agent retrieves the account data and the tariff rates, and provides a grounded answer across multiple turns.

<pre>
Customer: My bill seems a lot higher than usual. Can you tell me what I owe?

Agent: Your current bill for March 2026 is $287.45, covering 842 kWh of
usage over the period 1 to 31 March. It was due on 14 April and is
currently showing as overdue. Your previous three bills came in between
$178 and $201, so usage this month is noticeably higher. Would you like
me to look at your current rate to help explain the charge?

Customer: Yes please, what rate am I on?

Agent: You are on the Volta Flex Variable plan, which charges 28.4 cents
per kWh for all usage, plus a daily supply charge of 89.5 cents. At 842
kWh for the month, the usage component of your bill would be around
$239.17, with the supply charge adding approximately $27.75 for a
31-day billing period. If your usage has increased, that rate means even
a moderate spike has a clear impact on the total.
</pre>

### Conversation 2: Hardship escalation

A customer mentions they are unable to pay. The agent immediately escalates with a context summary rather than attempting to resolve the situation autonomously.

<pre>
Customer: I've just been made redundant and I have no idea how I'm going
to pay this bill.

Agent: I am sorry to hear that. I am going to transfer you now to one of
our specialist support team members who handles situations like this
directly. They will have full context of our conversation and can talk
you through Volta Energy's hardship program, which exists specifically
to help customers who are experiencing financial difficulty. You will
not need to repeat yourself.

--- [INTERNAL — Human agent handoff] ---
Reason:  Customer has disclosed they have recently been made redundant
         and cannot pay their current bill.
Summary: Customer on account ACC-10042 contacted about a $287.45
         overdue bill for March 2026. During the conversation they
         disclosed they have lost their job and cannot pay. Transferred
         for hardship assessment.
-----------------------------------------
</pre>

The second conversation is as important as the first. The agent does not offer an unauthorised payment extension, does not minimise the situation, and does not leave the customer on hold while it escalates internally. It transfers with context, which is what a well-trained human agent would do.

*This is a proof of concept. A production implementation would connect to a live CRM and billing system, operate over a web or app chat interface rather than a terminal, include identity verification before sharing account data, and integrate with the retailer's existing case management and complaint handling workflows. The intent here is to demonstrate that the tool-use pattern, context retention, and rules-based escalation logic all work correctly, and to show what a customer-facing AI agent in a regulated industry needs to get right before it can be trusted.*
