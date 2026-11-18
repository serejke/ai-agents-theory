# Guardrail

A Guardrail is the check point that each tool call of an agent goes through,
before its tool runs and after it. At this point, rules from the developer can
let the call continue, stop it, change its result or get approval from a person.
It is the boundary between the step that the LLM wants and the step that occurs
in the [Environment](../environment/environment.md).

**Composition**: a `pre` hook and a `post` hook at the dispatch of the
[AgentLoop](agent-loop.md) + the policies at these hooks.

<a id="why-it-matters"></a>

## Value

A Guardrail makes a rule that the LLM cannot break. The LLM can ignore the other
parts of an Agent, for example the system prompt, the tools and the memory. It
can also give them a different meaning. If the system prompt tells "don't push
to main", the LLM usually obeys, but this result is not sure. A Guardrail stops
the `push --force` command before the command gets to the shell. The LLM cannot
change this result.

This difference is the difference between a first test version of an Agent and
an Agent in production. A request that the model can ignore is sufficient for a
test. An Agent in production must have rules that the model cannot break.

### Each Agent has the hooks

Each tool call of an Agent goes through the dispatch:

1. The LLM selects a tool call.
2. The harness sends the call to the tool.
3. The tool runs.
4. The result goes back to the LLM.

The Guardrail is at the dispatch. Each AgentLoop that runs tools has the
dispatch. Thus the Guardrail is a part of the harness, not an optional pattern.

The hooks are a part of the structure. The user writes the policies. A policy
can let each call continue with no change. It can also stop a call, remove data
from a result or get approval from a person.

An Agent without policies also has the hooks. The hooks run for each tool call
and let the call continue. For example, PI has `beforeToolCall` and
`afterToolCall` in the API of `pi-agent-core`, its AgentLoop harness. PI calls
them for each tool call, also if no handler is attached. Thus the words "my
Agent has no Guardrails" tell that the Agent has no policies. They do not tell
that the Agent has no check point.

<a id="formal-definition"></a>

## Definition

```typescript
type PreVerdict =
  | { allow: true }
  | { deny: string } // reason shown to the user or logged
  | { askUser: string }; // question that the user must confirm

type PostVerdict =
  | { allow: true }
  | { redact: ToolResult } // the result, replaced or cleaned
  | { block: string }; // reason; the result is discarded

type Guardrail = {
  // Pre: check BEFORE the tool runs
  pre?: (action: ToolCall) => PreVerdict;

  // Post: check AFTER the tool runs
  post?: (action: ToolCall, result: ToolResult) => PostVerdict;
};
```

A Guardrail is a layer around a [Tool](../primitives/tool.md): a check before
the tool runs and a check after it. It is not a tool, because the LLM does not
call it. It is not a part of the context of the LLM, because the LLM does not
know about it. It is a part of the harness. It examines each step at the
boundary between the decisions of the LLM and the operations in the Environment.

```typescript
// Application: put Guardrails around a Tool
const guardedTool = (tool: Tool, guardrails: Guardrail[]): Tool => ({
  ...tool,
  execute: async (params) => {
    for (const g of guardrails) {
      const check = g.pre?.({ name: tool.name, params });
      if (check?.deny) throw new Error(check.deny);
      if (check?.askUser) await confirmWithUser(check.askUser);
    }
    const result = await tool.execute(params);
    for (const g of guardrails) {
      const check = g.post?.({ name: tool.name, params }, result);
      if (check?.redact) return check.redact;
      if (check?.block) throw new Error(check.block);
    }
    return result;
  },
});
```

## Examples

```typescript
const noForcePush: Guardrail = {
  pre: (action) => {
    if (
      action.name === "bash" &&
      action.params.command.includes("push --force")
    )
      return { deny: "force push is forbidden" };
    return { allow: true };
  },
};

const noSecrets: Guardrail = {
  post: (action, result) => {
    if (containsApiKey(result.output))
      return { redact: maskSecrets(result.output) };
    return { allow: true };
  },
};
```

In Claude Code, these parts are Guardrails:

- the permission system, for example a rule that lets the agent write files
  without approval
- safety rules, for example commands that the agent cannot run
- policies in hooks before and after each tool call.

In most systems, the developers write the Guardrails into the code of the agent.
A better configuration is a layer of policies that can compose. Then a user can
add rules without a change to the core of the agent.

<a id="guardrails-and-tool-expressiveness"></a>

## Guardrails on different types of tools

The range of operations of a tool controls the protection that a Guardrail can
give. [Tool](../primitives/tool.md#the-tool-expressiveness-spectrum) gives this
range, from specific tools to generative tools.

On a **specific tool**, a Guardrail stops all dangerous calls. The tool has a
closed set of operations, thus you can make a list of all its dangerous calls.
For example, a Guardrail on `file_write` can examine the path. Then it lets the
file operation continue or stops it, with the same result each time. No set of
parameters can go around this check.

On a **generative tool**, for example bash or code that the agent runs, a
Guardrail stops only the patterns that it knows. It catches known dangerous
commands, for example `rm -rf` and `push --force`. Thus the risk of accidental
damage is lower. For a full defense, Guardrails compose with limits in the
Environment:

1. **Specific tools for frequent operations**: the Guardrail stops all dangerous
   calls of these tools.
2. **Guardrails with patterns on bash**: they catch known dangerous commands at
   the tool boundary.
3. **Sandbox Environments**: they keep the damage small at the Environment
   boundary. They isolate the network, put limits on the filesystem and give no
   credentials.
4. **Approval from a person**: the agent must get approval for each command that
   agrees with a risk pattern.

Guardrails apply limits at the tool boundary. Sandboxes apply limits at the
Environment boundary. For specific tools, the tool boundary is sufficient. For
generative tools, the two layers operate together.
[Tool](../primitives/tool.md#the-tool-expressiveness-spectrum) gives all the
facts about the tool types.

<a id="modes-observe-and-control"></a>

## Monitor mode and control mode

The same hooks can monitor the tool calls or control them. The value that the
policy gives back selects the mode:

- **Monitor mode**: the policy records each tool call. Then a developer can
  debug the agent, do an inspection of its work or calculate the cost. It always
  gives back `allow`. The system records the step but does not change it. For
  example, some hooks write each call to the log and to a `.jsonl` file, and do
  not stop a call.
- **Control mode**: the policy applies rules. It stops dangerous steps, removes
  sensitive data from the output and gets approval from a person for steps with
  a high risk. For example, a policy stops each `rm -rf` command or removes API
  keys from the output.

The hooks are the same in the two modes: functions before and after the tool
runs. Make the hooks one time, and use them to monitor and to control. Only the
value that the policy gives back is different.

<a id="guardrails-vs-verification"></a>

## Guardrails and verification

Guardrails and [Verification](../verification/index.md) operate together. They
examine the same data: tool calls, outputs and traces. They operate at different
times:

- **Guardrails** stop a dangerous step while the agent runs. The system does not
  go into the bad condition.
- **Verification** examines the runs after their end. It finds if the system did
  the correct work.

A full agent system has the two. Guardrails stop very dangerous steps while the
agent runs. Verification shows correct operation across many runs. For example,
a Guardrail can stop `rm -rf` while the agent runs. A check can find `rm -rf` in
the traces after the run. The two have the same safety function from different
sides.

## Related

- [Tool](../primitives/tool.md): Guardrails are around the tools when they run
- [StateMachine](../patterns/state-machine.md): Guardrails put limits on the
  steps in a phase, and a StateMachine puts limits on the changes between phases
- [Verification](../verification/index.md): the check after the run, which
  operates together with the Guardrails during the run
