# VM and Network Monitoring

## Why it matters

Despite PaaS and containers, most enterprise estates still run thousands of virtual machines (including hybrid servers via Azure Arc), and every workload depends on networks that can silently drop, delay or misroute traffic. VM issues (disk filling, memory leaks, a stopped service, a failed patch) and network issues (an NSG change blocking a port, asymmetric routing, ExpressRoute degradation, DNS failures) are among the most common causes of incidents. Azure Monitor's VM monitoring with the **Azure Monitor Agent (AMA)** and **data collection rules**, plus **Network Watcher** tools (VNet flow logs, traffic analytics, Connection Monitor) and **Network insights**, give the visibility to detect and diagnose them.

## Key concepts

### VM monitoring data

| Data | Collected | Cost |
|---|---|---|
| **Host metrics** (CPU, network, disk I/O, availability) | Automatically by Azure | Free |
| **Activity log** (start, stop, resize, delete) | Automatically | Free |
| **Guest metrics** (memory, logical disk free space, processes) | AMA with DCRs; **OpenTelemetry metrics** recommended for new deployments, logs-based metrics (classic) for existing | Ingestion charges |
| **Guest logs** (Windows events, Syslog, IIS, text/JSON logs) | AMA with DCRs | Ingestion charges |
| **Dependencies / processes** (VM insights map) | Dependency agent (classic VM insights) | Ingestion charges |
| **Boot diagnostics, serial console** | Platform | Free/minimal |

Hybrid and other-cloud machines use **Azure Arc-enabled servers** with the Connected Machine agent and the same AMA/DCR approach.

### Azure Monitor Agent

AMA is the supported agent for guest data; the legacy Log Analytics agent (MMA/OMS) is retired, so migrate any remaining dependencies. AMA uses **DCRs** to define what to collect and where to send it, supports multiple destinations, ingestion-time transformations and Entra-authenticated, managed-identity-based ingestion. Deploy it at scale with Azure Policy initiatives, Bicep or **Essential machine management**.

### Network monitoring tools

| Tool | Purpose |
|---|---|
| **VNet flow logs** | Record IP flows through VNets (successor to NSG flow logs; capture encryption status and work across NSGs/AVNM rules) |
| **NSG flow logs** | **Retiring 30 September 2027**; no new creation; migrate to VNet flow logs |
| **Traffic analytics** | Analyses flow logs: top talkers, malicious IPs, open ports, traffic patterns |
| **Connection Monitor** | Continuous reachability and latency tests between endpoints (Azure, on-premises, internet) |
| **IP flow verify / NSG diagnostics / effective security rules** | Why is traffic allowed or denied? |
| **Next hop / effective routes** | Routing troubleshooting |
| **Packet capture** | Capture traffic on VMs for deep analysis |
| **Network insights** | Topology and health views across network resources (gateways, load balancers, firewalls, ExpressRoute, Private Link) |

## How it works on Azure

Data collection rule for Windows performance counters and errors from the System and Application logs (Bicep, simplified):

```bicep
resource dcr 'Microsoft.Insights/dataCollectionRules@2023-03-11' = {
  name: 'dcr-windows-baseline'
  location: resourceGroup().location
  properties: {
    dataSources: {
      performanceCounters: [ {
        name: 'perf'
        streams: [ 'Microsoft-Perf' ]
        samplingFrequencyInSeconds: 60
        counterSpecifiers: [ '\\Processor(_Total)\\% Processor Time', '\\Memory\\Available MBytes', '\\LogicalDisk(*)\\% Free Space' ]
      } ]
      windowsEventLogs: [ {
        name: 'events'
        streams: [ 'Microsoft-Event' ]
        xPathQueries: [ 'System!*[System[(Level=1 or Level=2)]]', 'Application!*[System[(Level=1 or Level=2)]]' ]
      } ]
    }
    destinations: { logAnalytics: [ { name: 'law', workspaceResourceId: law.id } ] }
    dataFlows: [ { streams: [ 'Microsoft-Perf', 'Microsoft-Event' ], destinations: [ 'law' ] } ]
  }
}
```

Associate the DCR with VMs through `Microsoft.Insights/dataCollectionRuleAssociations` or Azure Policy, which also installs AMA.

Low-disk query:

```kusto
Perf
| where TimeGenerated > ago(15m) and ObjectName == "LogicalDisk" and CounterName == "% Free Space" and InstanceName != "_Total"
| summarize freePct = avg(CounterValue) by Computer, InstanceName
| where freePct < 10
```

Enable VNet flow logs with traffic analytics:

```bash
az network watcher flow-log create -g NetworkWatcherRG -n fl-vnet-spoke1 -l westeurope \
  --vnet vnet-spoke1 --storage-account stflowlogsprod \
  --traffic-analytics true --workspace law-platform-prod --interval 10
```

Traffic analytics query (denied flows to a subnet):

```kusto
NTANetAnalytics
| where TimeGenerated > ago(1h) and FlowStatus == "Denied"
| summarize flows = sum(DeniedInFlows + DeniedOutFlows) by SrcIp, DestIp, DestPort
| top 20 by flows
```

## Design decisions and trade-offs

| Decision | Guidance |
|---|---|
| **Baseline DCRs** | One small baseline DCR per OS for all VMs (key counters, critical events); workload-specific DCRs on top. |
| **OpenTelemetry vs logs-based guest metrics** | OpenTelemetry metrics for new deployments; logs-based for existing classic VM insights. |
| **Event volume** | Collect errors/warnings, not all informational events; use XPath filters. |
| **Flow logs** | VNet flow logs on hubs and sensitive spokes; traffic analytics interval 10 or 60 minutes by need. |
| **Synthetic network tests** | Connection Monitor for critical paths (on-premises to Azure, app to database). |
| **Hybrid** | Arc-enable servers for consistent monitoring and policy. |

## Hands-on

1. Assign the built-in policy initiative to deploy AMA and associate a baseline DCR on test VMs.
2. Create the low-disk log alert.
3. Enable VNet flow logs with traffic analytics; review the traffic analytics dashboard.
4. Use IP flow verify and effective security rules to diagnose a blocked port.
5. Configure Connection Monitor between an on-premises (or Arc) machine and an Azure VM.

## Security, cost and reliability notes

- **Migrate off legacy agents and NSG flow logs** before retirement dates.
- **Flow logs cost** (storage plus traffic analytics ingestion); scope to where needed.
- **Event logs may include sensitive data**; filter and restrict access.
- **Agent health**: alert on missing heartbeats.

## Common mistakes

| Mistake | Fix |
|---|---|
| Relying on host metrics only (no memory/disk) | Enable guest monitoring |
| Collecting all event levels | XPath filters for errors/warnings |
| New NSG flow logs | VNet flow logs |
| No synthetic network tests | Connection Monitor |
| Monitoring Azure VMs but not on-premises | Azure Arc |

## Check yourself

1. Which VM data is free, and which requires the agent?
2. What is the Azure Monitor Agent configured with?
3. What replaces NSG flow logs, and when do they retire?
4. What does traffic analytics provide?
5. Which Network Watcher tools help diagnose a blocked connection?
6. How do you monitor on-premises servers the same way as Azure VMs?

<details>
<summary>Answers</summary>

1. Host metrics and activity logs are free; guest metrics and logs require AMA and incur ingestion charges.
2. Data collection rules (DCRs).
3. VNet flow logs; NSG flow logs retire on 30 September 2027.
4. Analysis of flow logs: top talkers, traffic patterns, malicious IPs, open ports, denied flows.
5. IP flow verify, NSG diagnostics/effective security rules, next hop, Connection Monitor, packet capture.
6. Azure Arc-enabled servers with AMA and DCRs.

</details>

## References

- [Monitor virtual machines](https://learn.microsoft.com/azure/azure-monitor/vm/vminsights-overview)
- [Azure Monitor Agent overview](https://learn.microsoft.com/azure/azure-monitor/agents/azure-monitor-agent-overview)
- [Network Watcher overview](https://learn.microsoft.com/azure/network-watcher/network-watcher-overview)
- [VNet flow logs](https://learn.microsoft.com/azure/network-watcher/vnet-flow-logs-overview)
- [NSG flow logs (retiring)](https://learn.microsoft.com/azure/network-watcher/nsg-flow-logs-overview)
- [Connection Monitor](https://learn.microsoft.com/azure/network-watcher/connection-monitor-overview)
