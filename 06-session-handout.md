# Session Handout: Diagnose an AI System

## Your Assignment

Act as the AI system team responsible for a generative AI feature that is
failing in production. Choose one system and one failure scenario, map how the
system probably works, diagnose what could be wrong, and recommend a defensible
response.

No coding or ML implementation is required. Your work should show how evidence
would distinguish possible causes. A list of generic AI improvements is not a
diagnosis.

## Core Deliverables

Submit four connected artifacts:

1. **System and trust-boundary map** using
   [`templates/system-map.md`](templates/system-map.md).
2. **Diagnostic register** using
   [`templates/diagnostic-register.md`](templates/diagnostic-register.md).
3. **Prioritized recovery plan** using
   [`templates/recovery-plan.md`](templates/recovery-plan.md).
4. **Three-slide decision summary** covering what is broken, why you think it
   is broken, and what should happen next.

## What Good Work Looks Like

The goal is not to guess the hidden technical answer. It is to make the best
decision that the available evidence supports.

- If the evidence is limited, recommend the next investigation and containment
  action. Do not claim that a root cause is verified.
- If one cause is strongly supported, recommend the smallest corrective action
  and explain how you would detect a regression.
- If the remaining risk is unacceptable, recommend pausing, restricting, or
  rolling back the feature.

Your four artifacts should tell one consistent story: the map identifies where
the failure could occur, the register distinguishes the causes, the recovery
plan responds to the evidence, and the slides communicate the decision.

## 1. Choose a System

Choose one of these systems or propose another realistic GenAI product. Groups
should use different combinations where possible.

| System | Purpose | Likely components and constraints |
|---|---|---|
| Customer-support copilot | Helps agents triage tickets and draft replies | RAG over help content and past cases; human approval; ticket integration |
| Internal policy assistant | Answers employee questions with traceable policy evidence | Role-based retrieval; document versions; strict citations and escalation |
| Sales-email assistant | Drafts outreach using account context and brand rules | CRM lookup; prompt templates; approval before send; personal data |
| Analytics-summary bot | Explains dashboards and changes in key metrics | BI queries; metric definitions; structured data; strong anti-speculation rules |
| HR-onboarding assistant | Guides new hires through tasks and policies | Conversation state; HR documents; task integration; sensitive employee data |
| Contract-review assistant | Surfaces clauses and evidence for legal review | Long documents; access controls; citations; no autonomous legal decision |
| E-commerce shopping assistant | Finds products and answers product questions | Catalogue retrieval; availability and price tools; ranking; multilingual input |
| Incident-response copilot | Summarizes alerts and proposes investigation steps | Live operational data; high urgency; tool permissions; audit trail |

Write down the user, task, supported decision, architecture owner, affected
people, and what the system must never do.

## 2. Add One Failure Scenario

Choose one primary failure. Make the symptom concrete enough to reproduce.

- **Outdated answers:** the system confidently uses an old policy or product
  version.
- **Irrelevant retrieval:** cited passages are real but do not answer the
  question.
- **Lost or contaminated context:** the system forgets a step, uses stale state,
  or mixes information between users.
- **Inconsistent behavior:** output format, tone, or decision varies under
  equivalent inputs.
- **Slow or unreliable responses:** latency, timeouts, or retry loops damage the
  workflow.
- **Unsupported claims:** the answer contains details absent from available
  evidence.
- **Privacy or permission failure:** the response exposes information the user
  should not access.
- **Unsafe action:** a tool executes an incorrect or insufficiently authorized
  operation.
- **Quality decline:** a previously acceptable workflow worsens after content,
  traffic, prompt, model, or product changes.

Add a user-visible example, the likely impact, the first containment action,
and an explicit statement of what is still unknown.

### Freeze the Case Facts

Write a short case statement before diagnosing. Treat only these items as
facts:

- one failing request and response or action;
- the expected behavior;
- who was affected and how;
- when the failure occurred;
- any known recent change.

If you invent a scenario detail for the exercise, label it **synthetic** and
keep it stable. Do not add convenient logs or architecture facts later simply
to support your preferred explanation.

## 3. Map the Architecture Before Diagnosing

Draw the request and data flows. Include only components that are relevant to
your chosen system, but consider:

- user, interface, and identity;
- application or orchestration logic;
- prompts and model configuration;
- source content, ingestion, retrieval, and citations;
- session state or memory;
- external tools and business systems;
- permissions and human approval;
- validation, fallback, and escalation;
- logs, evaluations, monitoring, and feedback.

Mark trust boundaries and label every uncertain component as an assumption.
Do not let an LLM silently invent missing architecture.

![A RAG system separates content ingestion from request serving](assets/google-rag-reference-architecture.png)

*Source: Google Cloud, [RAG infrastructure for generative AI](https://docs.cloud.google.com/architecture/rag-genai-gemini-enterprise-vertexai), licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Use the figure as a component checklist, not as a required vendor stack.*

### Identify the Evidence You Would Request

You are not expected to have access to a real production system. Use the map to
request the smallest useful evidence rather than inventing it.

| Evidence request | What it could reveal |
|---|---|
| Failing request with prompt, model, and configuration versions | Whether the behavior is reproducible and linked to a change |
| Retrieved passages, ranking, and permission filters | Whether the right evidence reached the model and was authorized |
| Session or memory state | Whether context was missing, stale, or shared incorrectly |
| Tool request, validated arguments, result, and approval record | Whether an unsafe or incorrect action crossed a trust boundary |
| Component timings, retries, and dependency status | Where latency or availability degraded |
| Fixed evaluation results before and after a change | Which behavior or user segment regressed |

![A trace view showing one request broken into component-level spans](assets/grafana-trace-view.png)

*Source: Grafana Labs, [Traces in Explore](https://grafana.com/docs/grafana/latest/visualizations/explore/trace-integration/). A trace is one example of evidence that can separate a slow model call from slow retrieval, tools, retries, or another dependency.*

## 4. Build Competing Hypotheses

List at least three materially different causes. Avoid naming the same idea at
different levels, such as "bad RAG," "bad retrieval," and "wrong documents."

For each hypothesis, record:

- the affected component;
- why it could produce the symptom;
- evidence supporting or weakening it;
- the smallest test or observation that would distinguish it;
- confidence before and after considering the evidence.

Use this question to prioritize investigation:

> Which next piece of evidence would most change what we decide to do?

Avoid diagnosis by association:

> Weak: "The answer hallucinated, so we should fine-tune the model."
>
> Stronger: "The claim is unsupported. We need to inspect the retrieved
> passages and final context first. If the required evidence was absent, test
> retrieval; if it was present but ignored, test generation and validation."

## 5. Use an LLM to Challenge the Diagnosis

First complete your own map and initial hypotheses. Then give an approved AI
assistant only non-sensitive scenario information and the relevant project
files.

```text
Act as a skeptical AI architecture review panel.

Review our system map, observed symptom, and diagnostic register. Separate
observations from assumptions. Find plausible causes we missed, evidence that
would distinguish them, and fixes that do not address the stated cause.

Do not invent system details, logs, user research, legal conclusions, or test
results. Mark anything that requires verification by an engineer, system owner,
security specialist, or primary source.
```

Record the most important suggestion and whether you accepted, edited, or
rejected it. An LLM-generated root cause remains a hypothesis until verified.

## 6. Propose and Prioritize the Recovery Plan

Include three to five actions across these horizons:

- **Contain:** reduce current user or business harm.
- **Investigate:** obtain the evidence needed to verify the cause.
- **Correct:** apply the smallest change linked to the evidence.
- **Prevent:** add regression evaluation, monitoring, ownership, or process
  changes that reduce recurrence.

Score each action on impact, effort, risk reduction, confidence in the
diagnosis, and reversibility. A high-impact fix with little supporting evidence
may belong after a cheap diagnostic test.

Use the dimensions consistently:

| Dimension | Question |
|---|---|
| Impact | How much user or business harm could this action reduce? |
| Effort | How much coordination, implementation, testing, or migration is required? |
| Risk reduction | Does it prevent recurrence or only hide the symptom? |
| Confidence | How strongly does current evidence connect the action to the cause? |
| Reversibility | Can the team stop or undo the action safely if it is wrong? |

Your plan must explicitly consider the non-ML baseline: workflow clarification,
content correction, permissions, user-interface changes, escalation, or manual
review may outperform a model change.

## 7. Define Proof of Recovery

State what would make the team confident enough to close the incident or run a
bounded release:

- a fixed evaluation case based on the original failure;
- representative and edge-case tests;
- user, retrieval, generation, action, and operational measures as relevant;
- guardrail thresholds and unacceptable outcomes;
- monitoring owner and review cadence;
- rollback or shutdown condition;
- remaining uncertainty and the person authorized to accept it.

![Evaluation during development connected to production monitoring and human feedback](assets/mlflow-evaluation-monitoring.png)

*Source: Microsoft Learn and Azure Databricks, [Evaluate and monitor AI agents](https://learn.microsoft.com/en-us/azure/databricks/mlflow3/genai/eval-monitor). Use the loop to connect the original failure, fixed evaluation cases, production evidence, and expert feedback rather than treating recovery as a one-time test.*

## 8. Prepare the Three-Slide Summary

### Slide 1: What Is Broken

Show the system purpose, relevant architecture flow, observed symptom, impact,
and immediate containment. Distinguish facts from assumptions.

### Slide 2: Why We Think It Is Broken

Show the ranked hypotheses, strongest evidence, missing evidence, and the next
test that best separates the causes.

### Slide 3: What We Recommend

Show the prioritized recovery plan, proof of recovery, owner, guardrails, and
what evidence would change the recommendation.

## Review Criteria

Use these criteria for a final peer review before presenting.

| Criterion | Strong evidence in the submission |
|---|---|
| Architecture clarity | The relevant request, data, permission, and fallback paths are visible without pretending unknown components are known. |
| Diagnostic quality | Several distinct causes are ranked, and each has evidence that could support or weaken it. |
| Decision quality | Containment, investigation, correction, and prevention actions follow from the diagnosis and include owners. |
| Proof of recovery | The original failure becomes a regression test with measurable pass, guardrail, and rollback conditions. |
| AI collaboration | The LLM challenged the work, while people verified claims and retained decision accountability. |

## Submission Check

Before presenting, confirm that:

- all four deliverables describe the same system, failure, and decision;
- facts, synthetic evidence, assumptions, unknowns, and verified causes are
  visibly distinct;
- each priority links to evidence or an explicit investigation need;
- external claims use checked authoritative sources;
- the LLM collaboration note shows what people accepted, edited, or rejected.

## Optional Extensions

Complete these only after the four core deliverables are coherent.

- **Architecture variants:** compare two credible designs, such as direct
  context versus RAG, synchronous versus asynchronous action, or hosted versus
  local model access.
- **Failure injection:** introduce a stale source, missing permission filter,
  unavailable tool, long context, or changed model version and predict which
  evidence would reveal it.
- **Evaluation set:** write ten representative, edge, refusal, and adversarial
  cases with expected evidence and pass criteria.
- **Threat model:** map sensitive assets, trust boundaries, prompt-injection
  paths, excessive permissions, and mitigations using OWASP guidance.
- **Incident runbook:** define detection, severity, containment, communication,
  rollback, recovery, and post-incident ownership.
- **Cost and latency budget:** allocate acceptable time and cost across
  retrieval, model calls, tools, validation, and retries.
- **Change-impact review:** assess what must be retested when the source corpus,
  prompt, model, tool contract, or permission policy changes.
- **Low-fidelity prototype:** illustrate the failed and recovered user journey
  using synthetic, non-sensitive content. Do not present simulated output as
  production evidence.

## Reflection

- Which evidence changed your ranking of root causes most?
- Which attractive fix did you reject because it was unsupported or too broad?
- Where does accountability sit when the model, data, and workflow have
  different owners?
- Which part of the architecture would be hardest to observe during a real
  incident?
