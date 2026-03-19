# Commune × n8n Integration: Research & Implementation Plan

> **Objective**: Ship Commune as a first-class n8n community node so automation builders can send and receive email, search conversations, and extract structured data from inbound emails — without writing code.

---

## Table of Contents

1. [What Makes n8n Integrations Succeed](#what-makes-n8n-integrations-succeed)
2. [Commune API Surface Map](#commune-api-surface-map)
3. [Node Architecture Design](#node-architecture-design)
4. [File Structure & Package Layout](#file-structure--package-layout)
5. [Node 1: Commune (Action Node)](#node-1-commune-action-node)
6. [Node 2: Commune Trigger (Webhook Node)](#node-2-commune-trigger-webhook-node)
7. [Credentials](#credentials)
8. [Implementation Code Reference](#implementation-code-reference)
9. [Publishing & Verification](#publishing--verification)
10. [Make.com Integration Path](#makecom-integration-path)
11. [Prioritised Roadmap](#prioritised-roadmap)

---

## What Makes n8n Integrations Succeed

Before designing anything, it's worth understanding what separates the 10-star nodes from the forgotten ones. Research across the n8n community and official n8n partner integrations reveals five consistent patterns.

### 1. Covers the "obvious" operations completely

Users drop a node into their workflow expecting it to do the thing that the product is known for. If they open the node and the operation they need isn't there, they close it and never come back. For Commune, the obvious operations are: **send an email**, **receive an email**, **search for an email**, and **get structured data out**. Every other feature is a bonus.

Successful nodes like the HubSpot node, Slack node, and GitHub node all lead with the 4–6 operations that cover 80% of user needs, and add advanced operations progressively.

### 2. Webhook triggers are the highest-retention feature

Nodes that only have action operations are useful but replaceable. Nodes that register a **webhook trigger** — so a workflow fires automatically when something happens — are sticky. Users build entire automation pipelines around them and can't easily switch away.

Discord, Stripe, GitHub, and Linear all have n8n webhook triggers and they are consistently the most-installed community nodes. Commune's inbound email webhook is the strongest candidate for this.

### 3. Credentials are frictionless

The node that forces users to dig through their dashboard for three different tokens will be rated 2 stars regardless of how many features it has. Successful credentials implementations require a single API key, show a clear label ("API Key" not "Bearer Token"), mark the field as `password` type so it's masked, and ideally include a "Test Credentials" button that pings a cheap endpoint like `GET /v1/inboxes`.

### 4. Output data matches what the next node expects

A node that returns a flat JSON object with human-readable keys (`thread_id`, `subject`, `from_address`) lets users immediately wire it to an If node, a Set node, or a Slack message without any data mapping gymnastics. Nodes that return nested or unpredictably structured responses cause friction. The Commune node should flatten and normalise output from every operation.

### 5. Clear descriptions + inline help = self-serve adoption

The best community nodes get adopted without documentation being read. Every field has a description. The operation selector is ordered by frequency-of-use. Rare or advanced fields are tucked into an "Additional Options" collection that is collapsed by default. When users hover a field they understand what to put in it.

### Key takeaways for Commune

- **Two nodes**: an Action node for sends/reads/search, and a Trigger node for receiving
- **One credential**: just the API key
- **Webhook trigger first**: this is Commune's biggest differentiator
- **Flatten output**: all responses normalised to a clean top-level structure
- **Inline copy**: every field has a 1-line description, every operation has a sentence

---

## Commune API Surface Map

The following table maps every Commune API endpoint to the corresponding n8n node operation. Priorities are based on impact and how commonly each feature appears in automation use-cases.

| Resource | Operation | HTTP | Endpoint | Priority | Node |
|---|---|---|---|---|---|
| **Inboxes** | Create Inbox | `POST` | `/v1/inboxes` | P0 | Action |
| **Inboxes** | List Inboxes | `GET` | `/v1/inboxes` | P0 | Action |
| **Inboxes** | Get Inbox | `GET` | `/v1/domains/:d/inboxes/:i` | P1 | Action |
| **Inboxes** | Update Inbox | `PUT` | `/v1/domains/:d/inboxes/:i` | P1 | Action |
| **Inboxes** | Delete Inbox | `DELETE` | `/v1/domains/:d/inboxes/:i` | P2 | Action |
| **Inboxes** | Set Webhook | `PUT` | `/v1/domains/:d/inboxes/:i` | P1 | Action |
| **Inboxes** | Set Extraction Schema | `PUT` | `/v1/domains/:d/inboxes/:i/extraction-schema` | P1 | Action |
| **Messages** | Send Email | `POST` | `/v1/messages/send` | P0 | Action |
| **Messages** | List Messages | `GET` | `/v1/messages` | P1 | Action |
| **Threads** | List Threads | `GET` | `/v1/threads` | P0 | Action |
| **Threads** | Get Thread Messages | `GET` | `/v1/threads/:id/messages` | P0 | Action |
| **Threads** | Update Thread Status | `PUT` | `/v1/threads/:id/status` | P1 | Action |
| **Threads** | Add/Remove Tags | `POST/DELETE` | `/v1/threads/:id/tags` | P2 | Action |
| **Threads** | Assign Thread | `PUT` | `/v1/threads/:id/assign` | P2 | Action |
| **Search** | Search Threads | `GET` | `/v1/search/threads` | P0 | Action |
| **Delivery** | Get Metrics | `GET` | `/v1/delivery/metrics` | P1 | Action |
| **Domains** | List Domains | `GET` | `/v1/domains` | P1 | Action |
| **Webhooks** | Receive Email | Inbound push | n8n webhook URL → Commune | P0 | Trigger |

---

## Node Architecture Design

### Two-node strategy

```
n8n-nodes-commune/
├── nodes/
│   ├── Commune/           ← Action node (Resource + Operation pattern)
│   └── CommuneTrigger/    ← Webhook trigger (registers webhook with Commune)
└── credentials/
    └── CommuneApi/        ← Single API key credential
```

**Why two nodes instead of one?** In n8n, trigger nodes and action nodes have fundamentally different runtime behaviour. Trigger nodes must implement `webhookMethods` (`create`, `delete`, `checkExists`) and run in a persistent "webhook listening" mode. Mixing this into a single node creates complexity and confuses users who are looking at the "Operations" dropdown. Separation also makes the nodes discoverable independently in the node panel.

### Resource / Operation structure

The Action node uses the industry-standard **Resource → Operation** two-level selector pattern, which every popular node follows:

```
Resource: [Inbox | Message | Thread | Search | Delivery]
  └── Operation: [specific action]
```

This is consistent with how Slack (`resource: channel`, `operation: send`), HubSpot, GitHub, and every other well-adopted n8n node works. Users already know this pattern.

### Webhook trigger architecture

The Trigger node:

1. **On workflow activate** — calls `create()`, which `PUT`s the n8n webhook URL onto the chosen Commune inbox via `PUT /v1/domains/:domainId/inboxes/:inboxId`
2. **When an email arrives** — Commune pushes the payload to the n8n webhook URL; n8n calls `trigger()`, which emits the normalised event data into the workflow
3. **On workflow deactivate** — calls `delete()`, which removes the webhook from the inbox by setting `webhook.endpoint` to `null`

This is the same pattern used by Stripe, GitHub, and Discord trigger nodes.

---

## File Structure & Package Layout

```
n8n-nodes-commune/
├── package.json
├── tsconfig.json
├── README.md
├── nodes/
│   ├── Commune/
│   │   ├── Commune.node.ts          # Main action node
│   │   ├── Commune.node.json        # Codex metadata
│   │   ├── CommuneDescription.ts   # All INodeProperties definitions
│   │   └── commune.svg             # Node icon
│   └── CommuneTrigger/
│       ├── CommuneTrigger.node.ts   # Webhook trigger node
│       ├── CommuneTrigger.node.json
│       └── commune.svg             # Same icon
└── credentials/
    └── CommuneApi.credentials.ts   # API key credential
```

### package.json

```json
{
  "name": "n8n-nodes-commune",
  "version": "0.1.0",
  "description": "n8n community node for Commune — email infrastructure for AI agents",
  "keywords": ["n8n-community-node-package", "email", "ai-agents", "commune"],
  "license": "MIT",
  "homepage": "https://commune.email",
  "repository": {
    "type": "git",
    "url": "https://github.com/commune-email/n8n-nodes-commune"
  },
  "main": "index.js",
  "scripts": {
    "build": "tsc && gulp build:icons",
    "dev": "tsc --watch",
    "format": "prettier nodes credentials --write",
    "lint": "eslint nodes credentials package.json"
  },
  "n8n": {
    "n8nNodesApiVersion": 1,
    "credentials": [
      "dist/credentials/CommuneApi.credentials.js"
    ],
    "nodes": [
      "dist/nodes/Commune/Commune.node.js",
      "dist/nodes/CommuneTrigger/CommuneTrigger.node.js"
    ]
  },
  "devDependencies": {
    "@n8n/node-cli": "*",
    "n8n-workflow": "*",
    "typescript": "^5.0.0"
  }
}
```

---

## Node 1: Commune (Action Node)

### Credentials

```typescript
// CommuneApi.credentials.ts
import { ICredentialType, NodePropertyTypes } from 'n8n-workflow';

export class CommuneApi implements ICredentialType {
  name = 'communeApi';
  displayName = 'Commune API';
  documentationUrl = 'https://commune.email/docs/authentication';
  properties = [
    {
      displayName: 'API Key',
      name: 'apiKey',
      type: 'string' as NodePropertyTypes,
      typeOptions: { password: true },
      required: true,
      default: '',
      description:
        'Your Commune API key. Find it in your dashboard under Settings → API Keys.',
      placeholder: 'comm_...',
    },
  ];
  authenticate = {
    type: 'generic' as const,
    properties: {
      headers: {
        Authorization: '=Bearer {{$credentials.apiKey}}',
      },
    },
  };
  test = {
    request: {
      baseURL: 'https://api.commune.email',
      url: '/v1/inboxes',
    },
  };
}
```

### Codex file

```json
// Commune.node.json
{
  "node": "n8n-nodes-commune.commune",
  "nodeVersion": "1.0",
  "codexVersion": "1.0",
  "categories": ["Communication"],
  "subcategories": {
    "Communication": ["Email"]
  },
  "resources": {
    "credentialDocumentation": [
      {
        "url": "https://commune.email/docs/authentication"
      }
    ],
    "primaryDocumentation": [
      {
        "url": "https://commune.email/docs"
      }
    ]
  },
  "alias": ["email", "agent email", "commune", "inbox", "ai email"]
}
```

### Node definition (Commune.node.ts)

Below is the full TypeScript implementation. The programmatic style is used because some operations (e.g. thread tags) require two successive API calls, and the flexible error-handling model is preferable over declarative for that.

```typescript
import {
  IExecuteFunctions,
  INodeExecutionData,
  INodeType,
  INodeTypeDescription,
  NodeApiError,
  NodeOperationError,
} from 'n8n-workflow';

const BASE_URL = 'https://api.commune.email/v1';

export class Commune implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'Commune',
    name: 'commune',
    icon: 'file:commune.svg',
    group: ['communication'],
    version: 1,
    subtitle: '={{$parameter["resource"] + ": " + $parameter["operation"]}}',
    description: 'Send and receive emails, manage inboxes, and search conversations with Commune',
    defaults: { name: 'Commune' },
    inputs: ['main'],
    outputs: ['main'],
    credentials: [{ name: 'communeApi', required: true }],
    properties: [
      // ──────────────────────────────────────────────────────────────────────
      // RESOURCE selector
      // ──────────────────────────────────────────────────────────────────────
      {
        displayName: 'Resource',
        name: 'resource',
        type: 'options',
        noDataExpression: true,
        options: [
          { name: 'Inbox',    value: 'inbox'    },
          { name: 'Message',  value: 'message'  },
          { name: 'Thread',   value: 'thread'   },
          { name: 'Search',   value: 'search'   },
          { name: 'Delivery', value: 'delivery' },
        ],
        default: 'message',
      },

      // ──────────────────────────────────────────────────────────────────────
      // MESSAGE operations
      // ──────────────────────────────────────────────────────────────────────
      {
        displayName: 'Operation',
        name: 'operation',
        type: 'options',
        noDataExpression: true,
        displayOptions: { show: { resource: ['message'] } },
        options: [
          {
            name: 'Send',
            value: 'send',
            description: 'Send an email from a Commune inbox',
            action: 'Send an email',
          },
          {
            name: 'List',
            value: 'list',
            description: 'List messages in an inbox or thread',
            action: 'List messages',
          },
        ],
        default: 'send',
      },

      // Send email fields
      {
        displayName: 'To',
        name: 'to',
        type: 'string',
        required: true,
        default: '',
        displayOptions: { show: { resource: ['message'], operation: ['send'] } },
        description: 'Recipient email address. Separate multiple addresses with commas.',
        placeholder: 'customer@example.com',
      },
      {
        displayName: 'Subject',
        name: 'subject',
        type: 'string',
        required: true,
        default: '',
        displayOptions: { show: { resource: ['message'], operation: ['send'] } },
        description: 'Email subject line',
      },
      {
        displayName: 'Email Body',
        name: 'html',
        type: 'string',
        typeOptions: { rows: 5 },
        default: '',
        displayOptions: { show: { resource: ['message'], operation: ['send'] } },
        description: 'HTML email body. Provide either this or Plain Text Body (or both).',
        placeholder: '<p>Hello!</p>',
      },
      {
        displayName: 'Plain Text Body',
        name: 'text',
        type: 'string',
        typeOptions: { rows: 3 },
        default: '',
        displayOptions: { show: { resource: ['message'], operation: ['send'] } },
        description: 'Plain text fallback. Recommended alongside the HTML body.',
      },
      {
        displayName: 'Additional Options',
        name: 'sendOptions',
        type: 'collection',
        placeholder: 'Add Option',
        default: {},
        displayOptions: { show: { resource: ['message'], operation: ['send'] } },
        options: [
          {
            displayName: 'Inbox ID',
            name: 'inboxId',
            type: 'string',
            default: '',
            description: 'Send from a specific inbox. Leave blank to use the account default.',
          },
          {
            displayName: 'Thread ID',
            name: 'thread_id',
            type: 'string',
            default: '',
            description: 'Reply within an existing conversation thread.',
          },
          {
            displayName: 'CC',
            name: 'cc',
            type: 'string',
            default: '',
            description: 'CC addresses (comma-separated)',
          },
          {
            displayName: 'BCC',
            name: 'bcc',
            type: 'string',
            default: '',
            description: 'BCC addresses (comma-separated)',
          },
          {
            displayName: 'Reply-To',
            name: 'reply_to',
            type: 'string',
            default: '',
            description: 'Override the reply-to address',
          },
          {
            displayName: 'From Name',
            name: 'from',
            type: 'string',
            default: '',
            description: 'Custom sender name or address',
          },
        ],
      },

      // ──────────────────────────────────────────────────────────────────────
      // INBOX operations
      // ──────────────────────────────────────────────────────────────────────
      {
        displayName: 'Operation',
        name: 'operation',
        type: 'options',
        noDataExpression: true,
        displayOptions: { show: { resource: ['inbox'] } },
        options: [
          { name: 'Create',          value: 'create',         action: 'Create an inbox'                  },
          { name: 'List',            value: 'list',           action: 'List all inboxes'                  },
          { name: 'Get',             value: 'get',            action: 'Get inbox details'                 },
          { name: 'Update',          value: 'update',         action: 'Update an inbox'                   },
          { name: 'Delete',          value: 'delete',         action: 'Delete an inbox'                   },
          { name: 'Set Webhook',     value: 'setWebhook',     action: 'Set the webhook on an inbox'       },
          { name: 'Set Extraction Schema', value: 'setSchema', action: 'Configure structured data extraction' },
        ],
        default: 'create',
      },
      // Create inbox fields
      {
        displayName: 'Local Part',
        name: 'localPart',
        type: 'string',
        required: true,
        default: '',
        displayOptions: { show: { resource: ['inbox'], operation: ['create'] } },
        description: 'The part before @ in the email address (e.g. "support" → support@yourdomain.com)',
        placeholder: 'support',
      },
      {
        displayName: 'Inbox Options',
        name: 'inboxCreateOptions',
        type: 'collection',
        placeholder: 'Add Option',
        default: {},
        displayOptions: { show: { resource: ['inbox'], operation: ['create'] } },
        options: [
          {
            displayName: 'Domain ID',
            name: 'domainId',
            type: 'string',
            default: '',
            description: 'The domain to create the inbox on. Auto-resolved if left blank.',
          },
          {
            displayName: 'Display Name',
            name: 'displayName',
            type: 'string',
            default: '',
            description: 'Sender name shown in email clients (e.g. "Acme Support")',
          },
          {
            displayName: 'Agent Name',
            name: 'agentName',
            type: 'string',
            default: '',
            description: 'Internal name for this agent inbox',
          },
          {
            displayName: 'Webhook URL',
            name: 'webhookEndpoint',
            type: 'string',
            default: '',
            description: 'URL to notify when emails arrive. Use the Commune Trigger node for n8n-native webhook handling.',
          },
        ],
      },
      // Get / Update / Delete: require domainId + inboxId
      {
        displayName: 'Domain ID',
        name: 'domainId',
        type: 'string',
        required: true,
        default: '',
        displayOptions: { show: { resource: ['inbox'], operation: ['get', 'update', 'delete', 'setWebhook', 'setSchema'] } },
        description: 'The ID of the domain the inbox belongs to',
      },
      {
        displayName: 'Inbox ID',
        name: 'inboxId',
        type: 'string',
        required: true,
        default: '',
        displayOptions: { show: { resource: ['inbox'], operation: ['get', 'update', 'delete', 'setWebhook', 'setSchema'] } },
        description: 'The ID of the inbox to act on',
      },
      // Set Webhook fields
      {
        displayName: 'Webhook Endpoint',
        name: 'webhookEndpoint',
        type: 'string',
        required: true,
        default: '',
        displayOptions: { show: { resource: ['inbox'], operation: ['setWebhook'] } },
        description: 'The URL that Commune will POST inbound email events to',
        placeholder: 'https://your-server.com/webhook/email',
      },
      // Set Extraction Schema fields
      {
        displayName: 'Schema JSON',
        name: 'schemaJson',
        type: 'json',
        required: true,
        default: '{"type":"object","properties":{"intent":{"type":"string"},"summary":{"type":"string"}}}',
        displayOptions: { show: { resource: ['inbox'], operation: ['setSchema'] } },
        description: 'A JSON Schema object defining what to extract from inbound emails',
      },
      {
        displayName: 'Schema Name',
        name: 'schemaName',
        type: 'string',
        default: 'extraction',
        displayOptions: { show: { resource: ['inbox'], operation: ['setSchema'] } },
      },

      // ──────────────────────────────────────────────────────────────────────
      // THREAD operations
      // ──────────────────────────────────────────────────────────────────────
      {
        displayName: 'Operation',
        name: 'operation',
        type: 'options',
        noDataExpression: true,
        displayOptions: { show: { resource: ['thread'] } },
        options: [
          { name: 'List',            value: 'list',          action: 'List threads in an inbox'           },
          { name: 'Get Messages',    value: 'getMessages',   action: 'Get messages in a thread'           },
          { name: 'Update Status',   value: 'updateStatus',  action: 'Update the status of a thread'      },
        ],
        default: 'list',
      },
      {
        displayName: 'Inbox ID',
        name: 'inboxId',
        type: 'string',
        required: true,
        default: '',
        displayOptions: { show: { resource: ['thread'], operation: ['list'] } },
        description: 'Filter threads by inbox',
      },
      {
        displayName: 'Thread ID',
        name: 'threadId',
        type: 'string',
        required: true,
        default: '',
        displayOptions: { show: { resource: ['thread'], operation: ['getMessages', 'updateStatus'] } },
        description: 'The thread to read or update',
      },
      {
        displayName: 'Status',
        name: 'status',
        type: 'options',
        options: [
          { name: 'Open',     value: 'open'     },
          { name: 'Closed',   value: 'closed'   },
          { name: 'Archived', value: 'archived' },
        ],
        default: 'open',
        displayOptions: { show: { resource: ['thread'], operation: ['updateStatus'] } },
        description: 'New status to set on the thread',
      },
      {
        displayName: 'Limit',
        name: 'limit',
        type: 'number',
        default: 20,
        displayOptions: { show: { resource: ['thread'], operation: ['list'] } },
        description: 'Maximum number of threads to return (1–100)',
      },

      // ──────────────────────────────────────────────────────────────────────
      // SEARCH operations
      // ──────────────────────────────────────────────────────────────────────
      {
        displayName: 'Operation',
        name: 'operation',
        type: 'options',
        noDataExpression: true,
        displayOptions: { show: { resource: ['search'] } },
        options: [
          {
            name: 'Search Threads',
            value: 'searchThreads',
            description: 'Semantic or keyword search across email threads',
            action: 'Search threads',
          },
        ],
        default: 'searchThreads',
      },
      {
        displayName: 'Query',
        name: 'query',
        type: 'string',
        required: true,
        default: '',
        displayOptions: { show: { resource: ['search'] } },
        description: 'What to search for. Commune uses semantic (vector) search when available, falling back to keyword matching.',
        placeholder: 'angry customer about refund',
      },
      {
        displayName: 'Inbox ID',
        name: 'inboxId',
        type: 'string',
        default: '',
        displayOptions: { show: { resource: ['search'] } },
        description: 'Narrow results to a specific inbox (recommended)',
      },
      {
        displayName: 'Limit',
        name: 'limit',
        type: 'number',
        default: 10,
        displayOptions: { show: { resource: ['search'] } },
        description: 'Maximum number of results (1–100)',
      },

      // ──────────────────────────────────────────────────────────────────────
      // DELIVERY operations
      // ──────────────────────────────────────────────────────────────────────
      {
        displayName: 'Operation',
        name: 'operation',
        type: 'options',
        noDataExpression: true,
        displayOptions: { show: { resource: ['delivery'] } },
        options: [
          {
            name: 'Get Metrics',
            value: 'getMetrics',
            description: 'Get delivery, bounce, and complaint rates for an inbox',
            action: 'Get delivery metrics',
          },
        ],
        default: 'getMetrics',
      },
      {
        displayName: 'Inbox ID',
        name: 'inboxId',
        type: 'string',
        required: true,
        default: '',
        displayOptions: { show: { resource: ['delivery'] } },
        description: 'The inbox to fetch metrics for',
      },
      {
        displayName: 'Period',
        name: 'period',
        type: 'options',
        options: [
          { name: 'Last 24 Hours', value: '24h' },
          { name: 'Last 7 Days',   value: '7d'  },
          { name: 'Last 30 Days',  value: '30d' },
        ],
        default: '7d',
        displayOptions: { show: { resource: ['delivery'] } },
        description: 'Time range for the metrics',
      },
    ],
  };

  // ──────────────────────────────────────────────────────────────────────────
  // EXECUTE
  // ──────────────────────────────────────────────────────────────────────────
  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];
    const credentials = await this.getCredentials('communeApi');
    const apiKey = credentials.apiKey as string;

    const headers = {
      Authorization: `Bearer ${apiKey}`,
      'Content-Type': 'application/json',
    };

    for (let i = 0; i < items.length; i++) {
      try {
        const resource  = this.getNodeParameter('resource',  i) as string;
        const operation = this.getNodeParameter('operation', i) as string;
        let response: any;

        // ── MESSAGE ──
        if (resource === 'message' && operation === 'send') {
          const to      = this.getNodeParameter('to',      i) as string;
          const subject = this.getNodeParameter('subject', i) as string;
          const html    = this.getNodeParameter('html',    i) as string;
          const text    = this.getNodeParameter('text',    i) as string;
          const opts    = this.getNodeParameter('sendOptions', i, {}) as any;

          if (!html && !text) {
            throw new NodeOperationError(
              this.getNode(),
              'Provide at least an HTML body or Plain Text body',
              { itemIndex: i },
            );
          }

          const body: any = { to: to.split(',').map((s: string) => s.trim()), subject };
          if (html)             body.html      = html;
          if (text)             body.text      = text;
          if (opts.inboxId)     body.inboxId   = opts.inboxId;
          if (opts.thread_id)   body.thread_id = opts.thread_id;
          if (opts.cc)          body.cc        = opts.cc.split(',').map((s: string) => s.trim());
          if (opts.bcc)         body.bcc       = opts.bcc.split(',').map((s: string) => s.trim());
          if (opts.reply_to)    body.reply_to  = opts.reply_to;
          if (opts.from)        body.from      = opts.from;

          response = await this.helpers.request({
            method: 'POST',
            url: `${BASE_URL}/messages/send`,
            headers,
            body: JSON.stringify(body),
          });

        } else if (resource === 'message' && operation === 'list') {
          response = await this.helpers.request({
            method: 'GET',
            url: `${BASE_URL}/messages`,
            headers,
          });

        // ── INBOX ──
        } else if (resource === 'inbox' && operation === 'create') {
          const localPart = this.getNodeParameter('localPart', i) as string;
          const opts      = this.getNodeParameter('inboxCreateOptions', i, {}) as any;
          const body: any = { local_part: localPart };
          if (opts.domainId)        body.domain_id    = opts.domainId;
          if (opts.displayName)     body.display_name = opts.displayName;
          if (opts.agentName)       body.name         = opts.agentName;
          if (opts.webhookEndpoint) body.webhook      = { endpoint: opts.webhookEndpoint, events: ['inbound'] };

          response = await this.helpers.request({
            method: 'POST',
            url: `${BASE_URL}/inboxes`,
            headers,
            body: JSON.stringify(body),
          });

        } else if (resource === 'inbox' && operation === 'list') {
          response = await this.helpers.request({
            method: 'GET',
            url: `${BASE_URL}/inboxes`,
            headers,
          });

        } else if (resource === 'inbox' && operation === 'get') {
          const domainId = this.getNodeParameter('domainId', i) as string;
          const inboxId  = this.getNodeParameter('inboxId',  i) as string;
          response = await this.helpers.request({
            method: 'GET',
            url: `${BASE_URL}/domains/${domainId}/inboxes/${inboxId}`,
            headers,
          });

        } else if (resource === 'inbox' && operation === 'delete') {
          const domainId = this.getNodeParameter('domainId', i) as string;
          const inboxId  = this.getNodeParameter('inboxId',  i) as string;
          response = await this.helpers.request({
            method: 'DELETE',
            url: `${BASE_URL}/domains/${domainId}/inboxes/${inboxId}`,
            headers,
          });

        } else if (resource === 'inbox' && operation === 'setWebhook') {
          const domainId = this.getNodeParameter('domainId',        i) as string;
          const inboxId  = this.getNodeParameter('inboxId',         i) as string;
          const endpoint = this.getNodeParameter('webhookEndpoint', i) as string;
          response = await this.helpers.request({
            method: 'PUT',
            url: `${BASE_URL}/domains/${domainId}/inboxes/${inboxId}`,
            headers,
            body: JSON.stringify({ webhook: { endpoint, events: ['inbound'] } }),
          });

        } else if (resource === 'inbox' && operation === 'setSchema') {
          const domainId   = this.getNodeParameter('domainId',   i) as string;
          const inboxId    = this.getNodeParameter('inboxId',    i) as string;
          const schemaName = this.getNodeParameter('schemaName', i) as string;
          const schemaJson = this.getNodeParameter('schemaJson', i) as string;
          response = await this.helpers.request({
            method: 'PUT',
            url: `${BASE_URL}/domains/${domainId}/inboxes/${inboxId}/extraction-schema`,
            headers,
            body: JSON.stringify({
              name: schemaName,
              enabled: true,
              schema: typeof schemaJson === 'string' ? JSON.parse(schemaJson) : schemaJson,
            }),
          });

        // ── THREAD ──
        } else if (resource === 'thread' && operation === 'list') {
          const inboxId = this.getNodeParameter('inboxId', i) as string;
          const limit   = this.getNodeParameter('limit',   i) as number;
          response = await this.helpers.request({
            method: 'GET',
            url: `${BASE_URL}/threads?inbox_id=${encodeURIComponent(inboxId)}&limit=${limit}`,
            headers,
          });

        } else if (resource === 'thread' && operation === 'getMessages') {
          const threadId = this.getNodeParameter('threadId', i) as string;
          response = await this.helpers.request({
            method: 'GET',
            url: `${BASE_URL}/threads/${threadId}/messages`,
            headers,
          });

        } else if (resource === 'thread' && operation === 'updateStatus') {
          const threadId = this.getNodeParameter('threadId', i) as string;
          const status   = this.getNodeParameter('status',   i) as string;
          response = await this.helpers.request({
            method: 'PUT',
            url: `${BASE_URL}/threads/${threadId}/status`,
            headers,
            body: JSON.stringify({ status }),
          });

        // ── SEARCH ──
        } else if (resource === 'search' && operation === 'searchThreads') {
          const query   = this.getNodeParameter('query',   i) as string;
          const inboxId = this.getNodeParameter('inboxId', i) as string;
          const limit   = this.getNodeParameter('limit',   i) as number;
          let url = `${BASE_URL}/search/threads?q=${encodeURIComponent(query)}&limit=${limit}`;
          if (inboxId) url += `&inbox_id=${encodeURIComponent(inboxId)}`;
          response = await this.helpers.request({ method: 'GET', url, headers });

        // ── DELIVERY ──
        } else if (resource === 'delivery' && operation === 'getMetrics') {
          const inboxId = this.getNodeParameter('inboxId', i) as string;
          const period  = this.getNodeParameter('period',  i) as string;
          response = await this.helpers.request({
            method: 'GET',
            url: `${BASE_URL}/delivery/metrics?inbox_id=${encodeURIComponent(inboxId)}&period=${period}`,
            headers,
          });
        }

        // Parse and flatten response
        const parsed = typeof response === 'string' ? JSON.parse(response) : response;
        const data = parsed?.data ?? parsed;

        if (Array.isArray(data)) {
          returnData.push(...data.map((d: any) => ({ json: d })));
        } else {
          returnData.push({ json: data });
        }

      } catch (error: any) {
        if (this.continueOnFail()) {
          returnData.push({ json: { error: error.message } });
          continue;
        }
        throw new NodeApiError(this.getNode(), error, { itemIndex: i });
      }
    }

    return [returnData];
  }
}
```

---

## Node 2: Commune Trigger (Webhook Node)

The Trigger node is the most valuable piece of the integration. It connects the n8n workflow to a Commune inbox, so every inbound email automatically starts an automation.

```typescript
// CommuneTrigger.node.ts
import {
  IHookFunctions,
  IWebhookFunctions,
  INodeType,
  INodeTypeDescription,
  IWebhookResponseData,
} from 'n8n-workflow';

const BASE_URL = 'https://api.commune.email/v1';

export class CommuneTrigger implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'Commune Trigger',
    name: 'communeTrigger',
    icon: 'file:commune.svg',
    group: ['trigger'],
    version: 1,
    description: 'Starts a workflow when an email arrives in a Commune inbox',
    defaults: { name: 'Commune Trigger' },
    inputs: [],
    outputs: ['main'],
    credentials: [{ name: 'communeApi', required: true }],
    webhooks: [
      {
        name: 'default',
        httpMethod: 'POST',
        responseMode: 'onReceived',
        path: 'commune-inbound',
      },
    ],
    properties: [
      {
        displayName: 'Domain ID',
        name: 'domainId',
        type: 'string',
        required: true,
        default: '',
        description: 'The domain ID that owns the inbox. Find this in your Commune dashboard.',
        placeholder: 'd_abc123',
      },
      {
        displayName: 'Inbox ID',
        name: 'inboxId',
        type: 'string',
        required: true,
        default: '',
        description: 'The inbox to listen for emails on',
        placeholder: 'inbox_xyz',
      },
      {
        displayName: 'Events',
        name: 'events',
        type: 'multiOptions',
        options: [
          {
            name: 'Inbound Email',
            value: 'inbound',
            description: 'Trigger when an email is received',
          },
        ],
        default: ['inbound'],
        description: 'Which events to listen for',
      },
      {
        displayName: 'Output',
        name: 'output',
        type: 'options',
        options: [
          {
            name: 'Full Payload',
            value: 'full',
            description: 'Return the complete webhook payload including raw email, security, and extraction data',
          },
          {
            name: 'Message Only',
            value: 'message',
            description: 'Return only the parsed message object (subject, body, sender, thread ID, extracted data)',
          },
        ],
        default: 'message',
        description: 'How much data to pass into the workflow',
      },
    ],
  };

  // Lifecycle: register webhook when workflow activates
  webhookMethods = {
    default: {
      async checkExists(this: IHookFunctions): Promise<boolean> {
        const domainId = this.getNodeParameter('domainId') as string;
        const inboxId  = this.getNodeParameter('inboxId')  as string;
        const credentials = await this.getCredentials('communeApi');
        const apiKey = credentials.apiKey as string;
        const webhookUrl = this.getNodeWebhookUrl('default');

        try {
          const response: any = await this.helpers.request({
            method: 'GET',
            url: `${BASE_URL}/domains/${domainId}/inboxes/${inboxId}`,
            headers: { Authorization: `Bearer ${apiKey}` },
          });
          const inbox = typeof response === 'string' ? JSON.parse(response) : response;
          return inbox?.data?.webhook?.endpoint === webhookUrl;
        } catch {
          return false;
        }
      },

      async create(this: IHookFunctions): Promise<boolean> {
        const domainId = this.getNodeParameter('domainId') as string;
        const inboxId  = this.getNodeParameter('inboxId')  as string;
        const events   = this.getNodeParameter('events') as string[];
        const credentials = await this.getCredentials('communeApi');
        const apiKey = credentials.apiKey as string;
        const webhookUrl = this.getNodeWebhookUrl('default');

        await this.helpers.request({
          method: 'PUT',
          url: `${BASE_URL}/domains/${domainId}/inboxes/${inboxId}`,
          headers: {
            Authorization: `Bearer ${apiKey}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            webhook: { endpoint: webhookUrl, events },
          }),
        });
        return true;
      },

      async delete(this: IHookFunctions): Promise<boolean> {
        const domainId = this.getNodeParameter('domainId') as string;
        const inboxId  = this.getNodeParameter('inboxId')  as string;
        const credentials = await this.getCredentials('communeApi');
        const apiKey = credentials.apiKey as string;

        try {
          await this.helpers.request({
            method: 'PUT',
            url: `${BASE_URL}/domains/${domainId}/inboxes/${inboxId}`,
            headers: {
              Authorization: `Bearer ${apiKey}`,
              'Content-Type': 'application/json',
            },
            body: JSON.stringify({ webhook: { endpoint: null } }),
          });
        } catch {
          // Silently fail on deregister; inbox may have been deleted
        }
        return true;
      },
    },
  };

  // Called each time Commune sends a webhook
  async webhook(this: IWebhookFunctions): Promise<IWebhookResponseData> {
    const body      = this.getBodyData() as any;
    const outputMode = this.getNodeParameter('output') as string;

    let data: any;
    if (outputMode === 'message') {
      // Flatten to the most useful fields for workflow builders
      const msg = body.message ?? {};
      const sender = (msg.participants ?? []).find((p: any) => p.role === 'sender');
      data = {
        // Core identifiers
        message_id:    msg.message_id,
        thread_id:     msg.thread_id,
        inbox_id:      body.inboxId,
        inbox_address: body.inboxAddress,
        // Content
        subject:       msg.metadata?.subject ?? '',
        body_text:     msg.content     ?? '',
        body_html:     msg.content_html ?? '',
        // Sender
        from:          sender?.identity ?? '',
        // Structured extraction
        extracted_data: body.extractedData ?? msg.metadata?.extracted_data ?? {},
        // Security
        spam_flagged:         body.security?.spam?.flagged ?? false,
        prompt_injection:     body.security?.prompt_injection?.detected ?? false,
        // Timestamps
        received_at:   msg.created_at ?? new Date().toISOString(),
        // Attachments
        has_attachments: (msg.attachments ?? []).length > 0,
        attachment_ids:  msg.attachments ?? [],
      };
    } else {
      data = body;
    }

    return {
      workflowData: [[{ json: data }]],
    };
  }
}
```

---

## Implementation Code Reference

### Normalised output schema

Every n8n node should output consistent, predictable data. Here is the normalised shape for the three most important outputs:

**Send message → output:**
```json
{
  "id": "msg_8f3a2b1c",
  "thread_id": "thread_abc123",
  "to": ["user@example.com"],
  "subject": "Order confirmation",
  "status": "sent"
}
```

**Trigger (message mode) → output:**
```json
{
  "message_id": "msg_xyz",
  "thread_id": "thread_abc",
  "inbox_id": "inbox_123",
  "inbox_address": "support@yourdomain.com",
  "subject": "Help with my order",
  "body_text": "Hi, I need help...",
  "body_html": "<p>Hi, I need help...</p>",
  "from": "customer@example.com",
  "extracted_data": { "intent": "support", "order_number": "12345" },
  "spam_flagged": false,
  "prompt_injection": false,
  "received_at": "2026-02-14T10:30:00Z",
  "has_attachments": false,
  "attachment_ids": []
}
```

**Search threads → output (one item per result):**
```json
{
  "thread_id": "thread_abc123",
  "subject": "Invoice #INV-001",
  "score": 0.89,
  "inbox_id": "inbox_abc",
  "participants": ["billing@company.com", "customer@example.com"]
}
```

---

## Publishing & Verification

### Step 1: Set up the repo

```bash
npx @n8n/node-cli new
# choose: community node
# name: commune
```

This scaffolds the correct `package.json`, `tsconfig.json`, and build setup.

### Step 2: Build and test locally

```bash
# In the n8n-nodes-commune directory:
npm run build

# Link to a local n8n instance:
npm link
cd ~/.n8n/nodes
npm link n8n-nodes-commune

# Start n8n:
n8n start
```

The Commune node appears in the n8n node panel under "Community".

### Step 3: Publish to npm

```bash
npm publish --access public
```

The package name `n8n-nodes-commune` and the keyword `n8n-community-node-package` ensure n8n can discover it.

### Step 4: Submit for n8n verification

1. Create an account at the n8n Creator Portal
2. Submit the node via the verified community nodes form
3. n8n reviews code quality, structure, and documentation
4. Once approved, the node appears in n8n's UI with a "Verified" badge

**Verification requirements:**
- No external npm runtime dependencies (only `n8n-workflow` as peer dep)
- All operations documented with descriptions
- Credentials include a `test` endpoint
- README covers authentication setup and operation examples
- Node icon is an SVG

---

## Make.com Integration Path

Make.com (formerly Integromat) uses a different model than n8n: integrations are called **Apps** and are built through a JSON-based declarative format, not TypeScript. The architecture parallels n8n but with different terminology.

### Make.com vs n8n comparison

| Concept | n8n | Make.com |
|---|---|---|
| Integration package | Community Node | Custom App |
| Authentication | `ICredentialType` | Connection definition (JSON) |
| Action | Node operation | Module (Instant action / Action) |
| Trigger | Trigger node | Instant Trigger (webhook) or Polling Trigger |
| Schema definition | TypeScript INodeProperties | Module parameters (JSON) |
| Publication | npm + Creator Portal | Make App Marketplace |

### Make.com implementation summary

A Make.com Commune App would consist of:

**Connection** — API key header (`Authorization: Bearer {{connection.apiKey}}`)

**Modules** (equivalent to n8n operations):
- **Send Email** (Action module) → `POST /v1/messages/send`
- **Create Inbox** (Action module) → `POST /v1/inboxes`
- **List Threads** (Action module) → `GET /v1/threads`
- **Search Threads** (Action module) → `GET /v1/search/threads`
- **Watch Inbound Emails** (Instant Trigger) → registers webhook on specified inbox, returns parsed email on each inbound event
- **Get Delivery Metrics** (Action module) → `GET /v1/delivery/metrics`

Make.com apps are built using their Make Developer Platform and submitted to the App Marketplace. The Instant Trigger uses the same webhook registration pattern as the n8n Trigger node — registering the Make-generated webhook URL with the Commune inbox on scenario activation.

**Recommended order**: Ship n8n first (TypeScript, more developer-friendly), then build Make.com in parallel since the API calls are identical and only the configuration format changes.

---

## Prioritised Roadmap

### Phase 1 — MVP (Week 1–2)

The minimum viable plugin that delivers real value and is worth publishing.

1. **`CommuneApi` credential** with API key + test endpoint
2. **Commune Trigger node** — webhook registration on a specified inbox, "message mode" output with flattened fields
3. **Commune Action node** — 4 operations: Send Email, List Threads, Get Thread Messages, Search Threads

**Why these four?** They cover the two most common automation patterns:
- *Inbound pipeline*: Email arrives → trigger → extract → route/respond
- *Outbound pipeline*: Some event → search for thread → reply

### Phase 2 — Full Coverage (Week 3–4)

5. Inbox: Create, List, Get, Set Webhook, Set Extraction Schema
6. Thread: Update Status
7. Delivery: Get Metrics
8. Message: full output mode on trigger (entire webhook payload)

### Phase 3 — Polish & Publish (Week 5)

9. Verify `checkExists()` prevents double-registration
10. Add `continueOnFail` handling to all operations
11. Test credentials button works
12. Write README with example workflows and screenshots
13. Submit to npm with `n8n-community-node-package` keyword
14. Submit to n8n Creator Portal for verification badge

### Phase 4 — Make.com (Week 6–8)

15. Replicate operations as Make.com modules using the Make Developer Platform
16. Submit to Make App Marketplace

---

## Example Workflows to Document

These are the three workflows that will appear in the README and on the Commune website, demonstrating real-world value:

**1. Customer Support Auto-Triage**
```
Commune Trigger (inbound email)
  → Commune (Search Threads: has customer emailed before?)
  → OpenAI (classify intent from extracted_data)
  → If (high urgency?)
     → Slack (alert support team)
  → Commune (Send: auto-acknowledge reply)
  → Commune (Update Thread Status: open)
```

**2. Sales Lead Qualification**
```
Commune Trigger (inbound email)
  → Commune (Get Thread Messages: read full context)
  → OpenAI (score lead quality)
  → HubSpot (Create/Update Contact)
  → Commune (Send: personalised follow-up)
```

**3. Inbox Health Monitor**
```
Schedule Trigger (daily 9am)
  → Commune (Get Delivery Metrics: last 24h)
  → If (bounce_rate > 5%?)
     → Slack (alert deliverability degradation)
```

---

*This document covers everything needed to build, test, publish, and grow the Commune n8n integration. The implementation code above can be dropped into the n8n-nodes-commune repository as a starting point — all API endpoints are verified against the live Commune backend (`https://api.commune.email/v1`).*
