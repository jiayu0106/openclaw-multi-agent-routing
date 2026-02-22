# multi-agent-routing (Skill)

Configure OpenClaw **multi-agent routing** as **Pattern B**:
- **DMs / private chats** → handled by the main agent (usually `main`)
- **Groups / guilds / teams** → each group is bound to a dedicated agent (isolated workspace, optional custom model)

> For the full guide and more examples, see: `SKILL.md`

## When to use

- You want **per-group isolation** (separate context and workspace per team/project)
- One bot serves many groups, but you want them **not to affect each other**
- You need deterministic routing in channels like Feishu / Discord / Telegram groups

## Supported channels (v1)

| Channel | Group ID format | Binding match key |
| --- | --- | --- |
| Feishu | `oc_xxx` (chat_id) | `peer.kind: "group"` + `peer.id` |
| Telegram | group chat id (stringified number, often `-100...`) | `peer.kind: "group"` + `peer.id` |
| Telegram (forum topic) | `<groupChatId>:topic:<topicId>` (e.g. `-100...:topic:99`) | `peer.kind: "group"` + `peer.id` (topic-aware) |
| Discord | guild ID (numeric string) | `guildId` |

## What you need to change in `openclaw.json` (3 places)

Pattern B requires updates in **three sections**:

1. **`agents.list`**: define agents (`id`, `workspace`, optional `model`)
2. **`bindings`** (top-level array): bind a group/guild to an agent
3. **`channels.<channel>`**: allow the group in (Feishu `groupPolicy/groupAllowFrom`, Telegram `channels.telegram.groups`, Discord `guilds`/policy)

## Quickstart (recommended flow)

1. **Confirm these 5 inputs** (to avoid silent routing misses)
   - Agent ID (recommended: `<channel>-<team>-<purpose>`)
   - Workspace path (recommended: `/root/.openclaw/workspace-<channel>-<team>`)
   - Model: use default, or pick an existing model already configured under `models.providers`
   - Group/Guild ID to bind (Feishu `oc_...` / Discord guild id / Telegram chat id)
   - Group access policy: allowlist (recommended) or open

2. **Back up `openclaw.json`** (strongly recommended)

```bash
cp ~/.openclaw/openclaw.json ~/.openclaw/openclaw.json.bak
```

3. Apply config in order: `agents.list` → `bindings` → `channels.*`
4. Validate JSON (a single missing comma can prevent the Gateway from starting)
5. Restart the Gateway to load the new config
6. Verify routing: DMs go to `main`, group messages go to the bound agent

## Minimal example (Feishu group → dedicated agent)

> This snippet uses placeholders. Replace `chat_id`, paths, and agent IDs with your real values.

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
  },
  "bindings": [
    {
      "agentId": "feishu-engineering-team",
      "match": {
        "channel": "feishu",
        "peer": { "kind": "group", "id": "oc_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx" }
      }
    }
  ],
  "channels": {
    "feishu": {
      "enabled": true,
      "groupPolicy": "allowlist",
      "groupAllowFrom": ["oc_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"]
    }
  }
}
```

## Common pitfalls

- **`bindings[].agentId` does not match any `agents.list[].id`**: the binding will never hit
- **`model.primary` references a non-existent provider/model**: the Gateway may fail to start, or the agent won’t work
  - Only reference models that already exist in `models.providers` as `<provider>/<modelId>`
- **You forgot to allowlist the group in the channel config**: messages will be rejected (especially common on Feishu/Telegram)

## How to get IDs

- **Feishu `chat_id` (`oc_...`)**: add the bot to the group → send a message → find `chatId` in gateway logs
- **Discord guild ID**: enable Developer Mode → right-click server → Copy Server ID
- **Telegram chat id**: read `chat.id` from incoming update/logs (often `-100...`)

## Further reading

- `SKILL.md`: full guide, more channel examples, routing priority, and checklist
