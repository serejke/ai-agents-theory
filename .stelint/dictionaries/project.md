# Dictionary

This dictionary gives the terms of this book and their meanings. It applies to
each file. A term with an uppercase letter is a concept of the theory or a name.
Each other term is a word of the technical field.

## The theory

| Term          | Meaning                                                                                                                                                                                       |
| ------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| theory        | The model of agent systems that this book gives. It tells which parts an agent system has and how these parts connect.                                                                        |
| concept       | One part of the theory with a name, for example LLM, Session or Memory. Each concept has one document.                                                                                        |
| tier          | A group of concepts with the same function in the theory. The tiers are the primitives, the harness, the Environment and the patterns.                                                        |
| composition   | The concepts that make a larger concept, and how they connect. Each harness document and each pattern document gives its composition in one line.                                             |
| to compose    | To make a larger concept from smaller concepts.                                                                                                                                               |
| harness       | The parts that each agent has when it runs, also if its author does not use these names: the AgentLoop, the Session, the Guardrail and the Agent.                                             |
| Agent         | A program that gets a task, uses an LLM to select steps, uses tools to do these steps, and continues until it completes the task. It is a Session with a frontend and an Environment.         |
| AgentLoop     | The loop that sends a prompt to the LLM, runs each tool call of the output, adds the results to the history and sends the prompt again. It stops when the LLM gives text without a tool call. |
| Session       | An AgentLoop with the history of its turns. The history stays between the turns.                                                                                                              |
| Guardrail     | The point that each tool call goes through before its tool runs. Policies at this point can let the call continue, stop it or change it.                                                      |
| Environment   | The resources that an Agent can use with its tools: files, processes, the network and credentials. It is not a part of the Agent.                                                             |
| Prompt        | The input that the LLM reads in one call: the system prompt, the tool schemas, the history and the inserts, in the format of the model.                                                       |
| Memory        | The pattern that keeps knowledge between Sessions with tools and loads it into the next Sessions.                                                                                             |
| StateMachine  | The pattern that divides a task into phases. A phase variable and the rules that change the phase control which tools the agent can use in each phase.                                        |
| Channel       | The pattern that moves data between agents in one run, with tools and an agreed format.                                                                                                       |
| PromptLoading | The pattern that controls when the harness puts a text source into the Prompt: at the start, on request, in steps or each turn.                                                               |
| Workspace     | The pattern that gives an agent a tree of items with names, for example files and folders. Its tools can read and write these items.                                                          |
| Planner       | The pattern that makes a plan with a Session that has only tools that read. A different Session does the plan.                                                                                |
| Router        | The pattern that sends each input to one of two or more handlers. The type of the input selects the handler.                                                                                  |
| Evaluator     | The pattern that uses an LLM to examine the output of an agent and sends the result back to the agent as feedback.                                                                            |
| Continuity    | The pattern that lets work continue across two or more Sessions. It uses Memory, PromptLoading, a Guardrail and tools.                                                                        |
| Topology      | The pattern that connects two or more Sessions in a shape, for example a tree or a pipeline. Channels move the data and Routers select the next Session.                                      |
| verification  | The work that shows that an agent system operates correctly, also if the LLM gives different outputs for the same input.                                                                      |
| trace         | The data that records one run of an agent: each prompt, each LLM output, each tool call and each tool result, in sequence.                                                                    |
| seam          | A point in an agent system where a test can replace one part with a different part, for example a recorded LLM.                                                                               |

## Agent systems

| Term                     | Meaning                                                                                                                                                                                                                 |
| ------------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| actor model              | A structure of software in which each part has a queue of messages. The parts operate only when they send and receive messages.                                                                                         |
| analysis                 | The work that examines data and finds facts in it.                                                                                                                                                                      |
| anchor                   | The part of a link after the `#` sign. It identifies one heading or one point in a document.                                                                                                                            |
| artifact                 | An output of an agent, for example a file, a report or a small program. A different agent or a person can use it.                                                                                                       |
| assistant                | A program that does tasks for a person. The person gives the task in words.                                                                                                                                             |
| authentication           | The process that makes sure that a user is the correct person.                                                                                                                                                          |
| author                   | The person who writes a document or a program.                                                                                                                                                                          |
| blackboard               | A shared state that all the agents of a system read and write. Its data has no schema.                                                                                                                                  |
| book                     | This repository: the documents of the theory.                                                                                                                                                                           |
| bootstrap                | A script or a tool that prepares the Environment for work in one step.                                                                                                                                                  |
| bottleneck               | A part of a system that all the work must go through. It makes the full system slower.                                                                                                                                  |
| broadcast                | A message that goes to all the parts of a system at the same time.                                                                                                                                                      |
| browser                  | A program that shows web pages.                                                                                                                                                                                         |
| bundle                   | One file that contains a program and all the code that the program uses.                                                                                                                                                |
| canonical history        | The history of a Session that Hermes keeps and records. It does not contain the text that Hermes adds to one API call only.                                                                                             |
| case study               | A document that examines one agent system that persons use, with the concepts of the theory.                                                                                                                            |
| chat                     | A conversation in text messages.                                                                                                                                                                                        |
| checkpoint               | Data that records the work that a workflow completed. After a crash, the workflow continues from the last checkpoint.                                                                                                   |
| class                    | A type in a program. It gives the data and the functions of each object of that type.                                                                                                                                   |
| clean state              | The condition of a project at the end of a Session when all changes have a commit, the progress log has a new entry and the build has no errors.                                                                        |
| cloud                    | The servers of a company that its customers use through the internet.                                                                                                                                                   |
| coding agent             | An agent that reads and changes the code of a project.                                                                                                                                                                  |
| cognitive affinity       | The property of an interface that agrees with the patterns of the training data of an LLM. An LLM uses such an interface correctly more frequently.                                                                     |
| cognitive facade         | A facade for the reasoning of an LLM. It shows a large API as a set of files and directories.                                                                                                                           |
| company                  | An organization that makes products for customers.                                                                                                                                                                      |
| compensation             | An operation that cancels the result of a previous operation, for example an operation that gives money back after an incorrect payment.                                                                                |
| conclusion               | A fact about a peer that Honcho derives from messages or from other conclusions.                                                                                                                                        |
| connector                | An adapter that connects Flue to the sandboxes of a different company.                                                                                                                                                  |
| context                  | All the text that the LLM reads in one call.                                                                                                                                                                            |
| context window           | The maximum quantity of tokens that the LLM can read in one call.                                                                                                                                                       |
| conversation             | The sequence of messages between a person and an agent.                                                                                                                                                                 |
| coordinator              | The agent of a Supervisor that gives tasks to the other agents and collects their results.                                                                                                                              |
| cost                     | The value that a customer gives to a company for a service.                                                                                                                                                             |
| cron                     | A Unix method that starts a program at specified times.                                                                                                                                                                 |
| customer                 | A person or an organization that uses the products of a company.                                                                                                                                                        |
| day                      | A period of 24 hours.                                                                                                                                                                                                   |
| deployment               | The installation of a new version of software on the servers that its customers use.                                                                                                                                    |
| description              | A text that tells the function of an item.                                                                                                                                                                              |
| developer                | A person who writes software.                                                                                                                                                                                           |
| dispatch                 | The step of the AgentLoop that sends a tool call from the LLM to its tool.                                                                                                                                              |
| durable execution        | A method to run a workflow that records each completed step. After a crash, the workflow continues from the last completed step.                                                                                        |
| durable execution engine | A system that runs workflows with durable execution.                                                                                                                                                                    |
| e2b                      | A company that gives sandboxes for agents in the cloud.                                                                                                                                                                 |
| email                    | A message that a person sends to a different person on the internet.                                                                                                                                                    |
| embedding                | A list of numbers that a model makes from a text. Two texts with near meanings have near embeddings.                                                                                                                    |
| engineer                 | A person who makes technical systems, for example software.                                                                                                                                                             |
| eval                     | A test that runs the LLM on a set of tasks and examines the quality of its outputs.                                                                                                                                     |
| event                    | A message that PI sends to its extensions at a specified point of its operation.                                                                                                                                        |
| feedback                 | Data about the result of a step that goes back to the agent, Then the agent can correct its work.                                                                                                                       |
| filesystem               | The files and the folders that a program can read and write.                                                                                                                                                            |
| follow-up message        | A message that the user sends while the agent operates. PI gives it to the agent at the point where the agent stops.                                                                                                    |
| fork                     | A copy of a Session that continues in a different direction from a point of its history.                                                                                                                                |
| framework                | A library that gives the parts of an agent system to a developer, for example LangGraph.                                                                                                                                |
| frontmatter              | The YAML data at the start of a Markdown file.                                                                                                                                                                          |
| full-text search         | A function of a database that finds each text that contains the given words.                                                                                                                                            |
| game                     | A program that a person uses for play.                                                                                                                                                                                  |
| generative tool          | A tool that runs each program or command that the LLM writes, for example bash. The agent can do each operation that its Environment lets it do.                                                                        |
| golden file              | A file that holds a known correct output. A test compares each new output with this file.                                                                                                                               |
| ground truth             | A file that gives the conditions that make a project done. Each item tells if it is done. The agent changes this only after a test.                                                                                     |
| guard                    | A function that examines the data of a workflow. It tells if a change of phase can occur.                                                                                                                               |
| handoff                  | The movement of a task from one agent to a different agent.                                                                                                                                                             |
| hierarchy                | A Topology in which Supervisors give tasks to lower Supervisors. Each lower Supervisor controls a team.                                                                                                                 |
| history                  | The messages of a Session in sequence: each input, each LLM output, each tool call and each tool result.                                                                                                                |
| hook                     | A function that the harness calls at a specified point, for example before each tool call.                                                                                                                              |
| hub                      | The part of a structure that connects to each other part. The other parts connect only to it.                                                                                                                           |
| idempotency              | The property of an operation that gives the same result when it runs two or more times.                                                                                                                                 |
| infrastructure           | The servers, the services and the software that a system uses when it runs.                                                                                                                                             |
| insert                   | A text that the harness adds to the Prompt for one turn, for example a short instruction about the task.                                                                                                                |
| internet                 | The public network that connects computers.                                                                                                                                                                             |
| JavaScript               | A language for programs.                                                                                                                                                                                                |
| jiti                     | A library that runs TypeScript files with no build step.                                                                                                                                                                |
| judge                    | An LLM that examines the output of a different LLM against a list of rules. It gives a number for each rule.                                                                                                            |
| knowledge                | The facts that an agent or a person knows.                                                                                                                                                                              |
| leaf                     | A node of a tree that has no node below it.                                                                                                                                                                             |
| loader                   | A rule of PromptLoading. It gives a source of text, a strategy and the Prompt slot that gets the text.                                                                                                                  |
| long-term memory         | The memory of a person that keeps facts for a long time. It is large, but the person must find each fact in it.                                                                                                         |
| mailbox                  | A queue of messages for one agent. The agent reads the messages when it can.                                                                                                                                            |
| managed workspace        | A Workspace that is a file facade on the storage of a company. The storage gives the permissions, the versions and the semantic search.                                                                                 |
| mesh                     | A structure in which each part can connect directly to each other part.                                                                                                                                                 |
| mini-SWE                 | A benchmark that examines agents on tasks that change software.                                                                                                                                                         |
| mock                     | A part that a test uses as an alternative to a part of the system, for example to the LLM. It gives prepared results and records the calls that it gets.                                                                |
| model                    | A large language model (LLM), for example Claude or GPT-5.                                                                                                                                                              |
| money                    | The units of value that persons and companies give for products.                                                                                                                                                        |
| month                    | A unit of time. One year has 12 months.                                                                                                                                                                                 |
| observer                 | A peer that sees the messages of a different peer. Honcho keeps a different set of facts about a peer for each observer.                                                                                                |
| orchestration            | The control of the sequence of the steps of a workflow.                                                                                                                                                                 |
| organization             | A group of persons that operate together, for example a company.                                                                                                                                                        |
| paper                    | A document of research that its authors publish.                                                                                                                                                                        |
| parameterized tool       | A tool that operates on one system, for example a database. The agent writes the parameters, thus it can make new queries on that system.                                                                               |
| parent agent             | An agent that starts subagents. It gives a task to each subagent and gets back the result.                                                                                                                              |
| payment                  | An operation that moves money from a customer to a company.                                                                                                                                                             |
| peer                     | A person or an agent. Honcho keeps facts about each peer.                                                                                                                                                               |
| peer card                | A short text about the life and the work of a peer. Honcho keeps it on top of the conclusions.                                                                                                                          |
| peer theory              | The knowledge with a structure that Honcho keeps for one peer. It is all the conclusions about the peer at each level and the peer card.                                                                                |
| permission               | A rule that lets an agent do one operation, for example write a file or run a command.                                                                                                                                  |
| persona                  | A text that tells an agent who it is and how it speaks.                                                                                                                                                                 |
| phase                    | One part of a task. Each phase has a set of tools and rules. A StateMachine moves the agent from one phase to the next phase.                                                                                           |
| phone                    | A small computer that a person can hold. It sends messages and voice calls.                                                                                                                                             |
| plan                     | A list of the steps that complete a task.                                                                                                                                                                               |
| post-training            | The training that changes a model after its first training.                                                                                                                                                             |
| prefetch                 | An operation that reads the memory providers at the start of a turn, before the LLM call.                                                                                                                               |
| price                    | The quantity of value that a customer gives for a product, or for each call to a model.                                                                                                                                 |
| product                  | An item that a company makes for its customers, for example an agent or a model.                                                                                                                                        |
| production               | The condition of a system when its customers use it, after its tests.                                                                                                                                                   |
| progress log             | A file that records the work of each Session. The agent adds one entry at the end of each Session.                                                                                                                      |
| prompt template          | A file of PI that holds a prompt. The user writes a command, and PI puts the prompt into the Prompt.                                                                                                                    |
| pseudocode               | Code that shows the structure of a program but does not run. This book writes it in a format that is almost the same as TypeScript.                                                                                     |
| reasoning                | The work of the LLM when it reads a prompt and selects the next step.                                                                                                                                                   |
| recursive cycle          | A cycle that starts again with the result of the previous cycle.                                                                                                                                                        |
| reducer                  | A function that merges two or more changes of one value into one value.                                                                                                                                                 |
| reinforcement learning   | A method of training. The model tries tasks, and a reward function gives a number to each result. Then the training changes the model to get higher numbers.                                                            |
| research                 | The work of persons who find new facts and publish them.                                                                                                                                                                |
| reservation              | Data that keeps a position on a flight or in a different service for a customer.                                                                                                                                        |
| resource                 | An item that a program can use, for example a file, a process, a network connection or a credential.                                                                                                                    |
| reviewer                 | A run of Hermes that examines the history of the last turns. It selects the facts and the skills to keep.                                                                                                               |
| reward function          | A function that examines the result of a rollout and gives it a number. A high number shows a good result.                                                                                                              |
| role                     | A label for the function of a message or of a subagent. In the Prompt, it tells which source wrote a message, for example the user or a tool. In Flue, a role is a Markdown file that gives instructions to a subagent. |
| rollout                  | One run of the agent on one task during reinforcement learning.                                                                                                                                                         |
| sandbox                  | An Environment with limits. It can stop access to files, processes, the network or credentials.                                                                                                                         |
| scoped runtime           | The method of Flue that changes the tools, the model and the system prompt of an agent for one call. After the call, it puts back the previous values.                                                                  |
| semantic search          | A method that finds the text with a meaning near the meaning of the query. It compares embeddings.                                                                                                                      |
| session graph            | A tree of Sessions that start with the same messages. Each node is a condition of the conversation, and a pointer selects the active Session.                                                                           |
| shared state             | Data that two or more agents or nodes read and change during one run.                                                                                                                                                   |
| signal                   | A message that tells a different part of a system that a condition occurred.                                                                                                                                            |
| social network           | A web service where persons publish short texts and links.                                                                                                                                                              |
| specialist               | An agent that does one type of task.                                                                                                                                                                                    |
| specific tool            | A tool that does one operation, for example a tool that reads one file. Its tool schema shows all the operations that it can do.                                                                                        |
| stack trace              | A list of the function calls that are active at one point of a program.                                                                                                                                                 |
| state                    | The data of one run of a LangGraph graph. Each node reads it and gives back changes to it.                                                                                                                              |
| state machine            | A system with a specified set of conditions and rules that move it from one condition to a different condition.                                                                                                         |
| steering message         | A message that the user sends while the tools of a turn run. PI gives it to the agent after these tools stop.                                                                                                           |
| strategy                 | The method of a loader that tells when its text goes into the Prompt: at the start, on request of the LLM, in layers or in each turn.                                                                                   |
| stream                   | A sequence of data that a program gives in small parts, one part after the other. The reader can use each part before the last part comes.                                                                              |
| structural navigation    | A method to find context. The agent moves through a tree of folders, and the names of the folders tell it where to go.                                                                                                  |
| subgraph                 | A graph that is one node of a larger graph.                                                                                                                                                                             |
| subscription             | A contract with a provider that lets a person use the models of the provider for a specified time.                                                                                                                      |
| summary                  | A short text that gives the important points of a longer text.                                                                                                                                                          |
| system prompt            | The part of the Prompt that gives the instructions of the agent. It stays the same for each turn of a Session.                                                                                                          |
| team                     | A group of agents that one Supervisor controls.                                                                                                                                                                         |
| tenant                   | One customer of a service that many customers use. The service keeps the data and the users of each tenant apart.                                                                                                       |
| terminal                 | A program that shows text. A person writes a command in it and then reads the output of the command.                                                                                                                    |
| terminal-bench           | A benchmark that examines agents on tasks in a terminal.                                                                                                                                                                |
| theme                    | A set of colors for the terminal interface of PI.                                                                                                                                                                       |
| title                    | The name of a document, a paper or a section.                                                                                                                                                                           |
| to publish               | To make a document available to all persons.                                                                                                                                                                            |
| to replay                | To run the turns of a recorded Session again.                                                                                                                                                                           |
| tool call                | A request in the output of the LLM to run one tool with arguments.                                                                                                                                                      |
| tool result              | The data that a tool gives back after a tool call. The harness adds it to the history.                                                                                                                                  |
| tool schema              | The name, the description and the parameters of a tool. The Prompt gives it to the LLM.                                                                                                                                 |
| training                 | The process that makes a model from large quantities of data.                                                                                                                                                           |
| trajectory               | The sequence of the steps of an agent in one run, with each LLM call and each tool call. Hermes records it in a file.                                                                                                   |
| tree                     | A structure in which each item, but not the first item, has one item above it.                                                                                                                                          |
| trigger                  | A condition that starts an agent, for example a webhook or a cron time.                                                                                                                                                 |
| trigram                  | A sequence of three characters. With trigrams, a database can find words also if the text has no spaces between the words.                                                                                              |
| trust                    | The condition in which a person thinks that a company or a system operates correctly.                                                                                                                                   |
| vector recall            | A memory method that finds the stored texts with the nearest meaning to a query.                                                                                                                                        |
| vector store             | A database that finds stored text with a meaning that is almost the same as a query.                                                                                                                                    |
| verdict                  | The result of an Evaluator. It tells if the output of an agent is correct, and it gives the problems and the corrections.                                                                                               |
| virtual machine          | A program that operates as a full computer with its operating system. It runs on a different computer.                                                                                                                  |
| webhook                  | An HTTP request that a different service sends to a program when a specified change occurs.                                                                                                                             |
| week                     | A period of 168 hours.                                                                                                                                                                                                  |
| workflow                 | A specified sequence of steps that a system follows to do a task.                                                                                                                                                       |
| working memory           | The memory of a person for the task that the person does at this time. It is fast and small, and its contents do not stay.                                                                                              |

## Security

| Term                      | Meaning                                                                                                                                          |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| adaptive attack           | An attack that knows the defense and changes to go through it.                                                                                   |
| adversarial training      | Training with examples of attacks. After it, the model rejects these attacks.                                                                    |
| architecture              | The structure of a system. It shows the parts of the system and how they connect.                                                                |
| attack                    | An operation that an attacker does against a system.                                                                                             |
| attacker                  | A person who tries to make a system do a dangerous operation, or to get its private data.                                                        |
| attention                 | The part of a model that calculates the effect of each input token on each output token.                                                         |
| attention weight          | A number in a model that tells how much one token has an effect on a different token.                                                            |
| backdoor                  | Code that lets an attacker go into a system after the attack.                                                                                    |
| blackmail                 | An attack on a person. The attacker makes the secrets of the person public if the person does not obey.                                          |
| blog                      | A web page where a person or a company publishes short articles in sequence.                                                                     |
| capability                | A value that lets a program do one operation. The program can do the operation only if it has the value.                                         |
| character smuggling       | An attack that puts instructions in characters that a person cannot see.                                                                         |
| classifier                | A model that examines an input and gives it one label from a set, for example "safe" or "attack".                                                |
| confidentiality           | The property of data that only the correct persons can read it.                                                                                  |
| confused deputy           | A program with permissions that does a dangerous operation for a person who does not have these permissions.                                     |
| constitution              | A list of rules that Anthropic uses in the training of Claude.                                                                                   |
| constrained decoding      | A method that lets the model give only the tokens that agree with a grammar or a schema.                                                         |
| containerd                | A program that runs containers, for example for Docker.                                                                                          |
| custom GPT                | A ChatGPT assistant with instructions and tools that a user adds.                                                                                |
| defense                   | A method that stops an attack or makes its damage smaller.                                                                                       |
| design pattern            | A known structure of a program that is a solution for a usual problem.                                                                           |
| direct prompt injection   | A prompt injection that the user writes into the conversation. The user is the attacker.                                                         |
| download                  | One copy of a package or a file that a person gets from a server.                                                                                |
| dual LLM pattern          | A structure of an agent in which one LLM reads the untrusted data and a different LLM has the tools.                                             |
| exfiltration              | The movement of private data from a system to an attacker.                                                                                       |
| first principle           | A basic fact. Other facts follow from it.                                                                                                        |
| guarantee                 | A property that stays correct because of the structure of a system, also if the LLM does an incorrect step.                                      |
| heuristic                 | A rule that usually gives a correct result, but not always.                                                                                      |
| homoglyph                 | A character that looks the same as a different character, for example the digit "0" and the letter "O".                                          |
| host mount                | A folder of the host that a container can read and write.                                                                                        |
| indirect prompt injection | A prompt injection from a different person in the data that the agent reads for its task. The user does not know about it.                       |
| information flow control  | A security method that follows the data from each source to each sink. It stops the data of a dangerous source at a sink that must stay safe.    |
| instruction hierarchy     | The instruction sources of a model in a sequence. The model obeys a higher source before a lower source, for example the system before the user. |
| integrity                 | The property of data that no attacker made it or changed it.                                                                                     |
| interpreter               | A program that reads the code of a different program and runs it step by step.                                                                   |
| jailbreak                 | A prompt that makes the model ignore its safety rules.                                                                                           |
| k-NN                      | A method that finds the nearest examples of an input in a set of known examples.                                                                 |
| lethal trifecta           | The risk of an agent that can read private data, can read untrusted data and can send data out. Simon Willison gave this name to the risk.       |
| lockfile                  | A file that records the correct version of each dependency of a project.                                                                         |
| misalignment              | The condition of a model that does operations that its developers do not want.                                                                   |
| mount namespace           | A function of the Linux kernel that gives a process a different view of the filesystem.                                                          |
| natural language          | A language of persons, for example English.                                                                                                      |
| object capability         | A permission that is an object. A program can do only the operations of the objects that it holds.                                               |
| pip                       | A tool that installs Python packages.                                                                                                            |
| privileged LLM            | The LLM of a dual LLM pattern that can use tools. It does not read the untrusted data.                                                           |
| probabilistic defense     | A defense in the model that decreases the risk of an attack. It cannot stop all attacks.                                                         |
| prompt injection          | An instruction in the data that an LLM reads, which tells the LLM to obey the instructions of an attacker.                                       |
| proxy                     | A server that gets requests from a client and sends them to a different server.                                                                  |
| quarantined LLM           | The LLM of a dual LLM pattern that reads the untrusted data. It cannot use tools.                                                                |
| raw socket                | A network socket that lets a program send and read packets directly.                                                                             |
| red team                  | A group of persons or a model that does attacks on a system to find its weak points before attackers do.                                         |
| rug pull                  | An attack in which a server changes its tool definitions after the user accepts the server.                                                      |
| sabotage                  | Damage that a person or an agent causes and wants to cause.                                                                                      |
| seccomp                   | A function of the Linux kernel that sets limits on the system calls of a process.                                                                |
| security                  | The protection of a system and its data from attackers.                                                                                          |
| sink                      | A point where data goes out of a system or causes an operation, for example a tool call that sends an email.                                     |
| social engineering        | An attack that gives incorrect information to a person or an agent. Then the person or the agent does a dangerous operation.                     |
| spotlighting              | A defense that puts a mark on the untrusted data in the Prompt.                                                                                  |
| success rate              | The percent of attacks that get the result that the attacker wants.                                                                              |
| supply chain              | The dependencies of a system, for example packages, plugins and services, and the persons who make them.                                         |
| system card               | A document in which a company tells the properties and the risks of its model.                                                                   |
| taint                     | A label on the data from a source that the system does not control. Information flow control puts it on each value that the data changes.        |
| threat model              | The attacks that a system must stop, and the operations that the attacker can do.                                                                |
| to pay                    | To give a company a quantity of value for a product or a service.                                                                                |
| to trust                  | To think that a source or a program is safe.                                                                                                     |
| tool poisoning            | An attack that puts dangerous instructions into the tool descriptions of a server.                                                               |
| trust boundary            | The point where data from a source that the system does not control goes into a part of the system that trusts its input.                        |
| typosquatting             | An attack that publishes a package with a name that is almost the same as the name of a known package.                                           |
| untrusted data            | Data from a source that the system does not control, for example a web page, an email or a tool result.                                          |
| year                      | A unit of time of approximately 8766 hours.                                                                                                      |

## Names

| Term                    | Meaning                                                                                                                                 |
| ----------------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| ACL Findings            | A series of papers that persons who do research on language and AI show at one time and location.                                       |
| ACP                     | The Agent Client Protocol: a standard protocol that connects an agent to an IDE.                                                        |
| Action-Selector         | A design pattern in which the LLM selects one operation from a list and does not read the result.                                       |
| Agent-as-Tool           | A mechanism in which a Session operates as a Tool, and a different Session calls it.                                                    |
| AgentDojo               | A benchmark that measures if agents stop prompt injection.                                                                              |
| AgentGuardian           | A defense that makes an access control policy from the operations of an agent.                                                          |
| AgentHarm               | A benchmark that measures if agents do dangerous tasks.                                                                                 |
| Aim Labs                | The security research group of the company Aim Security.                                                                                |
| Aim Security            | A security company for AI products.                                                                                                     |
| Alice                   | A person in the example of this case study.                                                                                             |
| Anthropic               | The company that makes the Claude models and Claude Code.                                                                               |
| AppArmor                | A security module of the Linux kernel that sets limits on the files and the operations of a program.                                    |
| Apple                   | A company that makes computers.                                                                                                         |
| April                   | Month 4 of the year.                                                                                                                    |
| arXiv                   | A public web service where persons publish research papers before other persons examine them. Each paper has a number.                  |
| ASR                     | "Attack success rate": the part of the attack tests in which the attack gets its result, in percent.                                    |
| Astro                   | A web framework for JavaScript. Its developers are the authors of Flue.                                                                 |
| Atlas                   | The browser of OpenAI with an agent.                                                                                                    |
| Atropos                 | The framework of Nous Research for reinforcement learning. It runs the rollouts and the reward functions.                               |
| Auto Mode               | A mode of Claude Code in which the agent can do more steps without approval.                                                            |
| AutoGen                 | A framework of agents. Its agents operate through conversations with each other.                                                        |
| Azure                   | A service of Microsoft that gives access to models on its servers.                                                                      |
| Azure AI Content Safety | The service of Microsoft that examines the inputs and the outputs of models for safety.                                                 |
| Azure Prompt Shields    | The service of Microsoft that finds prompt injections in the inputs of a model.                                                         |
| Bargury                 | A person who does research on security.                                                                                                 |
| bash                    | A Unix shell. An agent with a bash tool can run each command of the shell.                                                              |
| Bedrock                 | A service that gives access to the models of different companies on its servers.                                                        |
| Bedrock Converse        | The API of a service that gives the models of different companies through one interface.                                                |
| Bell-LaPadula           | A security model. Each item has a level, and data cannot go from a high level to a low level.                                           |
| Beurer-Kellner          | A person who does research on security.                                                                                                 |
| Black Hat USA           | A time and a location where persons who do security research come together and show their work.                                         |
| Bob                     | A person in the example of this case study.                                                                                             |
| Box                     | A company that stores the documents of other companies.                                                                                 |
| BrowseSafe              | A set of defenses for agents that use a browser.                                                                                        |
| ByteRover               | A Memory service for agents.                                                                                                            |
| C2PA                    | An organization and its standard specification. The specification tells the source of an image or of a different file with a signature. |
| CaMeL                   | A defense of DeepMind. It gives labels to the data and stops a tool call that sends untrusted data to a dangerous tool.                 |
| Carlini                 | A person who does research on security.                                                                                                 |
| Cato CTRL               | The security research group of a company.                                                                                               |
| Cerebras                | A company that makes special computers for models. It gives access to models on these computers.                                        |
| Charlie                 | A person in the example of this case study.                                                                                             |
| Chat Completions        | An API of OpenAI. It receives a list of messages and gives the next message.                                                            |
| ChatGPT                 | The chat assistant of OpenAI.                                                                                                           |
| China                   | A large area with many persons.                                                                                                         |
| CI/CD pipeline          | A pipeline that builds, tests and deploys code automatically after each change.                                                         |
| CISO                    | "Chief information security officer": the person who controls the security of a company.                                                |
| Claude                  | The models of Anthropic.                                                                                                                |
| Claude.ai               | The chat assistant of Anthropic on the web.                                                                                             |
| Cloud Hypervisor        | A program that runs microVMs.                                                                                                           |
| Cloudflare              | A company that operates a network of servers and sandboxes.                                                                             |
| Code Interpreter        | The tool of ChatGPT that runs Python code in a sandbox.                                                                                 |
| Code-Then-Execute       | A design pattern in which the LLM writes a program, and a different system runs the program.                                            |
| Codex                   | The coding agent of OpenAI.                                                                                                             |
| Cohen                   | The first author of a paper about CaMeL in companies.                                                                                   |
| Composer                | The mode of Cursor that changes two or more files for one request.                                                                      |
| Content Security Policy | A rule set of a web page that tells which servers the browser can get data from.                                                        |
| Context-Minimization    | A design pattern that removes untrusted data from the context before the next LLM call.                                                 |
| Copilot                 | The assistants of Microsoft, for example GitHub Copilot.                                                                                |
| Copilot Studio          | The service of Microsoft that makes Copilot agents.                                                                                     |
| Copilot Workspace       | A product of GitHub that does tasks on code in a container on a server of GitHub.                                                       |
| Costa                   | A person who does research on security.                                                                                                 |
| CrewAI                  | A framework of agents. Each agent has a function in a group of agents.                                                                  |
| CSP                     | "Content Security Policy": the rules of a web page about the servers that the browser can get data from.                                |
| Cursor                  | A coding agent in a program that changes code.                                                                                          |
| CurXecute               | A security problem of Cursor. A prompt injection could make a new MCP configuration without approval.                                   |
| CVE                     | The number of a known security problem in a public list.                                                                                |
| CVSS                    | A number from 0 to 10 that tells how dangerous a security problem is.                                                                   |
| Daytona                 | A company that operates sandboxes for agents.                                                                                           |
| Debenedetti             | A person who does research on security.                                                                                                 |
| December                | Month 12 of the year.                                                                                                                   |
| DeepMind                | The research organization of Google that makes models.                                                                                  |
| DeepSeek-R1             | A model with public weights.                                                                                                            |
| DEF CON                 | A time and a location where persons who do security research come together and show their work.                                         |
| Defender                | The security products of Microsoft.                                                                                                     |
| Deno                    | A runtime for JavaScript that gives no access to files or to the network without a permission.                                          |
| Deriver                 | The worker of Honcho that makes conclusions from each new batch of messages.                                                            |
| Dialectic               | The part of Honcho that makes the reply to each query.                                                                                  |
| DictAttack              | An attack that puts a dangerous payload in the grammar of constrained decoding.                                                         |
| Discord                 | A chat platform.                                                                                                                        |
| Doom                    | A computer game.                                                                                                                        |
| Dreamer                 | The worker of Honcho that makes deeper conclusions when no new messages come from a peer.                                               |
| Durable Object          | A service of Cloudflare that keeps the data of one item at one location. Each Durable Object has a SQLite database.                     |
| EchoLeak                | A security problem of Microsoft 365 Copilot. An email could send private data to an attacker also if the user did not open it.          |
| Edoardo Debenedetti     | A person who does research on security. This person is the first author of CaMeL and AgentDojo.                                         |
| Eric Wallace            | A person at OpenAI. This person is the first author of the paper about the instruction hierarchy.                                       |
| ETH Zurich              | A technical research organization.                                                                                                      |
| Express                 | A small JavaScript library for HTTP servers. The developer adds each other part.                                                        |
| F1                      | A number from 0 to 1 that tells the precision of a test. It counts the incorrect results of the two types.                              |
| F1 score                | A number from 0 to 1. It measures how correctly a classifier finds the items that it must find.                                         |
| FastAPI                 | A Python framework for HTTP APIs.                                                                                                       |
| Fides                   | A defense that gives security labels to the data of an agent and examines them before each tool call.                                   |
| Field Effect            | A security company.                                                                                                                     |
| Findings                | A group of papers that NAACL publishes each year.                                                                                       |
| Firecracker             | A program that runs microVMs in the cloud.                                                                                              |
| Flue                    | A framework that runs agents in sandboxes on the servers of a company.                                                                  |
| Fortune                 | A company that publishes articles about companies.                                                                                      |
| Galileo                 | A company that makes a classifier for the safety of LLM inputs.                                                                         |
| Gateway                 | The part of Hermes that connects the agent to platforms for chat.                                                                       |
| Gemini                  | The models of Google.                                                                                                                   |
| Gemini CLI              | The coding agent of Google. Its frontend is a terminal.                                                                                 |
| Gemini CloudCode        | An API of Google for the Gemini models.                                                                                                 |
| GenAI                   | AI models that make new text, images or code.                                                                                           |
| GitHub                  | The web service of Microsoft for the repositories of code.                                                                              |
| GitHub Action           | A workflow that GitHub runs automatically after a change in a repository.                                                               |
| GitHub Actions          | The service of GitHub that runs tasks in a repository at specified times or after changes in the repository.                            |
| GitLab                  | A web service for the repositories of code. It is almost the same as GitHub.                                                            |
| Google                  | The company that makes the Gemini models.                                                                                               |
| Google ADK              | The framework of agents of Google.                                                                                                      |
| Google Drive            | A service of Google that stores files.                                                                                                  |
| Google Workspace        | The services of Google for work, for example email and documents.                                                                       |
| GPT-4                   | A model of OpenAI from 2023.                                                                                                            |
| GPT-5                   | A model of OpenAI.                                                                                                                      |
| GPT-OSS-120B            | A model of OpenAI with public weights.                                                                                                  |
| GPU pod                 | A computer of a company with special parts that run models quickly. Persons can use it for a specified time.                            |
| Gray Swan               | A company that does red team tests of AI models.                                                                                        |
| Groq                    | A company that gives access to models on special computers.                                                                             |
| GroupChat               | A Topology in which all the Sessions use the same conversation.                                                                         |
| gVisor                  | A sandbox of Google for containers. It isolates the container from the operating system of the host.                                    |
| HashJack                | An attack that puts instructions in the part of a URL after the hash sign.                                                              |
| Hermes                  | The short name of Hermes Agent. Nous Research also gives this name to its models.                                                       |
| Hermes Agent            | A framework for an agent of one person. The agent operates all the time and keeps Memory across Sessions.                               |
| Hindsight               | A Memory service for agents.                                                                                                            |
| Hines                   | A person at Microsoft. This person is the first author of the spotlighting paper.                                                       |
| Holographic             | A Memory service for agents.                                                                                                            |
| Honcho                  | A Memory service for agents. It keeps facts about each person and each agent.                                                           |
| Honcho session          | A thread of messages between peers in Honcho. It is not the same as a Session of the theory.                                            |
| ICLR                    | A time and a location where persons who do AI research come together and show their papers.                                             |
| IDE                     | A program where a developer writes, changes and runs code.                                                                              |
| IMAP                    | A protocol that reads email from a server.                                                                                              |
| InjecAgent              | A benchmark of indirect prompt injection attacks against agents that use tools.                                                         |
| Invariant Labs          | A security company for AI agents.                                                                                                       |
| Japan                   | A large area with many persons.                                                                                                         |
| Jason Lemkin            | A person who writes about software companies.                                                                                           |
| Jira                    | A web service that keeps the tasks of a group of persons.                                                                               |
| Johann Rehberger        | A person who does research on security and writes about attacks on AI products.                                                         |
| JSONL                   | A text format. Each line of the file is one JSON object.                                                                                |
| July                    | Month 7 of the year.                                                                                                                    |
| June                    | Month 6 of the year.                                                                                                                    |
| Kata Containers         | A system that runs each container in a microVM.                                                                                         |
| Korea                   | A large area with many persons.                                                                                                         |
| Kubernetes              | A system that runs containers on a group of servers.                                                                                    |
| Köpf                    | A person who does research on security.                                                                                                 |
| LangChain               | The company and the library that make LangGraph.                                                                                        |
| LangGraph               | A framework of agents that shows each agent as a graph of nodes.                                                                        |
| LangSmith               | The service of LangChain that records the traces of agents and examines them.                                                           |
| LDAP                    | A protocol that gives the users of an organization and their groups.                                                                    |
| LibreChat               | A chat program with public source code that connects to MCP servers.                                                                    |
| LiteLLM                 | A server that sends requests to the APIs of different model providers.                                                                  |
| LiteLMGuard             | A small classifier model that finds dangerous prompts.                                                                                  |
| LLM Map-Reduce          | A design pattern in which an isolated LLM call reads each item of untrusted data, and a different step collects the results.            |
| Luca Beurer-Kellner     | A person who does research on security. This person is the first author of a paper about patterns for the security of LLM agents.       |
| M365                    | Microsoft 365: the products of Microsoft for email, documents and tables.                                                               |
| Maksym Andriushchenko   | A person who does research on security. This person is the first author of AgentHarm.                                                   |
| Manuel Costa            | A person at Microsoft Research. This person is the first author of Fides.                                                               |
| March                   | Month 3 of the year.                                                                                                                    |
| Mariner                 | An agent of Google that operates a browser.                                                                                             |
| Mario Zechner           | The author of PI.                                                                                                                       |
| May                     | Month 5 of the year.                                                                                                                    |
| MCP                     | The Model Context Protocol: a standard protocol that connects an agent to the tools of a different program.                             |
| MCP Inspector           | A tool that examines MCP servers.                                                                                                       |
| Mem0                    | A Memory service for agents.                                                                                                            |
| Michael Bargury         | A person at Zenity who does research on security.                                                                                       |
| Microsoft               | The company that makes GitHub Copilot.                                                                                                  |
| Microsoft Defender      | The security product of Microsoft that monitors computers and services.                                                                 |
| Microsoft Research      | The research organization of Microsoft.                                                                                                 |
| Microsoft Teams         | The chat service of Microsoft for organizations.                                                                                        |
| microVM                 | A small virtual machine that starts quickly.                                                                                            |
| MiniScope               | A defense that gives each task of an agent a smaller set of permissions.                                                                |
| Mistral                 | A company that makes models.                                                                                                            |
| Modal                   | A company that runs code and sandboxes on its servers for its customers.                                                                |
| NAACL                   | An organization of research on language and computers. It publishes papers each year.                                                   |
| NAACL Findings          | A series of papers that persons who do research on language and AI show at one time and location.                                       |
| NeurIPS                 | A time and a location where persons who do AI research come together and show their papers.                                             |
| NeuroTaint              | A method of information flow control for LLM agents that follows the effect of data in the model.                                       |
| NIST                    | An organization that writes technical standard documents.                                                                               |
| Nous Research           | The company that makes Hermes Agent and the Hermes models.                                                                              |
| npm                     | A tool that installs JavaScript packages.                                                                                               |
| OAuth                   | A standard protocol that gives a program a small set of permissions for the data of a user on a different service.                      |
| October                 | Month 10 of the year.                                                                                                                   |
| OneDrive                | A service of Microsoft that stores files.                                                                                               |
| OpenAI                  | The company that makes ChatGPT, Codex and models, for example GPT-5.                                                                    |
| OpenAI Agents SDK       | The framework of OpenAI for agents.                                                                                                     |
| OpenRouter              | A service that gives access to the models of many providers with one API.                                                               |
| OpenViking              | A Memory service for agents.                                                                                                            |
| Opus                    | The largest Claude models.                                                                                                              |
| OS-Harm                 | A benchmark that measures if agents that control a computer do dangerous tasks.                                                         |
| OWASP                   | An organization that publishes lists of the most dangerous risks of software.                                                           |
| OX Security             | A security company for software supply chains.                                                                                          |
| Pasquini                | The first author of a paper about adaptive attacks.                                                                                     |
| PI                      | A small coding agent. Its frontend is a terminal. Its code shows each concept of the theory as a point where a developer can add code.  |
| Pillar Security         | A security company for AI products.                                                                                                     |
| Plan-Then-Execute       | A design pattern in which the LLM makes a plan of tool calls before it reads untrusted data.                                            |
| Plastic Labs            | The company that makes Honcho.                                                                                                          |
| Pro                     | A large version of a Gemini model.                                                                                                      |
| Prompt Shields          | A service of Microsoft that finds prompt injection in the inputs of a model.                                                            |
| Prompt slot             | A part of the Prompt that gets text: the system prompt, the messages or the tool schemas.                                               |
| PromptArmor             | A security company for AI products.                                                                                                     |
| PromptGuard             | A small classifier model that finds prompt injections.                                                                                  |
| Pyodide                 | Python that runs in WASM.                                                                                                               |
| PyRIT                   | A red team tool of Microsoft for LLM programs.                                                                                          |
| Python                  | A language for programs.                                                                                                                |
| Qiusi Zhan              | A person at UIUC. This person is the first author of InjecAgent.                                                                        |
| R2                      | The file storage service of Cloudflare.                                                                                                 |
| RAG                     | A method that finds the documents for a query and adds them to the Prompt before the LLM call.                                          |
| RCE                     | "Remote code execution": an attack that runs the code of an attacker on a different computer.                                           |
| ReAct                   | A method in which an agent writes its reasoning before each tool call.                                                                  |
| Reconciler              | The worker of Honcho that writes a copy of its data in external indexes and corrects the differences.                                   |
| Redis                   | A database that keeps its data in memory.                                                                                               |
| Replit                  | A company that operates a web service where an agent writes and runs code.                                                              |
| RetainDB                | A Memory service for agents.                                                                                                            |
| Seatbelt                | The sandbox of the operating system of the computers of Apple.                                                                          |
| SharePoint              | A service of Microsoft that stores the documents of a company.                                                                          |
| Shumailov               | A person who does research on security.                                                                                                 |
| Siddiqui                | The first author of a paper about information flow control for LLMs.                                                                    |
| Simon Willison          | A developer who writes about the security of programs that use LLMs.                                                                    |
| Singularity             | A program that runs software in containers on computers that many persons use.                                                          |
| Slack                   | A chat service for organizations.                                                                                                       |
| Slack channel           | A group of messages in Slack with a name.                                                                                               |
| SMTP                    | A protocol that sends email.                                                                                                            |
| Snyk                    | A company that makes security tools for developers.                                                                                     |
| Solana                  | A public network that records the value that each person has.                                                                           |
| SPY Lab                 | A security research group of ETH Zurich.                                                                                                |
| SQL injection           | An attack that puts SQL instructions into the data of a database query.                                                                 |
| SQLite                  | A small database in one file.                                                                                                           |
| SSE                     | A protocol that lets an HTTP server send a sequence of messages to a client on one open connection.                                     |
| SSH                     | A protocol that lets a person or a program run commands on a different computer through the network.                                    |
| SSO                     | A service that identifies a person one time for all the services of an organization.                                                    |
| STDIO                   | The standard input and output of a process. MCP can send messages through it.                                                           |
| stelint                 | The tool that checks the documents of this book against the rules of STE and against this dictionary.                                   |
| Supermemory             | A Memory service for agents.                                                                                                            |
| Supervisor              | A Topology in which one Session gives tasks to other Sessions and collects their results.                                               |
| Swarm                   | A Topology in which each Session can give the task to a different Session, and no Session controls the other Sessions.                  |
| SWE-agent               | A coding agent from research. Its paper shows how the tools change the results of the agent.                                            |
| Tailwind                | A library that gives the format and the colors of web pages.                                                                            |
| TaintBench              | A benchmark that measures methods of information flow control for LLM agents.                                                           |
| Task tool               | The tool of Claude Code that starts a subagent and gives its output back as a tool result.                                              |
| Teams                   | The chat service of Microsoft.                                                                                                          |
| Telegram                | A chat platform.                                                                                                                        |
| The Decoder             | A web service with articles about AI.                                                                                                   |
| Thinking Machines Lab   | A company that makes models and services for the training of models.                                                                    |
| Tinker                  | The post-training service of Thinking Machines Lab.                                                                                     |
| Tramèr                  | A person who does research on security.                                                                                                 |
| UIUC                    | A research organization.                                                                                                                |
| UK AISI                 | A public organization that examines the security of AI models.                                                                          |
| Unicode normalization   | The change of Unicode characters with the same meaning into one standard shape.                                                         |
| V8 isolate              | An isolated part of the JavaScript runtime of Google.                                                                                   |
| Vercel                  | A company that operates web services and sandboxes for its customers.                                                                   |
| Vercel AI Gateway       | A service of Vercel that gives access to the models of many providers with one API.                                                     |
| VS Code                 | A program of Microsoft that persons use to write and change code.                                                                       |
| Wallace                 | A person who does research on models at OpenAI.                                                                                         |
| Wang                    | The first author of the NeuroTaint paper.                                                                                               |
| WASM                    | A format of code that runs in a runtime that isolates it, for example in a browser.                                                     |
| Watch Mode              | A mode of Atlas that stops the agent if the user does not look at it.                                                                   |
| WebExtensions           | The format for the extensions of web browsers.                                                                                          |
| xAI                     | A company that makes models.                                                                                                            |
| XML                     | A text format. It puts a name in special signs before and after each part of the text.                                                  |
| XPIA                    | The name that Microsoft gives to indirect prompt injection. Microsoft also gives this name to its classifier that finds such attacks.   |
| Zenity                  | A security company for AI agents.                                                                                                       |
| Zhan                    | A person who does research on security.                                                                                                 |
| Zhang                   | The first author of two papers about attacks on LLMs.                                                                                   |
