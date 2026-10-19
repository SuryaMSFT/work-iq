# Copilot Connectors

WorkIQ gives agents access to more than 400 Copilot Connectors for enterprise
systems such as ServiceNow KB, Jira Cloud, Confluence, and intranet sites. Any
Connector that is already connected in a Copilot surface is available through
WorkIQ with no extra setup:

- Connected sources take part in `ask` and in `retrieve` with `strategy: "copilot"`.
- Their operations are available through entity tools, discovered with
  `search_paths` and `get_schema`.

The same routing rules apply whether data lives in Microsoft 365 or in a
Connector: what matters is what the task needs. This reference adds
Connector-specific behavior to the [quick guide](../SKILL.md). General
confirmation, denial, and [recovery](troubleshooting.md) rules still apply.

## Common Connectors

The Connector catalog includes the Connectors below, but isn't limited to them;
any of the more than 400 Connectors can be connected. Which Connectors are
connected depends on the tenant; being listed here doesn't mean a Connector is
connected.

| Domain | Connectors | Typical content |
| --- | --- | --- |
| Files and knowledge | Google Drive, intranet sites, ServiceNow KB, Notion | Documents, internal and line-of-business sites, knowledge articles, wiki pages |
| Communication, meetings, and people | Slack, Zoom, Workday | Channel messages and threads, meetings and recordings, worker profiles, org, and time off |
| Developer tools | Jira Cloud, GitHub, Confluence Cloud, Azure DevOps | Issues, work items, pull requests, repositories, design and wiki pages |
| Financial services | LSEG, Moody's, S&P Global | Market data, research, credit ratings, and risk analysis |
| CRM and customer support | Salesforce, HubSpot, Intercom | Accounts, opportunities, deals, contacts, activity, and support conversations |
| Legal | iManage, NetDocuments, LexisNexis, Harvey | Matter documents, contracts, case law, and legal research |
| Accounting | Xero, Aiwyn Tax, SAP S/4HANA | Ledgers, invoices, payments, tax returns, and ERP records |
| Healthcare and life sciences | PubMed, ClinicalTrials.gov | Biomedical literature and clinical trial records |
| Small business | Shopify, Intuit QuickBooks, Stripe | Products, orders, inventory, invoices, payments, and subscriptions |

## Choose the API

| What the task needs | API |
| --- | --- |
| The user explicitly asks Copilot or a named agent for an answer | `ask`; see [delegation](ask-work-iq.md) |
| Semantic understanding: evidence, summaries, comparisons, investigations, themes, "why", or "related to", across sources or inside one named system | `retrieve` with `strategy: "copilot"`; name the systems in the query |
| Full records, every record matching exact field filters, counts, or a write in one connected system | Entity tools: `search_paths`, `get_schema`, then `call_function` for reads or `do_action` for writes |
| Semantic or cross-source context first, then full records or a write | `retrieve`, then entity tools on the cited records |

**Naming a system doesn't mean entity tools.** Entity tools match structured
fields such as status, assignee, dates, and IDs; they don't understand meaning.
When the user names a system but the request depends on semantics, such as
"Jira issues about login failures" or "Salesforce deals where the customer is
worried about pricing", start with `retrieve` using `copilot` and name the system
in the query. Retrieval finds records by meaning and returns citations such as
record keys and URLs.

**No system named.** Requests such as "my work items", "tickets assigned to me",
or "what's on my plate" look like field filters, but the items can sit in several
connected systems (Jira, Azure DevOps, GitHub, ServiceNow, and others). Start with
`retrieve` using `copilot` and no `capabilities` filter. Don't call `search_paths`
across Connectors to guess a system, and don't ask the user to pick one before
retrieving. This is a concrete connected-source need. Ordinary Microsoft 365
questions with no named source still use `grounding` as described in
[retrieval](retrieve-work-iq.md).

**Retrieve, then entity tools.** Use `retrieve` to gather cross-source or
semantically matched evidence, then entity tools to inspect the surfaced records
in depth or act on them. Iterate as new questions come up.

| Scenario | `retrieve` | Entity tools |
| --- | --- | --- |
| Pipeline review | CRM notes, email, and support history | Inspect at-risk deals and update next steps |
| Escalation investigation | Slack discussions, support conversations, and KB articles | Create a Jira issue with the findings |
| Ticket triage | Related incidents and known fixes | Update the ServiceNow ticket with the resolution |
| Themed search in one named system | Jira issues about SSO login failures | Fetch full issues, comments, linked bugs, and sibling issues in the same epic |

## Frame retrieve and ask queries

- **The user named the sources:** name the Connectors in the query, for example
  "Investigate the Contoso escalation using Intercom support conversations,
  Slack discussions, and Outlook email."
- **No source named:** append `Check for evidence across all connected sources.`
  to the query, for example "What is blocking the Contoso renewal? Check for
  evidence across all connected sources."

Naming a Connector guides retrieval but does not restrict results to it.

### Gather evidence across named connected systems

```json
{
  "query": ["Contoso escalation evidence from Intercom support conversations, Slack discussions, and Outlook email"],
  "strategy": "copilot"
}
```

### Gather work items across connected systems

```json
{
  "query": ["Work items and tickets assigned to me that are open or due this week. Check for evidence across all connected sources."],
  "strategy": "copilot"
}
```

## Retrieve strategy: `copilot` for Connectors

Set `strategy: "copilot"` whenever retrieval must include connected systems. The
`grounding` strategy searches only the Microsoft 365 index and does not return
Connector content. Don't try to work out which Connectors `grounding` might
cover. If the user requires Grounding only and also needs Connector content,
explain the conflict and ask which requirement to change.

## Go deeper with entity tools

Citations identify records by key, ID, or URL. When the task needs more than the
excerpts, such as full fields, comments, history, linked records, or every record
that shares a field value with a cited one, look the record up with the system's
read operation through `call_function`, using the cited key. Base any write on the
ID that lookup returns, never on a citation URL or excerpt.

## Link to source records

Always give the user the URL of every Connector record the answer relies on,
such as the Jira issue, ServiceNow incident, Confluence page, or Salesforce
opportunity. This applies to `retrieve` results, entity-tool reads, and
confirmed writes (link the created or updated record).

- Use the URL from the `retrieve` citation or from the record's URL field in
  the entity-tool response.
- Put the link next to the fact it supports, with the record's title or key as
  the link text.
- If no URL is returned, give the record key or ID instead. Never construct or
  guess a URL.

## Empty retrieve results

An empty result proves neither absence nor a broader-source need. For a Connector
task, one bounded fallback applies when `retrieve` succeeds but returns nothing
or only clearly irrelevant results:

1. Rephrase `retrieve` once with synonyms, broader terms, or the system named
   explicitly. Keep `strategy: "copilot"`.
2. If that is still empty and the user named a connected system, or the request
   clearly implies one (for example "Jira issues about SSO login failures"), run
   one targeted search in that system: `search_paths`, `get_schema`, then
   `call_function` with the system's keyword, text, or JQL/SOQL-style search and
   the relevant field filters (project, account, date range, status).
3. Fetch the full records you need and reason over them yourself.
4. Tell the user that retrieval returned nothing and that the answer comes from
   a direct search in that system, and state the searched scope.

Do not sweep several systems or broad collections. If no system is named or
clearly implied, report the searched scope as not found. The fallback does not
apply to errors, timeouts, unavailable retrieval, or denials; follow
[recovery](troubleshooting.md) for those and do not route around them.

## Connector operations through entity tools

Connector reads use `call_function`. Connector writes (create, update,
transition, delete) use `do_action`. Never use `create_entity`, `update_entity`,
or `delete_entity` for Connectors.

1. **Discover with [search_paths](search-paths-work-iq.md).** Connector paths are
   not in the known-path table, so always discover them. Search for the task, for
   example "search Salesforce opportunities", "Jira issues", or "ServiceNow
   incident update". Results include tool names, descriptions, and supported
   operations; choose the one that fits the task.
2. **Inspect the input with [get_schema](get-schema-work-iq.md).** Use the chosen
   path with `operationType: "fetch"` for reads or `"action"` for writes, then
   build a valid request with every required parameter.
3. **Run it.** Call [call_function](call-function-work-iq.md) for reads or
   [do_action](do-action-work-iq.md) for writes.
4. **Handle sign-in.** If the Connector needs sign-in, the tool shows a URL-mode
   elicitation; the user opens the URL and signs in, and the agent continues. If
   the host doesn't support elicitations, the tool returns the sign-in URL
   instead: give the user the link, ask them to finish signing in, and then retry
   the operation once. A sign-in request is not a denial. If sign-in fails or
   access is denied, stop and report it.
5. **Confirm writes.** Writes require explicit user confirmation through a
   form-based elicitation. Before any `do_action` call, summarize exactly what
   will change: system, record, fields, and new values. Continue only after the
   user confirms, execute once, and report the observed outcome.

## Routing examples

| Prompt | Connectors | API |
| --- | --- | --- |
| "Ask Copilot what changed on the Contoso renewal this week, considering CRM updates and customer conversations." | Salesforce, Slack | `ask` |
| "Ask the Sales Insights agent which Q4 opportunities are at risk and why." | HubSpot, Intercom | `ask` with the named agent |
| "Investigate recurring authentication complaints from enterprise customers over the past month." | Intercom, Jira Cloud, Slack | `retrieve` (`copilot`) |
| "Summarize what our internal documentation and previous incidents say about VPN certificate failures." | ServiceNow KB, Notion | `retrieve` (`copilot`) |
| "Compare what our internal research and market intelligence say about semiconductor demand." | Google Drive, LSEG, S&P Global | `retrieve` (`copilot`) |
| "Summarize the main themes in ServiceNow incidents about VPN issues this month." | ServiceNow | `retrieve` (`copilot`), named system with a semantic need |
| "Show work items assigned to me." | Not named; all connected systems | `retrieve` (`copilot`) |
| "What tickets and tasks are on my plate this week?" | Not named; all connected systems | `retrieve` (`copilot`) |
| "List all open P1 bugs in Jira project PLATFORM, grouped by assignee and status." | Jira Cloud | Entity tools |
| "Get the complete activity history and contacts of the Fabrikam renewal opportunity." | HubSpot | Entity tools |
| "Move incident INC0048217 to In Progress and assign it to Network Operations." | ServiceNow | Entity tools, confirmed `do_action` |
| "List every unpaid invoice older than 60 days with customer, balance, and due date." | Xero | Entity tools |
| "Find Jira issues related to SSO login failures and dig into the root cause." | Jira Cloud | `retrieve`, then entity tools |
| "Prepare the Q4 pipeline review, identify deals at risk from customer conversations, and update next steps." | HubSpot, Intercom, Slack | `retrieve`, then entity tools |
| "Investigate recurring customer login failures and create a Jira issue with supporting evidence." | Intercom, ServiceNow KB, Jira Cloud | `retrieve`, then entity tools |

## Common mistakes

- Using `ask` when the user didn't explicitly ask Copilot or a named agent.
- Using `retrieve` with `strategy: "grounding"` for Connector content.
- Assuming that naming a Connector in the query limits results to that source.
- Jumping to entity tools because a system was named when the request needs
  semantic understanding. Use `retrieve` first, then entity tools on the cited
  records.
- Probing Connectors with `search_paths`, or asking the user to pick a system,
  when no system was named. Retrieve first.
- Using `retrieve` when the user needs a complete list, a count, or a full record.
- Stopping at excerpts when the task needs full details, comments, history, or
  related records.
- Answering from Connector records without linking to them, or guessing a
  record URL that wasn't returned.
- Reporting "nothing found" after one empty result without the rephrase and, for
  a named or clearly implied system, the targeted direct search.
- Calling `call_function` or `do_action` without first using `search_paths` and
  `get_schema`.
- Using `create_entity`, `update_entity`, or `delete_entity` for a Connector write.
- Writing to a Connector without explicit user confirmation.
