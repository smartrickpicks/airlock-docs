# Discord Bot Studio — Code Analysis

> **Source:** `/Users/zacharyholwerda/Downloads/BotFiles/` — exported from Discord Bot Studio
> **Purpose:** Understand the working execution engine so we can adapt the architecture for Airlock's workflow engine.

---

## Architecture Summary

Bot Studio uses a **JSON-defined workflow graph** executed by a **sequential action runner** with **variable interpolation**. The architecture is clean and portable. Here's how it maps to Airlock.

### File Structure

```
BotFiles/
  bot.js                          <- Main runtime: event listeners + action executor
  DiscordFunctions.js             <- Utility: getDefaultChannel()
  package.json                    <- discord.js v13, flatted, papaparse, winston
  Handlers/
    Message.js                    <- Action execution engine (command-triggered)
    Events.js                     <- Action execution engine (event-triggered)
    guildMember.js                <- Member event handler
  BotData/
    commands/
      commands.json               <- Command definitions (trigger → action chain)
      events.json                 <- Event definitions (trigger → action chain)
      variables.json              <- (empty)
    nodes/
      nodes.json                  <- Visual canvas node positions + connections (for the UI)
      eventnodes.json             <- Visual canvas event node positions
    variables/
      globalvars.json             <- Global variable storage (persistent)
      servervars.json             <- Per-server variable storage (persistent)
    Settings/
      Settings.json               <- Bot token, prefix, client ID
      Rules.json                  <- Anti-spam rules
    user/
      user.json                   <- User data cache
    varcache.js                   <- Variable interpolation engine
    usercache.js                  <- User data cache module
  Classes/
    List.js                       <- Array helper
```

---

## Key Architecture Patterns (What We're Taking)

### 1. Dual-File Node System (Canvas vs Execution)

Bot Studio separates **canvas representation** from **execution data**:

**`nodes.json` (Canvas — for the visual builder UI):**
```json
{
  "type": "command",
  "name": "help",
  "guid": "7942fe4c-dffa-4146-8cab-d254b3d80d5d",
  "outputs": [
    {
      "label": "responses",
      "connections": ["37eda463-a170-44f9-af54-61378ffc9294"]
    }
  ],
  "x": 148,
  "y": 188
}
```

**`commands.json` (Execution — for the runtime):**
```json
{
  "name": "help",
  "actions": [
    {
      "name": "help menu",
      "category": "Message",
      "type": "Send Embed",
      "channelname": "${dbsVars.CommandChannel.id}",
      "color": "1FFF57",
      "title": "Help Menu",
      "description": "...",
      "savetovariable": "",
      "savetovariabletype": "temp"
    }
  ]
}
```

**What this means for Airlock:** We should do the same. React Flow stores node positions, connections, and visual metadata. The execution runtime reads a flattened version of the workflow that's just trigger → actions list. The visual builder writes both formats on save.

### 2. Sequential Action Chain with `callNextAction()`

The core execution loop in `bot.js`:

```javascript
DBS.callNextAction = async function (command, message, args, index) {
  var action = command.actions[index];
  // Execute the action at this index
  DBS.MsgHandler.Message_Handle(DBS, msg, command, index, args);
  // After action completes, it calls callNextAction(command, msg, args, index + 1)
};
```

**Actions are an ordered array.** Each action executes, then calls `callNextAction(command, msg, args, index + 1)` to advance. This is a simple linked-list execution model. No parallel branches in this implementation.

**What this means for Airlock:** Our BullMQ-based executor should follow the same pattern for linear workflows. For branching (which Bot Studio doesn't support well), we add a Branch node that creates separate BullMQ jobs for each path.

### 3. Variable Interpolation Engine (`varcache.js`)

The variable system uses `%%variableName%%` syntax for simple values and `%%object[property]%%` for nested access:

```javascript
// Two regex patterns:
var regex = /%%(\w+)\[([\w\s]+)\]%%/g;   // %%user[displayName]%%
var regex1 = /%%(.*?)%%/g;                // %%newuser%%

// Replace in all action fields:
newVal = newaction[e].replace(regex, (_match, group1, group2) => vars[group1][group2]);
newVal = newVal.replace(regex1, (_match, group1) => vars[group1]);
```

**Variable scopes:**
- **Temp (per-execution):** `cache[guild.id].variables` — lives only during the current action chain
- **Server vars:** `serverVars[guild.id]` — persisted to `servervars.json`, per-Discord-server
- **Global vars:** `globalVars` — persisted to `globalvars.json`, shared across all servers

**What this means for Airlock:** Our Liquid template syntax (`{{contact.name}}`, `{{vault.deal_value}}`) is more powerful but serves the same purpose. We keep:
- **Temp scope** (per-workflow-run) → `workflow_runs.context` JSONB column
- **Vault scope** (per-vault) → vault metadata and fields
- **Workspace scope** (per-workspace) → workspace settings
- **Global scope** → system-wide configuration

### 4. Event-to-Action Routing

`bot.js` registers Discord.js event listeners and routes them through `Event_Handle()`:

```javascript
DBS.Bot.on("messageCreate", (message) => DBS.checkMessage(message));
DBS.Bot.on("guildMemberAdd", (member) => {
  DBS.EventHandler.Event_Handle(DBS, DBS.EventsFile, 0, "User Joins Server", member);
});
DBS.Bot.on("channelCreate", (channel) => {
  let channelVars = {};
  channelVars.guild = channel.guild;
  channelVars.channel = channel;
  DBS.EventHandler.Event_Handle(DBS, DBS.EventsFile, 0, "Channel Create", channelVars);
});
```

**28 event types registered** in `eventnodes.json`: User Joins, User Kicked, User Banned, Any Message, Bot Init, Channel Create/Delete/Update/Pins, Emoji CRUD, Guild CRUD, Member Available/Speaking/Update, Message Delete/Update, Role CRUD, Typing Start, User Update, Button/Select/Command Interactions.

Each event finds its matching entry in `events.json` by name, then runs the attached action chain.

**What this means for Airlock:** Our event bus (BullMQ) already follows this pattern. When an event fires (inbound_message, vault_created, stage_changed, etc.), the workflow engine finds all active workflows with matching trigger types and creates execution runs for each.

### 5. Wait/Delay as First-Class Action Type

`Message.js` has a special `Wait_Handle` that pauses the action chain:

```javascript
module.exports.Wait_Handle = async function (dbs, msg, command, index, args, action) {
  const timer = (ms) => new Promise((res) => setTimeout(res, ms));
  let waitTime = parsedAction.waitduration;
  let unit = parsedAction.unit; // seconds | minutes | hours | days
  await timer(waitTime * timeMultiplier * 1000);
  DBS.callNextAction(command, msg, args, index + 1);
};
```

This is an in-memory delay — fine for a Discord bot, but doesn't survive restarts. For waits longer than seconds, it's fragile.

**What this means for Airlock:** This is exactly why we need BullMQ's delayed jobs. A `delay(5, 'minutes')` in our workflow becomes a BullMQ job with `delay: 300000`. If the server restarts, the job is still in Redis waiting. Bot Studio's approach would lose the delay state on restart.

### 6. Action Type Switch (`RunAction` — 40+ action types)

`Message.js` has a massive switch statement routing action types:

```javascript
module.exports.RunAction = async function (client, msg, action) {
  var parsedAction = ParseActionVariables(action, msg);
  switch (action.type) {
    case "Send Message": ...
    case "Send Embed": ...
    case "Add Role to User": ...
    case "Kick User": ...
    case "Create Channel": ...
    case "Store Value in Variable": ...
    case "Check Variable Value": ...
    case "Call API": ...
    // 40+ more...
  }
};
```

**Action categories discovered (from the switch cases):**
- **Message:** Send Message, Send Embed, Send Image, Send Random Image, Edit Message, Delete Message, Delete All Messages
- **Role:** Add Role to User, Remove Role From User, Add Role to Server
- **Channel:** Create Channel, Delete Channel, Create Category, Update Channel Permissions
- **Variable:** Store Value in Variable, Edit Variable, Check Variable Value, Check If Variable Exists
- **User Data:** Set User Data, Get User Data, Edit User Data, Check User Data
- **Condition:** Check User Permissions, Check If User Has Role, Check If Message Is In Channel, Check If Array Contains Value, Check If String Contains, Switch Case
- **Bot:** Set Bot Game, Set Bot Activity, Set Bot Status, Set Avatar
- **Other:** Kick User, Ban User, Add Reaction Listener, Role Reaction Menu, Generate Random Number, Get Row, Call API, Wait

**What this means for Airlock:** We need the same pattern but cleaner — a registry of action handlers instead of a switch statement:

```typescript
// Airlock pattern
const actionHandlers: Record<string, ActionHandler> = {
  'send_message': handleSendMessage,
  'create_task': handleCreateTask,
  'route_to_pool': handleRouteToPool,
  'ai_classify': handleAIClassify,
  'conversational_ask': handleConversationalAsk,
  // ...
};
```

### 7. Conditional Branching (True/False Actions)

The `Call API` handler shows how Bot Studio handles branching:

```javascript
module.exports.CallAPI_Handle = async function (msg, client, action) {
  let passActions = {};
  const response = await fetch(action.url, { ... });
  if (response.ok) {
    passActions.actions = action.trueActions;   // ← success branch
    DBS.callNextAction(passActions, msg, msg.args, 0);
  } else {
    passActions.actions = action.falseActions;  // ← failure branch
    DBS.callNextAction(passActions, msg, msg.args, 0);
  }
};
```

The `trueActions` and `falseActions` arrays contain sub-action chains. This is how conditions create branching — each branch is just another actions array.

**What this means for Airlock:** Our Branch node should work the same way. Each branch output has its own action chain. The workflow JSON stores:

```json
{
  "type": "branch",
  "condition": "{{lead_score}} >= 80",
  "true_path": ["action_1", "action_2"],
  "false_path": ["action_3", "action_4"]
}
```

### 8. Per-Execution Variable Context

When an event fires, Bot Studio creates a scoped variable context:

```javascript
// Events.js — on event trigger
switch (type) {
  case "Channel Create":
    vararray["createdchannel"] = varsE.channel;
    break;
  case "User Joins Server":
    vararray["newuser"] = varsE.member;
    break;
  case "Any Message":
    vararray["msguser"] = varsE.member;
    break;
}
```

Each event type injects specific variables that the action chain can reference. The event definition in `events.json` declares what variables it provides:

```json
{
  "name": "User Joins Server",
  "var": { "user": "newuser" }
}
```

This means actions can use `%%newuser%%` or `%%newuser[displayName]%%` to access event data.

**What this means for Airlock:** Our trigger nodes should declare their output variables the same way:

```json
{
  "type": "trigger",
  "trigger_type": "inbound_message",
  "provides": {
    "message": "trigger_data.message",
    "sender_phone": "trigger_data.from_address",
    "smart_line": "trigger_data.to_address",
    "channel": "trigger_data.channel"
  }
}
```

---

## What Bot Studio Gets Right (Keep)

1. **JSON-as-workflow** — workflows are portable, versionable, inspectable data. Not code.
2. **Sequential action chains** — simple, debuggable, easy to reason about.
3. **Variable interpolation** — actions reference dynamic data without hardcoding.
4. **Event → action routing** — declarative mapping from events to workflows.
5. **Separation of canvas (UI) from execution (runtime)** — the visual builder and the executor read different formats.
6. **Form data per action** — each action type has specific form fields (channelname, messagetext, color, etc.). The UI renders the right form based on action type.

## What Bot Studio Gets Wrong (Fix)

1. **In-memory delays** — `await timer()` doesn't survive restarts. Use BullMQ delayed jobs.
2. **No persistent execution state** — if the bot crashes mid-workflow, state is lost. Need `workflow_runs` table.
3. **Giant switch statement** — 40+ cases in one file. Use a registry pattern with per-action handler modules.
4. **No branching in the canvas** — `trueActions`/`falseActions` exist in JSON but the visual builder doesn't show branches well. React Flow handles this natively.
5. **File-based variable persistence** — `fs.writeFileSync` on every message. Use Postgres/Redis.
6. **No execution logging** — no audit trail of which actions ran, what they produced, or why a branch was taken. We need full execution logs.
7. **No concurrency control** — multiple events can trigger the same handler simultaneously with shared mutable state (`vars`, `cache`). Race conditions.
8. **No versioning** — overwriting JSON files directly. Need draft/published versioning.

---

## Mapping: Bot Studio → Airlock Workflow Engine

| Bot Studio Concept | Bot Studio Implementation | Airlock Equivalent |
|---|---|---|
| Command trigger | `commands.json` entries | Trigger nodes (teal) |
| Event trigger | `events.json` entries | Event subscriptions on BullMQ topics |
| Action chain | `actions[]` array on command/event | Node graph edges in React Flow → flattened to action list for executor |
| Variable interpolation | `%%varname%%` regex in `varcache.js` | `{{variable.path}}` Liquid templates |
| Temp variables | `cache[guild.id].variables` | `workflow_runs.context` JSONB |
| Server variables | `serverVars` persisted to JSON file | Vault metadata / workspace settings in Postgres |
| Global variables | `globalVars` persisted to JSON file | System-wide config in Postgres |
| Action handler | Switch cases in `Message.js` RunAction() | Action handler registry with per-type modules |
| Wait/Delay | In-memory `setTimeout` promise | BullMQ delayed job |
| True/False branching | `trueActions`/`falseActions` arrays | Branch node with multiple output ports |
| Canvas positions | `nodes.json` with `x`, `y`, `guid` | React Flow node positions + connections |
| Canvas connections | `outputs[].connections[]` GUIDs | React Flow edges |
| Event context vars | Switch case setting `vararray["newuser"]` | Trigger node `provides` declaration |
| Mod system | `mods/` directory, `loadMods()`, `isEvent`/`isResponse` | Plugin system with custom action types |
| API calls | `CallAPI_Handle` with fetch | HTTP Request action node |
| Anti-spam rules | `Rules.json` → discord-anti-spam | Rate limiting on workflow execution |

---

## Implementation Priority

The Bot Studio code confirms our architecture choices:

1. **JSON workflow storage** is correct (they do it, it works)
2. **Sequential execution with branching** is correct (their `callNextAction` pattern)
3. **Variable scoping** is correct (temp/server/global maps to run/vault/workspace)
4. **Event-driven triggers** are correct (their event listener → handler pattern)
5. **BullMQ over in-memory timers** is critical (their Wait handler breaks on restart)
6. **Postgres over file system** is critical (their JSON persistence is fragile)
7. **Action handler registry** is better than their switch statement
8. **React Flow handles branching** that their visual builder struggles with

The core insight: Bot Studio proves the JSON-workflow-with-sequential-executor pattern works at scale (BotGhost has 1M+ bots deployed). Our additions — persistent state, branching visualization, AI nodes, execution logging — are evolutionary, not revolutionary.
