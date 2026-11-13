# AI CRM Agent with Human Approval (n8n): n8n Workflow Code

Importable n8n workflow (n8n Cloud). Copy the JSON below into a file named `crm-agent-aiagent-cloud.json`, then in n8n use **Workflows → Import from File** (or paste it onto the canvas with Ctrl+V).

After import, connect your own credentials (no secrets are stored in this file) and follow the setup steps in the README.

````json
{
  "name": "AI CRM Agent / Autonomous sales operations assistant (AI Agent, n8n Cloud)",
  "nodes": [
    {
      "parameters": {
        "content": "## Setup (n8n Cloud)\n1. Credentials (free tiers): Google Gemini, Slack, Postgres (Supabase)\n2. Create table crm_actions in Supabase SQL Editor\n3. Slack channel: crm-approvals (change in the Slack node)\n4. Click Execute workflow: 4 mock requests (email draft, task, risky delete, big deal)\nReal client use: replace Mock Requests with a Slack Trigger or Schedule Trigger + CRM tools\nAI Agent node (Gemini) with a tool: deal_lookup",
        "height": 260,
        "width": 560
      },
      "id": "ecbd6569-46d4-497d-b7e2-0528b26b167e",
      "name": "Setup notes",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [
        -60,
        -220
      ]
    },
    {
      "parameters": {},
      "id": "a0c04eb0-7314-4506-94ae-f42027159001",
      "name": "When clicking Execute workflow",
      "type": "n8n-nodes-base.manualTrigger",
      "typeVersion": 1,
      "position": [
        0,
        200
      ]
    },
    {
      "parameters": {
        "jsCode": "return [\n  { json: { requester: 'Maya (Sales Rep)', request: 'Draft a follow-up email to Acme Corp about their stalled deal.' } },\n  { json: { requester: 'Jon (Sales Manager)', request: 'Create a task to review the Globex renewal quote before Friday.' } },\n  { json: { requester: 'Sam (Intern)', request: 'Delete all contacts for Initech from the CRM, they are no longer a client.' } },\n  { json: { requester: 'Priya (Account Exec)', request: 'Move the Umbrella Labs deal to Closed Won and notify finance.' } }\n];"
      },
      "id": "2ca6f1ee-5fd9-4a05-a916-dcf85500bbf1",
      "name": "Mock Requests",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        220,
        200
      ]
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=You are a CRM operations assistant. Read the request and decide: action_type (one of follow_up_email, create_task, update_deal, delete_records, other), target (the account or deal name), deal_value (number, 0 if none), summary (one sentence) and draft (a short draft message or task text, empty if not needed).\n\nRequester: {{ $json.requester }}\nRequest: {{ $json.request }}",
        "hasOutputParser": false,
        "options": {
          "systemMessage": "You are a CRM operations assistant. Decide what the request needs. Use the deal_lookup tool to check a deal's value and stage whenever the request is about a deal. Respond with ONLY a JSON object, no markdown, no extra text: {\"action_type\":\"follow_up_email|create_task|update_deal|delete_records|other\",\"target\":\"account or deal name\",\"deal_value\":0,\"summary\":\"one sentence\",\"draft\":\"short draft message or task text, empty if none\"}"
        }
      },
      "id": "009db895-c759-4470-9194-4cf2b4a0db8c",
      "name": "AI Agent",
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 1.7,
      "position": [
        460,
        200
      ],
      "retryOnFail": true,
      "maxTries": 3,
      "waitBetweenTries": 5000
    },
    {
      "parameters": {
        "modelName": "models/gemini-2.5-flash",
        "options": {}
      },
      "id": "83fe4a64-ced0-4860-9b8c-5b6d9b8e5342",
      "name": "Google Gemini Chat Model",
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        440,
        420
      ],
      "credentials": {
        "googlePalmApi": {
          "id": "REPLACE_GEMINI",
          "name": "Google Gemini(PaLM) Api account"
        }
      }
    },
    {
      "parameters": {
        "mode": "runOnceForEachItem",
        "jsCode": "const ctx = $('Mock Requests').item.json;\nlet r = $json.output;\nif (typeof r === 'string') {\n  try { const m = r.match(/\\{[\\s\\S]*\\}/); r = m ? JSON.parse(m[0]) : {}; } catch (e) { r = {}; }\n}\nr = r || {};\nconst type = String(r.action_type || 'other');\nconst value = Number(r.deal_value) || 0;\nconst reasons = [];\nif (type === 'delete_records') reasons.push('destructive action');\nif (type === 'update_deal' && value >= 50000) reasons.push('deal value over 50,000');\nif (type === 'other') reasons.push('unclear action');\nreturn { json: {\n  requester: ctx.requester,\n  request: ctx.request,\n  action_type: type,\n  target: r.target || '',\n  deal_value: value,\n  summary: String(r.summary || '').slice(0, 300),\n  draft: String(r.draft || '').slice(0, 600),\n  status: reasons.length ? 'pending_approval' : 'auto_executed',\n  approval_reason: reasons.join('; ')\n}};"
      },
      "id": "c23082e4-53d6-4ee9-9298-a4cf4eb5ef48",
      "name": "Apply Rules",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [
        820,
        200
      ]
    },
    {
      "parameters": {
        "conditions": {
          "options": {
            "caseSensitive": true,
            "leftValue": "",
            "typeValidation": "strict",
            "version": 2
          },
          "conditions": [
            {
              "id": "79aed0cc-8b95-4087-b731-a0dee53d5784",
              "leftValue": "={{ $json.status }}",
              "rightValue": "pending_approval",
              "operator": {
                "type": "string",
                "operation": "equals"
              }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "e2e1a4b2-1a98-48f1-92c4-bd826baa1ed7",
      "name": "Needs Approval?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2.2,
      "position": [
        1040,
        200
      ]
    },
    {
      "parameters": {
        "select": "channel",
        "channelId": {
          "__rl": true,
          "value": "crm-approvals",
          "mode": "name"
        },
        "text": "=:hourglass_flowing_sand: *Approval needed*\nRequested by: {{ $json.requester }}\nAction: {{ $json.action_type }} on {{ $json.target }}\nSummary: {{ $json.summary }}\nWhy: {{ $json.approval_reason }}",
        "otherOptions": {}
      },
      "id": "682c3f5b-e82d-4162-a601-28ebaceba3a3",
      "name": "Slack Approval Request",
      "type": "n8n-nodes-base.slack",
      "typeVersion": 2.3,
      "position": [
        1300,
        120
      ],
      "credentials": {
        "slackApi": {
          "id": "REPLACE_SLACK",
          "name": "Slack account"
        }
      },
      "onError": "continueRegularOutput"
    },
    {
      "parameters": {
        "operation": "insert",
        "schema": {
          "__rl": true,
          "value": "public",
          "mode": "list",
          "cachedResultName": "public"
        },
        "table": {
          "__rl": true,
          "value": "crm_actions",
          "mode": "name"
        },
        "options": {},
        "columns": {
          "mappingMode": "defineBelow",
          "value": {
            "requester": "={{ $json.requester }}",
            "request": "={{ $json.request }}",
            "action_type": "={{ $json.action_type }}",
            "target": "={{ $json.target }}",
            "deal_value": "={{ $json.deal_value }}",
            "status": "={{ $json.status }}",
            "approval_reason": "={{ $json.approval_reason }}",
            "summary": "={{ $json.summary }}",
            "draft": "={{ $json.draft }}"
          },
          "matchingColumns": [],
          "schema": [
            {
              "id": "requester",
              "displayName": "requester",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "request",
              "displayName": "request",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "action_type",
              "displayName": "action_type",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "target",
              "displayName": "target",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "deal_value",
              "displayName": "deal_value",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "number",
              "canBeUsedToMatch": true
            },
            {
              "id": "status",
              "displayName": "status",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "approval_reason",
              "displayName": "approval_reason",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "summary",
              "displayName": "summary",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            },
            {
              "id": "draft",
              "displayName": "draft",
              "required": false,
              "defaultMatch": false,
              "display": true,
              "type": "string",
              "canBeUsedToMatch": true
            }
          ]
        }
      },
      "id": "647bd3df-95bb-4cb3-a045-59a55838699c",
      "name": "Save Action",
      "type": "n8n-nodes-base.postgres",
      "typeVersion": 2.5,
      "position": [
        1300,
        320
      ],
      "credentials": {
        "postgres": {
          "id": "REPLACE_PG",
          "name": "Postgres account"
        }
      }
    },
    {
      "parameters": {
        "name": "deal_lookup",
        "description": "Looks up a CRM deal by account name. Input: the account name, for example 'Acme Corp'. Returns the deal stage, amount, days in stage and owner as JSON.",
        "jsCode": "const deals = {\n  'acme corp': { stage: 'Proposal', amount: 18000, days_in_stage: 21, owner: 'Maya' },\n  'globex': { stage: 'Renewal', amount: 35000, days_in_stage: 5, owner: 'Jon' },\n  'initech': { stage: 'Closed Lost', amount: 0, days_in_stage: 90, owner: 'Sam' },\n  'umbrella labs': { stage: 'Negotiation', amount: 62000, days_in_stage: 9, owner: 'Priya' }\n};\nconst key = String(query).toLowerCase().trim();\nreturn JSON.stringify(deals[key] || { error: 'deal not found' });"
      },
      "id": "60a3e01a-66aa-481e-961a-a423c7f103d2",
      "name": "Deal Lookup Tool",
      "type": "@n8n/n8n-nodes-langchain.toolCode",
      "typeVersion": 1.3,
      "position": [
        660,
        420
      ]
    }
  ],
  "connections": {
    "When clicking Execute workflow": {
      "main": [
        [
          {
            "node": "Mock Requests",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Mock Requests": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Apply Rules": {
      "main": [
        [
          {
            "node": "Needs Approval?",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Needs Approval?": {
      "main": [
        [
          {
            "node": "Slack Approval Request",
            "type": "main",
            "index": 0
          },
          {
            "node": "Save Action",
            "type": "main",
            "index": 0
          }
        ],
        [
          {
            "node": "Save Action",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Google Gemini Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "AI Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "AI Agent": {
      "main": [
        [
          {
            "node": "Apply Rules",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Deal Lookup Tool": {
      "ai_tool": [
        [
          {
            "node": "AI Agent",
            "type": "ai_tool",
            "index": 0
          }
        ]
      ]
    }
  },
  "pinData": {},
  "active": false,
  "settings": {
    "executionOrder": "v1"
  },
  "tags": []
}
````
