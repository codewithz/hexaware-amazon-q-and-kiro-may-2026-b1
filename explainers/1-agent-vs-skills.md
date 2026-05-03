Agent vs Skill — What's the Difference?
The one-sentence version
An Agent is who does the work. A Skill is how to do a specific task.

Agent — a specialist with a role and a toolbelt
When you define a custom agent in .kiro/agents/, you are creating a persistent, named specialist with:

A fixed identity — it always behaves as a code reviewer, or a test writer, or a docs writer
Controlled tool access — you decide exactly what it can touch: read only, or read + write, or read + write + shell
An invocation name — you call it explicitly with @agent-name

The code-review-agent you built in Lab 5 can only read. Even if someone asks it to fix the code, it cannot — the tool restriction is architectural, not just instructional.
Agent = identity + constraints + always available

Skill — a procedure manual loaded on demand
A Skill in .kiro/skills/ is a reusable instruction package with no identity of its own. It contains:

A description that tells Kiro when to activate it
Detailed instructions for one specific task or workflow
Optional scripts, reference docs, or templates

The key property: a Skill is dormant by default. Kiro loads only its name and description at startup. The full instructions only enter the context window when you invoke the skill — manually with /skill-name, or automatically when your prompt matches the description.
Skill = instructions + on-demand loading + portable

Side-by-side comparison
Custom AgentSkillLives in.kiro/agents/.kiro/skills/ or ~/.kiro/skills/Has an identityYes — named specialistNo — pure instructionsTool access controlYes — read, write, shellNo — inherits active contextHow you invoke it@agent-name explicitly/skill-name or auto-matchLoaded in contextAlways availableOnly when activatedPortable / shareableNo — project-specificYes — follows open Agent Skills standardCan be triggered by a HookYes — hook calls the agentNo — hooks don't invoke skills

When to use which
Use an Agent when:

You need to enforce a tool boundary (e.g. reviewer must never write files)
You want a persistent specialist invoked by name across many sessions
A Hook needs to delegate work to a specific role
You want Kiro to auto-select the right specialist based on the task type

Use a Skill when:

You have a reusable workflow you want to share across projects or teams
The instructions are task-specific and don't need to be loaded all the time
You want to import community-built procedures from GitHub
You need a personal workflow available in every project (global scope)


The analogy
Think of a hospital. The Agents are the doctors and nurses — they have defined roles, access badges, and are always on duty. A surgeon can enter the operating theatre (write access); a consultant can only review the chart (read access).
The Skills are the clinical procedures manual — the step-by-step guide for how to perform a lumbar puncture, or how to assess a stroke. Any doctor can pick up the relevant procedure when they need it. The manual doesn't have a role itself; it just carries the instructions.
A @code-review-agent (the doctor) might activate the spring-code-review skill (the procedure manual) when reviewing a Spring Boot class. The agent provides the role and the constraints. The skill provides the detailed steps.

Can they work together?
Yes — and that is the intended pattern in Kiro. A Hook fires → delegates to an Agent → the Agent activates a matching Skill for the detailed workflow. Each layer does one job:
Hook          → when to act        (fileEdited, postTaskExecution)
Agent         → who acts           (@docs-agent, @security-agent)
Skill         → how to act         (/spring-code-review, /api-doc-generator)
Steering      → what to know       (architecture.md, testing-standards.md)