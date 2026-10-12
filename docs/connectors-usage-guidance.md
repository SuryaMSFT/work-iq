# Using Connectors with Work IQ: when to use what, and how

This guide covers data in connected systems such as Jira, ServiceNow,
HubSpot, Salesforce, Slack, and Intercom. Microsoft 365 data such as mail,
calendar, and Teams follows the existing Work IQ skill guidance.

Agents reach Connector data in three ways: Ask, Retrieve, and Tools.

## When to use what

### Ask: the user wants Copilot or another agent to answer

Use Ask only when the user asks for Copilot's answer ("Ask Copilot...") or
names a specific agent. Copilot then does the work itself, combining
Connector content such as Slack threads and HubSpot notes with the user's
Microsoft 365 content. The calling agent passes the answer on as Copilot's
answer. Ask can be scoped to specific Connectors (see [Scoping](#scoping)).

Any other question goes to Retrieve or Tools, even when it spans several
systems. If those can't be used, the agent can offer Ask, but it calls Ask
only if the user agrees.

### Retrieve: find what the content says

Retrieve searches the content of connected systems by relevance. It returns
the most useful passages with citations, such as ServiceNow KB articles,
Confluence pages, Intercom conversations, Jira issue descriptions and
comments, or Slack threads. Use it when:
- **The question is about meaning, not fields.** For example, "What do our KB
  articles say about VPN certificate errors?" or "What are customers
  complaining about in support tickets?"
- **The answer may be spread across systems,** or the agent doesn't know which
  system has it.
- **The agent needs evidence to reason with,** not a complete data set.

Retrieve returns the most relevant results, not all of them, and it returns
excerpts rather than full records. Retrieve can be scoped to specific
Connectors and Microsoft 365 sources (see [Scoping](#scoping)).

### Tools: work with the system's own data and operations

A Connector's Tools are the system's own operations, such as search, query,
list, get, create, and update. They aren't only for records you already know.
Use them when:
- **The request can be described with filters on fields** such as status,
  owner, priority, stage, amount, or date. For example, "Open P1 bugs in TEL
  assigned to me" or "Deals in Negotiation closing this quarter."
- **The answer must be complete or counted.** For example, "How many tickets
  were escalated last week?" or "List every deal over $100K." Retrieve can't
  promise completeness.
- **The agent needs the full record,** including all fields, linked records,
  history, comments, or attachments. For example, "Show the full timeline of
  INC0012345."
- **The current state matters.** For example, "Is TEL-482 still blocked?"
  Tools read directly from the system.
- **The request uses the system's own terms,** such as a saved filter, a JQL
  query, a report, or a pipeline view. For example, "Run my 'Escalations this
  week' filter in Jira."
- **The user wants a change,** such as create, update, comment, assign, or
  transition. Tools make changes only after the user confirms.

Each Tools call works with one system. When a task spans several systems, the
agent combines the results itself.

### Retrieve and Tools together

Many tasks need both. The agent uses Retrieve to find what matters, then uses
Tools to get exact facts or to act. For example, for "Prepare the Q4 pipeline
review and update next steps," Retrieve finds the at-risk deals and the
reasons behind them in CRM notes, mail, and support history. Tools then read
each full HubSpot deal and update its next step after the user confirms.

### A quick test

- If a set of field filters describes what the agent needs, use Tools.
- If the agent is looking for what something says or means, use Retrieve.
- If the user asked Copilot or a named agent to answer, use Ask.

| Prompt | Use |
| --- | --- |
| "Ask Copilot what changed on the Contoso account this week." | Ask |
| "Ask the Sales Insights agent which Q4 deals are at risk." | Ask, with that agent |
| "What changed on the Contoso account this week?" | Retrieve, because the user didn't ask for Copilot |
| "What does our ServiceNow KB say about VPN certificate errors?" | Retrieve, scoped to ServiceNow |
| "Summarize customer feedback on telemetry from Jira, Slack, and Intercom." | Retrieve, scoped to those Connectors |
| "List all open P1 Jira issues in project TEL assigned to me." | Tools, using a filtered search |
| "How many HubSpot deals over $100K are closing this quarter?" | Tools, because the result must be complete |
| "Is TEL-482 still blocked, and who owns it?" | Tools, because the result must be current |
| "Show the full activity history of the Fabrikam deal." | Tools, for the full record |
| "Move INC0012345 to Resolved with a note." | Tools, after the user confirms |
| "Find the Jira bugs customers are most upset about and raise their priority." | Retrieve, then Tools |
| "Prepare the Q4 pipeline review and update next steps where they're clear." | Retrieve, then Tools |

**Common mistakes to avoid:**
- Calling Ask to gather information the agent will reason over itself.
- Sending Retrieve results to Ask to be summarized again.
- Using Retrieve for counts, complete lists, current status, or record IDs for
  updates.
- Paging through every record with Tools to find something by meaning. Use
  Retrieve instead.
- Treating Ask's answer as proof that something was done. Changes are made
  with Tools.
- Switching between Ask, Retrieve, and Tools on your own after an error, an
  empty result, or a sign-in prompt.

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
