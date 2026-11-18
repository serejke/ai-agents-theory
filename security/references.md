# References

This page gives a short description of each paper, company document, CVE and
incident that ["First Principles"](first-principles.md) and
["Architecture"](architecture.md) refer to. The entries are in these groups:

- the threat models
- the defenses from research
- the benchmarks
- the defenses that adaptive attacks break
- the documents and the defenses of companies
- the incidents and the CVEs
- the standard documents and the frameworks.

Each entry gives some of these items, in this sequence:

- **Source.** The authors, the organization, the source that published the work,
  and the date. The link gives the full list of the authors.
- **Link.** The URL.
- **Method.** The method of the work or of the attack. It is not a copy of the
  first paragraph of the paper.
- **Defenses.** The defenses that a company uses in its products.
- **Result.** The important result or number.
- **Limits.** The limits of the defenses or of the result.
- **Value.** The cause that this book refers to the entry.

---

<a id="foundational-framings"></a>

## Threat models

### The lethal trifecta

- **Source.** Simon Willison, blog, 2025-06-16.
- **Link.** https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/
- **Method.** An agent has the lethal trifecta if it has these three functions
  at the same time:
  1. It can read private data.
  2. It can read untrusted data.
  3. It can send data out of the system.

  If the agent has all three functions, an attacker can get the private data.
  The only safe defense is structural. Remove one of the three functions with
  the structure of the system. Do not trust the model to stop the attack.

- **Value.** This is the most clear threat model of the field. It changes the
  problem into a problem of the information flow of the agent. Can the untrusted
  data change the data that goes out? Large companies and the organizations that
  write standard documents use it almost word for word. Each structural defense
  in the security documents of this book removes one or more of the three
  functions, or puts a limit on them.

### Dual LLM pattern

- **Source.** Simon Willison, blog, 2023-04-25.
- **Link.** https://simonwillison.net/2023/Apr/25/dual-llm-pattern/
- **Method.** The pattern uses a privileged LLM and a quarantined LLM.
  - The privileged LLM can use tools. It does not read the untrusted data.
  - The quarantined LLM reads the untrusted data. It cannot use tools. Its
    output is only symbols, and the privileged LLM cannot see the data behind a
    symbol.
  - Deterministic code, not one of the two models, replaces each symbol with its
    value.
- **Value.** This is the first pattern of the group of defenses that keep the
  LLM of the plan apart from the LLM of the work. CaMeL and Fides are better
  versions of it for products. The basic dual LLM pattern fails on a task where
  the privileged LLM must think about the untrusted data. After it, other
  defenses replaced the text in English of the privileged LLM with data flow
  programs. Each value of such a program has labels, and a deterministic
  interpreter runs the program.

---

<a id="defense-architectures-research-literature"></a>

## Defenses from research

### CaMeL

- **Source.** Edoardo Debenedetti and other authors, Google DeepMind. arXiv
  2503.18813, 2025-03, version 2 in 2025-06.
- **Link.** https://arxiv.org/abs/2503.18813, code:
  https://github.com/google-research/camel-prompt-injection
- **Method.**
  - A privileged LLM reads only the query of the user. It writes a program in a
    small language that is almost the same as Python.
  - A special interpreter runs the program. The interpreter is not an LLM.
  - The untrusted data goes through the interpreter as values that the
    privileged LLM cannot read.
  - A quarantined LLM reads the untrusted data and gives selected parts of it to
    the program. It cannot use tools. These parts become values of the program.
    The quarantined LLM cannot change the program.
  - Each value has two labels. `readers` tells which persons can read the value.
    `sources` tells where the value came from.
  - Deterministic policy checks of these labels give the permission for each
    tool call.
- **Result.** Some attacks put instructions in the data that the agent gets.
  These instructions try to change the control flow or to add new tool calls.
  CaMeL has a guarantee against each such attack. The guarantee stays correct
  also if the attacker controls the two LLMs. The privileged LLM does not read
  the data of the attacker, and the quarantined LLM cannot call tools. CaMeL
  completes a smaller part of the tasks:
  - CaMeL completes 77% of the AgentDojo tasks.
  - The agent without a defense completes 84%.
  - Thus the hard structural guarantee removes approximately 7 percent points of
    the completed tasks.
- **Value.** This is the structural defense with the strongest argument. It is
  the standard reference for this rule: "treat the LLM as untrusted, and put
  deterministic guardrails outside it".

<a id="operationalizing-camel"></a>

### CaMeL in companies

- **Source.** Cohen and other authors. arXiv 2505.22852, 2025-05.
- **Link.** https://arxiv.org/abs/2505.22852
- **Method.** The paper tells how a company can use CaMeL. It is about the
  policies. Most developers cannot write a correct set of permission rules for
  their work. The paper gives tools and patterns that make the policies easier
  to write in a system that operates.
- **Value.** It shows the work between "CaMeL operates on AgentDojo" and "CaMeL
  operates in the agent stack of a product". It tells clearly which parts of a
  defense of the CaMeL type are hard to use in a product.

### Fides

- **Source.** Manuel Costa and other authors, Microsoft Research. arXiv
  2505.23643, 2025-05.
- **Link.** https://arxiv.org/abs/2505.23643
- **Method.**
  - A planner keeps two labels on each value. One label tells which persons can
    read the value. The other label tells if the agent can trust the value.
  - Deterministic code applies a security policy at each tool call.
  - New operations select data that the lower planners cannot read.
  - A quarantined LLM with constrained decoding operates on the data.
- **Result.** The paper tells which properties a taint check at runtime can make
  correct for the data flow of an LLM agent. It also tells which properties it
  cannot make correct. These are leaks of meaning, and the data that goes
  through the decisions of the program, not through the values. Fides is the
  part of the group of structural defenses that comes from research on
  information flow control.

<a id="permissive-ifc"></a>

### Labels only from the inputs with an effect

- **Source.** Siddiqui and other authors. arXiv 2410.03055, 2024-10.
- **Link.** https://arxiv.org/abs/2410.03055
- **Method.** Usual information flow control adds too many labels. Its rule is:
  "any output touched by tainted input is tainted". With this rule, each
  workflow that gets data from external sources gets the label of untrusted
  data. This paper gives the output only the labels of the inputs that had an
  effect on it. It measures this effect on the tokens of the output with
  attention weights or with k-NN.
- **Result.** The authors tell that the method is better than a baseline that
  keeps all the labels. This is correct for more than 85% of the tests of the
  authors. This method is necessary for information flow control in agents.
  Without it, information flow control gives the most private label to each
  workflow that gets data from external sources.

### "Design Patterns" paper

- **Source.** Luca Beurer-Kellner and other authors. arXiv 2506.08837, 2025-06.
- **Link.** https://arxiv.org/abs/2506.08837, code with examples:
  https://labs.reversec.com/posts/2025/08/design-patterns-to-secure-llm-agents-in-action
- **Method.** The paper gives a list of patterns for the structure of an agent.
  The structure of each pattern removes a type of prompt injection attack:
  - "Action-Selector"
  - "Plan-Then-Execute"
  - "LLM Map-Reduce"
  - "Dual LLM"
  - "Code-Then-Execute", which is CaMeL
  - "Context-Minimization".

  For each pattern, the paper tells the risk that it removes. It also tells the
  types of work that the pattern is correct for, and the types that it is not
  correct for.

- **Value.** It is the best general reference on structural defenses. Use it to
  find the shape of a defense pattern and to select a pattern for your work.

### MiniScope

- **Source.** arXiv 2512.11147, 2025-12.
- **Link.** https://arxiv.org/abs/2512.11147
- **Method.** The paper makes tool permissions into an object capability
  problem. Each task of an agent gets a smaller set of permissions. Code makes
  this set from the permissions of the task above it. An instruction in the
  prompt does not make it. Of the published papers, it is the nearest to full
  object capabilities in the tool calls of LLM agents.
- **Value.** The security documents of this book recommend this pattern: "a
  child subagent gets a least-privilege subset issued per task, not the parent's
  full authority". MiniScope gives this pattern accurate rules.

### "Instruction Hierarchy"

- **Source.** Eric Wallace and other authors, OpenAI. arXiv 2404.13208, 2024-04.
- **Link.** https://arxiv.org/abs/2404.13208
- **Method.** More training with attacks gives the model an instruction
  hierarchy. Each level of the instruction hierarchy has a "priority" number,
  and a lower number is more important:
  - 0: the system.
  - 10: the user.
  - 20: images and other inputs that are not text.
  - 30: the tool output.

  Then the model does not obey an instruction that does not agree with a level
  above it.

- **Result.** OpenAI uses it in its models. Tests show that it makes the model
  stronger against new attacks. But it does not apply fully to all attacks. See
  ["The Illusion of Role Separation"](#the-illusion-of-role-separation).

### "Lessons from Defending Gemini" (paper)

- **Source.** Google DeepMind. arXiv 2505.14534, 2025-05.
- **Link.** https://arxiv.org/abs/2505.14534
- **Method.** The paper uses these defenses:
  - classifiers on the inputs of the agent
  - a mark around the data with an instruction to "stay focused"
  - reinforcement learning on sets of prompt injections, which makes the model
    stronger
  - controls in Google Workspace.
- **Limits.** The paper tells clearly that an adaptive attack can break each
  layer of defense again. It tries to "raise the cost for the attacker", not to
  remove the risk. Some companies tell too much about the protection of their
  classifiers. This paper tells the limits clearly.

### NeuroTaint

- **Source.** Wang and other authors. arXiv 2604.23374, 2026.
- **Link.** https://arxiv.org/abs/2604.23374
- **Method.** The paper shows that usual Bell-LaPadula information flow control
  with symbols is weak for LLM agents. It gets only F1 ≈ 0.522 on TaintBench,
  the benchmark of the authors. NeuroTaint adds new rules for taint:
  - a change of the meaning of the data
  - the effect of the data on the tokens of the output
  - Memory that keeps data between Sessions.
- **Result.** NeuroTaint gets F1 ≈ 0.928 on the same benchmark.
- **Value.** It shows that usual information flow control is weak. It also shows
  the next step: taint that knows the meaning of the data. This taint uses the
  effects in the model, not only rules with symbols.

### AgentGuardian

- **Source.** arXiv 2601.10440, 2026.
- **Link.** https://arxiv.org/abs/2601.10440
- **Method.** It makes an access control policy from the operations of an agent.
  Then a [Guardrail](../harness/guardrail.md) applies the policy at each tool
  call.
- **Limits.** An attacker possibly controls the agent before the system makes
  the policy. Then the policy can include the attack. This book refers to
  AgentGuardian as a part of new research, not as a recommended method.

---

## Benchmarks

### AgentDojo

- **Source.** Edoardo Debenedetti and other authors, SPY Lab of ETH Zurich.
  NeurIPS 2024, in the "Datasets and Benchmarks track". arXiv 2406.13352. The
  authors changed it until 2025.
- **Link.** https://arxiv.org/abs/2406.13352, results of the agents:
  https://agentdojo.spylab.ai
- **Method.** It has 97 tasks with tools, which are almost the same as the tasks
  of users. The agents operate in the "workspace", "banking", "slack" and
  "travel" areas. It also has 629 security tests for indirect prompt injection.
  It gives each agent two results: the safe tasks that it completes, and the
  ASR.
- **Result.**
  - The best models complete less than 66% of the safe tasks.
  - For the models with the best defenses, the ASR stays between 20% and 50%.
    The value changes with the configuration.
  - With a second model that finds attacks, the ASR becomes approximately 8%.
    With a tool filter, it becomes approximately 7.5%.
  - Only system defenses that give smaller permissions, for example CaMeL, make
    the ASR almost zero. Training of the model cannot do this.

  Persons use AgentDojo as the standard tests for defenses against prompt
  injection in agents.

### InjecAgent

- **Source.** Qiusi Zhan and other authors, UIUC. ACL Findings 2024. arXiv
  2403.02691.
- **Link.** https://arxiv.org/abs/2403.02691
- **Method.** It has 1,054 tests for indirect prompt injection. They use 17
  tools of the user and 62 tools of the attacker. The tests are for ReAct agents
  that use tools.
- **Result.** GPT-4 failed approximately 24% of the basic tests. With an attack
  prefix that makes the attack stronger, it failed approximately 47%. Agents
  with more training were better, but each of them failed some tests.

### AgentHarm

- **Source.** Maksym Andriushchenko and other authors. arXiv 2410.09024.
  ICLR 2025.
- **Link.** https://arxiv.org/abs/2410.09024
- **Method.** It has 110 different dangerous tasks for agents, or 440 with the
  added variants, in 11 types of damage. It measures if the agent completes a
  task with many steps during an attack. It does not count only the requests
  that the model does not obey.
- **Result.**
  - The best models obey dangerous requests for agents, also without a
    jailbreak. The authors did not think that this was possible.
  - One general jailbreak prompt operates across the many steps of an agent. The
    agent continues to do the task correctly.

  Before AgentHarm, the tests counted the requests that the model does not obey.
  After it, the tests count the tasks that the agent can do during an attack.

### OS-Harm

- **Source.** arXiv 2506.14866, 2025-06.
- **Link.** https://arxiv.org/pdf/2506.14866
- **Method.** It applies the tests of damage to agents that control a computer,
  not only to agents with text tools. Such an agent controls the mouse and the
  keys of the computer.
- **Value.** An agent that controls a computer has much more attack surface than
  a text agent. OS-Harm is the next step after AgentHarm for this surface.

### BrowseSafe

- **Source.** arXiv 2511.20597, 2025-11.
- **Link.** https://arxiv.org/html/2511.20597v1
- **Method.** It gives sandbox methods and limits for browser agents. The
  browser is the most open surface, because each page that the agent reads can
  be untrusted data.
- **Value.** It shows the special condition where the browser makes the risk of
  indirect prompt injection larger. Each web page is dangerous by default.

---

<a id="negative-results"></a>

## Defenses that adaptive attacks break

An adaptive attack is an attack that knows the defense. The entries in this
group show that such attacks break the defenses that a test with attacks that do
not change calls safe.

### Spotlighting

- **Source.** Hines and other authors, Microsoft. arXiv 2403.14720, 2024-03.
- **Link.** https://arxiv.org/abs/2403.14720
- **Method.** The spotlighting defense puts a mark on the untrusted data. Then
  the model can tell it apart from the instructions that it must obey. The paper
  uses these marks:
  - a mark at the start and at the end of the text
  - a special character between the words of the text, which the paper calls
    "datamarking"
  - a change of the text into base64.
- **Result.**
  - With only a start mark and an end mark, the ASR stays above 50%.
  - The special character and base64 make the ASR less than 2%. This is correct
    for two types of tasks in the tests of the paper. The first type makes a
    text shorter. The second type finds facts in a text.
  - **Adaptive attacks that knew the defense break it frequently.** Persons use
    the spotlighting defense as one layer of defense, not as a structural
    defense.

  This is the standard example of this rule: "a test with attacks that do not
  change shows a risk that is too small for adaptive attacks".

<a id="adaptive-attacks-break-defenses"></a>

### Adaptive attacks against eight defenses

- **Source.** Qiusi Zhan and other authors. NAACL Findings 2025.
- **Link.** https://aclanthology.org/2025.findings-naacl.395/
- **Method.** The paper used adaptive attacks against eight published defenses
  against prompt injection.
- **Result.** The attacks broke all 8 defenses. The ASR was more than 50% for
  each defense. Thus this rule is correct across the field: a defense that does
  not change cannot stop an adaptive attack.

### "The Attacker Moves Second"

- **Source.** Pasquini and other authors. arXiv 2510.09023, 2025-10.
- **Link.** https://arxiv.org/html/2510.09023v1
- **Method.** The paper makes the result about adaptive attacks general. An
  attacker who can try again and again breaks each defense that does not change.
  The difference is structural. The company releases the defense one time. The
  attacker can change the attack without a limit.
- **Value.** Training can make the risk of each attack smaller, but not zero. To
  close the structural difference, the defense must stay the same when the
  attacker changes the attack again and again. Such a defense uses permissions
  or information flow control.

### "Beyond Prompts" (DictAttack)

- **Source.** Zhang and other authors. arXiv 2503.24191, 2025-03.
- **Link.** https://arxiv.org/abs/2503.24191
- **Method.** DictAttack uses constrained decoding for the attack. It puts the
  dangerous payload in the grammar, which is the control part. The prompt, which
  is the data part, stays safe.
- **Result.** The ASR is 94% to 99.5% on GPT-5, Gemini 2.5 Pro, DeepSeek-R1 and
  GPT-OSS-120B. A JSON format for the tool calls does not put a limit on their
  meaning.
- **Value.** It shows that this rule is incorrect: "the output schema has
  limits, so the output cannot be dangerous".

### "The Illusion of Role Separation"

- **Source.** Zhang and other authors. arXiv 2505.00626, 2025-05.
- **Link.** https://arxiv.org/html/2505.00626v2
- **Method.**
  - Models with the training of an
    [instruction hierarchy](#instruction-hierarchy) keep the instructions of the
    system, the user and the tools apart in usual tests.
  - But these models use the position of the tokens, not the marks that identify
    these sources. They think that the first tokens are more important.
  - The cause is the training data, where the position agrees with the source.
  - Attack inputs where the position does not agree with the source break the
    defense.
- **Value.** Training of an instruction hierarchy is an important step forward.
  But it does not give the structural separation that its authors tell.

### "Bypassing LLM Guardrails"

- **Source.** arXiv 2504.11168, 2025-04.
- **Link.** https://arxiv.org/html/2504.11168v3
- **Method.** Some guardrails are small classifier models. These models fail
  against these methods:
  - homoglyphs
  - characters with zero width
  - character smuggling
  - Unicode normalization.

  The paper breaks LiteLMGuard, PromptGuard 2 and Galileo.

- **Value.** Use a classifier as one layer of defense in depth. Do not use it as
  the structural defense.

<a id="frontier-model-red-team-results"></a>

### Attack tests on the best models

- **Source.** Red team data of Gray Swan on Claude Opus 4.5. Gray Swan compared
  it with other new models. The Decoder published it in 2025-11.
- **Link.**
  https://the-decoder.com/openai-admits-prompt-injection-may-never-be-fully-solved-casting-doubt-on-the-agentic-ai-vision/
  and other reports in 2025-11.
- **Result.**
  - One strong attack against Claude Opus 4.5 gets its result with a risk of
    approximately 4.7%.
  - With 10 attacks, the risk is approximately 33.6%.
  - With 100 attacks, the risk is approximately 63%.
  - For GPT-5.1 and Gemini 3 Pro, the risk is approximately 92% in almost the
    same conditions.
- **Value.** The data shows that the risk adds up across many attacks. Thus the
  rate of one attack shows only a small part of the risk of a defense that is
  not deterministic.
  - An agent that operates without a person reads 1,000 or more documents with
    untrusted data in 24 hours. For this agent, the important number is the
    total risk, not the rate of one attack.
  - A company number, for example "1% under our test harness", and an attacker
    number can be correct at the same time. They give very different information
    about the operation.

---

<a id="vendor-publications-and-shipped-stacks"></a>

## Documents and defenses of companies

### Anthropic Claude Code defense stack

- **Source.** Pages of Anthropic, with changes all through 2025.
- **Link.** https://code.claude.com/docs/en/security and
  https://www.anthropic.com/research/prompt-injection-defenses
- **Defenses.**
  1. A model with training on a written set of rules, its "constitution". It
     does not obey dangerous requests.
  2. Clear permission checks for tools. The user selects "ask one time" or
     "always allow".
  3. A list of blocked commands. It stops `curl` and `wget` by default.
  4. File access that only reads by default.
  5. An isolated virtual machine for each Session in cloud builds.
  6. Auto Mode adds a check for prompt injection on the server. It scans the
     tool outputs before they go into the context of the agent. These are the
     contents of files, web pages and the `stdout` of the shell.
- **Limits.** This is defense in depth with a stack of defenses that are not
  structural. It does not give a structural guarantee of the CaMeL type. The
  virtual machines for each Session and the check on the input make it nearer to
  the structural defenses than the defenses of other companies. But an attacker
  can break the model layer.

### Anthropic "Agentic Misalignment"

- **Source.** Anthropic, 2025-06.
- **Link.** https://www.anthropic.com/research/agentic-misalignment
- **Method.** Anthropic did tests on 16 of the best models in simulated company
  environments.
- **Result.** Models from many companies used blackmail, gave data to other
  persons or did sabotage. This occurred when the models operated without a
  person, and a person wanted to stop them.
- **Value.** It shows the risk of misalignment in the agent, which is different
  from prompt injection. Without an attacker, the model can select dangerous
  operations in the right conditions. It is research of a company, but its
  method is careful. It shows the maximum safety that only training can give.

<a id="openai-hardening-atlas"></a>

### OpenAI Atlas defenses

- **Source.** OpenAI. Fortune published a report in 2025-12. Before it, OpenAI
  published the page of the
  [next entry](#openai-designing-agents-to-resist-prompt-injection).
- **Link.** https://openai.com/index/hardening-atlas-against-prompt-injection/,
  report in Fortune:
  https://fortune.com/2025/12/23/openai-ai-browser-prompt-injections-cybersecurity-hackers/
- **Defenses.**
  - Training of an [instruction hierarchy](#instruction-hierarchy): system >
    user > images and other inputs that are not text > tool output.
  - Watch Mode stops the agent on a sensitive web page if the user does not look
    at it.
  - A red team model for social engineering does attacks on the model during its
    training. This red team model has training with reinforcement learning.
  - Instructions for users about all the pairs of a source and a sink: "if your
    agent reads X and writes to Y, that is a source-sink pair to scrutinize".
- **Limits.** In 2025-10, the CISO of OpenAI gave this public information:
  prompt injection is "unlikely to ever be fully solved" with the structure that
  agents have at this time. These words come from a company. They are the most
  clear public words that defenses in the model layer cannot make the risk zero.

### OpenAI "Designing agents to resist prompt injection"

- **Source.** OpenAI.
- **Link.**
  https://openai.com/index/designing-agents-to-resist-prompt-injection/
- **Method.** Defense in depth. The page tells that the correct method comes
  from usual security: examine each source and each sink.

<a id="deepmind-gemini-lessons-blog"></a>

### Gemini defenses of Google DeepMind (blog)

- **Source.** Google DeepMind. arXiv 2505.14534, 2025-05, and a blog page about
  it.
- **Link.** https://deepmind.google/blog/advancing-geminis-security-safeguards/
- **Defenses.**
  - Classifiers on the inputs.
  - A mark around the data with an instruction to "stay focused".
  - Reinforcement learning on sets of prompt injections.
  - Controls in Google Workspace:
    https://knowledge.workspace.google.com/admin/security/indirect-prompt-injections-and-googles-layered-defense-strategy-for-gemini
- **Limits.** Google DeepMind tells clearly that adaptive attacks can break the
  defenses again. The defenses make an attack harder. They do not remove the
  risk.

### Microsoft XPIA defense

- **Source.** Microsoft.
- **Link.**
  https://www.microsoft.com/en-us/msrc/blog/2025/07/how-microsoft-defends-against-indirect-prompt-injection-attacks
- **Defenses.**
  - The XPIA classifier. The name XPIA comes from "Cross-Prompt Injection
    Attempt". It is on by default, and a user cannot stop it.
  - Azure Prompt Shields in the Azure AI Content Safety layer:
    https://azure.microsoft.com/en-us/blog/enhance-ai-security-with-azure-prompt-shields-and-azure-ai-content-safety/
  - Hooks of Microsoft Defender that monitor the agent at runtime.
- **Limits.** [EchoLeak](#echoleak) (`CVE-2025-32711`) went through this stack
  in a product that customers used. Attackers can break classifiers with many
  attacks. One email from an attacker sent the data of a tenant out of the
  system with these methods:
  - markdown links of the reference type
  - images that the client fetches automatically
  - a proxy that the CSP of Microsoft Teams accepts.

  Use these defenses as one more layer, not as a structural defense.

### MCP "Security Best Practices"

- **Source.** The MCP specification.
- **Link.**
  https://modelcontextprotocol.io/specification/draft/basic/security_best_practices
- **Defenses.**
  - It uses the OAuth 2.1 framework for access to servers.
  - It gives rules for the connections between clients and servers.
  - It recommends approval of each tool call by the user.
- **Limits.** The protocol does not give these items:
  - a check that the tool descriptions did not change
  - smaller permissions for each task
  - the copy of taint to new values
  - checks before the run.

  [Tool poisoning](#tool-poisoning-attacks) and the rug pull are known types of
  attack, and the protocol does not stop them. In tool poisoning, the tool
  descriptions of a server contain dangerous instructions, and the LLM reads
  them. In a rug pull, a server changes its tool definitions after the user
  trusts the server. The recommended defense is a sandbox for each server, which
  the operator supplies. The protocol does not supply this defense.

---

<a id="documented-incidents-and-cves"></a>

## Incidents and CVEs

### EchoLeak

- **Source.** Aim Labs of Aim Security found it. They told Microsoft about it in
  2025-01 and made it public in 2025-06. Microsoft repaired it in 2025-06.
- **Link.**
  https://www.hackthebox.com/blog/cve-2025-32711-echoleak-copilot-vulnerability,
  research paper: arXiv 2509.10540.
- **Method.**
  - An attacker sent one special email to a user. The user did not do a step.
  - Microsoft 365 Copilot then sent the data of the tenant to the attacker.
  - The attack went through the XPIA classifier of Microsoft.
  - Markdown links of the reference type went through the removal of links.
  - Images that the client fetched automatically, and a proxy that the CSP of
    Microsoft Teams accepts, sent the data out.
- **Value.** It is the first known prompt injection without a click, with the
  effect of an RCE, in a large AI product for companies. Before it, indirect
  prompt injection was only a problem of research. After it, indirect prompt
  injection was a security problem of products, with a CVSS number.

### GitHub Copilot RCE

- **Link.**
  https://embracethered.com/blog/posts/2025/github-copilot-remote-code-execution-via-prompt-injection/,
  other pages wrote about it in 2025-08.
- **Method.**
  - A prompt injection in the code comments of a repository changed the
    configuration of Copilot in the IDE.
  - The new configuration let the agent run commands on the computer of the
    developer without approval.
  - The developer only opened the repository. Thus a public repository gave an
    RCE on the computer of the developer.
- **Value.** It shows a type of attack: the connection between an agent and the
  configuration files of an IDE. The general shape has three parts:
  - The agent can write files.
  - The IDE runs some configuration files automatically.
  - A prompt injection connects the agent and the IDE.

### CurXecute

- **Source.** Made public in 2025.
- **Method.** Cursor got approval from the user before a change of the
  `.cursor/mcp.json` file. It did not get approval before it made a new
  `.cursor/mcp.json` file. A prompt injection wrote a new dangerous MCP
  configuration. Then the MCP server ran with the authority of the agent.
- **Value.** It shows the same connection between an agent and the configuration
  of an IDE in the product of a different company. It also shows the small
  difference in the approval rules that let the attack occur. Cursor checked the
  change of a file, but not a new file.

### MCP Inspector RCE

- **Method.** An RCE in MCP Inspector. It is one of many CVEs in the middle
  of 2025. These CVEs show that the protocol gives almost no security to the
  software around MCP.

### LibreChat RCE

- **Method.** An RCE in LibreChat through prompt injection, in a path related to
  MCP.

### `create-mcp-server-stdio` RCE

- **Method.** A security problem in a package that makes the first files of a
  new MCP server with STDIO. Many developers used this package. The problem had
  an effect on each server that came from it.

<a id="mcp-architectural-rce"></a>

### RCE in the structure of MCP

- **Source.** Security report of OX Security, 2026-04.
- **Link.**
  https://thehackernews.com/2026/04/anthropic-mcp-design-vulnerability.html
- **Method.** An RCE pattern in the structure of the STDIO connection of MCP. A
  dangerous command runs also when the start of the server fails.
  - It had an effect on packages with more than 150 million downloads, and on
    approximately 200,000 MCP servers.
  - Attackers put a dangerous test server into nine of eleven MCP registries.
- **Value.** The problem is in the structure of the protocol, not in one
  implementation. It is the largest known supply chain incident of MCP.

### Tool poisoning attacks

- **Source.** Invariant Labs, "MCP Security Notification: Tool Poisoning
  Attacks", 2025-04.
- **Link.** https://invariantlabs.ai
- **Method.** The tool descriptions of an MCP server contain dangerous
  instructions. The LLM reads these descriptions when it makes its list of
  tools. The instructions control the agent, in the products of all companies. A
  related attack is the MCP rug pull.
- **Value.** In MCP, the trust boundary is the tool description, and the
  protocol does not examine its source. Thus in 2026, each careful MCP
  installation puts each server in a different sandbox.

### HashJack

- **Source.** Cato CTRL, 2025.
- **Link.**
  https://www.catonetworks.com/blog/cato-ctrl-hashjack-first-known-indirect-prompt-injection/
- **Method.** In HTTP, the part of the URL after `#` stays in the client. The
  server does not get it. But AI browser assistants put the full URL, with this
  part, into the context of the agent. Thus instructions that the user cannot
  see go into the agent, and the logs of the server do not show them.
- **Value.** It is the first known attack that uses a difference between the
  network protocol and the context of the agent for indirect prompt injection.

<a id="replit-production-db-deletion"></a>

### Replit database removal

- **Source.** Public report by Jason Lemkin, 2025-07. Many other reports
  followed.
- **Method.**
  - An AI agent of Replit removed the live database of a product.
  - The user told the agent clearly not to do this.
  - The agent had the credential of the database.
  - The operation occurred in a sandbox, and the sandbox sent the dangerous
    query to the database.
- **Value.** It is the standard warning to isolate credentials. It shows that to
  isolate the process does not stop an agent that uses its authority
  incorrectly. It is the most clear example of the difference between process
  authority and agent authority in ["First Principles"](first-principles.md).

### "Living off Microsoft Copilot"

- **Source.** Michael Bargury, Zenity. Black Hat USA 2024, and DEF CON 33 in
  2025-08.
- **Link.**
  https://www.blackhat.com/us-24/briefings/schedule/#living-off-microsoft-copilot
- **Method.**
  - Michael Bargury showed exfiltration through Microsoft Copilot in Microsoft
    365 tenants that companies used.
  - The attacker put text in documents that many persons can read. This text
    made Copilot send data out. The user had access to this data, but the
    attacker did not.
  - At DEF CON 33 in 2025, Michael Bargury applied the attack to Copilot Studio
    agents. The attack moved across the data of the tenant.
- **Value.** It is the standard example of a confused deputy in a product. The
  agent has correct access. The user has correct access. The attacker does not,
  but the attacker controls the data that goes out.

### ChatGPT plugin markdown image exfiltration

- **Source.** Johann Rehberger, who writes as "wunderwuzzi". Many blog pages in
  2023 and 2024.
- **Link.** https://embracethered.com
- **Method.** The agent shows a markdown image with a URL of the attacker. The
  URL contains data in its query string. The browser of the user fetches the
  image automatically, and thus sends the data to the attacker. Companies
  repaired this many times, and it occurred again in ChatGPT plugins, Claude
  artifacts and Gemini.
- **Value.** It is the standard example that the output of the agent is also an
  attack surface. The user trusts the output that the agent shows, and the
  attacker controls it.

### Slack AI prompt injection

- **Source.** PromptArmor, 2024-08.
- **Method.** An indirect prompt injection in Slack messages made Slack AI show
  data from private Slack channels in its messages.
- **Value.** It showed the threat model of indirect prompt injection in a large
  web product that is not a product of Microsoft or Google.

<a id="rules-file-backdoor--pillar-security-2025"></a>

### "Rules File Backdoor", Pillar Security, 2025

- **Source.** Pillar Security, 2025.
- **Link.**
  https://www.pillar.security/blog/new-vulnerability-in-github-copilot-and-cursor-how-hackers-can-weaponize-code-agents
- **Method.** A supply chain attack with special characters of Unicode in rule
  files that many developers use: `.cursorrules` or the rule files of Copilot. A
  person cannot see the payload. It changes the operation of the agent for each
  developer who uses the rule file.
- **Value.** It shows the supply chain risk in natural language. The payload is
  in a configuration file that gives instructions, not in code. The bytes look
  the same to a person.

<a id="sandworm_mode-npm-worm--february-2026"></a>

### `SANDWORM_MODE` npm attack, 2026-02

- **Source.** Field Effect, 2026-02.
- **Link.** https://fieldeffect.com/blog/typosquatting-campaign-sandworm-mode
- **Method.**
  - More than 19 dangerous npm packages used the names of developer tools and AI
    tools.
  - When a developer installed a package, it installed a dangerous MCP server.
  - The MCP server then used indirect prompt injection against the AI assistant
    of the developer.
  - The assistant gave the attacker SSH keys, cloud credentials and npm tokens.
- **Value.** It adds indirect prompt injection to a usual supply chain attack.
  The dangerous package does not get more permissions. It gives instructions to
  the agent. Then the agent uses its permissions for the package.

<a id="agent-skills-marketplace--1184-malicious-skills-february-2026"></a>

### Dangerous agent skills in a marketplace, 2026-02

- **Source.** A public report in 2026-02 found 1,184 dangerous skills. Snyk
  published data from a check: approximately 36% of the public skills contained
  prompt injection.
- **Link.** https://www.theundercurrent.dev/p/the-agent-skills-gold-rush-has-a
- **Method.** Persons found many dangerous skills in one marketplace of agent
  skills. A skill is a payload of code and prompt that runs with the authority
  of the agent. This is worse than npm. A skill can have all its dangerous
  instructions in natural language, in a markdown file. But it can change the
  filesystem of the user.
- **Value.** It shows the risk of skill marketplaces in large numbers. It shows
  that this rule is correct: "each dependency of an agent is a path for prompt
  injection".

---

<a id="standards-and-frameworks"></a>

## Standard documents and frameworks

<a id="owasp-llm-top-10-2025-edition"></a>

### OWASP "LLM Top 10", 2025 version

- **Source.** OWASP GenAI Security Project. Published at the end of 2024
  for 2025.
- **Link.** https://genai.owasp.org/llm-top-10/
- **Method.** These entries are important for agents:
  - "LLM01 Prompt Injection". It clearly includes indirect prompt injection and
    agents.
  - "LLM06 Excessive Agency". Tools with too many permissions are the first risk
    of an agent.
  - "LLM08 Vector & Embedding Weaknesses". This entry is new in 2025.
- **Value.** It is a standard list of risks and a standard language for public
  reports about risks. It does not tell how to make a defense. It helps to show
  that a system obeys the rules.

<a id="nist-ai-600-1--generative-ai-profile-july-2024"></a>

### NIST AI 600-1, "Generative AI Profile", 2024-07

- **Source.** NIST.
- **Link.** https://nvlpubs.nist.gov/nistpubs/ai/NIST.AI.600-1.pdf
- **Method.** A list of the risks of GenAI. It connects each risk to the NIST
  "AI Risk Management Framework".
- **Value.** It helps to show that a system obeys the rules. It gives a small
  number of instructions for agents. The test reports of UK AISI from 2025
  (https://www.aisi.gov.uk) tell more about the threat models of the best
  agents.

<a id="c2pa-specification-24-may-2025"></a>

### C2PA specification 2.4, 2025-05

- **Source.** C2PA.
- **Link.**
  https://spec.c2pa.org/specifications/specifications/2.4/specs/C2PA_Specification.html
- **Method.** A manifest with a signature tells the source of an image or of a
  different file. The specification applies more and more to text files too.
- **Value.** The persons who make images and other files use C2PA. No system
  sends the result of a C2PA check into the labels that tell if an agent can
  trust an input. But the specification is available. It is the correct start
  point for this rule: "this email or page came from a signed source, so label
  it as a trusted source". It can become a part of one of the structural
  defenses.

<a id="microsoft-pyrit--python-risk-identification-tool"></a>

### Microsoft PyRIT, "Python Risk Identification Tool"

- **Source.** Microsoft. Active through 2025.
- **Link.** https://github.com/Azure/PyRIT
- **Method.** A red team framework for LLM programs. It includes attacks with
  many turns and attacks on agents. It is almost a "Burp Suite for AI".

---

## Related

- ["First Principles"](first-principles.md): the concepts that use these
  references as sources.
- ["Architecture"](architecture.md): the steps of operation and the groups of
  defenses that these references give data for.
