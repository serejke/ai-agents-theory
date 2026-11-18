# Trace

A trace records all the steps of one run of an agent, in sequence: each LLM
call, each tool call and each tool result. A **trace** shows the work of an
[AgentLoop](../harness/agent-loop.md) as a stack trace shows the work of a
function call. It tells the steps that occurred, and also how the agent got to
its result.

<a id="why-it-matters"></a>

## Value

Before you can do the verification of an agent, you must see the work that it
did. The last output, for example a report, a code change or a message, shows
only a part of this work. Two runs can give the same output by very different
paths. The reasoning of one run is correct. The other run gets the correct
output with incorrect reasoning. Without a trace, you cannot see the difference
between these runs.

If a system does not keep traces, it can examine only the outputs. If it keeps
traces, it can examine these items as different checks:

- the prompts
- the sequence of steps
- the quantity of tokens, LLM calls and time
- the quality of the output.

<a id="formal-definition"></a>

## Definition

```typescript
type Trace = {
  steps: Step[];
  finalOutput: string;
  metadata: {
    totalLlmCalls: number;
    totalTokens: number;
    durationMs: number;
  };
};

type Step = {
  llmCall: {
    messagesIn: Message[]; // what the LLM saw
    response: string; // what the LLM gave
    toolCalls: ToolCall[]; // the tool calls that the LLM selected
  };
  toolResults: Array<{
    tool: string;
    params: unknown;
    result: ToolResult;
  }>;
};
```

<a id="what-each-verification-type-inspects"></a>

## The parts of the trace that each check examines

Each [verification method](strategies.md) examines a part of the trace:

| Verification of                        | The part of the trace that it examines                      |
| -------------------------------------- | ----------------------------------------------------------- |
| The output                             | `trace.finalOutput` only                                    |
| The sequence of steps                  | `trace.steps`: the sequence of the tool calls               |
| The quantity of tokens, calls and time | `trace.metadata`: the tokens, the calls and the time        |
| The prompt                             | `trace.steps[0].llmCall.messagesIn`: the request to the LLM |

It is easy to keep a trace during a run. After the run, it is not easy to make
the trace again. If a pipeline keeps traces from the first day, it can add new
checks and then use them on the previous runs. If a pipeline does not keep
traces, it must do each test run again for each new check.

## Related

- [Verification](index.md): the work that uses the traces. Verification is not a
  tier, and it applies to all the tiers.
- [Seams](seams.md): the points where a test replaces a part, to get traces that
  you can compare.
- The [verification methods](strategies.md): the tests that do checks on traces.
- [AgentLoop](../harness/agent-loop.md): the harness that a trace records.
