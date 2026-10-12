# Using Connectors with Work IQ: when to use what, and how

Connectors let agents work with data in systems such as Jira, ServiceNow,
HubSpot, Slack, and Intercom, alongside Microsoft 365. Agents use them through
three Work IQ capabilities: Ask, Retrieve, and Tools.

## When to use what

**Use Ask only when the user deliberately hands the question to Copilot or
another agent.** That happens in three cases:
- The user explicitly asks for Copilot's answer, as in "Ask Copilot...".
- The user names a particular agent.
- Retrieve can't be used, the agent explains why, and the user chooses to have
  Copilot answer instead.

Ask returns that agent's answer. The calling agent presents it as the other
agent's answer and doesn't claim to have checked it. A question, summary, or
investigation by itself isn't a reason to use Ask; use Retrieve for those.

**Use Retrieve when the agent needs information to reason with.** Retrieve
returns relevant content with citations from across the user's sources.
The agent then works out the answer itself. Use Retrieve for investigations,
summaries, comparisons, and catch-up. It's especially useful when the agent
doesn't know which system or record has the information.

**Use Tools when the agent needs exact records or needs to make a change.**
Tools read and write records directly in a specific system. Use them for a
known record, a complete or filtered list, or a create or update.

**Use Retrieve and then Tools when the agent must find something first and
then act on it.** Retrieve finds what matters, and Tools read the full records
or make the changes.

| Prompt | Use |
| --- | --- |
| "Ask Copilot what changed on the Contoso account this week." | Ask |
| "Ask the Sales Insights agent which Q4 deals are at risk." | Ask, with that agent |
| "Ask that same agent which of those deals is most urgent." | Ask, continuing the same conversation |
| "What changed on the Contoso account this week?" | Retrieve, because the user didn't ask for Copilot |
| "Investigate the Contoso escalation and prepare a response plan." | Retrieve |
| "Summarize customer feedback on telemetry from Jira, Slack, and email." | Retrieve, scoped to those sources |
| "What does our ServiceNow KB say about VPN certificate errors?" | Retrieve, scoped to ServiceNow |
| "Ask Copilot whether ServiceNow ticket SNW-4421 met its SLA." | Ask, pointed at that ticket |
| "Get HubSpot deal 18234 with its stage, contacts, and next step." | Tools |
| "List all open P1 Jira issues in project TEL." | Tools |
| "Set the next step on the Fabrikam deal to 'Security review on Oct 20'." | Tools, after the user confirms |
| "Prepare the Q4 pipeline review and update next steps where they're clear." | Retrieve, then Tools |

**Common mistakes to avoid:**
- Calling Ask to gather information the agent will reason over itself.
- Sending Retrieve results to Ask to be summarized again.
- Calling Ask automatically when Retrieve is unavailable, fails, or comes back
  empty. Offer Ask to the user as an option instead.
- Calling Retrieve or listing agents before an "Ask Copilot..." request. Call
  Ask directly.
- Treating Ask's answer as proof that something was done. Changes are made
  with Tools.
- Using Retrieve to get complete lists or record IDs for updates.
- Making many Tools calls to imitate a search.
- Switching between Ask and Retrieve on your own after an error, an empty
  result, or a sign-in prompt.

## How to use them

### Scoping

By default, don't scope. Without a scope, Ask and Retrieve use all the
sources they can reach. A topic such as "Contoso deals" isn't a reason to
scope.

Scope only in these cases:
- The user names a source, such as "check Slack and the Outlook thread".
- The task has a clear boundary, such as "support tickets only".
- The agent is filling one specific gap, such as the status of one Jira issue.

When you scope, include every source the user named, including Microsoft 365
sources such as Outlook or Teams.

To scope to a Connector, the agent needs its ID. To get it, call
`search_paths`, then call `fetch` to list the user's connected Connectors. Use
only IDs returned this way, and never guess an ID from a name. If the
Connector the user named isn't connected, tell the user.

### Retrieve

Always set `strategy` explicitly. Use `copilot` when the request needs
evidence from a Connector, including requests that also need Microsoft 365
content. Otherwise use `grounding`, which is the default for ordinary or
unspecified sources. Don't choose `copilot` just because you don't know where
the information is. Retrieve with `copilot` still returns evidence; it isn't
Ask.

To scope, list the sources in `capabilities`. Connectors go under
`GraphConnectors`, and Microsoft 365 sources use their own names, such as
`Email`.

*"Investigate the Contoso escalation using Intercom support history and
customer emails."*

```json
{
  "query": ["Contoso escalation timeline: support issues and customer communications"],
  "strategy": "copilot",
  "capabilities": [
    { "name": "GraphConnectors", "connections": [ { "connection_id": "<intercom-id>" } ] },
    { "name": "Email" }
  ]
}
```

Inside a Connector, you can narrow to a container, path, or item.

*"What does the IT-Network ServiceNow KB say about VPN certificate errors?"*

```json
{
  "query": ["VPN certificate error troubleshooting steps"],
  "strategy": "copilot",
  "capabilities": [
    { "name": "GraphConnectors", "connections": [
      { "connection_id": "<servicenow-id>", "items_by_container_name": [ { "container_name": "IT-Network" } ] }
    ] }
  ]
}
```

### Ask

Use Ask only for a request the user has handed to Copilot or a named agent.
Don't call Retrieve or `list_agents` first for an "Ask Copilot..." request.
Leave out `agentId` to use Copilot.

Scope Ask only when the user's request names sources. To limit which
Connectors Ask uses, set `dataSources.connectorIds`:
- Leave `dataSources` out to use all of the user's Connectors.
- Pass an empty list to use no Connectors.
- Pass IDs to use only those Connectors.

`dataSources` doesn't limit Microsoft 365 content. To focus on a Microsoft 365
source, name it in the question. Never send Retrieve parameters such as
`query`, `strategy`, or `capabilities` to Ask.

*"Ask Copilot whether we committed to moving the Contoso launch to Nov 15.
Have it check the internal Slack discussion and the latest Outlook thread with
the customer."*

```json
{
  "question": "Using our internal Slack discussion and the latest Outlook thread with Contoso, did we commit to moving the launch to November 15? Cite both.",
  "dataSources": { "connectorIds": ["<slack-id>"] }
}
```

To point Ask at specific items, use `resources` for Connector records or
`fileUrls` for OneDrive and SharePoint files. Don't widen a request about one
item into a broad search.

*"Ask Copilot whether ServiceNow ticket SNW-4421 was resolved within SLA."*

```json
{
  "question": "Was this ticket resolved within our SLA?",
  "resources": [ { "external": { "itemId": "SNW-4421", "contentSource": "ServiceNow" } } ]
}
```

To use a named agent, pass its exact `agentId`. Reuse an ID you already have,
or look it up with `list_agents`. Never guess an ID. If the agent can't be
found or the match is unclear, stop or ask the user to choose; don't quietly
use Copilot instead. For a follow-up to the same agent, pass the
`conversationId` from its previous answer. Don't reuse that ID with a
different agent or for an unrelated task.

*"Ask the Sales Insights agent which Q4 deals are at risk."*

```json
{ "question": "Which Q4 deals are at risk and why?", "agentId": "<id from list_agents>" }
```

Present the answer as that agent's answer and keep its citations. Don't
claim you checked the sources yourself. If Ask fails, don't switch to Retrieve
on your own. Explain the alternative and let the user decide. If the user also
asked for a change, such as updating a record, Ask's answer doesn't make the
change; use Tools after the user confirms.

### Tools

Connector operations appear in `search_paths` results next to Microsoft 365
paths. To use one:
1. Search for the operation, such as "HubSpot deals by close date".
2. Check the request body with `get_schema` if you need to.
3. Read with `call_function`.
4. Write with `do_action`, but only after the user confirms the exact change.
5. If a sign-in is required, show it to the user. When they've signed in,
   make the same call again.

*"Review Q4 strategic deals in HubSpot, find ones with no next step, and
update the next step where it's clear."* The agent finds the HubSpot search and
update operations with `search_paths`. It reads the deals with
`call_function` and proposes updates. After the user confirms the updates, it
makes them with `do_action`.

## Open questions

1. **Connector ID lookup isn't available yet.** Agents can't get Connector IDs
   through Work IQ today. Until they can, should agents skip scoping, or name
   the source in the question or query?
2. **Ask and Retrieve use different IDs.** Ask uses `connectorIds` and Retrieve
   uses `connection_id`. Can one ID work for both?
3. **Ask and Retrieve scope differently.** Retrieve can scope Microsoft 365
   sources, but Ask can scope only Connectors. Should Ask match Retrieve, like
   the source picker in Copilot Chat?
4. **Federated Connectors in Retrieve.** How are they scoped in
   `capabilities`? Does the `grounding` strategy support Connectors?
5. **Default Retrieve strategy.** The skill defaults to `grounding`, which
   doesn't include Connectors. Should the default change to `copilot` when the
   user has Connectors?
6. **Unavailable Connectors.** Ask quietly drops Connector IDs that aren't
   connected or that the user can't access. Should it tell the agent which IDs
   it dropped?
7. **Named agents.** Does Ask scoping work when the call targets a specific
   agent? Retrieve doesn't allow scoping in that case.
8. **Availability.** When will Ask scoping and Connector Tools reach tenants
   that use third-party agent hosts?
9. **The two skills choose Ask differently.** This guide follows
   `workiq-preview`: use Retrieve by default and Ask only when the user hands
   the question off. The public `workiq` skill does the opposite and prefers
   Ask for summaries and reasoning. Should both skills follow the same rule?
   And should hosts that only display an answer, such as a Copilot bot in
   Slack, be treated as handing off by default?
