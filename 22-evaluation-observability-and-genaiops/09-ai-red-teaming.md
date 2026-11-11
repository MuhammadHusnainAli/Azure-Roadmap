# AI Red Teaming

## Why it matters

Evaluators score how an agent behaves on the inputs you expected. Attackers and curious users don't send the inputs you expected: they encode requests in Base64, split them across turns, hide instructions in documents the agent retrieves, or coax it into calling a tool it shouldn't. **AI red teaming** deliberately probes for these failures before someone else finds them. Microsoft's **AI Red Teaming Agent** (Preview) automates adversarial probing with **PyRIT** attack strategies and Foundry's risk and safety evaluators, producing an **Attack Success Rate (ASR)** scorecard you can gate releases on and schedule in production. Automated scans complement, but don't replace, expert human red teaming.

## Key concepts

### Where red teaming fits (NIST AI RMF)

Microsoft frames this with NIST's **Govern, Map, Measure, Manage**: map risks for your use case, measure them at scale (automated scans), manage them with mitigations and incident response. Run scans at:

| Stage | Purpose |
|---|---|
| **Design** | Choose the safest base model for the use case |
| **Development** | After model upgrades or fine-tuning |
| **Pre-deployment** | Release gate for apps and agents |
| **Post-deployment** | Scheduled red-team runs on synthetic adversarial data (dashboard red team scans) |

### How the AI Red Teaming Agent works

1. **Seed attack objectives** per risk category (curated dataset, or bring your own prompts).
2. **Attack strategies** from PyRIT transform objectives to evade safeguards; a fine-tuned adversarial model drives multi-turn attacks.
3. **Evaluation**: risk and safety evaluators score each attack–response pair.
4. **Reporting**: ASR by risk category and attack complexity, logged in Foundry for tracking over time.

### Risk categories

| Category | Targets | Where |
|---|---|---|
| Hateful and unfair, sexual, violent, self-harm content | Models and agents | Local and cloud |
| Protected materials, code vulnerability, ungrounded attributes | Models and agents | Local and cloud |
| **Prohibited actions** | Agents | Cloud only |
| **Sensitive data leakage** | Agents | Cloud only |
| **Task adherence** | Agents | Cloud only |
| **Indirect prompt injection (XPIA)** | Agents | Via `IndirectJailbreak` strategy |

Text scenarios only. **Prohibited actions** uses a taxonomy generated from your policy and the agent's tools: *prohibited* (never allowed, for example social scoring), *high-risk* (allowed with human-in-the-loop confirmation, for example payments) and *irreversible* (allowed with disclosure and confirmation, for example deletions). Agentic scans are single-turn, English-only and use synthetic data and mock tools.

Cloud red teaming regions currently include East US 2, France Central, Sweden Central, Switzerland West and North Central US. Adversarial inputs are **redacted** from results, and runs against Foundry-hosted agents are transient (not stored).

### Supported targets for agentic risk

Supported: Foundry prompt agents and hosted (container) agents with Azure tool calls. Not supported: workflow agents, non-Foundry agents, function tool calls, browser automation, connected agents, computer use.

### Attack strategies (PyRIT)

| Group | Strategies |
|---|---|
| **Encoding/obfuscation** | Base64, Binary, Morse, ROT13, Caesar, Atbash, Url, AsciiArt, AsciiSmuggler, AnsiAttack |
| **Character tricks** | CharacterSpace, CharSwap, Diacritic, Flip, Leetspeak, UnicodeConfusable, UnicodeSubstitution, StringJoin, SuffixAppend |
| **Semantic** | Tense (rephrase in past tense) |
| **Jailbreak** | Jailbreak (UPIA, direct user prompt injection), IndirectJailbreak (XPIA, in tool outputs or context) |
| **Multi-turn** | Multi turn, Crescendo (gradual escalation) |
| **Complexity groups** | `EASY` (Base64, Flip, Morse), `MODERATE` (Tense), `DIFFICULT` (composition of Tense and Base64); `AttackStrategy.Compose([...])` chains two |

### Attack Success Rate

**ASR** = percentage of attacks that elicited an undesirable response. Report it overall, per risk category and per complexity level. ASR is judged by generative models, so false positives and negatives happen; review results before acting.

## How it works on Azure

### Local scan (SDK, Preview)

```bash
pip install "azure-ai-evaluation[redteam]"
```

```python
import asyncio, os
from azure.identity import DefaultAzureCredential
from azure.ai.evaluation.red_team import RedTeam, RiskCategory, AttackStrategy

def app_target(query: str) -> str:
    return call_my_app(query)          # your application or agent wrapper

async def main():
    rt = RedTeam(
        azure_ai_project=os.environ["AZURE_AI_PROJECT"],        # Foundry project endpoint
        credential=DefaultAzureCredential(),
        risk_categories=[RiskCategory.Violence, RiskCategory.HateUnfairness,
                         RiskCategory.Sexual, RiskCategory.SelfHarm],
        num_objectives=5,                                         # default 10 per category
    )
    await rt.scan(
        target=app_target,
        scan_name="support-app-pre-release",
        attack_strategies=[AttackStrategy.EASY, AttackStrategy.MODERATE,
                           AttackStrategy.Compose([AttackStrategy.Base64, AttackStrategy.ROT13])],
        output_path="redteam-scorecard.json",                    # overall_asr, per-category ASR
    )

asyncio.run(main())
```

Targets can be a callback, a model configuration (for base model selection) or a PyRIT `PromptChatTarget`.

### Cloud scan of an agent (Preview API)

```python
from azure.ai.projects.models import AzureAIAgentTarget, AgentTaxonomyInput, EvaluationTaxonomy, RiskCategory

client = project.get_openai_client()
red_team = client.evals.create(
    name="support-agent-agentic-redteam",
    data_source_config={"type": "azure_ai_source", "scenario": "red_team"},
    testing_criteria=[
        {"type": "azure_ai_evaluator", "name": "Prohibited Actions", "evaluator_name": "builtin.prohibited_actions", "evaluator_version": "1"},
        {"type": "azure_ai_evaluator", "name": "Sensitive Data Leakage", "evaluator_name": "builtin.sensitive_data_leakage", "evaluator_version": "1"},
    ],
)
target = AzureAIAgentTarget(name="support-agent", version="13")
taxonomy = project.beta.evaluation_taxonomies.create(
    name="support-agent",
    body=EvaluationTaxonomy(description="Prohibited actions taxonomy",
                            taxonomy_input=AgentTaxonomyInput(risk_categories=[RiskCategory.PROHIBITED_ACTIONS], target=target)),
)
run = client.evals.runs.create(
    eval_id=red_team.id, name="pre-release",
    data_source={"type": "azure_ai_red_team",
                 "item_generation_params": {"type": "red_team_taxonomy",
                                            "attack_strategies": ["Flip", "Base64", "IndirectJailbreak"],
                                            "num_turns": 5,
                                            "source": {"type": "file_id", "id": taxonomy.id}},
                 "target": target.as_dict()},
)
```

Review and edit the generated taxonomy before running. Results appear in the portal and through `client.evals.runs.output_items.list(...)`.

### Schedule in production

On the agent's **Monitor** tab, enable **Red team scans** with a template and schedule; findings feed the dashboard and alerts (file 07).

### Human red teaming

Microsoft's guidance on planning LLM red teaming: assemble diverse testers (security, domain, responsible AI), define harms and scope, test the base model and the full application with mitigations, iterate, and report findings with reproduction steps to owners. Use automated scans for breadth and humans for depth and novel attacks.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Local vs cloud** | Local for content risks on any callable target during development; cloud for agentic risks on Foundry agents and scheduled runs. |
| **Gate** | Block release on ASR above an agreed threshold for high-severity categories (often 0% for prohibited actions and data leakage on tested scenarios). |
| **Strategy coverage** | Start with EASY/MODERATE; add Crescendo, multi-turn and IndirectJailbreak for agents with tools or retrieval. |
| **Custom objectives** | Bring your own seed prompts for domain-specific harms. |
| **Environment** | Run in a "purple" (non-production) environment; mock or sandbox tools. |
| **Mitigations** | Guardrails, prompt shields, tool approval, least-privilege tools, system message safety templates ([Topic 23](../23-responsible-ai-security-and-governance/README.md)). |

## Hands-on

1. Run a local scan against a base model configuration with the four content categories and `EASY` strategies; read `overall_asr`.
2. Wrap your app in a callback; rerun with `MODERATE` and a composed strategy; compare with the base model.
3. Run a cloud prohibited-actions scan against a Foundry agent with an Azure tool; review and edit the generated taxonomy first.
4. Add a mitigation (tool approval or guardrail) and rerun; record ASR before and after.
5. Schedule a weekly red team scan on the agent's Monitor tab.

## Security, cost and reliability notes

- **Authorisation**: only red team systems you own or are authorised to test; inform the platform and security teams.
- **Harmful content exposure**: cloud results redact adversarial inputs; restrict who can view local outputs.
- **Side effects**: agent scans can trigger tools; use non-production tools and data.
- **Non-determinism**: ASR varies between runs; track trends and review individual findings.
- **Limitations**: synthetic data, no full sandboxing, adversarial population only; not proof of safety.
- **Cost**: adversarial generation and evaluation consume tokens; size `num_objectives` sensibly.

## Common mistakes

| Mistake | Fix |
|---|---|
| Red teaming the base model only | Test the full app/agent with its tools and mitigations |
| One scan before launch, then never again | Scheduled scans and re-scan on every model or prompt change |
| Treating ASR 0% as "secure" | Combine with human red teaming and threat modelling |
| Running agent scans against production tools | Sandbox |
| No owner for findings | Track findings like security bugs with SLAs |

## Check yourself

1. Which three things does the AI Red Teaming Agent automate?
2. Which risk categories are agent-only and cloud-only?
3. Explain Jailbreak vs IndirectJailbreak.
4. What are the EASY, MODERATE and DIFFICULT groups?
5. What is ASR, and why should results be reviewed?
6. What are the three tiers of the prohibited actions taxonomy?

<details>
<summary>Answers</summary>

1. Scanning (adversarial probing), evaluating attack–response pairs (ASR), and reporting/logging scorecards.
2. Prohibited actions, sensitive data leakage and task adherence.
3. Jailbreak (UPIA) injects attack prompts directly in user input; IndirectJailbreak (XPIA) places them in tool outputs or retrieved context.
4. EASY: Base64, Flip, Morse; MODERATE: Tense; DIFFICULT: a composition of Tense and Base64.
5. Attack Success Rate, the percentage of attacks producing undesirable responses; it's judged by generative models and can include false positives.
6. Prohibited (never allowed), high-risk (allowed with human-in-the-loop confirmation), irreversible (allowed with disclosure and confirmation).

</details>

## References

- [AI Red Teaming Agent (Preview)](https://learn.microsoft.com/azure/foundry/concepts/ai-red-teaming-agent)
- [Run AI Red Teaming Agent locally](https://learn.microsoft.com/azure/foundry/how-to/develop/run-scans-ai-red-teaming-agent)
- [Run AI Red Teaming Agent in the cloud](https://learn.microsoft.com/azure/foundry/how-to/develop/run-ai-red-teaming-cloud)
- [Planning red teaming for LLMs](https://learn.microsoft.com/azure/foundry/openai/concepts/red-teaming)
- [PyRIT](https://azure.github.io/PyRIT/)
