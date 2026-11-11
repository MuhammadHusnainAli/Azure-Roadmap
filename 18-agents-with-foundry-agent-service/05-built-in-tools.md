# Built-In Tools

## Why it matters

Tools turn a model into an agent. Foundry Agent Service ships a catalogue of managed tools that need no code from you: search your files, query an Azure AI Search index, run Python, search the web, drive a browser, generate images, ground on SharePoint, Fabric or Work IQ. Choosing the right built-in tool (and knowing each one's status, cost model, data boundary and limits) is often the difference between a working agent in an afternoon and weeks of custom integration. Architects especially need to know which tools leave the compliance boundary and which are still in preview.

## Key concepts

### Two ways to attach tools

| Mode | How | Notes |
|---|---|---|
| **Direct tool integration** | Add the tool to the agent definition (`tools=[...]`) | Simplest for prompt agents |
| **Toolbox** | Curate tools once; expose them through one managed MCP endpoint | Reuse across agents and frameworks, central auth and governance, tool search (file 07) |

Not every tool supports both. Function calling is direct only (client-side execution); Grounding with Bing, computer use, image generation, SharePoint, Fabric data agent and Azure Functions are direct only; tool search and the reminder tool are toolbox only.

### Knowledge tools

| Tool | Status | What it does | Key facts |
|---|---|---|---|
| **File search** | GA | Searches uploaded files in a **vector store** (service parses, chunks, embeds and indexes) | One vector store per agent and one per conversation; up to 10,000 files per vector store; standard setup stores vectors in your own AI Search |
| **Azure AI Search** | GA | Queries an existing index via a project connection | `query_type`: `simple`, `vector`, `semantic`, `vector_simple_hybrid`, `vector_semantic_hybrid` (default); optional filter and `top_k`; use project managed identity (not keys) for private networking |
| **Foundry IQ knowledge base** | Preview (via MCP) | Agentic retrieval across many sources | [Topic 17 file 08](../17-rag-and-knowledge/08-foundry-iq.md) |
| **SharePoint** | Preview | Grounds on a SharePoint site or folder with user permissions | Uses the user's identity (OBO) |
| **Fabric data agent** | Preview | Asks a Fabric data agent for answers over business data | Direct only |
| **Fabric IQ** / **Work IQ** | Preview | Business-data and Microsoft 365 context | [Topic 17 file 09](../17-rag-and-knowledge/09-work-iq-fabric-iq-and-web-grounding.md) |

### Web tools

| Tool | Status | Notes |
|---|---|---|
| **Web Search** | GA (recommended) | No Bing resource to manage; `user_location`, `search_context_size`, domain allow/block lists |
| **Grounding with Bing Search** | GA | Your own Bing resource; `count`, `freshness`, `market`, `set_lang`; supports non-OpenAI models sold by Azure |
| **Grounding with Bing Custom Search** | Preview | Restrict to chosen domains |

All web tools send data **outside the Azure compliance boundary**, aren't covered by the DPA, ignore private endpoints, and incur extra cost.

### Action and execution tools

| Tool | Status | What it does | Key facts |
|---|---|---|---|
| **Code interpreter** | GA | Runs Python in a sandbox to analyse data, make charts, process files | Extra charge per session; session active up to 1 hour with 30-minute idle timeout; concurrent conversations create separate sessions; generated files via container file citations |
| **Custom code interpreter** | Preview | Code execution in a sandbox you control, connected via MCP | For custom packages or isolation needs |
| **Browser automation** | **GA** | Managed remote browser: navigate, read, fill forms, upload, with **Live View** and **Take control** | Can take unintended actions; test and monitor; exposed as an MCP tool set |
| **Computer use** | Preview | Interprets screenshots and proposes clicks, typing, scrolling using the `computer-use-preview` model | Your code executes actions in your environment; strong isolation needed |
| **Image generation** | Preview | Generates images within the agent loop | Direct only |
| **Reminder tool** | Preview | Schedule follow-ups | Toolbox only |
| **Azure Speech** | — | Speech capabilities as a tool | See Foundry Tools ([Topic 20 file 03](../20-foundry-tools/03-azure-speech.md)) |

**Deep research**: earlier previews offered a deep research tool; it doesn't appear in the current (late 2026) agent tool catalogue. For research-style agents, combine web search with a reasoning model at high effort and background mode, or a hosted agent.

### Built-in vs custom

Built-in tools run in Microsoft-managed infrastructure with managed auth. Custom tools (functions, OpenAPI, Azure Functions, MCP) connect your own systems (file 06, 07).

## How it works on Azure

Attaching built-in tools in Python (`azure-ai-projects` 2.x):

```python
from azure.ai.projects.models import (
    PromptAgentDefinition, FileSearchTool, CodeInterpreterTool, AutoCodeInterpreterToolParam,
    WebSearchTool, AzureAISearchTool, AzureAISearchToolResource, AISearchIndexResource,
    AzureAISearchQueryType)

tools = [
    FileSearchTool(vector_store_ids=[vector_store.id]),
    CodeInterpreterTool(container=AutoCodeInterpreterToolParam(file_ids=[csv_file.id])),
    WebSearchTool(),
    AzureAISearchTool(azure_ai_search=AzureAISearchToolResource(indexes=[
        AISearchIndexResource(project_connection_id=search_conn_id, index_name="policies",
                              query_type=AzureAISearchQueryType.SIMPLE)])),
]
# omit query_type to get the default, vector_semantic_hybrid (needs a vectoriser and semantic config)
```

Files and vector stores are managed with the OpenAI-compatible client from `project.get_openai_client()` (`files.create`, `vector_stores.create`, `vector_stores.files.upload_and_poll`).

Tool availability varies by **region and model**; check the tool-support table in the limits article before choosing a region.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **File search vs Azure AI Search tool vs Foundry IQ** | File search for a few uploaded documents per agent or conversation; Azure AI Search tool for an existing, governed index; Foundry IQ for multi-source, permission-aware knowledge shared by many agents. |
| **Web Search vs Bing tools** | Web Search to start; Grounding with Bing when you need market/freshness control or non-OpenAI models; custom search for domain restriction. |
| **Code interpreter** | Great for data analysis and file generation; budget per-session charges and avoid sending sensitive data unless the sandbox meets your requirements. |
| **Browser automation vs computer use vs APIs** | Prefer APIs (OpenAPI, MCP) where they exist. Browser automation (GA) for websites without APIs; computer use (Preview) for desktop/UI automation in your own sandbox. |
| **Number of tools** | Up to 128 per agent, but more tools reduce selection accuracy and add tokens; use a toolbox with tool search for large sets. |
| **Instructions** | Tell the agent when to use each tool and to cite sources; use `tool_choice="required"` when a tool must be used. |

## Hands-on

Build a "data analyst" agent with file search and code interpreter.

```python
import os
from azure.identity import DefaultAzureCredential
from azure.ai.projects import AIProjectClient
from azure.ai.projects.models import (PromptAgentDefinition, FileSearchTool,
                                      CodeInterpreterTool, AutoCodeInterpreterToolParam)

project = AIProjectClient(endpoint=os.environ["FOUNDRY_PROJECT_ENDPOINT"], credential=DefaultAzureCredential())
oai = project.get_openai_client()

vs = oai.vector_stores.create(name="sales-policies")
with open("discount-policy.pdf", "rb") as f:
    oai.vector_stores.files.upload_and_poll(vector_store_id=vs.id, file=f)
with open("sales-2026.csv", "rb") as f:
    csv = oai.files.create(file=f, purpose="assistants")

agent = project.agents.create_version(agent_name="sales-analyst", definition=PromptAgentDefinition(
    model="gpt-5-mini",
    instructions=("Use code interpreter for any calculation on sales-2026.csv. "
                  "Use file search for policy questions and cite the policy. "
                  "Never guess numbers."),
    tools=[FileSearchTool(vector_store_ids=[vs.id]),
           CodeInterpreterTool(container=AutoCodeInterpreterToolParam(file_ids=[csv.id]))]))

agent_client = project.get_openai_client(agent_name=agent.name)
r = agent_client.responses.create(
    input="Which region exceeded the maximum discount allowed by policy in Q3? Show a bar chart.")
print(r.output_text)
for item in r.output:
    print(item.type)            # look for file_search_call, code_interpreter_call
```

Download the generated chart via the container file citation, then open the trace to see each tool call and its latency and cost.

## Security, cost and reliability notes

- **Data boundary**: web tools leave the Azure compliance boundary; SharePoint and Work IQ act with user identity; code interpreter runs your data in a managed sandbox.
- **Indirect prompt injection** via files, web pages and browser content is the main risk; keep content filters (XPIA protection) on, and require approvals for actions.
- **Browser automation** can act on websites; restrict to approved sites, monitor with Live View, and avoid logged-in sessions to sensitive systems without controls.
- **Costs**: code interpreter sessions, web/Bing calls, file search storage (vector store size per day) and tokens for retrieved content, browser sessions.
- **Limits**: vector store file limits, one vector store per agent/conversation, session timeouts; handle tool errors gracefully in instructions.
- **Admin control**: tool governance lets administrators restrict which tools can be used.

## Common mistakes

| Mistake | Fix |
|---|---|
| Uploading thousands of documents to file search for an enterprise knowledge base | Use Azure AI Search or Foundry IQ |
| Key-based AI Search connection in a private network | Use project managed identity |
| Assuming web grounding stays in your region | It doesn't; review compliance |
| Leaving code interpreter on for agents that never need it | Remove unused tools |
| Using computer use where an API exists | Prefer API-based tools |
| No citation instruction | Require citations for knowledge tools |

## Check yourself

1. Which tools support both toolbox and direct integration, and which are direct only?
2. When would you choose file search, the Azure AI Search tool or Foundry IQ?
3. What query types does the Azure AI Search tool support, and which is the default?
4. What are code interpreter's session limits and cost implications?
5. What is the status of browser automation and of computer use?
6. Name three compliance facts about web grounding tools.
7. How many vector stores can be attached to an agent and to a conversation?
8. Why limit the number of tools on an agent?

<details>
<summary>Answers</summary>

1. Both: MCP, web search, Azure AI Search, code interpreter, file search, OpenAPI, A2A, browser automation, Fabric IQ, Work IQ. Direct only: function calling, Grounding with Bing, computer use, image generation, SharePoint, Fabric data agent, Azure Functions.
2. File search for small per-agent document sets; Azure AI Search tool for an existing governed index; Foundry IQ for multi-source, permission-aware knowledge shared across agents.
3. `simple`, `vector`, `semantic`, `vector_simple_hybrid`, `vector_semantic_hybrid` (default).
4. Active up to one hour, 30-minute idle timeout; separate sessions per concurrent conversation; charged per session on top of tokens.
5. Browser automation is GA; computer use is Preview.
6. Data leaves the Azure compliance and geo boundary; the DPA doesn't apply; private endpoints aren't respected (plus extra cost).
7. One per agent and one per conversation.
8. More tools add tokens and reduce tool-selection accuracy; use focused tool sets or a toolbox with tool search.

</details>

## References

- [What is Toolbox in Foundry? (supported tools)](https://learn.microsoft.com/azure/foundry/agents/concepts/toolbox-overview)
- [File search tool](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/file-search)
- [Vector stores for file search](https://learn.microsoft.com/azure/foundry/agents/concepts/vector-stores)
- [Azure AI Search tool](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/ai-search)
- [Code interpreter tool](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/code-interpreter)
- [Browser automation tool](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/browser-automation)
- [Computer use tool (Preview)](https://learn.microsoft.com/azure/foundry/agents/how-to/tools/computer-use)
- [Tool best practices](https://learn.microsoft.com/azure/foundry/agents/concepts/tool-best-practice)
