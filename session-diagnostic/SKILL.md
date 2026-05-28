---
name: session-diagnostic
description: |
  Captures a complete telemetry log of a Cowork session — request interpretation,
  routing decisions, task graph, tool-call ledger, subagent dispatches, execution
  metrics, detected issues, cross-output consistency, latency/token outliers, and
  known unknowns — into a single markdown deliverable at output/session-diagnostic.md.
  Designed to be model-agnostic, topic-agnostic, and input/output-agnostic so the
  same skill applies to any future session.
  Use when user asks to "log diagnostics", "diagnostic log", "diagnostic report",
  "track this session", "trace the session", "session telemetry", "capture execution
  stats", "execution trace", "audit this run", "what did the agents do", "what tools
  did you use", "what happened in this session", "build a running log", "tell me
  everything you do", or pairs another request with "and also track everything" /
  "and create a diagnostic file" / "and log all the activity".
  Do NOT use for: TaskCreate-style live progress tracking (the task system handles
  the spinner UX), conversation transcript export as HTML (use debug-trajectory
  instead), or memory snapshots (use the memory store directly).
---

# Session Diagnostic

A reusable runbook for producing a complete execution telemetry log of any Cowork session, regardless of model, topic, inputs, or outputs.

## Output

A single markdown file at `output/session-diagnostic.md`. Written in **two or more passes**:

1. **Frame** — before any other work, write sections 1–5 with placeholders for the rest.
2. **Live appends** — after each significant phase (subagent return, file delivery, error, retry). **Degenerate when no subagents are dispatched** — see "Inline-only sessions" below; in that case the workflow collapses to frame + close-out and you must say so honestly rather than fabricate appends.
3. **Close-out** — final pass after delivery gate confirms the deliverables (sections 7–15).

If the main task fails midway, the frame still ships — never block diagnostics on success of the primary work.

### Inline-only sessions (zero subagents)

A legitimate session shape: user forbids subagents, work is trivial enough to run inline, or the chosen skill doesn't dispatch. When this happens:

- **Section 5** is still written, but as an explicit declaration ("no subagents dispatched") with rationale (user constraint, inline-trivial, etc.) and a list of which downstream sections will be N/A as a consequence.
- **Section 7** says so explicitly and notes the harness-exposed metrics that would have been available with subagents (`total_tokens`, `tool_uses`, `duration_ms`, `agentId`).
- **Section 10** replaces "subagent median" math with a main-agent step count and a one-line note that medians aren't computable.
- **Section 11** marks token/tool-use rows as "not exposed for inline main-agent work" rather than fabricating.
- The 3-pass workflow becomes 2-pass (frame + close-out). State this in the close-out so the cadence is honest.

## Required sections (in this order)

1. **Session Header** — start time (from `Current local time` in system context), user identity, model identifier, workspace cwd, topic / objective, primary deliverables planned.
2. **Request Interpretation & Routing Decisions** — user's stated goal, agent's parse, routing-decisions table (`Concern | Decision | Rationale`), meta-cognitive checkpoint (VALIDATE → DECOMPOSE → SOLVE → VERIFY → SYNTHESIZE → REFLECT — from the implicit deep-reasoning skill), confidence score for entering SOLVE.
3. **Task Graph** — table of TaskCreate entries: `ID | Subject | Owner (main / subagent name) | Starting Status`.
4. **Environment Snapshot** — platform, shell, knowledge cutoff, internet availability, pre-installed packages relevant to the work, output-sync rules, git status.
5. **Subagent Dispatch Plan** — for each planned subagent: skill name (if any), environment facts embedded in the prompt, anti-fabrication directives, expected output path. State the dispatch pattern (parallel vs serial), the dispatch mode (**foreground blocking** vs **`run_in_background: true`** — background is mandatory when the main agent has independent work to do during the dispatch window, e.g. an inline docx/pptx build, OR when fan-out exceeds the harness concurrency pool so completion notifications stream in over multiple turns), and rationale. **If no subagents will be dispatched, say so explicitly and name the cause** (user constraint, inline-trivial, single-skill no-dispatch path) — then list which downstream sections become N/A.
6. **Tool Call Ledger (live)** — running table: `# | Phase | Actor (main / subagent name) | Tool | Purpose | Result`. Use **Phase** labels (Setup / Frame / Build / Verify / Write-Action / Close — or whatever maps to the session) — NOT wall-clock times. The harness does not expose per-call timestamps; `T+0s`-style placeholders are fabrication, so the older `Time` column was dropped in favor of Phase. **Result** is a one-word status (OK / error / pending). Append rows as calls happen. The ledger MAY collapse a multi-tool turn into one phase row for readability (e.g. "TaskUpdate ×3 + CreateDraftMessage" on a single line); when it does, Section 12's per-tool count must remain un-collapsed and reflect every individual invocation. Cannot capture subagent-internal tool calls — only the aggregate count returned by the harness. **Ledger hygiene:** log failed probes AND their recovery as separate rows so the discovery process is visible. Collapsing them into one "retry" row hides signal.
7. **Subagent Execution Results** — for each returned subagent: agent ID, `total_tokens`, `tool_uses`, `duration_ms`, status. Verbatim quote of the subagent's own summary (abridge with `[...]` only if very long). **Background dispatch:** results arrive as `<task-notification>` blocks containing `<result>` and `<usage>` — capture those verbatim as they stream in. **Do NOT Read the `.output` file path** referenced in the notification: it is a symlink to the full sub-agent JSONL transcript and will overflow main-agent context if read directly. The notification's `<result>` and `<usage>` fields are the primary telemetry. **Optional richer capture:** the JSONL transcript at `.output` contains the per-assistant-turn input/output/cache token split that the harness does NOT include in `<usage>`. If you want that split, run the streaming parser described in **Token-Split Capture** below, with bounded shell-side processing only — never via the Read tool, and never as a `cat`/`tail` whose output streams into your context. For high-fan-out runs (≥~12 subagents) prefer a compact table per-subagent (`# | name | tokens | duration | excerpt`) rather than one full block per agent, or the section blows past sensible length. **Inline-only sessions:** state explicitly that no subagents were dispatched and note that `total_tokens` / `tool_uses` / `duration_ms` / `agentId` are not exposed by the harness for inline main-agent work.

    ### Token-Split Capture (optional sub-procedure)

    The harness `<usage>` block returns only `total_tokens` for each subagent. The `input_tokens` / `output_tokens` / `cache_creation_input_tokens` / `cache_read_input_tokens` split is recorded **inside the JSONL transcript** on each assistant-turn message. You can recover it without ingesting the raw transcript by running a streaming parser that emits only aggregates:

    ```python
    # scripts work/extract-usage-split.py <agent_output_path>
    import json, sys, pathlib
    totals = {"input_tokens": 0, "output_tokens": 0,
              "cache_creation_input_tokens": 0, "cache_read_input_tokens": 0,
              "assistant_turns": 0}
    for line in pathlib.Path(sys.argv[1]).open():
        try:
            obj = json.loads(line)
        except Exception:
            continue
        u = (obj.get("message") or {}).get("usage") or {}
        if not u: continue
        totals["assistant_turns"] += 1
        for k in ("input_tokens", "output_tokens",
                  "cache_creation_input_tokens", "cache_read_input_tokens"):
            totals[k] += int(u.get(k) or 0)
    print(json.dumps(totals))
    ```

    Critical lifecycle constraint, confirmed by direct probe on 2026-05-28: **transcript files in `/tmp/claude-*/.../tasks/<agentId>.output` are cleaned during or after the session.** Of 50 subagent transcripts dispatched earlier in a session, all were already gone by close-out time — only the most recent (an in-flight bash task) survived. Implication for this skill:

    - **Extract at notification time, not at close-out.** Each time a `<task-notification>` arrives, immediately run the parser against the named `output-file` if you want the split for that subagent. Append the four-field split to Section 7 next to that subagent's row.
    - **If close-out runs before extraction, the split is lost** — report it as "not captured" in Section 7 and explain in Section 8 that the transcript was already rotated. Do not pretend a missed window is a harness limitation.
    - The parser emits only aggregates (one short JSON line per subagent), so its stdout is safe for the main agent's context regardless of transcript size.
    - Never substitute `Read`, `cat`, `tail`, `head`, or `grep` against the `.output` file when the goal is capturing token splits — those stream raw content. Use the streaming parser pattern above, which discards every byte except the summed counters.
8. **Issues, Warnings & Errors** — **ALWAYS include this section.** Capture:
   - Retry loops or auto-corrections reported by subagents (e.g. visual-QA second pass, regenerated chart, re-validated formulas).
   - Spec deviations (subagent added unrequested fields, omitted a requested element, used different formatting).
   - Tool failures and recovery attempts.
   - **Skill-doc vs runtime drift** — e.g., a documented script path that doesn't exist at the documented location (`scripts/office/...` vs actual `scripts/scripts/office/...`), a documented binary name that's symlinked elsewhere, a documented env var that isn't set. Capture both the documented path/name and the actual one found. **Confirmed instance (record as a known drift in any session that uses the docx/pptx/xlsx skills):** the office helpers documented at `scripts/office/validate.py` and `scripts/office/soffice.py` actually live at `scripts/scripts/office/validate.py` and `scripts/scripts/office/soffice.py` in the Cowork sandbox. The first call following the SKILL.md path fails with `[Errno 2] No such file or directory`. If you hit this, recover by running the script at the doubled path AND log both paths in Section 8 — even if recovery is silent to the user, downstream sessions benefit from the record.
   - Permission denials or sandbox rejections.
   - Resource warnings (large file sizes, near-limit token usage, near-limit context).
   - Anti-fabrication near-misses (subagent had to invent data because retrieval failed).
   - If genuinely none observed: write `"No issues, warnings, or errors observed during execution."` — explicit "none" is itself a useful signal.
9. **Cross-Output Consistency Check** — when 2+ deliverables share a data domain (numbers, names, dates, headings), verify they are internally consistent. If subagents generated illustrative data independently, **explicitly flag** that figures across files will likely NOT agree and call this out as a known limitation of independent parallel dispatch. **Temporal-build drift is NOT inconsistency** — if deliverable A's status table reports "B: In progress" because A was built before B finished, and B's own copy reports "B: Complete," that is sequential build order showing through, not a defect. Note which build moment each deliverable's data reflects rather than flagging the difference as a failure. Reference observation from a Cowork session on 2026-05-28: a docx built first reported the PDF as "In progress" while the PDF's own table reported "Complete" — both correct as of their respective build moments. Single deliverable → write `"Single deliverable — N/A."`
10. **Latency & Token Outliers** — compute median `total_tokens` and median `duration_ms` across subagents. Flag any subagent at ≥1.5× either median. State the observed multiplier and a one-sentence hypothesis (e.g. *"PPT took 2.1× the median duration — likely the visual-QA render+inspect+fix loop"*). **For bulk dispatch (N ≥ ~12 subagents from a single message): also produce a Wave Analysis sub-table** — see Wave Analysis below. **Inline-only sessions:** medians aren't computable; replace with a main-agent step count and a one-line note flagging any single step that visibly stalled or required rework.

    ### Wave Analysis (bulk-dispatch concurrency probe)

    When ≥~12 subagents are dispatched in a single message and the underlying work is approximately uniform (similar token output, no tool use), the harness's effective concurrency pool can be estimated from the `duration_ms` distribution. This is the closest thing to a direct probe of the concurrency cap, which the harness does not otherwise expose.

    Method:
    1. Sort subagents by `duration_ms` ascending.
    2. Treat the **first plateau** of similar low durations as Wave 1 (concurrent slots that started immediately). Note its size W and its max duration t1.
    3. Treat the next plateau as Wave 2 (max duration t2), etc.
    4. **Estimated concurrency pool size = W** (size of Wave 1).
    5. **Estimated intrinsic task time = t1** (Wave 1's longest, since it had no queue contention).
    6. **Estimated effective parallelism cap = N / number_of_waves** (sanity check against W).
    7. **Optimal batch size for minimum per-agent wall-clock = W.** Dispatch in batches of W rather than one mega-batch.

    Document the analysis as a small table:

    | Dispatch positions | Duration range | Implied wave | Implied start time |
    |--------------------|---------------|--------------|-------------------|
    | ... | t_min – t_max | Wave 1 | T0 |
    | ... | ... | Wave 2 | ~t1 |
    | ... | ... | ... | ... |

    Always state the inference as an **estimate**, not a measurement — the cap may be tenant- or load-dependent and may shift between sessions. Reference observation from a Cowork session on 2026-05-28: 50 uniform paragraph-generation subagents dispatched in a single message produced four waves of ~12, with intrinsic task time ~10 s and last-wave wall-clock ~39.5 s. Concurrency cap was estimated at ~12 slots; optimal batch size ~12.
11. **Aggregate Session Stats** — table summing subagent tokens and tool uses, and reporting **parallel wall-clock duration as the max (not sum) of concurrent subagents**. Compute parallel-vs-serial savings. **If Section 10's Wave Analysis was triggered**, add two extra rows: *Estimated concurrency-pool size* (W from wave 1) and *Recommended batch size for fastest per-agent wall-clock* (= W). Briefly note the trade-off: bigger batches reduce dispatch overhead but increase per-agent wall-clock past wave 1. **Inline-only sessions:** mark token/tool-use sum rows as "not exposed for inline main-agent work" rather than fabricating; report only main-agent tool call count.
12. **Main-Agent Tool Usage** — un-collapsed count of each tool called by the main agent during this session. If Section 6 collapsed multi-tool turns into single phase rows for readability, this count must still reflect every individual tool invocation (a "TaskUpdate ×3" Section 6 row contributes 3 to TaskUpdate here, not 1).
13. **Reasoning Process Recap** — reconstructed numbered narrative of how the main agent moved from request → topic/scope → dispatch → verification. Note: reconstructed from outputs, not internal chain-of-thought.
14. **Known Unknowns** — explicit list of telemetry the Cowork harness does NOT expose directly. Always include at minimum:
    - **per-subagent input/output/cache token split** — not in the `<usage>` block, but *recoverable* per Token-Split Capture (Section 7) **if** the parser runs before the transcript is rotated. If you didn't run the parser in time, list this row as "not captured this session" rather than as fully unknown.
    - **main-agent input/output/cache token split** — not exposed by the harness for inline main-agent work, and no `.output` transcript is written for the main agent's own turns. Genuinely unknown.
    - subagent internal chain-of-thought
    - subagent system prompt / loaded skills / memory context
    - exact wall-clock timestamps per tool call
    - sandbox CPU / RAM / network deltas
    - retry attempts internal to a subagent (only externally-reported retries are visible)
    - **exact concurrency-pool size** — not exposed, but *can be estimated* via Wave Analysis (Section 10) when bulk dispatch is performed with uniform work. Estimate only; the cap may shift between sessions, tenants, or load conditions.
    - **scheduling policy of the dispatch queue** — FIFO vs weighted vs other; only the resulting per-subagent `duration_ms` is observable.
    - **transcript retention window** — `.output` files in `/tmp/claude-*/.../tasks/` are cleaned at some point during/after the session. The exact trigger and TTL are not documented; the only reliable strategy is to extract any per-transcript data at notification time.
15. **Deliverables Confirmed** — two parts (the second is mandatory whenever the session produced anything that did not land on disk):
    - **On-disk deliverables:** final `Glob output/**/*` listing. Confirms the file delivery gate passed.
    - **Side-effect deliverables (not on disk):** explicit list of any non-file actions that were themselves the deliverable — emails sent, drafts saved, Teams messages posted, calendar events created/updated/cancelled, scheduled tasks set up, external API state changes. Cite the confirming tool result verbatim for each (e.g. `"Email sent successfully."`, the draft id returned by `CreateDraftMessage`, the event id returned by `CreateEvent`) so a future reader can verify the action happened. A session whose only deliverable was a non-file action still passes the delivery gate if the side-effect confirmations are present.

## Workflow

### Step 0: Chip-alongside invocation (common case)

When the user pastes a `/session-diagnostic` chip in the same turn as a real request ("create X, Y, Z [/session-diagnostic]"), the diagnostic runs as **bookkeeping** that wraps the real work. The user wants the deliverables; the log is captured alongside, not instead of. Therefore:

- Do NOT pause to ask clarifying questions about the diagnostic itself — the chip is the invocation.
- Do NOT run extensive environment probes before producing any user-visible progress.
- Do NOT treat the diagnostic file as the priority deliverable in the user-facing summary; mention it last, alongside the actual deliverables.

Pattern: `TaskCreate` (real-work tasks + one diagnostic task) → write frame quickly → do the real work → close out the log → final summary.

### Step 1: Open the log immediately

Before doing any other work, write the frame (sections 1–5) to `output/session-diagnostic.md`. Set `_to be filled after dispatch_` placeholders for sections 6–15.

### Step 2: Track tool calls as they happen

Each main-agent tool call → append a row to section 6. Each subagent dispatch → log the dispatch itself.

### Step 3: Append subagent results on return

For each subagent return, append to section 7 with the verbatim summary (or `[...]`-abridged version if extremely long) plus harness-reported metrics.

### Step 4: Always populate sections 8, 9, 10

Empty sections are a smell — explicit `"none observed"` / `"N/A"` is informative. For section 10, compute medians and flag outliers with a hypothesis.

### Step 5: Delivery gate + close-out

Run `Glob output/**/*` at **two** moments, not one:

1. **Mid-build probe** — at any phase boundary where the *next* phase is an external write action (sending email, posting to Teams, creating calendar events, calling an external API that mutates remote state). Verify the prior build phase actually landed on disk before triggering an irreversible action. Catching a missing file before sending an email that references it is much cheaper than recovering after.
2. **Final delivery gate** — before announcing completion to the user. Confirms `session-diagnostic.md` and all other on-disk deliverables exist.

Then:

3. Append section 15 (both the on-disk listing AND any side-effect deliverable confirmations — see section spec).
4. Mark the diagnostic task complete via TaskUpdate.

### Step 6: Tell the user where to find it

Name the diagnostic file alongside the primary deliverables in the user-facing summary. Do not paste the diagnostic content into chat — the file is the deliverable.

## Anti-patterns

- **Claiming "continuous" updates when actually batched.** If writes happen in 3 passes, say so in the log honestly.
- **Pretending live appends happened in an inline-only session.** Zero subagents → no append triggers → the cadence is frame + close-out, not three passes. Say so in the close-out.
- **Echoing user-input content as if it were telemetry.** The log captures what the agent did, not user content beyond the initial intent parse.
- **Fabricating metrics not exposed by the harness** (e.g. inventing prompt/completion split, estimating temperature, inventing total_tokens for main-agent work). Use the "Known Unknowns" section instead.
- **Skipping section 8 because "nothing went wrong."** The section being present with `"none observed"` is itself a useful diagnostic signal.
- **Forgetting cross-output consistency** when multiple files share a data domain.
- **Reporting wall-clock as the sum of parallel subagent durations.** For concurrent dispatch, use `max`, not `sum`.
- **Collapsing failed probes into "retry" rows.** Log the failed call AND the recovery call as separate ledger entries — discovery is signal, not noise.
- **Reading the `.output` file path of a backgrounded subagent.** It is a symlink to the full sub-agent JSONL transcript and will blow up main-agent context. Capture only the `<result>` and `<usage>` fields from each `<task-notification>` block — OR run the Token-Split Capture parser (Section 7), which emits only aggregated counters.
- **Waiting until close-out to extract token splits.** Transcript `.output` files are cleaned during the session; by close-out they are typically gone. If you want input/output split per subagent, run the parser at notification time, not at the end.
- **Treating a uniform-N bulk dispatch as a flat parallel ensemble.** When ≥~12 subagents are dispatched from one message, durations almost always rise with dispatch order — that is the concurrency cap showing through. Run Wave Analysis (Section 10) and report the estimated pool size and recommended batch size; don't silently average over the wave structure.
- **Logging dispatch-order vs duration-order without distinguishing them.** Section 7's per-subagent rows should be in dispatch order so Wave Analysis works. If you sort the table by duration for readability, keep a dispatch-position column or you lose the throttling signal.
- **Treating the final `Glob output/**/*` as the only delivery gate when the actual deliverable was a non-file action.** Sent emails, drafts, Teams posts, calendar events, and scheduled tasks never appear in `output/`; Section 15 must record their confirming tool results separately. A "PASSED" delivery gate that lists only on-disk files for a session whose user-facing deliverable was an email is a false signal.
- **Flagging temporal-build drift as a cross-output consistency failure.** If deliverable A was built before deliverable B finished, A's status table will (correctly) report B as in progress while B's own copy reports it as complete. That is sequential build order, not divergence — call it out as intentional in Section 9, do not record it as an inconsistency.
- **Fabricating wall-clock times in Section 6.** The harness does not expose per-call timestamps. `T+0s` / `T+15s` placeholders are invented telemetry — use a **Phase** column instead and record `Time` only when the harness actually returned one (e.g. a subagent's `duration_ms`).

## High-fan-out dispatch — preferred pattern

When the session legitimately needs ≥~12 subagents (stress test, bulk independent research, multi-format generation):

1. **Use `run_in_background: true` on every subagent.** Foreground blocks the main agent until all return; background lets the main agent continue independent work and receive completions as `<task-notification>` blocks.
2. **Dispatch all from a single message** so they enter the queue together. This gives Wave Analysis a clean signal.
3. **Keep each subagent prompt small and uniform** (single short return value, no tool use, no file writes). Non-uniform work makes Wave Analysis noisy and obscures the concurrency cap.
4. **Capture `<task-notification>` results directly into Section 7.** Do not Read the symlinked `.output` file. Do not re-Agent on the same agent ID just to fetch the result.
4a. **Optional, same turn as the notification:** run the Token-Split Capture parser (Section 7) against the notification's `output-file` path to extract the input/output/cache token split. Must happen in the same turn the notification arrives — the transcript may be rotated before close-out.
5. **After all return, run Wave Analysis** (Section 10) and record the estimated concurrency cap and recommended batch size in Section 11.
6. **For production workflows** (not stress tests): split the work into batches of ~W (the estimated cap) and dispatch waves serially. This minimizes per-agent wall-clock; the trade-off is more main-agent dispatch overhead.

## Multi-session reuse

This skill is intentionally model-agnostic, topic-agnostic, and IO-agnostic:

- **Different models** — captures whatever `total_tokens` / `tool_uses` / `duration_ms` the harness exposes, regardless of underlying model identity.
- **Different prompts** — the section list does not assume a specific task type. Sections 5, 7, 9 adapt to whatever the dispatch plan looks like.
- **Different inputs** — environment snapshot adapts to whatever the runtime exposes.
- **Different outputs** — Cross-Output Consistency Check (section 9) only fires when there are 2+ deliverables sharing a data domain.

## When NOT to use

- **TaskCreate-style live progress tracking** — the platform task system handles the spinner UX; the diagnostic file is a deliverable, not a live indicator.
- **Conversation transcript export** — use the `debug-trajectory` skill instead; that produces an HTML render of the actual user/assistant turns.
- **Memory snapshots** — use the memory store directly; this skill does not introspect saved memories.
- **A single trivial request** (one-shot lookup, simple question) — overkill for anything that fits in one tool call.

## Cowork-specific environment notes

- **Output path:** `output/session-diagnostic.md` is the only diagnostic deliverable. User-visible. Synced to OneDrive.
- **Time source:** `Current local time` from system context. Do not run `date -u`; never re-derive timezones.
- **Subagent metrics:** `total_tokens`, `tool_uses`, `duration_ms`, `agentId` are returned in the tool result. Capture them verbatim.
- **Delivery gate is mandatory:** never announce the file as ready before `Glob output/**/*` confirms it exists.
