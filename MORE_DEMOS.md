# More demos

What to try after the [README](README.md)'s first demo: the chat front, the five-questions pipeline, and pointing System 2 at a different model. Get the two triage workflows from the README running first. For the full file list, the System 1 vs System 2 evaluations and the fallback plan, see [REPO_DETAILS.md](REPO_DETAILS.md).

## Before you start

### Choosing the System 2 model

The Ollama System 2 workflows read the model name from the `S2 Config` workflow (default `gemma4:12b-mlx`) through a **Get S2 Model** node. To use a different model, such as one you already have from `ollama list`, edit the value in the **S2 Model Name** node of `S2 Config` and save. No activation and no other workflow edits are needed.

- If a Get S2 Model node shows an error, open it and pick `S2 Config` from the workflow list.
- If the lookup fails, the workflows fall back to `gemma4:12b-mlx` and ignore your override. Run one System 2 path and check that the node's output shows your `model` name.

The OpenAI-compatible variants (`*-openai-*.json`) and `five-questions-chat-demo.json` don't use `S2 Config`. Each has its own **Endpoint Config** node (base URL, model, and API key if required) to edit instead.

### Publishing webhook and chat workflows

- A webhook or chat workflow only listens while published (**Publish**, called **Activate** in older n8n). "Active" below means published and listening, not merely open in the editor.
- Re-importing a file in the n8n UI creates a duplicate workflow. Delete or deactivate old copies first, so only one copy listens on each path.
- The Ollama and OpenAI-compatible terminal five-questions workflows share the `/five-questions` webhook path. Publish only the one you intend to use.

Prefer the terminal? [REPO_DETAILS.md](REPO_DETAILS.md#bulk-import-from-the-terminal) has a bulk-import command that reloads every workflow after a `git pull`.

## Chat front

`system1-ticket-triage-chat.json` screens each chat message with a System 1 gate before doing any triage:

- Safe casual message: a short canned conversational reply. No LLM call, no conversation memory - a small chat demo, not a general assistant.
- Support ticket: the usual System 1 triage and routing; a confident decision gets the same team/confidence/urgency/route/latency reply.
- Malicious message: score >= 0.7 gets a polite refusal before triage. Anger or swearing about a legitimate support issue is not malicious by itself; the gate question says so explicitly.
- Uncertain ticket triage: System 1 team confidence < 0.6 escalates to System 2. The reply says System 1 was unsure, then shows both systems' answers and latencies. A failed or malformed System 2 response stays `human_review` instead of inventing a decision.

The intent choice needs confidence >= 0.6 to enter triage; an uncertain intent asks for more detail, an invalid or unavailable gate asks you to retry. Maliciousness takes precedence over intent. These thresholds are demo defaults to play with, not a security boundary. Routes are labels only: nothing is paged or created.

1. Import `workflows/system1-ticket-triage-chat.json` and publish it.
2. Open http://localhost:5678/webhook/system1-triage-chat/chat and type into the chat page.
3. Try `hello`, then a real ticket such as `I was charged twice for my subscription this month` or `Production server is down, customers cannot log in, revenue impacted`. Check that an angry but legitimate complaint still gets support.

The gate and confident-ticket paths need only ollaya (port 11435). The low-confidence escalation also needs Ollama (port 11434) with the model set in `S2 Config`.

## Five-questions pipeline

`five-questions-demo.json` runs the five System 1 decisions as one pipeline. One ticket goes in through a webhook. System 1 decides at every stage, in a different way; sticky notes on the canvas name each step; plain code applies policy to what the model decided.

1. CLASSIFY - a choice call picks the team (infra, billing or sales) with a confidence.
2. ROUTE - a noul call scores urgency from 0 to 1. Code routes: >= 0.7 pages on-call, otherwise the team queue.
3. RANK - a score call picks a business-impact level (low, medium or high), mapped to a priority: P1 >= 1.5, P2 >= 0.5, otherwise P3.
4. GATE - a choice call: automate, or a human must handle it (legal threat, security incident, large refund, very upset customer). A human verdict stops the run at `human_review`; an unusable answer fails safe to human.
5. CHOOSE - a choice call: resolve with a help article, hand to the team, or escalate to System 2. Code policy on top: an urgent ticket is never auto-answered, and a CLASSIFY confidence under 0.6 always escalates. If System 2 is also under 0.6 or fails to answer, the verdict is `human_review`.

A normal ticket makes five System 1 calls in a row (each roughly 0.5 s on the demo Mac) and System 2 only when escalated.

Run it:

1. Import `s2-config.json` first if you haven't, then `five-questions-demo.json`. The OpenAI-compatible `five-questions-openai-demo.json` uses its own Endpoint Config node instead of `s2-config.json` - and shares the `/five-questions` webhook path, so publish only one of the two.
2. Click **Execute workflow** (it then waits for one call on the test URL), or publish the workflow to use the production URL. A webhook is an HTTP address that starts the workflow when it receives a request.
3. Send a ticket from a terminal:

   ```bash
   curl -X POST http://localhost:5678/webhook-test/five-questions \
     -H 'Content-Type: application/json' \
     -d '{"ticket":"Every Tuesday around 2pm our export job fails, but only for customers on the German login page. Rest of the week it works fine."}'
   ```

   Use `/webhook/five-questions` instead of `/webhook-test/five-questions` when the workflow is published. The test URL listens only during a manual test; click Execute again before each new test request. Leave out `ticket` to use a built-in example.
4. The JSON reply has `final_route`, `decided_by`, a `stages` object with each step's output, `laya_calls`, `laya_total_ms` and `total_ms`. Three terminal verdicts: System 1 decided, human via gate, and System 2 answered - the last can still be `human_review` if System 2 is unsure or unavailable.

To tell the paths apart: four Laya calls with CHOOSE skipped is the gate-human terminal; five calls with a System 1 verdict is System 1 alone; `stages.choose.system_2_usable` distinguishes a usable System 2 answer from its human-review fallback.

[DEMO.md](DEMO.md) (sections 3.1 and 3.2) has three rehearsed paths with exact prompts and PowerShell commands, plus a chat variant of this pipeline (`five-questions-chat-demo.json`).

The thresholds (0.7 urgent, 0.6 confidence, impact bands) are demo values to tune for your own tickets; routes are labels only; the gate is a demo guardrail, not a security boundary. System 1 can be confidently wrong, and a confident answer passes the gate.

## Compare the two systems

Want numbers? Three evaluation setups score System 1 against System 2 over labelled tickets, including the recommended pair of independent runs over the same 10 tickets with side-by-side correctness and latency. They are in [REPO_DETAILS.md](REPO_DETAILS.md#compare-system-1-and-system-2).
