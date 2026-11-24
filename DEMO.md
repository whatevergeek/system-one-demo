# Demo-day runbook (Nov 24)

A checklist, not documentation. Setup lives in the [README](README.md) (getting started) and [MORE_DEMOS.md](MORE_DEMOS.md) (chat front and five-questions).

## Before the talk

- [ ] `docker compose up -d` in this repo, then open http://localhost:5678
- [ ] ollaya running on port 11435 (System 1)
- [ ] Ollama running on port 11434 (System 2; model: `gemma4:12b-mlx`, set in the `S2 Config` workflow)
- [ ] Required workflows imported. Publish only the intended terminal five-questions variant; both terminal workflows use the same webhook path (`/five-questions`). Keep this copy untouched until the talk.
- [ ] Eval setup done once (REPO_DETAILS.md, recommended eval pair): `tickets eval 10` data table imported and selected in **When fetching a dataset row** of both eval workflows, `comparison runs` table created
- [ ] Fallback ready: `fallback/capture-canned.sh` re-run, backup recording captured (see the end of this file)

### Warm up both tiers (do this right before going on stage)

Both models pay a cold-start tax. Fire one throwaway call at each, then keep them warm.

1. **System 1 warmup** - run the Tuesday ticket command (section 3.2) once.
2. **System 2 warmup** - run the enterprise command (section 3.2) once and wait for it to finish. A cold gemma can take two minutes or more (the node timeout is 300 s); if the reply says `human_review`, run it again. Do not go on stage until it returns a System 2 answer.
3. gemma is dropped from memory after a few idle minutes. If you wait longer than that, repeat step 2.

## Live run order

### 1. S2 triage (canvas)

Open `system2-ticket-triage`, hit **Execute workflow**, and watch System 2 reason through the four tickets (Ollama, model from the `S2 Config` workflow, default `gemma4:12b-mlx`). Slow and deliberate: this is what deep reasoning costs, and it sets up the contrast. The first run can take a few minutes while Ollama loads the model; later runs are much faster. (An OpenAI-compatible variant, `system2-openai-ticket-triage`, is also in the repo.)

### 2. S1 triage (canvas)

Run `system1-ticket-triage` from the canvas on the same four tickets. Instant answers: System 1 gets 4/4 right in well under a second per ticket. The speed and token contrast with step 1 is the point.

### 3.1 Five questions - chat

Use `workflows/five-questions-chat-demo.json` to run the five-question pipeline from an n8n chat window instead of the terminal. On a fresh instance, edit the chat workflow's **Endpoint Config** node for your OpenAI-compatible endpoint (base URL, model and API key if required); it does not use `s2-config.json`. Publish the chat workflow, then open http://localhost:5678/webhook/five-questions-chat/chat and paste a ticket. The reply shows the five stages, per-call latency and verdict. This workflow was exercised on the Mac on Nov 23, including the human-gate and System 2 escalation paths.

The chat accepts only ticket text, so CLASSIFY scoring via `expected_team` is off. The small model can vary between runs, so use the exact examples in section 3.2 and rehearse them before the talk.

**Sample prompts** - paste these exactly; the small model can vary between runs.

- **Path 1 - Gate to human:** `hmm`
- **Path 2 - System 1 alone:** `Every Tuesday around 2pm our export job fails, but only for customers on the German login page. Rest of the week it works fine.`
- **Path 3 - Escalation to System 2:** `We are a 500-seat enterprise: your export fails every Tuesday, we were charged twice this month, and we need volume discount pricing. Can someone own all of this?`

Note: the "production server down" outage ticket also lands on human, so don't use it as the path 2 example.


### 3.2 Five questions - webhook (terminal)

The existing `workflows/five-questions-openai-demo.json` workflow is kept as a reference and runs from PowerShell. Its three paths were confirmed on the Mac on Nov 20, before RANK switched from noul to score on the morning of Nov 20 - re-run to re-confirm with the current workflow. Click **Listen for test event** before each `/webhook-test/` request (use `/webhook/` instead when the workflow is active). The Ollama and OpenAI-compatible terminal workflows both use `/five-questions`; publish only the intended variant, not both at once.

```powershell
# Path 1 - Human via GATE
Invoke-RestMethod -Method Post -Uri http://localhost:5678/webhook-test/five-questions -ContentType 'application/json' -Body '{"ticket":"hmm"}' | ConvertTo-Json -Depth 6

# Path 2 - System 1 alone (Tuesday ticket)
Invoke-RestMethod -Uri http://localhost:5678/webhook-test/five-questions -Method Post -ContentType 'application/json' -Body '{"ticket":"Every Tuesday around 2pm our export job fails, but only for customers on the German login page. Rest of the week it works fine."}' | ConvertTo-Json -Depth 6

# Path 3 - System 1 escalates to System 2 (enterprise ticket)
Invoke-RestMethod -Uri http://localhost:5678/webhook-test/five-questions -Method Post -ContentType 'application/json' -Body '{"ticket":"We are a 500-seat enterprise: your export fails every Tuesday, we were charged twice this month, and we need volume discount pricing. Can someone own all of this?"}' | ConvertTo-Json -Depth 6
```

Expected, as recorded on the Mac:

- **Path 1:** `human_review` - `decided_by` = System 1 gate -> human, four Laya calls, CHOOSE skipped. A fine side demo of "knowing when not to automate".
- **Path 2:** Verdict (System 1 decided) - GATE automate, CHOOSE `hand_to_team`, `final_route` = `queue:infra`, about 150 ms in total (classify confidence about 0.78).
- **Path 3:** CLASSIFY confidence about 0.28, so CHOOSE escalates (`policy_overrides` shows "classify confidence < 0.6"); `decided_by` = System 2 (Ollama), after System 1 escalated. Warm, roughly 15 to 110 s. Narrate the wait: "this is what deep reasoning costs. System 1 knew it was unsure in about 300 ms."

RANK now uses the score type (low / medium / high business impact, mapped to P1 / P2 / P3); the routing rules and the other four stages are unchanged. Say: "five decisions from all three kinds of System 1 call (choice, noul and score), and the code only applies the rules."

Do not use the outage ticket ("Production server is down... revenue impacted") as a path-2 example: on the Mac it stopped at the gate and went to `human_review` after four calls, same as `hmm`. It never reaches CHOOSE.

### 4. S2 10-question eval (canvas)

Run `eval-s2-tickets-10`: n8n walks all 10 labelled tickets (`tickets eval 10` table), one execution per row, and writes the `s2_*` columns - computed route, per-field correctness, token counts, wall and server latency. S2 runs through Ollama (model from the `S2 Config` workflow). (An OpenAI-compatible variant, `eval-s2-openai-tickets-10`, is also in the repo.)

### 5. S1 10-question eval (canvas)

Run `eval-s1-tickets-10`: the same 10 tickets, the same expected answers, the same routing policy, latency measured the same way on ollaya. The shared table now shows both systems side by side - route correctness and wall latency per ticket. Run the two evals one at a time.

### 6. Chat demo (last)

Open http://localhost:5678/webhook/system1-triage-chat/chat (workflow active). Use these exact messages: the chat gate wants a ticket-shaped message, and a bare ticket gets "Could you tell me a little more?".

1. `hello` - a short canned reply, no triage.
2. System 1 handles it:
   `Hi, I need help with an infrastructure issue: every Tuesday around 2pm our export job fails, but only for customers on the German login page. Rest of the week it works fine.`
   Expect: gate pass, routed `queue:infra` at about 0.95 confidence, in roughly 50 to 100 ms.
3. Escalation to System 2:
   `Hi, we're a 500-seat enterprise and I need help: your export fails every Tuesday, we were charged twice this month, and we need volume discount pricing. Can someone own all of this?`
   Expect: gate pass, System 1 team confidence about 0.37, the reply says "System 1 was unsure, so I asked System 2", then System 2 answers billing 0.80, urgent 0.90, `page_oncall:billing`. It takes about 100 s.

Caution: the System 2 chat reply waits on gemma. Keep it warm (see the warmup section); a cold gemma can take two minutes or more (the node timeout is 300 s).

## If something breaks

- Slow or hung System 2: wait up to two minutes, then re-run (the model is loaded by then).
- ollaya down: use the canned responder (REPO_DETAILS.md, "Fallback plan").
- Everything down: play the recorded backup (`fallback/recording-shot-list.md` has the shot order).

## Fallback pointers

- `fallback/canned-ollaya-responder.json` - import, activate, point **Laya Decide** at it (one field).
- `fallback/capture-canned.sh` - regenerate the canned responses from a healthy ollaya.
- `fallback/recording-shot-list.md` - the backup video script.
- The canned responder covers the original ticket demos, not the gated chat or the five-questions workflow.
