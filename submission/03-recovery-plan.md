# Prioritized Recovery Plan: HR-Onboarding Assistant (Slack/Teams bot)

Legend: **[S]** synthetic exercise detail, **[A]** assumption, **[Proposed]** a threshold or role suggested by this team, to be confirmed by the owner. Scores are judgments, not measurements.

## Decision

- **Recommendation:** **Contain now, investigate, then apply the smallest fix the evidence supports. Do not pause the whole assistant.** Restrict it to private delivery and to approved documents until the cause is verified.
- **Decision owner:** Head of People Operations (accountable owner of the data and the risk), with IT/Security as co-approver. Notification decisions belong to Security, Legal and HR, not this team.
- **Reason in one sentence:** Private reminders were shown to a whole cohort and other symptoms suggest wider exposure, a disclosure cannot be recalled, and the cause is not verified, so we close the exposure paths first, capture the evidence, and only then correct.
- **Evidence that would change the recommendation:**
  - The scope query shows exposure beyond the shared channel (DMs, exports, logs, the model provider) or many affected people → pause the whole assistant and open the incident process.
  - The trace shows the model received only task name, status and date → lower the priority of data minimisation and refocus on the corpus, memory and injection hypotheses.
  - The trace shows a broad service account returning other people's data on request (H5) → add per-user authorization to the correction and delay relaunch.
  - Injection-like text appears before a leak (H6) → add input handling and channel-context limits, and raise its priority.
  - A cause proves single and clean, and its fix passes the tests → bring the bounded release forward.

## Actions

Scores: High / Medium / Low. For effort, Low is cheap. For confidence, it is how strongly current evidence links the action to the cause.

| Priority | Horizon | Action | Linked cause or unknown | Impact | Effort | Risk reduction | Confidence | Reversibility | Owner |
|---|---|---|---|---|---|---|---|---|---|
| 1 | Contain | **Lock out public replies.** The bot answers only in private DMs. In a group channel it replies with an ephemeral "I have sent you a DM". Pause proactive channel reminders. Export the exposed posts for evidence, then remove them with Security and HR. | H1 (delivery path). Also unknown: who saw the posts. | High | Low | Medium (stops the public exposure; does not fix what the bot knows) | High (the primary case is a public post) | High | Platform lead, with IT and HR |
| 2 | Contain | **Suspend retrieval over unapproved sources.** Snapshot the vector index and its inventory as evidence, then disable retrieval over the shared-drive index. Answer policy questions from a short approved list of documents or link to them. Restrict access to logs and traces. | H3 and the reported policy-answer symptom (b). Unknown: index contents. | High | Low | Medium | Medium (symptom (b) is reported, not confirmed) | High (retrieval can be switched back on) | Platform lead, with HR and Security |
| 3 | Investigate | **Run the evidence in the register, in order:** (a) message metadata and assembled prompt for the failing request, plus a replay with synthetic users for a mention, a scheduled reminder and a thread reply; (b) raw checklist API response and retrieved passages with source paths; (c) index inventory and PII scan; (d) Security's scope query for other exposures and copies in logs, exports and the model provider; (e) memory key, credential scope and channel context. | All of H1 to H6. Unknown: how far exposure has spread. | High | Low-Medium | Low (it finds the cause; it does not remove risk) | High (it directly separates the hypotheses) | High (read-only, using synthetic users) | Platform lead (trace), Security (scope), HR systems owner (API) |
| 4 | Correct | **Apply only the fix the evidence supports.** Payload contains sensitive fields (H2): allow-list `task_name`, `status`, `due_date` in the backend before the prompt is built. Records in the index (H3): purge and re-index from an approved list of general-audience documents, banning raw spreadsheets and intake templates. Memory keyed by channel (H4): key it by person and shorten its lifetime. Broad service account (H5): scope calls to the requesting user. Instruction-like text (H6): stop passing channel history to the model and treat retrieved text as untrusted. | H2 to H6, chosen by the trace. Depends on action 3. | High | Medium | High | Medium now, High after action 3 | Medium (config and index changes are easy to undo; an API contract change is harder) | Owner of the failing component |
| 5 | Prevent | **Make the failure a permanent test and add ownership.** Fixed regression cases from the original failure, a cross-user and cross-channel test set, an ingestion gate (approved sources only, PII scan on every crawl), a PII-in-output alert as a last line, a named owner for the data path, and a change review for bot settings, API contracts and sources. | Recurrence of any hypothesis. Change-impact gap. | High | Medium | High | High (tests the failure directly) | High | AI platform team, with Security review of the cases; Head of People Ops for data ownership |

## Options Considered

| Option | Why it could help | Why not first? | Reversible? |
|---|---|---|---|
| Workflow or non-ML change | DM-only delivery, HR sends sensitive reminders manually, and a short approved list of documents for policy questions. Needs no model change and starts today. | It is the containment (actions 1 and 2), not the end state. It adds manual work for HR. | Yes |
| Small architecture correction | A field allow-list, a curated index and a memory key change fix the actual data path at low cost. | The path is not verified. It follows the trace (action 4). | Mostly |
| Larger platform change | Enterprise-wide permission tags on every chunk, synced from the identity system, and a custom red-team suite give stronger guarantees. | Deliberately deferred. Once the corpus is a curated list of general-audience documents, document-level tagging adds cost for little risk reduction. Revisit if the corpus grows to include non-public documents, or if H5 is confirmed. | Hard |
| Tighten the system prompt | Cheap and quick. | Rejected. If sensitive data is in the context, a prompt rule is not a control. | Yes |
| Banner asking users not to enter sensitive IDs | Cheap warning. | The leaks are outbound from the system, so it does not address the reported symptoms. Acceptable only as minor guidance. | Yes |
| Output filter only | A last line of defence. | Kept as a backstop in action 5, not the fix. It would hide the data-access faults. | Yes |
| Full pause of the assistant | Eliminates recurrence risk. | Held in reserve until the scope query shows wider exposure. It also stops the safe, useful parts of onboarding. | Yes |

## Proof of Recovery

All thresholds are **[Proposed]**. The baseline for the original failure is "fails": a private reminder appears in the shared channel.

| Level | Measure or test | Baseline | Pass or guardrail threshold | Owner |
|---|---|---|---|---|
| User or workflow | Hires still receive correct task reminders, in DM only; policy questions get correct answers from approved documents | Channel replies; reminders posted publicly | Reminders correct at least at the pre-incident rate; 0 reminders posted to a shared channel; hires can still reach HR for personal questions | Head of People Ops |
| Retrieval or context | Assembled prompt for task requests contains only `task_name`, `status`, `due_date`; index inventory shows only approved sources; PII scan of the index is clean | Prompt and index contents unknown | 100% of sampled prompts within the allow-list; 0 files from unapproved sources; 0 PII hits on scan | AI platform team |
| Generation or action | The original case plus a cross-user and cross-channel set (for example 30 to 50 synthetic cases: mention, thread reply, schedule, DM, group channel, a user asking about another, policy questions about salary and relocation, instruction-like text in channel) | The original case fails | Original case passes; 0 disclosures of personal or other-employee data across the whole set; ephemeral notice shown in group channels | AI platform team, with Security |
| Operations | Trace coverage of routing, payload, prompt, retrieved sources and memory key; log access limited; copies of exposed data located and handled | Coverage unknown; exposure in logs unknown | Every request traceable end to end; log access limited to named roles; copies removed or documented under Security's direction | Security, AI platform lead |
| Risk | PII-in-output alert and ingestion PII scan active; owner named for the data path; change review in place | No known monitoring or owner | Any confirmed disclosure in production is a stop event (see rollback); alerts reviewed at an agreed cadence | Head of People Ops, Security |

Note: a clean run of the test set is a release gate, not proof of safety. About 50 clean cases bound the failure rate only to roughly 6% at 95% confidence, so the set must include independent adversarial cases from Security.

## Sequencing

Order and rough timing are a proposal. Each phase starts only when the previous gate is met, not on a calendar date.

| Phase | Timing [Proposed] | Gate to move on |
|---|---|---|
| Contain | Hours 0 to 4 | Public replies off; exposed posts exported and removed; index snapshotted and retrieval suspended; log access restricted |
| Investigate | Days 1 to 2 | Trace, payload, index scan and scope query completed; hypotheses updated |
| Correct | Days 2 to 4 | The fix for each confirmed cause applied and reviewed; index re-built from the approved list if H3 confirmed |
| Verify and relaunch | Day 5 or later | Regression and cross-user tests pass; Security and HR sign off; risk owner accepts remaining uncertainty |

## Release and Rollback

- **Release scope:** A bounded release to one cohort, DM-only, with a sample of responses reviewed manually. Widen only after a period with no disclosure. **[Proposed]**
- **Monitoring required:** PII-in-output alert on every reply; weekly review of blocked and redirected requests; trace sampling; ingestion scan on every crawl; test set rerun on every change to bot settings, API contract, sources, prompt, model or memory design. Review cadence: daily during the pilot, weekly afterwards. **[Proposed]**
- **Rollback or shutdown trigger:** Any confirmed disclosure of personal or other-employee data → switch the feature off within minutes and start incident handling. Also: any test-set failure in a pre-release run, or a discovered path the fix did not cover.
- **Fallback experience:** The assistant sends task reminders in DM and answers from a short approved list of documents. Personal or HR-case questions get a plain message and a route to the HR help desk. Static onboarding links as the backstop. **[A]**
- **Remaining risk and authorized risk owner:** Remaining risk: paths not covered by the test set; unknown extent of past exposure; hypotheses still unverified. The risk owner who may accept it is the Head of People Operations, with Security's sign-off. This team cannot accept it alone, and Legal decides on notification.
