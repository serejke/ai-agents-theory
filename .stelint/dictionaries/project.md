# Dictionary

This dictionary gives the terms of this book and their meanings. It applies
to each file. A term with an uppercase letter is a concept of the theory or
a name. Each other term is a word of the technical field.

## The theory

| Term | Meaning |
|---|---|
| theory | The model of agent systems that this book gives. It tells which parts an agent system has and how these parts connect. |
| concept | One part of the theory with a name, for example LLM, Session or Memory. Each concept has one document. |
| tier | A group of concepts with the same function in the theory. The tiers are the primitives, the harness, the Environment and the patterns. |
| composition | The concepts that make a larger concept, and how they connect. Each harness document and each pattern document gives its composition in one line. |
| to compose | To make a larger concept from smaller concepts. |
| harness | The parts that each agent has when it runs, also if its author does not use these names: the AgentLoop, the Session, the Guardrail and the Agent. |
| Agent | A program that gets a task, uses an LLM to select steps, uses tools to do these steps, and continues until it completes the task. It is a Session with a frontend and an Environment. |
| AgentLoop | The loop that sends a prompt to the LLM, runs each tool call of the output, adds the results to the history and sends the prompt again. It stops when the LLM gives text without a tool call. |
| Session | An AgentLoop with the history of its turns. The history stays between the turns. |
| Guardrail | The point that each tool call goes through before its tool runs. Policies at this point can let the call continue, stop it or change it. |
| Environment | The resources that an Agent can use with its tools: files, processes, the network and credentials. It is not a part of the Agent. |
| Prompt | The input that the LLM reads in one call: the system prompt, the tool schemas, the history and the inserts, in the format of the model. |
| Memory | The pattern that keeps knowledge between Sessions with tools and loads it into the next Sessions. |
| StateMachine | The pattern that divides a task into phases. A phase variable and the rules that change the phase control which tools the agent can use in each phase. |
| Channel | The pattern that moves data between agents in one run, with tools and an agreed format. |
| PromptLoading | The pattern that controls when the harness puts a text source into the Prompt: at the start, on request, in steps or each turn. |
| Workspace | The pattern that gives an agent a tree of items with names, for example files and folders. Its tools can read and write these items. |
| Planner | The pattern that makes a plan with a Session that has only tools that read. A different Session does the plan. |
| Router | The pattern that sends each input to one of two or more handlers. The type of the input selects the handler. |
| Evaluator | The pattern that uses an LLM to examine the output of an agent and sends the result back to the agent as feedback. |
| Continuity | The pattern that lets work continue across two or more Sessions. It uses Memory, PromptLoading, a Guardrail and tools. |
| Topology | The pattern that connects two or more Sessions in a shape, for example a tree or a pipeline. Channels move the data and Routers select the next Session. |
| verification | The work that shows that an agent system operates correctly, also if the LLM gives different outputs for the same input. |
| trace | The data that records one run of an agent: each prompt, each LLM output, each tool call and each tool result, in sequence. |
| seam | A point in an agent system where a test can replace one part with a different part, for example a recorded LLM. |

## Agent systems

| Term | Meaning |
|---|---|
| model | A large language model (LLM), for example Claude or GPT-5. |
| training | The process that makes a model from large quantities of data. |
| reasoning | The work of the LLM when it reads a prompt and selects the next step. |
| context | All the text that the LLM reads in one call. |
| context window | The maximum quantity of tokens that the LLM can read in one call. |
| system prompt | The part of the Prompt that gives the instructions of the agent. It stays the same for each turn of a Session. |
| tool call | A request in the output of the LLM to run one tool with arguments. |
| tool result | The data that a tool gives back after a tool call. The harness adds it to the history. |
| tool schema | The name, the description and the parameters of a tool. The Prompt gives it to the LLM. |
| insert | A text that the harness adds to the Prompt for one turn, for example a short instruction about the task. |
| history | The messages of a Session in sequence: each input, each LLM output, each tool call and each tool result. |
| conversation | The sequence of messages between a person and an agent. |
| description | A text that tells the function of an item. |
| hook | A function that the harness calls at a specified point, for example before each tool call. |
| phase | One part of a task. Each phase has a set of tools and rules. A StateMachine moves the agent from one phase to the next phase. |
| state machine | A system with a specified set of conditions and rules that move it from one condition to a different condition. |
| workflow | A specified sequence of steps that a system follows to do a task. |
| handoff | The movement of a task from one agent to a different agent. |
| framework | A library that gives the parts of an agent system to a developer, for example LangGraph. |
| developer | A person who writes software. |
| engineer | A person who makes technical systems, for example software. |
| filesystem | The files and the folders that a program can read and write. |
| sandbox | An Environment with limits. It can stop access to files, processes, the network or credentials. |
| permission | A rule that lets an agent do one operation, for example write a file or run a command. |
| terminal | A program that shows text. A person writes a command in it and then reads the output of the command. |
| chat | A conversation in text messages. |
| assistant | A program that does tasks for a person. The person gives the task in words. |
| email | A message that a person sends to a different person on the internet. |
| internet | The public network that connects computers. |
| case study | A document that examines one agent system that persons use, with the concepts of the theory. |
| research | The work of persons who find new facts and publish them. |
| paper | A document of research that its authors publish. |
| author | The person who writes a document or a program. |
| to publish | To make a document available to all persons. |
| book | This repository: the documents of the theory. |
| knowledge | The facts that an agent or a person knows. |
| plan | A list of the steps that complete a task. |
| feedback | Data about the result of a step that goes back to the agent, Then the agent can correct its work. |
| tree | A structure in which each item, but not the first item, has one item above it. |
| resource | An item that a program can use, for example a file, a process, a network connection or a credential. |
| coding agent | An agent that reads and changes the code of a project. |
| organization | A group of persons that operate together, for example a company. |
| customer | A person or an organization that uses the products of a company. |
| JavaScript | A language for programs. |
| company | An organization that makes products for customers. |
| product | An item that a company makes for its customers, for example an agent or a model. |
| anchor | The part of a link after the `#` sign. It identifies one heading or one point in a document. |
| title | The name of a document, a paper or a section. |
| social network | A web service where persons publish short texts and links. |
| pseudocode | Code that shows the structure of a program but does not run. This book writes it in a format that is almost the same as TypeScript. |

## Security

| Term | Meaning |
|---|---|
| security | The protection of a system and its data from attackers. |
| attacker | A person who tries to make a system do a dangerous operation, or to get its private data. |
| attack | An operation that an attacker does against a system. |
| threat model | The attacks that a system must stop, and the operations that the attacker can do. |
| prompt injection | An instruction in the data that an LLM reads, which tells the LLM to obey the instructions of an attacker. |
| untrusted data | Data from a source that the system does not control, for example a web page, an email or a tool result. |
| exfiltration | The movement of private data from a system to an attacker. |
| defense | A method that stops an attack or makes its damage smaller. |

## Names

| Term | Meaning |
|---|---|
| Anthropic | The company that makes the Claude models and Claude Code. |
| Claude | The models of Anthropic. |
| Claude Code | The coding agent of Anthropic. Its frontend is a terminal. |
| OpenAI | The company that makes ChatGPT, Codex and models, for example GPT-5. |
| ChatGPT | The chat assistant of OpenAI. |
| Codex | The coding agent of OpenAI. |
| GPT-5 | A model of OpenAI. |
| Google | The company that makes the Gemini models. |
| Gemini | The models of Google. |
| Gemini CLI | The coding agent of Google. Its frontend is a terminal. |
| Microsoft | The company that makes GitHub Copilot. |
| GitHub | The web service of Microsoft for the repositories of code. |
| Copilot | The assistants of Microsoft, for example GitHub Copilot. |
| Cursor | A coding agent in a program that changes code. |
| LangGraph | A framework of agents that shows each agent as a graph of nodes. |
| LangChain | The company and the library that make LangGraph. |
| PI | A small coding agent. Its frontend is a terminal. Its code shows each concept of the theory as a point where a developer can add code. |
| Hermes Agent | A framework for an agent of one person. The agent operates all the time and keeps Memory across Sessions. |
| Honcho | A Memory service for agents. It keeps facts about each person and each agent. |
| Flue | A framework that runs agents in sandboxes on the servers of a company. |
| Cloudflare | A company that operates a network of servers and sandboxes. |
| Vercel | A company that operates web services and sandboxes for its customers. |
| MCP | The Model Context Protocol: a standard protocol that connects an agent to the tools of a different program. |
| Python | A language for programs. |
| bash | A Unix shell. An agent with a bash tool can run each command of the shell. |
| grep | A Unix command that finds lines of text that agree with a pattern. |
| stelint | The tool that checks the documents of this book against the rules of STE and against this dictionary. |
| prettier | A tool that gives a standard format to the text of a Markdown file. |
| npm | A tool that installs JavaScript packages. |
