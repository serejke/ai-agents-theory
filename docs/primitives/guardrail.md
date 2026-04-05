# Guardrail

A Guardrail is a hard constraint enforced in code that the LLM cannot override, bypass, or reinterpret. It is the only primitive in the system that is not controlled by the LLM.

## Why It Matters

Everything else in an agent system — the system prompt, tools, memory — the LLM can ignore or interpret its own way. Tell it "don't push to main" in the system prompt and it will usually comply, but there's no guarantee. A Guardrail makes compliance structural: the `push --force` command is intercepted and blocked before it reaches the shell, regardless of what the LLM intended.

This distinction — suggestions the model might follow vs. constraints the model cannot violate — is what separates a prototype from a production system. Any agent system that takes real-world actions (writes files, calls APIs, sends messages) needs Guardrails.

## Formal Definition

```typescript
type Guardrail = {
  // Pre: check BEFORE executing an action
  pre?: (action: ToolCall) => Allow | Deny | AskUser;

  // Post: check AFTER execution
  post?: (action: ToolCall, result: ToolResult) => Allow | Redact | Block;
};
```

Guardrails are middleware — they wrap [Tool](tool.md) execution with pre- and post-checks. They are not tools themselves (the LLM doesn't call them) and not part of the LLM's context (the LLM doesn't reason about them). They are infrastructure that intercepts actions at the boundary between the LLM's decisions and the world's execution.

```typescript
// Application — wraps a Tool with Guardrails
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

---

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

In Claude Code, Guardrails manifest as: the permission system ("can it write files without asking?"), safety rules (restricted commands), and hook-based policies (pre/post tool-call hooks). In most current systems Guardrails are hardcoded, whereas ideally this should be a composable layer — so the user can add their own constraints without modifying the agent's core.

---

## Guardrails and Tool Expressiveness

How complete a Guardrail's coverage is depends on where the tool sits on the [expressiveness spectrum](tool.md).

Over **specific tools** (closed action space), Guardrails provide complete coverage. The set of dangerous actions is enumerable. A Guardrail on `file_write` can inspect the path and deterministically allow or deny — no combination of parameters can circumvent it.

Over **generative tools** (open action space — bash, code execution), Guardrails provide pattern-level coverage. They catch known dangerous commands (`rm -rf`, `push --force`) and raise the bar against accidental damage. For complete defense over generative tools, Guardrails compose with environment-level containment:

1. **Specific tools for common operations** — Guardrail interception provides complete coverage
2. **Pattern-based Guardrails on bash** — catch known dangerous commands at the tool boundary
3. **Sandboxed environments** — contain blast radius at the environment boundary (network isolation, filesystem restrictions, no credentials)
4. **Human-in-the-loop** — require approval for commands matching risk patterns

Guardrails enforce constraints at the tool boundary; sandboxes enforce constraints at the environment boundary. For specific tools, the tool boundary is sufficient. For generative tools, both layers work together. See the [Tool Expressiveness Spectrum](tool.md) for the full analysis.

---

## Two Modes: Observe and Control

The same middleware infrastructure serves two distinct purposes depending on what the callback returns:

- **Observe mode**: Log every tool call for debugging, audit, or cost tracking. Always returns `allow`. The system captures what happened without intervening. Example: hooks that log to transcript + JSONL but never deny.
- **Control mode**: Enforce constraints. Deny dangerous actions, redact sensitive output, require human confirmation for high-stakes operations. Example: blocking `rm -rf` commands, masking API keys in output.

The infrastructure is identical — pre/post callbacks wrapping tool execution. Build hooks once, use for both observability and enforcement. The difference is purely in what the callback returns.

---

## Guardrails vs. Verification

Guardrails and [Verification](../verification.md) are complementary — both operate on the same action space (tool calls, outputs, trajectories) but at different points in the lifecycle:

- **Guardrails**: runtime prevention. Block dangerous actions _during_ execution. The system never reaches the bad state.
- **Verification**: offline detection. Confirm correct behavior _after_ execution. The system already ran; you're checking whether it did the right thing.

A complete agent system has both: Guardrails that prevent catastrophic actions during execution, and Verification that confirms correct behavior across runs. A Guardrail that blocks `rm -rf` at runtime and a trajectory assertion that checks for `rm -rf` in traces serve the same safety goal from different angles.

---

## Related

- [Tool](tool.md) — Guardrails wrap tool execution
- [StateMachine](state-machine.md) — Guardrails constrain actions within a phase; StateMachine constrains transitions between phases
- [Verification](../verification.md) — offline counterpart to runtime Guardrails
