---
layout: post
title: "Agent 3: Controlled Document Interpreter"
date: 2026-04-25
---

*View the project on GitHub: [controlled-document-interpreter](https://github.com/aleatthayter/controlled-document-interpreter)*

## The Challenge

Mining and energy operations run on controlled documents. Isolation procedures, confined space entry requirements, hot work permit rules, environmental licence conditions. These documents define how work is done safely and within regulatory requirements. Sites can carry hundreds of them, each with its own version history, approval chain, and review date.

The problem is access. Not physical access, because most sites have these documents stored somewhere. The problem is practical access. A field worker preparing to enter a confined space needs to know exactly what the permit requirements are and what atmospheric conditions are acceptable before entry. A supervisor issuing a hot work permit needs to know whether the work falls within a confined space and what that triggers. An environment officer needs to know what the discharge turbidity limit is under the site licence. In practice, finding the right answer in the right document, in the right version, quickly enough to be useful, is harder than it should be.

The consequences of getting it wrong range from regulatory non-compliance to serious injury. When workers cannot find answers easily, they rely on memory, ask colleagues, or make reasonable assumptions. Sometimes those assumptions are wrong.

## How It Works

This agent ingests a set of approved controlled documents and answers plain-language questions drawn strictly from them. Here is what it does:

- Loads controlled documents (procedures, permit conditions, safety protocols) and indexes them into a local vector store for fast retrieval
- When a question is asked, retrieves the most relevant document sections using semantic search
- Passes those sections to Claude with a strict instruction: answer only from what is in the documents, cite the source and section, and refuse to answer if the information is not covered
- Returns a plain-language answer with a clear citation so the worker can verify it in the source document

The agent does not guess, infer, or draw on general knowledge. If the information is not in the loaded documents, it says so and directs the worker to their supervisor or document controller. That refusal behaviour is intentional and important. An agent that improvises answers to safety-critical questions is more dangerous than one that admits the limits of what it knows.

The agent is built in Python using LangChain as the orchestration layer, Claude (Anthropic) as the underlying model, and ChromaDB as the local vector store for document retrieval. Pydantic models enforce the output schema for the evaluation pipeline, ensuring every result is consistently structured and comparable across runs. The document search is also exposed as an MCP (Model Context Protocol) server, which means any MCP-compatible client including Claude Desktop can query the controlled documents directly without going through the command line. MCP is a standardised protocol for connecting AI models to external data sources, and exposing the document index this way means it can be consumed as a tool by other agents or systems without any custom integration work.

Here is a simple view of how the agent works:

<pre>
+----------------------------------------------------------+
|                     DATA SOURCES                         |
|                                                          |
|     Controlled Documents (Procedures, Permits, SOPs)     |
|                         |                                |
|                         v                                |
|         CHUNK AND INDEX INTO VECTOR STORE                |
|                         |                                |
|                         v                                |
|          WORKER ASKS A PLAIN-LANGUAGE QUESTION           |
|                         |                                |
|                         v                                |
|        RETRIEVE RELEVANT DOCUMENT SECTIONS               |
|                         |                                |
|                         v                                |
|    AI ANSWERS FROM DOCUMENTS ONLY, CITES SOURCE         |
|                         |                                |
|                         v                                |
|         REFUSE IF INFORMATION NOT IN DOCUMENTS           |
+----------------------------------------------------------+
</pre>

### Evaluation and Tracing

Deploying an AI agent against safety-critical documents introduces a question that does not apply in lower-stakes contexts: how do you know it is working correctly before you trust it?

This agent includes a formal evaluation suite that measures three things. Behaviour accuracy tracks whether the agent answers when it should and refuses when it should. Faithfulness measures whether the answer is genuinely supported by the retrieved document sections, using a second Claude call to judge whether the response introduces any claim not found in the source. Relevance measures whether the retrieval step surfaced the right document sections in the first place. Each of these can fail independently and for different reasons, so measuring them separately is more informative than a single pass or fail.

Every query and response is traced through LangSmith, which logs the full chain: the question received, the document chunks retrieved, the prompt constructed, and the model output returned. This makes it possible to inspect exactly what happened when an answer is wrong or a retrieval misses the relevant section, rather than having to reproduce the failure from scratch. In a regulated environment, that audit trail also serves a compliance function: you can demonstrate what the agent was given and what it returned for any interaction.

The evaluation suite runs 12 test cases against the four loaded documents, covering factual questions, safety-critical edge cases, and deliberate out-of-scope questions that should trigger a refusal.

## Why It Matters

Getting controlled document access right has direct safety and compliance consequences:

- Reduced reliance on memory and word-of-mouth: Workers currently fill knowledge gaps by asking colleagues or relying on what they remember from training. An agent grounded in current, approved documents reduces that dependency and the risk that comes with it.
- Faster permit preparation: Supervisors preparing confined space or hot work permits often need to cross-reference multiple documents to confirm requirements. An agent that surfaces the relevant conditions quickly reduces that time and the likelihood of a missed requirement.
- Consistent answers across the workforce: A new starter and a twenty-year veteran asking the same question will receive the same answer, drawn from the same version of the same document. The quality of safety knowledge does not depend on who you happen to ask.
- Measurable reliability: The evaluation suite provides a quantitative view of how well the agent is performing before it is trusted in the field. Faithfulness and relevance scores can be tracked over time as documents are updated or the document set expands.
- Document version control: The agent only answers from the documents it has been given. Keeping those documents current is the same governance challenge as keeping a document management system current, but the answer returned is always traceable to a specific version.
- Composable integration: Exposing the document search through MCP means the controlled document knowledge becomes a standardised integration point. A planning tool, a permit management system, or another agent can query the same documents through a consistent interface without bespoke integration work for each connection.

## The Agent in Action

### Before: The Lookup Problem

Here is a sample of the kinds of questions that arise during day-to-day work on a mine site, alongside the document a worker would need to locate and read to find the answer.

| Question | Document Required | Typical Resolution |
|---|---|---|
| Can the standby person enter the confined space to help? | Confined Space Entry Procedure, Section 5.3 | Ask the supervisor or safety officer |
| How long after hot work must the fire watch continue? | Hot Work Permit Procedure, Section 5.2 | Check the procedure document if you can find it |
| Who is authorised to perform electrical isolation? | Isolation Procedure, Section 5.1 | Ask the electrical supervisor |
| What is the nighttime noise limit at the boundary? | Environmental Permit Conditions, Section 3.2 | Contact the environment team |
| Can blasting happen on a public holiday? | Environmental Permit Conditions, Section 3.3 | Check the licence document |

Each answer exists in a document. The friction is in finding it quickly under operational pressure.

### Agent Output

Here is a sample of how the agent responds to the same questions.

**Question:** Can the standby person enter the confined space to assist an injured worker?

**Answer:** No. According to the Confined Space Entry Procedure (WMS-HSE-011), Section 5.3, the standby person must not enter the confined space under any circumstances. If a worker requires assistance, the standby person must initiate emergency response as outlined in Section 5.4. Retrieval lines and harnesses are required for this purpose (Section 6.1).

---

**Question:** How long after hot work is completed must the fire watch continue?

**Answer:** The fire watch must continue for a minimum of 30 minutes after hot work is completed, per the Hot Work Permit Procedure (WMS-HSE-019), Section 5.2. The fire watch person must have access to a fire extinguisher and be trained in its use during this period.

---

**Question:** What is the approved contractor rate for scaffolding labour?

**Answer:** This information is not covered in the current controlled documents. Please consult your supervisor or the document controller.

The last example is the refusal case. The agent does not attempt to answer a question outside its scope. That behaviour is tested explicitly in the evaluation suite and is as important as getting factual answers right.

### Evaluation Results

Running the evaluation suite against 12 test cases produced the following results:

| Metric | Score |
|---|---|
| Behaviour accuracy (answer vs refuse) | 100% |
| Average faithfulness | 0.96 |
| Average retrieval relevance | 0.94 |

Faithfulness and relevance are scored from 0.0 to 1.0 by a separate Claude call acting as judge, assessing whether the answer is supported by the retrieved chunks and whether the right chunks were retrieved. LangSmith traces each run end-to-end, making it straightforward to inspect any case where scores fall below expected thresholds.

*This is a proof of concept. A production implementation would connect to a live document management system to pull current approved versions automatically, would include access controls so workers can only query documents relevant to their role and site, and would carry a more extensive evaluation suite updated each time new documents are added. The intent here is to demonstrate that this kind of grounded, auditable document access is achievable and to show how evaluation and tracing are built into the approach from the start rather than added afterwards.*
