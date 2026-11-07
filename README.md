# Dynatrace → Jira: automatic incident from problems

A Dynatrace Automations workflow that creates a Jira incident when Dynatrace Intelligence detects a problem, including a short AI-generated summary. After creating the ticket, it comments on the Dynatrace problem with the Jira ticket ID and link. If the problem recurs, it comments on the existing ticket instead of creating a duplicate.

## Files

| File | Contents |
|---|---|
| `workflow-jira-es.json` | Spanish version (texts, AI summary and ticket description in Spanish). |
| `workflow-jira-en.json` | English version (same logic, everything in English). |

Both are clean templates: no credentials, and no connection IDs, projects, users or tenant URLs. The trigger ships **inactive**.

## What it does

```
Dynatrace Intelligence problem (CREATED / UPDATED / REOPENED)
        │
        ├─► search_existing_incident   JQL by problem ID (summary or label)
        └─► incident_summary        2-sentence AI summary (optional)
                    │
        ┌───────────┴────────────┐
   no ticket found?         ticket found?
        │                        │
 create_jira_incident    comment_on_existing_incident
        │
 annotate_problem_with_jira   (comment on the problem with ticket ID + link)
```

1. **Trigger:** active Dynatrace Intelligence problems (availability, error, slowdown, resource contention, custom alert and monitoring unavailable), excluding duplicates.
2. **Search:** looks in Jira for a not-done ticket whose summary or label contains the problem ID (for example `P-26115`).
3. **AI summary:** Dynatrace Assist summarizes the problem in at most 2 sentences. It keeps host, service and ID names exactly as received and highlights key items in bold using Jira wiki syntax. The language disclaimer that Assist appends at the end is stripped.
   - If this task fails, the ticket is still created, just without the summary block.
4. **Create ticket:** if no open ticket exists, creates the incident with the summary, problem details, a link to the problem in Dynatrace, impact, description and metadata. It adds the labels `<problem ID>` and `dynatrace`.
5. **Comment:** if a ticket already exists, adds a recurrence comment with the summary and the link.
6. **Comment on the Dynatrace problem:** right after the ticket is created, the problem in Dynatrace is annotated with the Jira ticket **ID** (for example `ABC-123`) and a clickable **link** to it, so anyone viewing the problem can jump straight to the ticket.

## How to use it

1. Make sure you have a Jira connection in Dynatrace (**Settings → Connections → Jira**) with permission to search and create issues.
2. Deploy the template (pick your language). You can either upload the JSON file directly in the Dynatrace UI (**Workflows → Upload**) or use `dtctl`:

   ```bash
   dtctl apply -f workflow-jira-en.json
   # or
   dtctl apply -f workflow-jira-es.json
   ```

3. In the created workflow, fill in:
   - **Jira connection** in `search_existing_incident`, `comment_on_existing_incident` and `create_jira_incident`.
   - **Project** and **issue type** in `create_jira_incident`. Optional: priority, assignee and reporter.
4. Test with a manual run using a real problem and check the ticket.
5. Activate the trigger.

## Notes

- The search does not filter by project. If you need that, add `project = "KEY" AND` to the JQL in `search_existing_incident`.
- Summary timestamps are requested in UTC-5. Change the time zone in the `incident_summary` instruction if needed.
- The `incident_summary` prompt must stay short: with a long prompt, Dynatrace Assist replied that it was not a valid question. Formatting rules belong in the instruction field.
- The problem link is built from `environment().url`, so it works on any tenant.
- Task names in this README are the English ones. The Spanish version keeps Spanish names: `buscar_incidente_existente`, `resumen_del_incidente`, `comentar_incidente_existente`, `crear_incidente_jira` and `anotar_problema_con_jira`.
