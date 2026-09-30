# Diagnostic Register: HR-Onboarding Assistant (Slack/Teams bot)

Legend: **[S]** synthetic exercise detail, **[A]** assumption, **[Requested]** evidence we would ask for but do not have. No cause below is verified.

## Incident Frame

- **Observed symptom:** The assistant posted a private task reminder about one hire's visa document in the shared `#onboarding-cohort` channel, where every member could read it. **[S]**
- **Reproducible example:** "Hi Alex, your visa document is still missing. Please upload it by Friday." posted in the channel by the bot. **[S]** Whether it reproduces on demand, and under which trigger (a tag, a proactive reminder, a reply in a thread), is **not yet confirmed**.
- **Reported but unconfirmed symptoms [S]:** (a) the bot repeats a hire's own home address, phone number or tax status when asked about their status; (b) general policy answers quote real employee names, past compensation packages or manager notes.
- **Affected users or workflow:** Alex (immigration paperwork disclosed) and every member of the channel. Symptoms (a) and (b) suggest other hires and other employees may be affected. Unknown how many.
- **Impact and severity:** High. Immigration, tax, contact and compensation data are sensitive. Exposure to a whole cohort is a privacy incident with possible legal and trust consequences. Whether it is reportable is for Security, Legal and HR to decide, not this team.
- **First observed:** Reported in the current cohort's first weeks. Earlier occurrences are unknown. **[S]**
- **Recent relevant changes:** **None known.** This is a gap, not a finding. The register does not assume a change caused the failure.
- **Immediate containment:** (1) The bot answers only in private DMs. In a group channel it replies with an ephemeral "I have sent you a DM". Channel reminders are paused. (2) Preserve an export of the affected posts, then remove them, with Security and HR. (3) Snapshot the vector index, then suspend retrieval over unapproved sources and point users to the approved documents. (4) Restrict access to logs and traces.
- **Facts frozen for this case:**
  1. One public post with a private reminder. **[S]**
  2. Expected: a private message to the hire only.
  3. Affected: Alex and the channel members who could read it.
  4. Timing: reported in the current cohort's first weeks. **[S]**
  5. Recent change: none known.

## Evidence Available

Only the first row exists. Everything else is a request, and none is assumed to exist or invented.

| Evidence | Source | What it shows | Limitation | Observed, inferred, or verified? |
|---|---|---|---|---|
| The public post (text, channel, time) | Workspace export or screenshot | The symptom, and that it was visible to the channel | Says nothing about why the bot posted or what it saw | **Observed** (synthetic) |
| Message metadata: channel type, thread, reply visibility, triggering event (mention, schedule, event) | Workspace API and bot logs | Whether the reply was routed to the channel by design, by trigger, or by a fallback | Depends on log coverage | **Requested** |
| Bot app permissions and reply settings | Workspace admin, bot config | Whether DM-only or ephemeral delivery is enforced | Configuration may differ from behavior | **Requested** |
| Raw checklist API response for the failing request | Orchestrator logs, HRIS API | Which fields the HRIS returns | Response logging may be off or summarised | **Requested** |
| Assembled prompt sent to the model | Orchestrator debug capture, or a replay | Whether address, tax, bank or immigration fields were in the model's context | May not be stored; a replay uses synthetic users | **Requested** |
| Retrieved passages with source file paths | Retriever logs | Whether real records were retrieved, and from which source | Only covers retrieval | **Requested** |
| Index inventory: files crawled, source folders, PII scan | Ingestion config, index export | Whether raw records and templates were indexed | A scan finds patterns, not every record | **Requested** |
| Memory key, contents and lifetime for two users | Session store | Whether one user's data was reused for another | Data may have expired | **Requested** |
| Credential scope and identity mapping of HRIS calls | IAM configuration, trace | Whether the assistant can read more than the asker may | Configuration only; needs a test to confirm behavior | **Requested** |
| Channel context passed to the model | Prompt builder, bot scopes | Whether other members' messages reach the prompt (untrusted text) | May not be logged | **Requested** |
| Scope query: other posts, DMs and answers containing personal data, and who could see them | Security, log and workspace search | Severity and number of affected people | Depends on retention and search | **Requested** |
| Change log for bot, API, index and prompt | Release notes | A trigger, or absence of one | May be incomplete | **Requested** |

## Evidence Priority

1. **Message metadata and the assembled prompt for the failing request.** Together they show how the reply reached the channel and what the model was given.
2. **Raw API response and retrieved passages with source paths.** Show which store the sensitive content came from.
3. **Scope query.** Runs in parallel because it drives containment and notification.
4. **Memory, credentials and channel context.** Tests the lower-ranked hypotheses.
5. **Change log.**

## Competing Hypotheses

Six causes at different points on the path. The ranking is **provisional**. It orders hypotheses by how many of the reported symptoms each explains and by whether it needs an adversary. No recent change is known, so nothing here rests on one. Hallucination is not listed: the failure is that real personal data is disclosed, and nothing reported suggests invented content. If the leaked values later turn out not to match any source, that assessment should be revisited.

| Rank | Hypothesis and component | Why it fits | Evidence against it | Next discriminating test | Confidence |
|---|---|---|---|---|---|
| 1 | **Reply visibility is not enforced (B1, B8).** The bot replies, or sends reminders, in the shared channel instead of privately, by design or by configuration. | The primary case is itself a public post, so the delivery path is the channel. Reminders and mentions in a group channel commonly default to channel replies unless private delivery is set. It also makes symptoms (a) and (b) visible to everyone. | It explains **where** the content appeared, not **why the content was sensitive**, so it cannot be the only cause. It is unclear whether the bot was ever meant to reply privately. | Read the failing post's metadata (channel type, thread, visibility, triggering event). In a test channel with synthetic users, send a mention, a schedule-triggered reminder and a thread reply, and note where each lands. | Medium-High → Medium-High |
| 2 | **The checklist API returns the full profile and the orchestrator passes it to the model (B3, B7).** The task endpoint returns address, tax, bank and immigration fields, and the raw payload is placed in the prompt. | It matches symptom (a) precisely: the bot repeats the hire's own address, phone and tax status. A model quotes what it is given, so fields in context can surface in any answer or reminder. It would also explain why a reminder carries visa detail. | No API response or prompt has been seen. If the assembled prompt contains only task name, status and date, this is wrong. It does not explain symptom (b) unless payloads contain other people's data. | Capture the raw API response and the assembled prompt for the failing request and a status question. Are sensitive fields present before generation? | Medium → Medium |
| 3 | **The vector index contains real employee records (B4, B5).** The crawler read an unfiltered shared HR folder and indexed offer letters, claim templates and tracking sheets. | It explains symptom (b), where general questions return real names and compensation. An uncleaned shared drive is a known way for such files to enter an index. | Nothing reported ties the primary case to it. Symptom (b) could also come from memory (H4) or an API payload (H2), so the source needs confirming. | Scan the index export for PII patterns and source paths. For the policy question in (b), read the retrieved passages and their source files. | Medium → Medium |
| 4 | **Session memory carries one user's data into another's context (B6).** Memory is keyed by channel or thread instead of by person, or lives too long. | In a shared channel several hires share a thread or channel, so a channel-level key would mix them. It would explain a private detail reaching someone else, and stale details reappearing. | No sign of how memory is keyed. It does not explain the primary case, which is a routing failure. If two sessions in a test show isolated memory, it fails. | Inspect the memory key and stored contents. Run two synthetic users in one channel, one after the other, and check whether the second sees the first's data. | Low-Medium → Low-Medium |
| 5 | **No per-user authorization on the data path (B2).** The assistant uses a broad service account or has no reliable Slack-to-employee mapping, so a user can obtain another person's tasks or profile, or the bot answers about whoever is mentioned. | Assistants often call HR systems with one shared credential for convenience. In a shared channel, "what is Alex's status?" could be answered for anyone. | No report of one hire asking about another. The three reported symptoms arise without that request. | Check the credential and its scope. As synthetic user A, ask about synthetic user B and see what is returned. | Low-Medium → Low-Medium |
| 6 | **Instructions hidden in untrusted text steer the bot (B1, B5).** A channel message, or text in an uncleaned document, tells the model to reveal or post data. | The channel is multi-user, so any member can write text the bot may read, and an unfiltered drive may contain instruction-like text. It would be one way to obtain a public post. | The symptoms recur with ordinary inputs and need no adversary, which is better explained by H1 to H3. No instruction-like text has been seen. Its likelihood depends on whether channel history reaches the prompt. | Search the failing prompt and retrieved passages for instruction-like text. Replay the request with channel context removed and see if the behavior changes. | Low → Low |

### Ranking rationale

H1 leads because the primary case is a delivery failure and is observed directly. H2 and H3 come next because each explains a reported symptom with a concrete mechanism and no adversary. H4 to H6 rank lower because they need a specific flaw (memory keying, a broad credential, injected text) for which no evidence exists yet. They stay because the shared channel and the uncleaned drive make each plausible, and because a correct fix for H1 to H3 would not remove them.

The causes are not mutually exclusive. The likeliest picture is several at once: a public channel (H1) carrying a raw payload (H2) and unclean documents (H3). The tests separate them so each gets the right fix.

### Fixes that would not address the stated cause

- **A stricter system prompt** ("never share personal data"). If the data is in the context, a prompt rule is not a control.
- **Asking users not to type sensitive IDs.** The leaks are outbound, not caused by user input.
- **A session time limit alone.** It shortens exposure to H4 but does not fix a wrong memory key, and does nothing for H1 to H3.
- **A new or fine-tuned model.** Nothing points at model behavior.
- **An output filter alone.** Worth having as a last line, but it would hide the data-access faults.

## Next-Best Evidence

- **The next observation or test we would request:** For the failing request, the message metadata, the raw checklist API response and the assembled prompt, with a replay using synthetic users for a mention, a scheduled reminder and a thread reply.
- **Why it best separates the leading hypotheses:**
  - Reply lands in the channel by design or trigger → H1 confirmed as the delivery path.
  - Sensitive fields in the raw payload and the prompt → H2.
  - Sensitive text in retrieved passages, from files that should not be indexed → H3.
  - Data in context that came from another session → H4.
  - Data returned for a user who should not be allowed it → H5.
  - Instruction-like text before the leak → H6.
- **Who can provide or run it:** The platform team pulls the trace and runs the replay. IT/Security provides the workspace metadata and runs the scope query.
- **Decision that depends on it:** Which fixes to apply and which containment to relax. A prompt with only task name, status and date turns attention to the corpus and interface. A prompt with the full profile makes data minimisation the first fix.

## AI Challenge Note

**Sources:** Two AI inputs, both treated as hypotheses until an engineer, system owner, security specialist or primary source verifies them.
- **Part A:** a separate analysis of this system produced with a different LLM and supplied by the team. It provided the scenario and a proposed diagnosis and fix.
- **Part B:** a review of our own work by Claude (this assistant), plus the decisions the team made in response. Claude's review of the earlier version of the case is included where its findings carried over.

The panel prompt from the handout has **not** yet been run on the current map and register. Dispositions are the team's proposed decisions and are not yet confirmed by a system owner or Security.

### Part A: the other LLM's analysis

| LLM suggestion | How it was checked | Accepted, edited, or rejected | Reason |
|---|---|---|---|
| **1. The root causes are API over-fetching, a dirty corpus and missing interface boundaries, stated as settled.** | Compared with the handout, which asks for competing hypotheses and no claim of a verified cause. No logs, payloads or index contents were seen. | **Edited** | Kept as H1 to H3, ranked, each with a test. They stay hypotheses until the trace and index scan are run. |
| **2. Prompt injection and hallucination are ruled out.** | Hallucination: the failure is disclosure of real personal data, and nothing reported suggests invented content, so it is not listed. Injection: not excluded by any observation. The channel is multi-user and the corpus unfiltered. | **Edited** | Hallucination dropped. Injection kept as H6 at low confidence, with the argument for why it ranks last and a test that could still reveal it. |
| **3. Channel lockout: DM-only, with an ephemeral notice in group channels.** | Matches the primary case and the interface boundary (B8). Feasibility in the workspace is to do. **[verify: IT/Platform]** | **Accepted** | Cheapest, most reversible way to stop the public exposure. It is the first containment action. |
| **4. Minimise the checklist payload to task name, status and due date.** | Fits H2 if the payload carries the profile. Not yet shown. | **Accepted, conditional** | Becomes the first correction if the trace shows sensitive fields in the prompt. Not applied until then. |
| **5. Wipe the vector database, then re-index only a whitelist of general documents.** | Correct for H3 if confirmed. Wiping first would destroy the evidence for H3 and for the scope query. | **Edited** | Snapshot the index and record its inventory first. Then suspend, purge and re-index from an approved list. |
| **6. Defer enterprise-wide vector permissions and custom red-teaming as overkill.** | Reasonable for document-level tagging when documents are curated to general audience. It does not answer H5, where the question is whether the assistant can read more than the asker may. | **Edited** | Enterprise-wide permission sync stays deferred, with an explicit revisit condition. A basic per-user authorization check on the data path (H5) is tested, not assumed away. |
| **7. Set the memory time limit to 30 minutes.** | Shortens exposure but does not fix a memory keyed by channel. The key and lifetime are unknown. | **Edited** | Test the memory key first. A time limit is added only as a secondary control. |
| **8. Add a banner telling users not to enter sensitive IDs.** | The leaks are outbound from the system, not caused by input. | **Rejected as a control** | Moves responsibility to the user and does not stop any of the reported symptoms. Acceptable only as minor guidance. |
| **9. A fixed five-day timeline ending with relaunch.** | A calendar date is not evidence. | **Edited** | Relaunch depends on passing the regression tests and on Security and HR sign-off, not on a day count. |

**Most important suggestion:** Number 5. It reveals a conflict between fast containment and preserving evidence, and the plan resolves it with a snapshot first.

### Part B: Claude's review and the team's decisions

| Input | How it was checked | Accepted, edited, or rejected | Reason and effect on the deliverables |
|---|---|---|---|
| **Claude: "The ranking is circular."** In the earlier version, the leading hypothesis fit only an invented recent change. | Re-read the earlier register: the lead came from fit with a synthetic change, not from evidence. | **Accepted** | The current case states "recent change: none known", so no ranking rests on an invented trigger. Ranking is by how many symptoms each cause explains, and labelled provisional. |
| **Claude: State the assumptions and the evidence gap for each fix.** A fix chosen before the trace may target the wrong layer. | Compared the recovery plan's actions with the hypotheses. | **Accepted** | Action 4 (Correct) is chosen by the trace and lists one fix per hypothesis. The register lists fixes that would not address the cause. |
| **Claude: Exposure may extend beyond the chat** (logs, exports, the model provider). | Compared the containment and scope query with the map: logs are marked "may contain PII". | **Accepted** | The scope query and evidence table cover logs, exports and the model provider. Log access is restricted in containment. |
| **Claude: A clean test run is a release gate, not proof of safety,** and a set written by the same team tests only imagined paths. | Arithmetic check (about 50 clean cases bound the failure rate to roughly 6% at 95% confidence); this is a calculation, not a cited source. | **Accepted** | The recovery plan words the test set as a gate and asks for independent adversarial cases from Security. |
| **Claude: Snapshot evidence before wiping the index** (from the conflict between the other LLM's "wipe in hours 0 to 4" and preserving evidence). | Compared the other LLM's step 5 with the need to test the corpus hypothesis. | **Accepted** | Contain and Correct steps snapshot the index and record its inventory before purging. |
| **Claude proposed keeping hallucination as a hypothesis** (as a check that the leaked values match a real record). | The team pointed out the failure is a privacy or permission failure and that there is no supporting evidence for invented content. Nothing reported suggests fabricated values. | **Rejected (team decision)** | Hallucination is not listed. The register states why, and what would bring it back (leaked values matching no source). |
| **Claude proposed keeping injection, session memory and missing permissions as low-ranked hypotheses.** The team asked that each be defensible and well argued. | Each was given a reason it fits, evidence against it, and a discriminating test, and a short account of why it ranks low. | **Accepted (team decision)** | H4 (memory), H5 (permissions) and H6 (injection) stay, each with an argument and a test. |
| **Team decision: freeze one primary case.** The scenario has three symptoms and the handout asks for one reproducible symptom. | The team chose the public broadcast as the frozen case. | **Team decision** | The other two symptoms stay as reported but unconfirmed evidence patterns that separate the hypotheses. |

**Still to do before presenting:**
- Run the handout's skeptical-panel prompt on the current map and register with an approved assistant, and record the result here.
- Have an engineer or system owner check the technical assumptions (reply visibility, payload contents, index sources, memory key).
- Check the unsourced general claims against primary sources: Slack and Teams documentation, and OWASP guidance on excessive agency and sensitive-data disclosure.
- Confirm whether the assistant can send ephemeral messages in Teams as well as Slack.


## Current Diagnosis

- **Verified cause, if any:** None. The only observed fact is that the reminder was posted publicly.
- **Best-supported hypothesis if not verified:** H1 for the delivery path, in combination with H2 and/or H3 for the sensitive content. The order comes from how many symptoms each explains, not from logs.
- **Important uncertainty:** What the model was given, which store the sensitive content came from, and how many others were exposed.
- **Evidence that would change the diagnosis:** A prompt containing only task name, status and date weakens H2 and shifts attention to H3, H4 or H6. Retrieved passages from files that should not be indexed make H3 the lead content cause. Data reaching one user from another session raises H4. Data returned for a user who should not receive it raises H5.
