---
name: multi-agent-routing
description: Configure multi-agent routing (Pattern B) where the main agent handles all DMs and each group/guild/team binds to a separate agent. Use when setting up isolated workspaces per group chat for Feishu or Discord channels.
metadata: {"openclaw":{"emoji":"🔀"}}
---

# Multi-Agent Routing (Pattern B)

Configure **Pattern B** routing: main agent handles all private messages (DMs); each group chat binds to a dedicated agent with isolated workspace and optional custom model.

## Supported Channels (v1)

| Channel | Group ID Format | Binding Key |
|---------|----------------|-------------|
| Feishu  | `oc_xxx` (chat_id) | `peer.kind: "group"`, `peer.id` |
| Telegram | Group chat id (stringified number, often negative like `-100123...`) | `peer.kind: "group"`, `peer.id` |
| Telegram (forum topic) | `<groupChatId>:topic:<topicId>` (example: `-1001234567890:topic:99`) | `peer.kind: "group"`, `peer.id` (topic-aware) |
| Discord | Guild ID (numeric string) | `guildId` |

---

## ⚠️ Code of Conduct (Read Before Proceeding)

### 1. Confirm Information with User Before Making Changes

Before modifying the configuration file, **you MUST confirm the following with the user**:

| Item | Description | Example Question |
|------|-------------|------------------|
| **Agent ID** | Unique identifier for the new agent | "Please confirm the agent ID. Recommended format: `feishu-engineering-team`" |
| **Workspace Path** | Workspace path for the new agent | "Please confirm the workspace path. Recommended: `/root/.openclaw/workspace-feishu-engineering`" |
| **Model to Use** | Read available models from `models.providers` first, then let user choose | See "Model Selection" section below |
| **Group ID** | The group/guild ID to bind (Feishu: `oc_xxx`, Discord: guild ID) | "Please provide the Feishu chat_id (starts with `oc_`) or Discord guild ID to bind" |
| **Group Access Policy** | `allowlist`: only allow specified group IDs in `groupAllowFrom`; `open`: allow all groups | "Do you want to use allowlist (recommended, requires group IDs) or open (allow all groups)?" |

#### Model Selection

**IMPORTANT**: Before asking about models, **first read the user's config file** to get the list of already configured models from `models.providers`.

**Step 1**: Read the config file and extract available models:

```
Available models in your config:
1. minimax/MiniMax-M2.1 (MiniMax M2.1)
2. minimax/MiniMax-M2.1-lightning (MiniMax M2.1 Lightning)
3. minimax/MiniMax-M2 (MiniMax M2)
4. kitcoding/claude-opus-4-5-20251101 (claude-opus)
```

**Step 2**: Ask the user to choose:

> "Which model should this agent use?"
> - [ ] Use default model (inherits from `agents.defaults.model`)
> - [ ] Select from existing models: _______________

**DO NOT ask for API key or base URL** unless the user explicitly says they want to add a completely new provider.

> 💡 **Confirmation Template** (copy and use):
>
> Before starting the configuration, please confirm the following:
> 1. **Agent ID**: _______________ (recommended format: `<channel>-<team>-<purpose>`)
> 2. **Workspace Path**: _______________ (recommended format: `/root/.openclaw/workspace-<channel>-<team>`)
> 3. **Model**: [ ] Use default model / [ ] Select from existing: _______________ (list available models from config)
> 4. **Group ID to bind**: _______________ (Feishu: `oc_xxx`, Discord: guild ID)
> 5. **Group Policy**: [ ] allowlist (recommended, only allow specified group IDs) / [ ] open (allow all groups)

### 2. Backup Configuration File Before Modifying

> 🛡️ **Strongly Recommended**: Before modifying `openclaw.json`, **always backup the original config file** so you can quickly rollback if something goes wrong.

**Backup command examples**:

```bash
# Linux/macOS - with timestamp
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.backup.$(date +%Y%m%d_%H%M%S)

# Simple backup
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
```

**Verify backup exists**:

```bash
# Check backup file exists
ls -la ~/.openclaw/openclaw.json*
```

> ⚠️ **Rollback method**: If Gateway fails to start or behaves abnormally after config changes, restore from backup:
> ```bash
> cp ~/.openclaw/openclaw.json.bak ~/.openclaw/openclaw.json
> ```

### 3. Operation Flow Summary

```
1. Confirm Agent ID, Workspace, Model, Group ID with user
       ↓
2. Backup existing config file
       ↓
3. Modify config file (agents.list + bindings + channels)
       ↓
4. Validate JSON format is correct
       ↓
5. Restart Gateway to load new config
       ↓
6. Test and verify routing works correctly
```

---

## Configuration Overview

Pattern B requires changes in **three sections** of `openclaw.json`:

1. **`agents.list`** — Define agent entries (id, workspace, model)
2. **`bindings`** — Route groups/guilds to agents
3. **`channels.<channel>`** — Allow groups (groupPolicy + groupAllowFrom/guilds)

## Part 1: Configure Agents (`agents.list`)

Add agents to `agents.list[]`. Each agent can have:

| Field | Required | Description |
|-------|----------|-------------|
| `id` | ✅ | Unique agent identifier |
| `default` | ❌ | Set `true` for the fallback agent (usually `main`) |
| `workspace` | ❌ | Isolated workspace path (inherits from `agents.defaults.workspace` if omitted) |
| `model` | ❌ | Custom model config (inherits from `agents.defaults.model` if omitted) |

### Naming Convention Recommendations

> 💡 **Agent ID Naming**:
> - Format: `<channel>-<team/purpose>-<description>`
> - Examples: `feishu-engineering-team`, `discord-product-community`, `feishu-hr-recruitment`
> - Benefit: Immediately shows which channel and team/purpose this agent serves
>
> 💡 **Workspace Path Naming**:
> - Format: `<base-path>/workspace-<channel>-<team/purpose>`
> - Example: `/root/.openclaw/workspace-feishu-engineering`
> - Benefit: Clear organization in file system, easy to manage and backup

### Example: Basic two-agent setup

```json
{
  "agents": {
    "defaults": {
      "model": { "primary": "minimax/MiniMax-M2.1-lightning" },
      "workspace": "/root/.openclaw/workspace"
    },
    "list": [
      { "id": "main", "default": true },
      {
        "id": "feishu-engineering-team",
        "workspace": "/root/.openclaw/workspace-feishu-engineering"
      }
    ]
  }
}
```

### Example: Agent with custom model

> ⚠️ **Important**: When assigning a custom model to an agent, **you MUST use a provider/model that is already configured in `models.providers`**.
>
> Model reference format is `<provider>/<modelId>`, where:
> - `provider` must be an existing key in `models.providers`
> - `modelId` must be the `id` of a model in that provider's `models[]` array
>
> If you reference a non-existent provider or model, Gateway will fail to start or the agent won't work properly.

**Step 1**: First, configure the provider and model in `models.providers`:

```json
{
  "models": {
    "mode": "merge",
    "providers": {
      "kitcoding": {
        "baseUrl": "https://kitcoding.com/",
        "apiKey": "<YOUR_API_KEY>",
        "api": "anthropic-messages",
        "models": [
          {
            "id": "claude-opus-4-5-20251101",
            "name": "claude-opus",
            "reasoning": false,
            "input": ["text"],
            "cost": { "input": 0, "output": 0, "cacheRead": 0, "cacheWrite": 0 },
            "contextWindow": 200000,
            "maxTokens": 8192
          }
        ]
      }
    }
  }
}
```

**Step 2**: Then reference it in the agent using `<provider>/<modelId>` format:

```json
{
  "agents": {
    "list": [
      { "id": "main", "default": true },
      {
        "id": "feishu-engineering-team",
        "workspace": "/root/.openclaw/workspace-feishu-engineering",
        "model": { "primary": "kitcoding/claude-opus-4-5-20251101" }
      }
    ]
  }
}
```

> 💡 **Tip**: If an agent doesn't need a custom model, you can omit the `model` field and the agent will inherit from `agents.defaults.model`.

## Part 2: Configure Bindings (`bindings`)

`bindings` is a **top-level array** (same level as `agents`, `channels`, etc.).

### Feishu Group Binding

```json
{
  "bindings": [
    {
      "agentId": "feishu-engineering-team",
      "match": {
        "channel": "feishu",
        "peer": {
          "kind": "group",
          "id": "oc_7953e99214cc0c26012402796d304aaf"
        }
      }
    }
  ]
}
```

- `agentId`: Must match an agent's `id` in `agents.list[]`
- `peer.kind`: `"group"` for group chats, `"direct"` for DMs
- `peer.id`: Feishu chat_id (starts with `oc_`)

### Telegram Group Binding

```json
{
  "bindings": [
    {
      "agentId": "telegram-community",
      "match": {
        "channel": "telegram",
        "peer": { "kind": "group", "id": "-1001234567890" }
      }
    }
  ]
}
```

- `agentId`: Must match an agent's `id` in `agents.list[]`
- `peer.kind`: Use `"group"` for Telegram groups/supergroups
- `peer.id`: Telegram `chat.id` as a string (often a negative number like `-100...`)

### Telegram Forum Topic Binding (optional)

If the group is a forum (topics enabled), you can bind either:

- **Whole group**: bind base group id (`-100...`) and let topics inherit via parent binding.
- **A specific topic**: bind the topic-specific peer id (`-100...:topic:<topicId>`).

```json
{
  "bindings": [
    {
      "agentId": "telegram-topic-99",
      "match": {
        "channel": "telegram",
        "peer": { "kind": "group", "id": "-1001234567890:topic:99" }
      }
    },
    {
      "agentId": "telegram-community",
      "match": {
        "channel": "telegram",
        "peer": { "kind": "group", "id": "-1001234567890" }
      }
    }
  ]
}
```

> Tip: Parent binding inheritance is supported for Telegram forum topics, so binding the base group id is usually enough unless you want per-topic isolation.

### Discord Guild Binding

```json
{
  "bindings": [
    {
      "agentId": "discord-product-community",
      "match": {
        "channel": "discord",
        "guildId": "1234567890123456789"
      }
    }
  ]
}
```

- `agentId`: Must match an agent's `id` in `agents.list[]`
- `guildId`: Discord server ID (numeric string, 18-19 digits)

### Example: Multiple bindings

```json
{
  "bindings": [
    {
      "agentId": "feishu-engineering-team",
      "match": {
        "channel": "feishu",
        "peer": { "kind": "group", "id": "oc_7953e99214cc0c26012402796d304aaf" }
      }
    },
    {
      "agentId": "feishu-operations-team",
      "match": {
        "channel": "feishu",
        "peer": { "kind": "group", "id": "oc_8a64f88325dd1d37023503897e415bbf" }
      }
    },
    {
      "agentId": "discord-product-community",
      "match": {
        "channel": "discord",
        "guildId": "1234567890123456789"
      }
    }
  ]
}
```

## Part 3: Channel Access Control

### Feishu Group Access

```json
{
  "channels": {
    "feishu": {
      "enabled": true,
      "groupPolicy": "allowlist",
      "groupAllowFrom": [
        "oc_7953e99214cc0c26012402796d304aaf",
        "oc_8a64f88325dd1d37023503897e415bbf"
      ]
    }
  }
}
```

- `groupPolicy`: `"allowlist"` (default, recommended) | `"open"` | `"disabled"`
- `groupAllowFrom`: Array of allowed chat_ids (only when `groupPolicy: "allowlist"`)

### Telegram Group Access

Telegram uses `channels.telegram.groups` as the primary group allowlist + mention gating surface.

- If you set `channels.telegram.groups`, it becomes an **allowlist** (only listed groups, or `"*"`, are accepted).
- Use `requireMention` to control whether the bot replies only when mentioned.

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "groups": {
        "-1001234567890": { "requireMention": true }
      }
    }
  }
}
```

(Forum topics inherit their parent group config unless you add per-topic overrides under `topics`.)

### Discord Guild Access

Discord guilds are typically allowed by default. Use per-guild config if needed:

```json
{
  "channels": {
    "discord": {
      "enabled": true,
      "groupPolicy": "open",
      "guilds": {
        "1234567890123456789": {
          "requireMention": true
        }
      }
    }
  }
}
```

## Complete Example

Full Pattern B config for one Feishu group + one Discord guild:

```json
{
  "bindings": [
    {
      "agentId": "feishu-engineering-team",
      "match": {
        "channel": "feishu",
        "peer": { "kind": "group", "id": "oc_7953e99214cc0c26012402796d304aaf" }
      }
    },
    {
      "agentId": "discord-product-community",
      "match": {
        "channel": "discord",
        "guildId": "1234567890123456789"
      }
    }
  ],
  "agents": {
    "defaults": {
      "model": { "primary": "minimax/MiniMax-M2.1-lightning" },
      "workspace": "/root/.openclaw/workspace"
    },
    "list": [
      { "id": "main", "default": true },
      {
        "id": "feishu-engineering-team",
        "workspace": "/root/.openclaw/workspace-feishu-engineering"
      },
      {
        "id": "discord-product-community",
        "workspace": "/root/.openclaw/workspace-discord-product"
      }
    ]
  },
  "channels": {
    "feishu": {
      "enabled": true,
      "groupPolicy": "allowlist",
      "groupAllowFrom": ["oc_7953e99214cc0c26012402796d304aaf"]
    },
    "discord": {
      "enabled": true,
      "groupPolicy": "open"
    }
  }
}
```

## Routing Priority

When a message arrives, the router checks bindings in this order:

1. **Exact peer match** (Feishu: `peer.kind` + `peer.id`)
2. **Guild match** (Discord: `guildId`)
3. **Team match** (Slack: `teamId`)
4. **Account match** (`accountId`)
5. **Channel match** (any account on that channel)
6. **Default agent** (`agents.list[].default: true`, else first in list, fallback to `main`)

DMs without explicit binding fall back to the default agent.

## How to Get IDs

### Feishu chat_id

1. Add bot to the group
2. Send a message, check gateway logs for `chatId`
3. Or use Feishu Open Platform API: `GET /im/v1/chats`

### Discord guild ID

1. Enable Developer Mode in Discord settings
2. Right-click server name → "Copy Server ID"
3. Or check gateway logs when receiving a message

## Checklist

- [ ] **Naming**: Use semantic Agent ID (e.g., `feishu-engineering-team`) that clearly indicates channel and purpose
- [ ] **Naming**: Use clear Workspace path (e.g., `/root/.openclaw/workspace-feishu-engineering`)
- [ ] Add agents to `agents.list[]` with unique `id` and `workspace`
- [ ] **If using custom model**: First configure the provider and model in `models.providers`
- [ ] **If using custom model**: Reference model in agent's `model.primary` using `<provider>/<modelId>` format for an **already configured** model
- [ ] Add bindings to top-level `bindings[]` array, ensure `agentId` matches an `agents.list[].id`
- [ ] For Feishu: set `groupPolicy: "allowlist"` + `groupAllowFrom`
- [ ] Restart gateway to load new config
- [ ] Test: DM should go to `main`, group message should go to bound agent

> ⚠️ **Common Mistakes**:
> - Referencing a model ID in agent's `model.primary` that is not configured in `models.providers` - Gateway will fail to find the provider/model
> - `bindings[].agentId` not matching any `agents.list[].id` - routing will fail
