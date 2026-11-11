# Routines and Long-Running Agents

## Why it matters

Not every agent waits for a user to type. Many valuable agents run **on a schedule** (daily summaries, weekly risk reports), **on events** (a GitHub issue opens, a Teams message arrives), or **for a long time** (deep analysis, multi-hour investigations, waiting days for an approval). Building that trigger and durability layer yourself means stitching together schedulers, Functions, queues, storage and auth. Foundry Agent Service provides **routines** for triggers, **background mode** for asynchronous responses, and **resilient execution** for hosted agents that must survive crashes and restarts.

## Key concepts

### Routines

A routine is a project-scoped automation rule: **one trigger, one action**.

| Component | Description |
|---|---|
| **Trigger** | When it runs: timer (once), recurring schedule (cron), or event |
| **Action** | Invoke one prompt agent or hosted agent (Responses or Invocations API) |
| **Input** | Text or JSON sent to the agent |
| **Lifecycle** | Enable, disable, update or delete without recreating the agent |
| **Run history** | Each run's input, output, status and link to the response and trace |

Trigger types:

| Type | Example | Notes |
|---|---|---|
| **Timer** | Run once at `2030-09-01T09:00:00Z` | Becomes inactive after firing |
| **Recurring (`schedule`)** | Weekdays at 07:00 (`0 7 * * 1-5`) | Cron with IANA time zone; **minimum interval five minutes** |
| **Event** | `github_issue` opened/closed; `custom` trigger with the `teams` provider for channel messages | Uses project connections (GitHub, Teams connectors) |

Routines answer *when should this agent run?*. They aren't orchestration: for branching, multiple agents or approvals, use a workflow (file 09).

### Identity and constraints

| Aspect | Behaviour |
|---|---|
| Dispatch identity | **Agent identity by default**; optionally the **creator's identity** (only the creating principal) for tools needing delegated access; fixed at creation |
| Endpoint auth | Routine dispatch doesn't support the preview AgentBlueprint scheme; keep Azure RBAC on the agent endpoint |
| Networking | Works with VNet-secured projects (reuses the agent invocation path) |
| CMK | **Not supported**; don't use routines for CMK-required workloads |
| Regions | All Foundry regions except UK West, Switzerland West, Japan West, UAE North and Norway East |
| Agent types | Prompt and hosted agents (not workflow agents) |
| SDK | Python `azure-ai-projects>=2.4.0` via `client.beta.routines` (beta surface) |

### Agent-scheduled reminders

Where routines are external triggers, a hosted agent can **schedule its own follow-up** with the built-in **reminder tool** (Preview, toolbox only): the agent decides it needs to check back and the service re-invokes it later.

### Background mode

For long single requests (complex reasoning, report generation, image generation), set `background=True` on the response, then poll `responses.retrieve(id)` until the status leaves `queued`/`in_progress`. Background mode also lets MCP tool calls exceed the 100-second synchronous limit (file 07).

### Resilient long-running hosted agents

Microsoft distinguishes three capabilities:

| Capability | Provides | Doesn't provide |
|---|---|---|
| **Background execution** | Work continues without the original connection; clients poll or reconnect | Recovery after the owning process stops |
| **Resilient execution** | Durable work identity, persisted input, process-loss detection, handler re-entry | Automatic preservation of every intermediate state or side effect |
| **Stream replay** | Reconnecting clients receive retained events from a cursor | A checkpoint of internal workflow state |

Key points:

- Recovery **re-enters the handler from the beginning**; it isn't deterministic replay. Use durable checkpoints or watermarks (state store, `$HOME`) to skip completed work, and make side effects idempotent.
- For the Responses protocol, crash recovery applies to **stored background responses** when the server opts in.
- For human-in-the-loop waits of hours or days, use **multi-turn durable tasks** (`@multi_turn_task`) that suspend and resume on the same `task_id` (file 09).
- **Steering** lets a client adjust an in-flight turn; **cancellation** stops a hosted-agent turn.

## How it works on Azure

Create a recurring routine in Python:

```python
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient

client = AIProjectClient(endpoint=PROJECT_ENDPOINT, credential=DefaultAzureCredential())
routine = client.beta.routines.create_or_update(
    routine_name="daily-summary",
    description="Weekday morning summary of support activity.",
    enabled=True,
    triggers={"weekday-morning": {"type": "schedule",
                                  "cron_expression": "0 7 * * 1-5",
                                  "time_zone": "Europe/London"}},
    action={"type": "invoke_agent_responses_api",
            "agent_name": "support-summariser",
            "input": "Summarise new and escalated tickets from the last 24 hours."})
print(routine.name, routine.enabled)
```

The Azure Developer CLI supports the same (`--trigger recurring --cron "0 7 * * 1-5" --time-zone UTC`), and the portal shows run history linked to traces.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Routine vs Logic Apps vs Functions timer** | Routines for simple "when X, run this agent" with Foundry governance and traces; Logic Apps when you need connectors, branching and approvals; Functions/Event Grid for custom event plumbing. |
| **Agent vs creator identity** | Agent identity for unattended jobs (least privilege, auditable); creator identity only when tools need delegated user access, and remember it's tied to one person. |
| **Synchronous vs background** | Background for anything that may exceed tens of seconds; give users progress and notifications. |
| **Resilience** | Opt into resilient execution for jobs where losing work is costly; design idempotent steps and checkpoints. |
| **Event noise** | Filter events (labels, channels) so agents aren't triggered by everything; agents cost money per run. |
| **Output destination** | Write results somewhere useful: Teams post, ticket comment, email, storage, not just the run history. |

## Hands-on

1. Create a prompt agent `support-summariser` with a tool that reads recent tickets (OpenAPI or function) and a Teams or email output tool.
2. Create the recurring routine above with a five-minute cron (`*/5 * * * *`) for testing; watch run history and open a linked trace.
3. Switch to the weekday-morning schedule and disable/enable the routine.
4. Create a **GitHub issue** event routine (`github_issue` trigger via a GitHub connection) that invokes a triage agent to label and comment on new issues.
5. Run a long analysis in background mode:

```python
oai = client.get_openai_client(agent_name="support-summariser")
r = oai.responses.create(input="Analyse the last 90 days of tickets and find the top 5 root causes.",
                         background=True)
import time
while r.status in ("queued", "in_progress"):
    time.sleep(5)
    r = oai.responses.retrieve(r.id)
print(r.status, r.output_text[:500])
```

## Security, cost and reliability notes

- **No secrets in routine input or prompts**; use connections and Entra ID.
- **Least privilege** for the agent identity used by unattended routines; they run without a human watching.
- **Idempotency**: event triggers and recovery can cause repeated processing; design tools to tolerate duplicates.
- **Runaway cost**: frequent schedules and noisy events multiply model and tool spend; set budgets and alerts.
- **Monitoring**: alert on failed routine runs; review run history regularly.
- **Compliance**: no CMK support for routines; choose another trigger mechanism for CMK-bound workloads.

## Common mistakes

| Mistake | Fix |
|---|---|
| Using routines to orchestrate multi-step processes | Use workflows or Logic Apps |
| Cron every minute | Minimum is five minutes; choose a sensible cadence |
| Creator identity for production jobs | Use agent identity unless delegation is essential |
| Long synchronous requests timing out | Use background mode |
| Assuming recovery resumes mid-function | Recovery re-enters the handler; use checkpoints |
| Results only visible in run history | Deliver outputs to users' channels |

## Check yourself

1. What are the three routine trigger types?
2. What identity does a routine use by default, and when would you change it?
3. Name two constraints of routines (security, region or encryption).
4. What's the difference between routines and workflows?
5. How do you run a long response asynchronously?
6. Distinguish background execution, resilient execution and stream replay.
7. Why must side effects be idempotent in resilient agents?
8. How can a hosted agent schedule its own follow-up?

<details>
<summary>Answers</summary>

1. Timer (once), recurring schedule (cron), and event (for example GitHub issue or Teams message).
2. The agent identity; switch to creator identity only when tools need delegated user access (fixed at creation).
3. Any two: no CMK support; not in some regions; no AgentBlueprint endpoint scheme; five-minute minimum cron interval; prompt and hosted agents only.
4. Routines decide when an agent runs (trigger → one agent); workflows decide how multiple steps, decisions or agents connect.
5. Set `background=True` on `responses.create` and poll `responses.retrieve` until it completes.
6. Background: work continues after the client disconnects; resilient: work survives process loss via durable identity and handler re-entry; stream replay: reconnecting clients receive retained events.
7. Recovery re-enters handlers from the beginning, so steps may run more than once.
8. By calling the reminder tool (Preview) exposed through a toolbox.

</details>

## References

- [Routines in Foundry Agent Service](https://learn.microsoft.com/azure/foundry/agents/concepts/routines)
- [Automate agents with routines](https://learn.microsoft.com/azure/foundry/agents/how-to/use-routines)
- [Long-running agent resilience](https://learn.microsoft.com/azure/foundry/agents/concepts/long-running-agent-resilience)
- [Background responses (runtime components)](https://learn.microsoft.com/azure/foundry/agents/concepts/runtime-components)
- [Reminder tool (Preview)](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/reminder-tool)
- [Add a human-in-the-loop approval step](https://learn.microsoft.com/azure/foundry/agents/how-to/add-human-in-the-loop)
