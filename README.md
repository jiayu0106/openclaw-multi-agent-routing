# multi-agent-routing (OpenClaw Skill)

This skill documents **Multi-Agent Routing** for OpenClaw:

- **DMs** → handled by the **main** agent (shared session)
- **Group chats** → each group/guild routes to a **dedicated agent** (isolated workspace, optional custom model)

It’s intended for setups where you want strict isolation between group conversations (different memory, tools, and even different models).

## Supported channels (v1)

| Channel | Group ID format | Binding key |
| --- | --- | --- |
| Feishu | `oc_xxx` (`chat_id`) | `match.peer.kind: "group"`, `match.peer.id` |
| Discord | Guild ID (numeric string) | `match.guildId` |

## Where to put this skill

OpenClaw loads skills from (highest precedence first):

1. `<agent-workspace>/skills/<skill-name>`
2. `~/.openclaw/skills/<skill-name>`
3. Bundled skills

To use this as a workspace-only skill:

```bash
mkdir -p ~/.openclaw/workspace/skills
cp -R ./skills/multi-agent-routing ~/.openclaw/workspace/skills/multi-agent-routing
```

Then restart the gateway (or refresh skills) and verify:

```bash
openclaw skills list
openclaw skills info multi-agent-routing
openclaw skills check
```

## What Pattern B changes in `openclaw.json`

Pattern B requires **three** coordinated changes:

1. **`agents.list`** — define per-group agents (id/workspace/model)
2. **`bindings`** — bind each group/guild to an agent
3. **`channels.<channel>`** — allow the group to reach the bot (groupPolicy / allowlists)

## Quickstart checklist

1. **Pick an `agentId`** (unique, semantic)
2. **Pick an isolated `workspace` path** for that agent
3. **(Optional) Pick a custom model** that already exists in `models.providers`
4. **Add a `bindings[]` entry** for the target group/guild
5. **Ensure the channel access control allows the group**
6. Restart gateway and verify routing in logs

## Examples

### Feishu: bind one group to a dedicated agent

```json5
{
  agents: {
    defaults: { workspace: "/root/.openclaw/workspace" },
    list: [
      { id: "main", default: true },
      { id: "feishu-engineering", workspace: "/root/.openclaw/workspace-feishu-engineering" }
    ]
  },
  bindings: [
    {
      agentId: "feishu-engineering",
      match: { channel: "feishu", peer: { kind: "group", id: "oc_xxx" } }
    }
  ],
  channels: {
    feishu: {
      enabled: true,
      groupPolicy: "allowlist",
      groupAllowFrom: ["oc_xxx"]
    }
  }
}
```

### Discord: bind one guild to a dedicated agent

```json5
{
  agents: {
    list: [
      { id: "main", default: true },
      { id: "discord-community", workspace: "/root/.openclaw/workspace-discord-community" }
    ]
  },
  bindings: [
    { agentId: "discord-community", match: { channel: "discord", guildId: "123456789012345678" } }
  ]
}
```

## Common pitfalls (important)

- **Routing is not access control**:
  - `bindings` only decides *which agent* handles a message.
  - The message can still be blocked earlier by channel policies like `channels.feishu.groupPolicy`.

- **Feishu `groupAllowFrom` must contain group IDs (`oc_...`), not user IDs (`ou_...`)**:
  - `groupAllowFrom` controls **which groups are allowed at all**.
  - If you want to restrict *who* inside a group can trigger the bot, use:
    - `channels.feishu.groups.<chat_id>.allowFrom: ["ou_xxx", ...]`

- **Mention gating**:
  - Many channels default to “respond only when @mentioned”. If the bot is silent in groups, check `requireMention` (global or per-group).
