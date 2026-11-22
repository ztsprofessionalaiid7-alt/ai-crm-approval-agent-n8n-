# AI CRM Agent with Human Approval (n8n)

An n8n workflow where an AI Agent (Gemini 3.5 Flash Lite) reads sales-team requests, looks up deal data with a tool, plans the CRM action, and applies hard business rules. Safe actions run automatically, risky ones go to Slack for human approval.

▶ **Demo video:** https://drive.google.com/file/d/1fDYsmNdxW_0SD-Q1NOcpa9sdVdisN8XF/view

## How it works

1. Mock requests arrive (follow-up email, task, risky delete, big deal update)
2. The AI Agent classifies the request (action type, target, deal value, summary, draft) and can call the `deal_lookup` tool to check a deal's stage and amount
3. A rules step (not the AI) decides the status: deleting records, deals of 50,000 or more and unclear requests become `pending_approval`
4. An IF node sends pending actions to a Slack approval request
5. Every action is saved to Postgres with its status and approval reason

```
Manual Trigger -> Mock Requests -> AI Agent (Gemini + deal_lookup tool)
-> Apply Rules -> Needs Approval?
   yes -> Slack Approval Request -> Save Action
   no  -> Save Action
```

## AI Agent

AI Agent node with one tool (`deal_lookup`). Approval rules are deterministic code, so the AI never decides on its own whether a destructive action may run.

## Tech stack

n8n, AI Agent, Gemini 3.5 Flash Lite, CRM, Slack, Postgres

## Setup

1. Copy the JSON from `workflow.md` into a file named `crm-agent-aiagent-cloud.json` and import it into n8n
2. Create credentials (all have free tiers):
   - Google Gemini (API key)
   - Slack (Bot token and signing secret)
   - Postgres (Supabase Session pooler, SSL: Require)
3. In the Slack node set channel `crm-approvals`, then run `/invite @your-bot` in that channel
4. Create the database tables (below)
5. In the Postgres save node remove the `id` and `created_at` fields, the database fills them
6. Click **Execute workflow**: 4 mock inputs run through all branches

## Database

```sql
create table if not exists public.crm_actions (
  id bigint generated always as identity primary key,
  requester text,
  request text,
  action_type text,
  target text,
  deal_value numeric,
  status text,
  approval_reason text,
  summary text,
  draft text,
  created_at timestamptz default now()
);
```

## Demo data

| Mock input | Expected result |
|---|---|
| Maya: follow-up email to Acme Corp | Auto-executed, draft written |
| Jon: task to review Globex quote | Auto-executed |
| Sam (intern): delete all Initech contacts | Pending approval (destructive action) |
| Priya: move Umbrella Labs to Closed Won | Pending approval (deal value over 50,000) |

## Notes

- Replace Mock Requests with a Slack Trigger or Schedule Trigger and real CRM tools for client use
- The approval step is a Slack message only. Closing the loop (approve button or label) is a planned next step
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
