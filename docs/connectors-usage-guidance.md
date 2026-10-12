# Using Connectors with Work IQ: when to use what, and how

Connectors let agents work with data in third-party and line-of-business
systems, such as Jira, ServiceNow, HubSpot, Slack, Intercom, and many others,
alongside Microsoft 365. Agents use them through three Work IQ capabilities:
Ask, Retrieve, and entity tools (`search_paths`, `get_schema`,
`call_function`, `do_action`, and others). The examples in this guide use a
few well-known systems, but the same guidance applies to any connected system.

## When to use what

The same rules apply whether the data lives in Microsoft 365 or in a
Connector. What matters is what the agent needs back.

**Use Ask to delegate a question to Copilot or another agent.** Use it only
when the user asks for that agent's answer, either by saying "Ask Copilot..."
or by naming a specific agent. Copilot then works across the user's Microsoft
365 content and connected systems itself. For example, it can combine Slack
threads, CRM deal notes, Outlook mail, and other sources into one answer. Ask
can be scoped to specific Connectors (see [Scoping](#scoping)). If Retrieve
isn't available, the agent can offer Ask as an option, but it uses Ask only if
the user agrees.

Ask returns that agent's answer, which the calling agent passes on as the other
agent's work. An ordinary question, summary, or investigation isn't a reason
to use Ask, even when it spans several connected systems. Use Retrieve for
those.

**Use Retrieve when the agent needs semantic understanding across sources.**
Retrieve finds content by meaning across connected systems and Microsoft 365,
such as ServiceNow KB articles, Jira issues, Intercom conversations, Slack
messages, Outlook mail, and similar content from other connected systems. It
returns compact, relevant excerpts with citations, sized so the agent can
reason over them without spending tokens on whole records. Use Retrieve for
investigations, summaries, comparisons, and catch-up, especially when the
answer is spread across systems or the agent doesn't know where it lives.
Retrieve can be scoped to specific Connectors and Microsoft 365 sources (see
[Scoping](#scoping)).

**Use entity tools when the work is specific to one connected system.**
With entity tools, the agent can search for the operations that every
connected system exposes and call them directly to create, read, update, and
delete records. It finds the right tools and their supported operations with
`search_paths`, checks the input schema for the operation with `get_schema`,
and then calls the operation (see [Entity tools](#entity-tools) for details). Entity tools provide depth and
completeness that Retrieve doesn't. Use entity tools to:
- Get full records, with all their fields, history, and related items.
- Get every record that matches field filters, or a count of them.
- Make changes, such as creating, updating, or transitioning a record.

For example, use entity tools to get a deal's complete activity history from a
CRM such as Salesforce or HubSpot, list every open P1 issue in a tracker such
as Jira, or update a ticket in a service desk such as ServiceNow.

**Use Retrieve and then entity tools when the agent must find something first
and then act on it.** For example, Retrieve finds the at-risk deals from CRM
notes, mail, and support history. Entity tools then read each full CRM record
and update its next step.

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
| "Show the full activity history of the Fabrikam deal in HubSpot." | Entity tools |
| "List all open P1 Jira issues in project TEL." | Entity tools |
| "Set the next step on the Fabrikam deal to 'Security review on Oct 20'." | Entity tools, after the user confirms |
| "Prepare the Q4 pipeline review and update next steps where they're clear." | Retrieve, then entity tools |

**Common mistakes to avoid:**
- Calling Ask to gather information the agent will reason over itself.
- Sending Retrieve results to Ask to be summarized again.
- Calling Ask automatically when Retrieve is unavailable, fails, or comes back
  empty. Offer Ask to the user as an option instead.
- Calling Retrieve or listing agents before an "Ask Copilot..." request. Call
  Ask directly.
- Treating Ask's answer as proof that something was done. Changes are made
  with entity tools.
- Using Retrieve to get complete lists or record IDs for updates.
- Making many entity tool calls to imitate a search.
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
change; use entity tools after the user confirms.

### Entity tools

Connector operations appear in `search_paths` results next to Microsoft 365
paths. To use one:
1. Find the right tools and their supported operations with `search_paths`,
   such as "HubSpot deals by close date".
2. Check the input schema for the operation with `get_schema`.
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
8. **Availability.** When will Ask scoping and Connector entity tools reach tenants
   that use third-party agent hosts?
9. **The two skills choose Ask differently.** This guide follows
   `workiq-preview`: use Retrieve by default and Ask only when the user hands
   the question off. The public `workiq` skill does the opposite and prefers
   Ask for summaries and reasoning. Should both skills follow the same rule?
   And should hosts that only display an answer, such as a Copilot bot in
   Slack, be treated as handing off by default?
