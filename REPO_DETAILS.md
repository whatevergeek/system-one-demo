# Repository details

Reference material: what each file is, how to compare System 1 and System 2 over labelled tickets, and what to do if the live demo dies. Start with the [README](README.md); the chat and five-questions demos are in [MORE_DEMOS.md](MORE_DEMOS.md); the talk's runbook is [DEMO.md](DEMO.md).

## Workflows

| File | What it does | Runs from | Needs |
| --- | --- | --- | --- |
| `system1-ticket-triage.json` | System 1 triages four sample tickets; code routes them | Canvas | ollaya |
| `system2-ticket-triage.json` | The same tickets and routing, answered by an LLM | Canvas | Ollama |
| `system1-ticket-triage-chat.json` | Chat front: System 1 gate, triage, escalation when unsure | Chat | ollaya, Ollama |
| `five-questions-demo.json` | Classify, route, rank, gate, choose in one pipeline | Webhook | ollaya, Ollama |
| `five-questions-chat-demo.json` | The same pipeline behind a chat window | Chat | ollaya, OpenAI-compatible |
| `system1-choice-poc.json` | Smallest possible question + choices call | Canvas | ollaya |
| `eval-s1-tickets-10.json`, `eval-s2-tickets-10.json` | Recommended System 1 vs System 2 comparison | Canvas | ollaya / Ollama |
| `compare-s1-s2-evaluation.json` | Older comparison: both systems in one run | Canvas | ollaya, Ollama |
| `compare-s1-s2-metrics-evaluation.json` | Older comparison: scored runs in the Evaluations tab | Evaluations tab | ollaya, Ollama |
| `s2-config.json` | Holds the System 2 model name for the Ollama workflows | Called by others | - |

Files with `openai` in the name (`system2-openai-ticket-triage.json`, `five-questions-openai-demo.json`, `eval-s2-openai-tickets-10.json`) are the same workflows pointed at an OpenAI-compatible endpoint, set in their **Endpoint Config** node instead of `S2 Config`.

## Datasets

| File | Contents | Used by |
| --- | --- | --- |
| `tickets-eval-10.csv` | 10 labelled tickets, plus blank result columns | Recommended comparison, Evaluations-tab comparison |
| `comparison_runs.csv` | Header only: the run-status table | Recommended comparison |
| `tickets-eval-remaining-9.csv` | The same tickets without row 1 | Evaluations-tab comparison (step 6) |
| `tickets.csv` | 4 labelled tickets | Same-run comparison |
| `eval-check-expressions.txt` | The five checks to paste | Evaluations-tab comparison |

All tickets and labels are synthetic and hand-picked: they show how the evaluation works, they are not an accuracy benchmark.

## Bulk import from the terminal

`docker-compose.yml` mounts `./workflows` into the container, so you can import everything at once:

```bash
docker exec system-one-demo-n8n n8n import:workflow --separate --input=/data/workflows

# or one file:
docker exec system-one-demo-n8n n8n import:workflow --input=/data/workflows/five-questions-demo.json
```

Workflow ids are fixed, so a CLI import updates workflows in place (the UI's Import from File creates a duplicate instead). A re-import also resets `S2 Config` to the default model, clears the table picks in the comparison workflows, and unpublishes webhook and chat workflows.

## Compare System 1 and System 2

Every comparison scores both systems with the same routing rule: team confidence below 0.6 routes to `human_review`, urgency of 0.7 or more routes to `page_oncall:<team>`, anything else to `queue:<team>`.

To load a CSV, open **Data tables** in n8n, create a table and use **Import CSV**. To point a workflow at a table, double-click the node and set its Data table field.

### Recommended: two runs over the same 10 tickets

`eval-s1-tickets-10.json` and `eval-s2-tickets-10.json` run independently over the same tickets, so neither system runs inside the other's test.

1. Import both workflows.
2. Import two Data tables and keep the names n8n gives them. Some nodes find the tables by name.
   - `datasets/tickets-eval-10.csv` - becomes `tickets eval 10`
   - `datasets/comparison_runs.csv` - becomes `comparison runs`
3. In each workflow, select `tickets eval 10` in **When fetching a dataset row**. Leave every other table field alone.
4. Click **Execute workflow** on one canvas, wait for it to finish, then run the other.

Each run fills only its own `s1_*` or `s2_*` columns, so the same 10 rows end up showing both systems side by side:

| Columns | Meaning |
| --- | --- |
| `*_route`, `*_route_correct` | The computed route, and `correct` / `WRONG` against `expected_route` (likewise for team and urgency) |
| `*_wall_ms` | Client-side time around each call, measured the same way for both. The headline number |
| `*_server_ms` | Each backend's own reported total, including model load. Corroboration only |
| `*_input_tokens`, `*_output_tokens` | As reported by each backend. `0` is real (System 1 writes no text); blank means not reported |
| `*_status` | `STALE since ...` while a run is in progress, `current as of ...` once that row is done |
| `s2_parse_error` | Blank, or why System 2's reply couldn't be parsed |

**Is a run finished?** In `comparison runs`, the `latest_s1` / `latest_s2` row goes from `running` to `complete`, with `processed` equal to `expected`. A row stuck at `running` blocks the next run: stop that execution in the Executions list, set the status to `aborted`, and run again.

<details>
<summary>Repairs for older tables and blanked nodes</summary>

**No `s1_status` / `s2_status` columns** (table imported from an earlier CSV): add both as Text columns by hand. Don't re-import, or the tickets are duplicated. Without them the first run stops before any model call.

**A run-state node lost its column mapping** (usually after picking a table in it by hand): restore only the blank values, switching the field to Expression when the value starts with `={{`. Leave every other field empty, especially `ticket` and `expected_*` in Mark Rows Stale.

- Claim Run: `key` = `latest_s1`; `status` = `running`; `batch` = `={{ $execution.id }}`; `expected` = `={{ String($('When fetching a dataset row').first().json._rowsLeft + 1) }}`; `processed` = `0`; `started_at` = `={{ $now.toISO() }}`; `completed_at` empty.
- Mark Rows Stale (S1): `s1_status` = `={{ 'STALE since ' + $now.toISO() + ' - a new S1 run is in progress; s1_* values on this row are from a previous run' }}`.
- Record Progress: `key` = `latest_s1`; `status` = `={{ $('When fetching a dataset row').first().json._rowsLeft === 0 ? 'complete' : 'running' }}`; `processed` = `={{ String($('When fetching a dataset row').first().json.row_number + 1) }}`; `completed_at` = `={{ $('When fetching a dataset row').first().json._rowsLeft === 0 ? $now.toISO() : '' }}`.
- S2 workflow: the same, with `key` = `latest_s2` and Mark Rows Stale (S2) setting `s2_status` (replace `S1` / `s1_*` with `S2` / `s2_*` in the message).

</details>

### Older: both systems in one run

`compare-s1-s2-evaluation.json` sends each ticket through both systems in a single canvas run and adds an `agree` column.

1. Create a table `tickets` from `datasets/tickets.csv`, and a table `comparison_runs` with Text columns `key`, `status`, `batch`, `expected`, `processed`, `started_at`, `completed_at`.
2. Select `tickets` in **When fetching a dataset row** and **Write Results to Dataset**, and `comparison_runs` in **Get Run Status**, **Claim Run** and **Record Progress**. Re-pick them after every re-import.
3. Execute the whole workflow from the trigger. Running single nodes writes nothing, and a partial run leaves the status stuck at `running`.

The results are current once `comparison_runs` shows `complete` with `processed` equal to `expected`. Recovery is the same as above.

### Older: scored runs in the Evaluations tab

`compare-s1-s2-metrics-evaluation.json` gives each run named scores in n8n's Evaluations tab. It can't measure System 2 latency, and it needs a registered Community edition (free: Settings > Usage and plan > Unlock).

1. Import the workflow. Create a table `tickets_eval10` from `datasets/tickets-eval-10.csv` and select it in **When fetching a dataset row**.
2. On the **Ollama Chat Model** node, create an Ollama credential with Base URL `http://host.docker.internal:11434` and no API key.
3. Set the model in this workflow's own **S2 Config** code node (it doesn't use the shared `S2 Config` workflow).
4. Execute the workflow once by hand, so the Evaluations tab can see the row shape.
5. In the Evaluations tab, set up tests with **S2 Decide (AI)** as the AI node. Remove the preselected Correctness check, and add the five custom checks from `datasets/eval-check-expressions.txt`.
6. Saving the checks creates a table `Evaluation: System 1 vs System 2 - Ticket Triage (Evaluation + Metrics)` with one example row. If that row is the "Production server is down..." ticket, label it `infra` / `1` / `page_oncall:infra` and import `tickets-eval-remaining-9.csv`. Otherwise import `tickets-eval-10.csv` and delete or label the example row. Map `ticket` and the three `expected_*` columns only.
7. Run the test.

Only `expected_route` is scored. `s1_no_answer` flags System 1 returning nothing usable; `s2_invalid_output` flags System 2 replies that aren't valid JSON or are off the allowed options.

## Fallback plan

If the live demo dies, `fallback/` has two independent safety nets (details in [fallback/README.md](fallback/README.md)):

- **Canned responses:** import and publish `fallback/canned-ollaya-responder.json`, then change the **Laya Decide (System 1)** URL to `http://host.docker.internal:5678/webhook/canned-ollaya`. Run `fallback/capture-canned.sh` beforehand so the answers are real captures. It covers `system1-ticket-triage` and the choice PoC only, not the chat or five-questions workflows.
- **Recording:** `fallback/recording-shot-list.md` is a shot-by-shot script for a backup video.
