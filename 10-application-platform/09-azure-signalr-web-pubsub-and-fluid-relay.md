# Azure SignalR, Web PubSub and Fluid Relay

> Topic: 10 · Application Platform · Tracks: E ● A ○ C – · Status checked: 2026-10

## Why it matters

Modern applications push data to users in real time: live dashboards, notifications, collaborative editing, multiplayer experiences and, above all for AI, **streaming model output token by token**. Holding thousands of persistent connections on your own app servers is hard to scale. Azure SignalR Service and Azure Web PubSub manage those connections for you; Azure Fluid Relay supports real-time collaborative data structures. Knowing when to use each, and when plain server-sent events are enough, saves both effort and cost.

## Key concepts

### Real-time options compared

| Option | Protocol | Best for |
|---|---|---|
| **Server-sent events (SSE) from your API** | HTTP, one-way server → client | Streaming one model response to one client during a request. Simplest, no extra service |
| **Azure SignalR Service** | ASP.NET Core SignalR protocol over WebSockets (with fallbacks) | .NET applications using SignalR hubs; broadcast and groups at scale |
| **Azure Web PubSub** | Raw WebSockets with JSON or binary subprotocols; also **Socket.IO** support | Any language, publish/subscribe to groups, IoT and browser clients, Socket.IO apps |
| **Azure Fluid Relay** | Fluid Framework | Collaborative data structures (shared text, maps) with low latency merge |
| Event Grid MQTT broker | MQTT | Device messaging (topic 11 and 32) |

### Azure SignalR Service

| Feature | Detail |
|---|---|
| Modes | **Default** (your ASP.NET Core hub server holds hub logic; clients connect to the service), **Serverless** (no hub server; Azure Functions bindings or REST API send messages), Classic (legacy) |
| Scale | Units of concurrent connections and messages; Free, Standard, Premium tiers (Premium adds autoscale and zones) |
| Groups and users | Send to all, a group, a user or a connection |
| Auth | Clients get access tokens from your negotiate endpoint; Entra auth between your server and the service |
| Networking | Private endpoints |

### Azure Web PubSub

| Feature | Detail |
|---|---|
| Clients | Any WebSocket client; client SDKs; **Socket.IO** compatibility (serverless and default modes) |
| Server side | REST API and server SDKs (Python, JavaScript, .NET, Java) to send to all, groups, users or connections |
| Event handlers | Upstream webhooks to your server (or Functions) on connect, message and disconnect events |
| Subprotocol `json.webpubsub.azure.v1` | Clients can join groups and publish directly, with permissions encoded in their token |
| Tiers | Free, Standard, Premium (autoscale, zones) |
| Auth | Client access URLs with signed tokens issued by your server; Entra for server-to-service |

### Fluid Relay

Azure Fluid Relay hosts sessions for applications built on the open-source **Fluid Framework** (the technology behind Microsoft Loop components): distributed data structures that merge concurrent edits in real time. Use it for collaborative editors and whiteboards. Check its current status and roadmap before starting new projects; many teams also consider alternatives such as Web PubSub with CRDT libraries (for example Yjs).

### Streaming LLM output: choosing an approach

```text
A. One user, one request:   Browser ──HTTP POST──► API ──stream──► Foundry
                            Browser ◄──SSE (tokens)── API ◄────────┘
B. Long-running agent task: Browser ─► API: 202 Accepted (taskId), join group "task-123" in Web PubSub
                            Worker / Durable orchestration streams progress + tokens ─► Web PubSub group
                            Browser receives updates even if it reconnects or the work moves between servers
C. Many viewers:            One generation broadcast to everyone watching (live summaries, collaborative AI)
```

| Need | Use |
|---|---|
| Single response, interactive chat | SSE directly from the API (OpenAI-style `stream=true`) |
| Work outlives the HTTP request (agents, Durable orchestrations, queue workers) | Web PubSub or SignalR groups keyed by task or conversation |
| Several devices or users see the same stream | Web PubSub or SignalR broadcast to a group |
| Proxies or gateways that buffer SSE | WebSockets via Web PubSub |
| Reconnect and resume | Store progress server-side; send a replay on reconnect |

## How it works on Azure

### SSE streaming from a FastAPI endpoint

```python
from fastapi import FastAPI
from fastapi.responses import StreamingResponse
from azure.identity import DefaultAzureCredential, get_bearer_token_provider
from openai import OpenAI

tp = get_bearer_token_provider(DefaultAzureCredential(), "https://cognitiveservices.azure.com/.default")
client = OpenAI(base_url="https://<foundry-resource>.openai.azure.com/openai/v1/", api_key=tp)
app = FastAPI()

@app.post("/chat")
def chat(body: dict):
    def gen():
        stream = client.chat.completions.create(model="<deployment>", messages=body["messages"], stream=True)
        for chunk in stream:
            if chunk.choices and (delta := chunk.choices[0].delta.content):
                yield f"data: {delta}\n\n"
        yield "data: [DONE]\n\n"
    return StreamingResponse(gen(), media_type="text/event-stream")
```

Check that any gateway in the path (API Management, Application Gateway, Front Door) does not buffer the response and that idle timeouts exceed generation time.

### Streaming from a worker through Web PubSub

```python
from azure.messaging.webpubsubservice import WebPubSubServiceClient
from azure.identity import DefaultAzureCredential

wps = WebPubSubServiceClient(endpoint="https://<wps>.webpubsub.azure.com", hub="agent",
                             credential=DefaultAzureCredential())

# API: give the browser a client URL allowed to join only its task group
def client_url_for(task_id: str, user_id: str) -> str:
    token = wps.get_client_access_token(user_id=user_id, roles=[f"webpubsub.joinLeaveGroup.task-{task_id}"],
                                        minutes_to_expire=60)
    return token["url"]

# Worker: publish tokens and progress to the group
def publish(task_id: str, kind: str, text: str):
    wps.send_to_group(f"task-{task_id}", {"type": kind, "text": text}, content_type="application/json")
```

The browser connects with the URL, joins `task-<id>` and renders messages as they arrive.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| SSE vs managed real-time service | SSE for request-scoped streaming; Web PubSub or SignalR when work is asynchronous, multi-device or broadcast |
| SignalR vs Web PubSub | SignalR for ASP.NET Core SignalR apps; Web PubSub for other languages, raw WebSockets, Socket.IO and simpler pub/sub |
| Serverless mode | With Functions for event-driven notification without hub servers |
| Authorisation | Short-lived client tokens scoped to specific groups; never let clients join arbitrary groups |
| Scale and resilience | Premium tier for autoscale and zone redundancy; geo-replicas for multi-region (check availability) |

## Hands-on

1. Run the FastAPI SSE sample locally against a Foundry deployment and consume it with `curl -N`.
2. Create a Web PubSub resource (Free tier), and build a tiny page that connects with a client URL from your API and joins a group.
3. Write a worker script that publishes fake "agent progress" messages to that group every second, then real streamed tokens from a model.
4. Close and reopen the browser tab mid-stream; implement a "replay last N messages" endpoint so the client catches up.

```bash
az webpubsub create -g $RG -n wps-lab-$RANDOM -l $LOC --sku Free_F1
```

## Security, cost and reliability notes

- **Security:** issue client tokens from an authenticated endpoint; scope roles to the user's own groups; keep token lifetimes short.
- **Security:** disable access keys and use Entra between your servers and the service.
- **Security:** do not stream sensitive data to groups that others can join.
- **Cost:** billed by units (concurrent connections) and messages; token-by-token streaming creates many messages. Batch tokens into small chunks (for example every 50–100 ms) to cut message counts.
- **Reliability:** clients disconnect; design reconnect, resume and idempotent rendering. Use Premium tiers with zones for production scale.

## Common mistakes

- Adding Web PubSub for a simple single-request chat where SSE would do.
- Long synchronous HTTP requests for agent tasks instead of async work plus real-time updates.
- Gateways buffering SSE so tokens arrive all at once.
- Sending every single token as a separate message at high scale.
- Clients allowed to join any group by name.

## Check yourself

1. When is server-sent events the right choice for streaming LLM output?
2. When should you use Web PubSub or SignalR instead?
3. What is the difference between SignalR Service default and serverless modes?
4. How do you stop a client from receiving another user's stream?
5. How can you reduce message costs when streaming tokens?
6. What is Fluid Relay for?

<details>
<summary>Answers</summary>

1. When a single client receives the response to its own request while the HTTP request is open, and no gateway buffers the stream.
2. When work continues beyond the request (agents, background workers, durable orchestrations), when several clients or devices need the same stream, or when WebSockets are needed.
3. Default mode uses your ASP.NET Core hub server for hub logic; serverless mode has no hub server and uses Functions bindings or the REST API to send messages.
4. Issue short-lived client tokens that only permit joining that user's own task or conversation groups.
5. Batch tokens into chunks sent every short interval rather than one message per token.
6. Hosting real-time collaborative sessions built with the Fluid Framework's distributed data structures.

</details>

## References

- [Azure SignalR Service overview](https://learn.microsoft.com/azure/azure-signalr/signalr-overview)
- [SignalR Service modes](https://learn.microsoft.com/azure/azure-signalr/concept-service-mode)
- [Azure Web PubSub overview](https://learn.microsoft.com/azure/azure-web-pubsub/overview)
- [Web PubSub for Socket.IO](https://learn.microsoft.com/azure/azure-web-pubsub/socketio-overview)
- [Choose between SignalR Service and Web PubSub](https://learn.microsoft.com/azure/azure-web-pubsub/resource-faq)
- [Azure Fluid Relay](https://learn.microsoft.com/azure/azure-fluid-relay/overview/overview)
- [Stream chat completions (OpenAI SDK)](https://learn.microsoft.com/azure/ai-foundry/openai/how-to/streaming)
