# Azure Maps and Notification Hubs

## Why it matters

Two specialised services appear in many applications: **maps and location** (store finders, delivery tracking, field service, logistics, geofencing) and **push notifications** to mobile devices. Azure Maps and Azure Notification Hubs provide them as managed APIs. Neither is central to AI, but both show up in AI solutions: agents that answer "where is my delivery?", route planning tools for field technicians, or push notifications triggered by an agent's actions.

## Key concepts

### Azure Maps

| Capability | Examples |
|---|---|
| **Render** | Map tiles (road, satellite, hybrid), static images, Web SDK and mobile SDKs |
| **Search** | Geocoding (address → coordinates), reverse geocoding, points of interest, autocomplete |
| **Route** | Directions for car, truck (vehicle constraints), pedestrian; route matrices; reachable range (isochrones); traffic-aware ETAs |
| **Traffic** | Flow and incidents |
| **Weather** | Current conditions, forecasts, alerts |
| **Geolocation** | IP to country |
| **Spatial** | Geofencing, closest point, great-circle distance (some spatial APIs are being consolidated; check status) |
| **Timezone** | Time zone by coordinates |
| Data residency | Geographic scope settings (for example EU) for data processing |

Status notes:

- **Bing Maps for Enterprise** is retiring; Microsoft directs customers to Azure Maps. Plan migrations if you find Bing Maps keys in existing applications.
- **Azure Maps Creator** (indoor maps) has been retired.
- Azure Maps uses a **Gen2 pricing tier** (pay per transaction with a free monthly allowance); older Gen1 tiers are retired.

**Authentication**: Microsoft Entra ID (recommended; roles such as `Azure Maps Data Reader`), shared keys (avoid), or SAS tokens for browser clients. Web apps typically get a short-lived Entra token from their back end for the Web SDK.

### Azure Maps as an agent tool

Location questions are natural agent tools:

```text
User: "Which of our depots can reach Utrecht Centraal within 30 minutes by van right now?"
Agent → tool: geocode("Utrecht Centraal") → route_matrix(depots → point, travelMode=truck, traffic=true)
      → filter ETAs ≤ 30 min → answer with names and ETAs
```

Expose a few well-described functions (geocode, route, search nearby) through MCP or OpenAPI, and keep keys out of the agent by using managed identity on the tool host.

### Azure Notification Hubs

A push notification engine that sends to iOS, Android and other platforms through their platform notification services (PNS).

```text
Your back end ──► Notification Hub ──► APNs (Apple) ──► iPhones
                                    ──► FCM v1 (Google) ──► Android devices
                                    ──► WNS (Windows), browser push, others
```

| Concept | Detail |
|---|---|
| **Installations / registrations** | Device handle (PNS token) plus tags, stored in the hub |
| **Tags** | Target segments (`user:42`, `region:emea`, `topic:orders`) and tag expressions (`user:42 && platform:ios`) |
| **Templates** | Platform-independent payloads filled in per device |
| Broadcast and targeted sends | To all, by tags, or to an installation |
| Scheduled sends | Standard tier |
| Telemetry | Per-message outcomes (Standard) |
| Tiers | Free, Basic, Standard (scale, telemetry, scheduled push, SLA); namespaces group hubs |

Platform credentials: **FCM v1** (the legacy FCM API has been shut down by Google; hubs must use FCM v1 credentials) and **APNs token-based authentication** (.p8 keys) rather than certificates where possible.

### When to use Notification Hubs

| Use Notification Hubs | Consider instead |
|---|---|
| Your own mobile apps need push at scale with tags and templates | Firebase or APNs directly for a single-platform, simple app |
| Multi-platform apps, segmentation, millions of devices | ACS SMS or email when push is not available or not opted in |
| Server-side triggers (order shipped, agent finished a task) | Web Push via a service for browser-only apps |

## How it works on Azure

### Geocode and route with Entra authentication (REST)

```bash
MAPS_CLIENT_ID=$(az maps account show -g $RG -n $MAPS --query properties.uniqueId -o tsv)
TOKEN=$(az account get-access-token --resource https://atlas.microsoft.com/ --query accessToken -o tsv)

# Geocode
curl -s "https://atlas.microsoft.com/geocode?api-version=2025-01-01&query=Utrecht%20Centraal" \
  -H "Authorization: Bearer $TOKEN" -H "x-ms-client-id: $MAPS_CLIENT_ID" | jq '.features[0].geometry.coordinates'

# Route (lat,lon:lat,lon)
curl -s "https://atlas.microsoft.com/route/directions/json?api-version=1.0&query=52.0894,5.1101:52.3676,4.9041&travelMode=truck&traffic=true" \
  -H "Authorization: Bearer $TOKEN" -H "x-ms-client-id: $MAPS_CLIENT_ID" | jq '.routes[0].summary'
```

API versions differ by service and change over time; check the REST reference.

### Sending a tagged push notification (Python REST outline)

```python
import requests
# Notification Hubs uses SAS tokens generated from an access policy; keep the key server-side (in Key Vault)
headers = {"Authorization": sas_token, "Content-Type": "application/json;charset=utf-8",
           "ServiceBusNotification-Format": "fcmV1", "ServiceBusNotification-Tags": "user:42"}
payload = {"message": {"notification": {"title": "Order shipped", "body": "Your order ORD-1042 is on its way."}}}
requests.post(f"https://{namespace}.servicebus.windows.net/{hub}/messages/?api-version=2015-01", json=payload, headers=headers)
```

SDKs for .NET, Java and JavaScript wrap this; templates let you send one message for all platforms.

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| Azure Maps vs other map providers | Azure Maps for Azure-integrated billing, Entra auth and data residency options; other providers when specific coverage or features are required |
| Maps authentication | Entra for servers; short-lived tokens for browsers; avoid embedding keys |
| Location data | Treat precise location as personal data: minimise, aggregate, set retention |
| Notification Hubs tier | Standard for production (telemetry, SLA, scale) |
| Notification content | Avoid sensitive data in push payloads (they appear on lock screens); send a prompt to open the app instead |
| Device targeting | Tags per user and segment; templates for multi-platform |

## Hands-on

1. Create an Azure Maps account (Gen2), assign yourself `Azure Maps Data Reader`, and run the geocode and route commands above.
2. Wrap geocode and route into two HTTP functions (file 03) with managed identity, then expose them as MCP tools (file 05) and ask an agent for travel times between two addresses.
3. Read the Notification Hubs FCM v1 migration guide and list what an existing app must change if it still used legacy FCM credentials.

```bash
az maps account create -g $RG -n maps-lab-$RANDOM --sku G2 --kind Gen2 --accept-tos
```

## Security, cost and reliability notes

- **Security:** Entra auth for Maps; disable local (key) auth where possible. Store Notification Hubs access keys in Key Vault; use listen-only policies on devices and send policies only on the back end.
- **Security:** location and device tokens are personal data; protect and expire them.
- **Cost:** Maps is per transaction; cache geocoding results for repeated addresses. Notification Hubs is per tier and push volume.
- **Reliability:** push delivery depends on Apple and Google; it is best-effort. Do not rely on push for critical messages without a fallback channel.

## Common mistakes

- Embedding an Azure Maps subscription key in a public web page.
- Leaving applications on Bing Maps for Enterprise without a migration plan.
- Using legacy FCM credentials that no longer work.
- Sensitive details in push notification text.
- Treating push notifications as guaranteed delivery.

## Check yourself

1. Name four Azure Maps capabilities.
2. How should a browser application authenticate to Azure Maps?
3. What happened to Bing Maps for Enterprise and Azure Maps Creator?
4. How do tags work in Notification Hubs?
5. Which FCM API must Notification Hubs use now?
6. Why should push notifications not carry sensitive content?

<details>
<summary>Answers</summary>

1. Any four of: map rendering, search and geocoding, routing and route matrices, traffic, weather, geolocation, time zones, spatial operations such as geofencing.
2. With short-lived Microsoft Entra tokens (or SAS tokens) obtained from its own back end, rather than an embedded key.
3. Bing Maps for Enterprise is retiring with customers directed to Azure Maps; Azure Maps Creator has been retired.
4. Devices register with tags (user, segment, topic); sends target tags or tag expressions to reach matching devices.
5. FCM v1; the legacy FCM API was shut down.
6. Notifications can appear on lock screens and pass through third-party platforms; sensitive content should stay in the authenticated app.

</details>

## References

- [Azure Maps overview](https://learn.microsoft.com/azure/azure-maps/about-azure-maps)
- [Authentication with Azure Maps](https://learn.microsoft.com/azure/azure-maps/azure-maps-authentication)
- [Azure Maps pricing tier (Gen2)](https://learn.microsoft.com/azure/azure-maps/understanding-azure-maps-transactions)
- [Migrate from Bing Maps to Azure Maps](https://learn.microsoft.com/azure/azure-maps/migrate-bing-maps-overview)
- [Azure Notification Hubs overview](https://learn.microsoft.com/azure/notification-hubs/notification-hubs-push-notification-overview)
- [Migrate to FCM v1](https://learn.microsoft.com/azure/notification-hubs/notification-hubs-gcm-to-fcm)
- [Routing and tag expressions](https://learn.microsoft.com/azure/notification-hubs/notification-hubs-tags-segment-push-message)
