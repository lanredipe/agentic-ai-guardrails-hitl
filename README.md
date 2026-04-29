# The Sentinel – Governed Agentic Workflow & HITL Guardrails

## Overview
The Sentinel is a production-oriented Agentic RAG architecture designed to wrap autonomous multi-hop reasoning in a strict Safety & Governance Layer. Moving beyond simple retrieval, this system acts as a governance layer that intercepts adversarial inputs, enforces algorithmic budgeting, and requires explicit Human-in-the-Loop (HITL) authorization for ambiguous or high-cost execution paths.

Built on **LangGraph**, the system utilizes persistent checkpointers (`MemorySaver`) not just for pausing execution, but to enable manual state mutation (injecting facts directly into the agent's memory mid-run) to ensure absolute brand safety and factual grounding.

---

## System Design: Control Plane vs Execution Plane
To maintain strict governance over the LLM, the architecture enforces a rigid separation of concerns:
* **Control Plane (Sentinel):** Handles policy enforcement, cost governance, HITL interruptions, and safety validation prior to and following execution.
* **Execution Plane (Investigator):** Performs planning, retrieval, source grading, and generation.

This separation ensures that autonomous reasoning remains strictly constrained within enforceable, deterministic safety boundaries.

---

## Risk Model & Mitigation Strategy
The system is engineered to mitigate key risks inherent in autonomous agent execution:
* **Prompt Injection Risk:** Mitigated via the `admission_controller` using deterministic pattern checks and LLM-based semantic validation.
* **Data Exfiltration Risk (PII):** Prevented through regex-based masking of sensitive data (e.g., emails, phone numbers) before any external tool invocation.
* **Cost Overrun Risk:** Controlled via the `financial_brake` which enforces a hard-coded `cost_threshold`.
* **Hallucination Risk:** Addressed through a post-generation Natural Language Inference (NLI) Fact Checker that serves as the final epistemic validation layer.
* **Untrusted Source Risk:** Managed by the `source_grader` via strict exact-match domain blocklisting.

---

## Core Engineering Features
* **Dual-Layered Guardrails:** Combines deterministic (zero-latency regex/blocklists) and probabilistic (LLM-evaluated) checks.
* **The Financial Brake (Algorithmic Budgeting):** Tracks continuous API token expenditure (`current_cost`), triggering a circuit breaker if limits are exceeded.
* **Stateful HITL & "Time Travel" Editing:** Leverages LangGraph's `interrupt_before` functionality. Operators can approve paths or use `graph.update_state` to manually overwrite the `research_ledger`, actively steering the agent without restarting the run.
* **Dual-Memory Persistence (Neo4j + FAISS):** Validated answers are embedded into FAISS, while a dedicated extraction parser maps the output into semantic triples (`subject`, `predicate`, `object`) for persistent **Neo4j Knowledge Graph** storage.

---

## Failure Handling Strategy
The system enforces strict, non-speculative behavior when encountering edge cases:
* Policy violations result in an immediate request denial.
* Budget overruns trigger HITL intervention before execution can continue.
* Hallucination detection appends explicit warning tags to the final output rather than discarding the run.
* Insufficient retrieved data results in a bounded refusal response instead of fabricated or hallucinated answers.

---

## Observability & Compliance Audit
The system maintains a structured audit trail, ensuring full traceability and accountability for every agent decision. Custom LangChain callbacks capture:
* Policy decisions (Passed/Failed)
* Cost accumulation across execution steps
* Human intervention actions (Approve/Edit)
* Safety enforcement triggers

**Forensic Execution Log:**
```text
--- NODE: ADMISSION CONTROLLER ---
Policy Check: Passed.
--- NODE: FINANCIAL BRAKE ---
[System] Budget exceeded ($5.00 > $0.50). Interrupting for HITL.

[SYSTEM PAUSED] Cost Limit Breached: $5.00
[PIVOT LOGIC] Ambiguous entity detected. Presenting Plan for Approval:
--- AGENT PLAN ---
1. Collect Authoritative Sources...
2. Synthesize Narrative...
AMBIGUOUS: Yes

Enter choice (c/e): e
User Intervention: State Edited
INJECT FACTS INTO LEDGER: [Operator manually injects required context]
```

---

## Technical Architecture & State Contracts
Execution is controlled via a directed **StateGraph** utilizing an `AgentState` schema (`masked_query`, `current_cost`, `research_ledger`, `policy_violation`).

### Node Contracts
1. **Admission Controller**: Mutates `masked_query` via regex; sets `policy_violation` if injection is detected.
2. **Planner**: Evaluates query scope and mutates the `is_ambiguous` boolean flag.
3. **Financial Brake**: Evaluates `current_cost` against `cost_threshold`.
    * *Conditional Edge*: Cost exceeded OR ambiguous → **human_intervention_node**.
4. **HITL Node**: Checkpoint node waiting for operator input (Approve/Edit).
5. **Source Grader**: Filters retrieved documents against an exact-match `BLOCKLIST`.
6. **Fact Checker**: Ensures all generated claims are strictly entailed by verified context.
7. **Memory Updater**: Commits data to FAISS and executes Cypher queries to update Neo4j.

---

## Tech Stack
* **Orchestration:** LangGraph, LangChain
* **LLMs:** Groq (Llama-3.1-70B/8B Architecture)
* **Knowledge Graph:** Neo4j (Cypher Integration)
* **Vector Store:** FAISS (HuggingFace Embeddings)
* **Search Engine:** Tavily API

---

## Author
**Olanrewaju Stephen Amudipe**
*System Architect & Orchestrator*
*lanreamudipe@gmail.com*

Architected the defense-in-depth safety layer, designed the Control Plane vs Execution Plane logic, and implemented the LangGraph checkpointer system for active human-in-the-loop state mutation.
