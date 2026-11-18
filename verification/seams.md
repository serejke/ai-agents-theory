# Seams

A seam is a point in an agent system where you can replace one part with a
different part. The other parts of the system do not change. For example, a test
can replace the LLM with a mock that always gives the same output. An agent
system is a composition of primitives, and each point of this composition is a
**seam**.

<a id="why-it-matters"></a>

## Value

Seams make it possible to do tests of an agent system. An agent system is made
of small parts that a test can replace. If an agent had no seams, a test could
examine only its input and its output. An agent with seams is a set of parts,
and a test can examine each part one at a time. Only its core is not
deterministic, and the verification of this core must use the results of many
runs.

Each [verification method](strategies.md) replaces a part at a seam. For
example:

- It replaces the LLM with a mock.
- It puts test data into a Channel before the run.
- It replaces a tool with a mock that records each call.

When you know where the seams are, you know where a test can connect to the
system.

<a id="the-primitive-seams"></a>

## The seams of the concepts

```
AgentLoop = LLM + Tool[]
                 ↑ seam: replace LLM with a mock

Session = AgentLoop + history
                       ↑ seam: inject pre-built history

StateMachine = Phase[] + Transition[]
                          ↑ seam: test transitions with synthetic phase outputs

Channel = write → (artifact) → read
                    ↑ seam: pre-populate artifacts, verify what's read

Workspace = filesystem operations
             ↑ seam: use a temporary directory with fixture data
```

<a id="the-llm-seam"></a>

## The LLM seam

The most important seam of each agent system is between the
[LLM](../primitives/llm.md) and all other parts. The LLM is the only primitive
that is not deterministic. All the other parts are deterministic:

- the tool calls
- the operations that read and write the Channels
- the checks of the Guardrails
- the phase changes of the StateMachine.

```typescript
// The LLM seam: a protocol that each implementation can obey
type LlmCall = (
  system: string,
  messages: Message[],
) => AsyncStream<TextDelta | ToolCall>;

// Live system: the actual model
const realLlm: LlmCall = (system, messages) =>
  anthropic.stream({ model: "claude-sonnet", system, messages });

// Test: a deterministic mock
const mockLlm: LlmCall = (system, messages) =>
  Stream.of({ type: "text", text: "Mock response." });

// Test: gives again the outputs of a recorded trace
const replayLlm: LlmCall = (system, messages) =>
  playbackFromTrace(recordedTrace, messages);
```

If you replace the LLM with a mock, the system divides into two parts:

- **The deterministic shell.** This part holds the tool calls, the changes to
  the data and the input and output of the Channels. It also holds the assembly
  of the Prompt, the output files and the phase changes. Usual software tests
  can examine it.
- **The core that is not deterministic.** This part holds the reasoning of the
  LLM, the selection of the tools and the text that the LLM gives. It must have
  different verification methods, for example evals, checks of the properties of
  the output, and checks of the sequence of steps.

**Use mocks for the tests of the shell, and use evals for the tests of the
core.** The two types of test operate together. One type does not replace the
other type.

<a id="the-channel-seam"></a>

## The Channel seam

If two or more agents operate in a pipeline, the
[Channel](../patterns/channel.md) between two agents is a seam. Each operation
that writes to a Channel makes an item. Each operation that reads from a Channel
uses an item. Thus a test can examine each agent without the other agents. Put
test data into its input Channel. Then examine the items in its output Channel.

```typescript
// Test agent B alone:
const inputChannel = createChannel();
inputChannel.write("notes/topic-1.md", fixtureNote); // test data, put in before the run

const outputChannel = createChannel();
await agentB.run({ input: inputChannel, output: outputChannel });

assert(outputChannel.read("analysis.md").includes("key finding"));
```

This method operates with each type of Channel: a filesystem, data that all the
agents can read and write, or a store of items.

<a id="the-tool-boundary-seam"></a>

## The Tool seam

A [Tool](../primitives/tool.md) has side effects: it changes items that are
external to the LLM. In the live system, a tool can call an external API, write
to a database or run a shell command. In a test, the tools are the outer seam of
the system. A test can replace each tool with a mock. Each call goes to the
mock. The mock writes the call to a log and gives a prepared result.

```typescript
const fakeTools: Tool[] = [
  {
    name: "query_database",
    execute: async (params) => {
      callLog.push({ tool: "query_database", params });
      return { rows: fixtureData };
    },
  },
  {
    name: "write_file",
    execute: async (params) => {
      callLog.push({ tool: "write_file", params });
      return { success: true };
    },
  },
];

// After the agent runs, check the tool calls:
assert(callLog.some((c) => c.tool === "query_database"));
assert(
  callLog.find((c) => c.tool === "write_file")?.params.path === "report.md",
);
```

This test examines the decisions of the agent: did it call the correct tools
with the correct parameters? The tools do not change files, databases or other
systems during the test.

## Related

- [Verification](index.md): the work that uses the seams. Verification is not a
  tier, and it applies to all the tiers.
- The [trace](trace.md): the data that the replaced parts give to the test.
- The [verification methods](strategies.md): the tests that use the seams.
- The [LLM](../primitives/llm.md): the most important seam is between the LLM
  and all other parts.
- [Channel](../patterns/channel.md): the seams between agents in a pipeline.
- [Tool](../primitives/tool.md): the outer seam of the system.
