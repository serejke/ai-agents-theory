# AgentLoop

An **AgentLoop** is the loop that lets a language model use tools. The
[LLM](../primitives/llm.md) selects a [Tool](../primitives/tool.md) call, the
tool runs, and the LLM reads the result and selects the next step. This part of
the harness makes the LLM and the Tool primitives into an agent that runs. It is
the smallest set of parts that lets an agent operate without a person.

**Composition**: LLM + Tool[] in a recursive cycle.

## Value

<a id="why-it-matters"></a>

An agent uses the LLM and the tools together. The LLM can select a step, but it
cannot do the step. A tool can do a step, but it cannot select the step. The
AgentLoop connects the two:

1. The LLM selects a tool to call.
2. The tool runs and gives a result.
3. The LLM reads the result and selects the next step.

The loop continues until the LLM gives a text without a tool call.

The AgentLoop is a part of the harness, not an optional pattern. Each AI
assistant that can do tasks runs on an AgentLoop, for example Copilot, ChatGPT
and Claude.ai. These systems connect different tools and a different system
prompt to the loop. But the loop is the same in each system.

## Definition

<a id="formal-definition"></a>

```typescript
// The events of a run: text chunks while the LLM writes,
// tool calls found in the output, tool results after each tool runs.
type Event =
  | { kind: "text"; delta: string }
  | { kind: "tool_call"; call: ToolCall }
  | { kind: "tool_result"; callId: string; result: ToolResult };

type AgentLoop = (
  llm: LLM,
  tools: Tool[],
  system: string,
) => (messages: Message[]) => AsyncGenerator<Event>;

// Semantics:
// 1. Assemble a Prompt from (system, messages, tool schemas)
// 2. Call llm(prompt); stream the text; parse the tool calls from the output
// 3. If there are no tool calls → return
// 4. Run each tool; append the results to messages; go to 1
```

The AgentLoop is a `while` loop. It is not a graph, a pipeline or a state
machine. It is the control flow with the smallest number of parts that lets an
agent operate without a person:

```typescript
// Implementation sketch (~20 lines)
async function* agentLoop(llm, tools, system, messages) {
  while (true) {
    const prompt = {
      system,
      tools: tools.map((t) => t.schema),
      messages,
    };
    // llm(prompt) streams text; the parser splits it into content + tool calls
    // with the rules of the active protocol.
    const { text, toolCalls } = await runAndParse(llm, prompt);
    if (toolCalls.length === 0) {
      yield { kind: "text", delta: text };
      return;
    }
    messages.push({ role: "assistant", content: text, toolCalls });
    for (const call of toolCalls) {
      const tool = tools.find((t) => t.name === call.name);
      const result = await tool.execute(call.args);
      messages.push({ role: "tool_result", callId: call.id, result });
    }
  }
}
```

<a id="the-minimal-agent"></a>

## The smallest agent

The AgentLoop is the smallest possible agent. Only the tools and the system
prompt set the work that the agent can do. The structure of the agent stays the
same:

- With an LLM and the tools `[fileRead, fileWrite, bash]`, the AgentLoop is a
  coding agent.
- With the tools `[webSearch, webFetch]`, the AgentLoop is a research agent.

```typescript
// A coding agent
const coder = agentLoop(
  claude_sonnet,
  [fileRead, fileWrite, bash, grep, glob],
  codingPrompt,
);

// A research agent: the same loop, different tools and prompt
const researcher = agentLoop(
  claude_sonnet,
  [webSearch, webFetch, noteWrite],
  researchPrompt,
);
```

Thus a coding agent and a research agent do not have different structures. They
are the same loop with different configurations. Each agent uses this loop.

<a id="what-agentloop-lacks"></a>

## No history between calls

The AgentLoop keeps no data between two calls. It gets `messages` as input and
gives `Event` items as output, but it does not keep the history of the
conversation. Each call starts again without data from the previous call. This
is a decision of the structure: the AgentLoop only calculates and keeps no data.

The AgentLoop is the first layer, and the other concepts compose on top of it:

- For an agent that keeps the previous messages of a chat, use a
  [Session](session.md). The Session adds a `history: Message[]` around the
  AgentLoop and keeps it between the calls.
- For an agent that keeps facts across conversations, add
  [Memory](../patterns/memory.md).
- For an agent with hard safety limits, put [Guardrails](guardrail.md) around
  its tools.
- For a workflow with two or more phases, connect AgentLoops with a
  [StateMachine](../patterns/state-machine.md).

## The loop as a graph

Some frameworks show the AgentLoop as a graph with two nodes: `call_model` and
`call_tools`, with an edge in each direction. The graph adds steps that compile
it, node definitions and edge routing. But the loop only calls the LLM, runs the
tool calls of the output, and then calls the LLM again.

A graph helps for a workflow with many phases, for example in the
[StateMachine](../patterns/state-machine.md) pattern. For the basic loop of tool
calls, a `while` loop is the correct structure.

## Related

- [LLM](../primitives/llm.md): the part of the loop that selects the steps.
- [Tool](../primitives/tool.md): the part of the loop that does the tasks.
- [Session](session.md): an AgentLoop with a history, which gives an agent that
  can continue a conversation.
- [Guardrail](guardrail.md): the limits around each tool call in the loop.
