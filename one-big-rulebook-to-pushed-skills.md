# Why I replaced one big rulebook with small pushed skills to make an LLM follow its rules, and how I keep it honest with a two-agent check pipeline

I gave an LLM one giant instruction file and told it to follow the rules. It could not. The file had grown past two thousand lines of behavioural rules, task procedures, and house conventions, all stuffed into one place the model was expected to read and obey. The real ask on every task was three things at once: hold all of that in the context window, select the rules that apply to this task, and then keep applying them for the full length of the task. It failed at all three, and it failed in a way that passed a glance and broke under inspection.

So I rebuilt the control layer around the model. The single file is gone. In its place is a small always-on preamble injected on every turn, a retrieval step that loads exactly one task procedure on demand, and two independent verifier agents that grade the work before and after it runs. This is why a monolithic instruction file breaks at the mechanism level, what I replaced it with, and the failure mode I have not closed.

## The four failure modes of a monolithic instruction file

I did not theorise the failure modes. I logged them from repeated runs and wrote them down as the anchor of the tracking issue for this work.

The first is capacity. Every token of instruction competes for the same finite context and the same finite attention. A two-thousand-line file is not only long, it dilutes the model's attention across rules that have nothing to do with the current task. The model reads the head, reads the tail, and the middle degrades. This is the well-known lost-in-the-middle behaviour, and a big static rulebook walks straight into it.

The second is selection. Even with every rule present in context, the model cannot reliably decide which rules govern the task in front of it. A debugging procedure is pure noise during a planning question, and a planning rule is noise during a debug. When everything is loaded at once, the model has to filter relevant from irrelevant on its own, and the filtering step is where it slips.

The third is the one that cost the most trust. The model would load the correct rule, begin the task, and shed half the constraints partway through. Not defiance. Instruction decay over a long generation. Ten tool calls in, the constraint from the first step is no longer in effect, because it is no longer effectively in context.

The fourth is ungrounded output. The model would assert with full confidence and no backing. A file path that did not exist. A function signature that was never defined. A claim about the code that read as certain and was false.

Here is the split that reorganised the whole design. Two of these four are retrieval problems, and two are not. Capacity and selection are both about getting the right instruction into context at the right moment, and better retrieval solves them. Decay and ungrounded output are not retrieval problems. You can place the exact correct rule in context and the model will still shed it mid-task or emit an unbacked claim. Retrieving a rule and complying with a rule are different operations. Compliance needs a separate verification stage, and no amount of better retrieval substitutes for it.

## The control layer I replaced it with

The new design has three stages, wired into the request path so the model cannot silently bypass them.

The first stage is injection. A preprocessing step at the harness layer runs on every user turn, before the model generates, and prepends three blocks to the prompt: the always-set, a routing table, and the task flow the model must follow. This is deliberately not the system prompt. The system prompt is static and set once. This is a per-turn injection that re-asserts the constraints on every single call, which is the direct counter to instruction decay. I chose injection over retrieval-on-demand for the base constraints for one blunt reason. Retrieval only fires if the model elects to retrieve, and the failure I am fixing is precisely that it does not always elect to. Injection does not ask.

The always-set is the fixed overhead. It is paid in tokens on every request, so it stays small by force. A dozen or so short rules, each in its own file, grouped by when they fire: output conventions, a decision protocol, an evidence-before-claims rule, safety and housekeeping. The discipline here is ruthless minimalism, because anything added to the always-set is taxed on every call for the life of the system.

The second stage is routing. The table maps a task class to exactly one procedure file. The model matches the incoming task against the table by meaning rather than by keyword, names the procedure it intends to load, and halts for approval before executing it. Only the matched procedure enters context. Everything else stays on disk. This is just-in-time context assembly. The working set for any task is the small always-set plus one procedure, never the whole corpus. That is the fix for capacity and selection together. The rules that do not apply are not occupying the window.

The third stage is verification, and it is the stage that matters most. After the model emits a plan, a separate agent with its own clean context grades that plan against the loaded rules and returns approve or redo. After the model produces the result, a second independent agent grades the output and re-opens every file and line the model cited, confirming each citation actually resolves. That second agent is the grounding check against ungrounded output. Both verifiers run on fresh context on purpose, because the agent that produced the work is the worst possible judge of it. A model rationalises its own output. It reads its own false claim as self-evidently true because it generated it. An independent judge with no stake in the answer catches what the author reasons itself past. This is LLM-as-judge, but the load-bearing detail is context isolation, not the judging itself.

The request path for any real task is now this:

```
user turn
  -> harness injects always-set + routing table + task flow
  -> route: match task class, load one procedure
  -> PLAN
  -> verifier A grades plan against loaded rules (redo | approve)
  -> EXECUTE
  -> verifier B grades result + resolves every cited source
  -> present
```

## The enforcement wall I built first, then deleted

My first attempt at stopping instruction decay was not a verifier. It was a hard interceptor.

I put a gate in front of tool execution. Before any tool call ran, the model had to attest that it had followed the required steps, and the gate physically blocked the call if the attestation was missing. On paper a hard block beats a soft check. In practice it was brittle and it produced constant false positives, blocking work I had already approved. In one session it walled correct, approved tool calls roughly eight times. The interceptor became the thing fighting the task instead of the thing protecting it.

So I removed it. The replacement is intentionally soft. The model announces its route and halts for approval, and the verifier agents grade the work, but nothing physically blocks execution. This is a genuine tradeoff and I want to be precise about the direction it cuts. A soft gate is skippable by a model under time or token pressure in a way a hard interceptor is not. I took the soft version anyway, because the hard interceptor's false stops cost more real throughput than the decay it prevented. If decay returns badly enough, the escalation path is a smarter, state-aware interceptor. I am not reaching for it until the soft design measurably fails.

## The failure mode I have not closed

I want to end on the open hole, not a recap, because the hole is the honest part.

The verifiers grade the work against whatever rules were loaded. Neither of them checks whether the correct rules were loaded in the first place. If the routing stage selects the wrong procedure, both verifier A and verifier B will grade the output against the wrong standard and pass it cleanly. Routing correctness rests entirely on the model announcing its selection and a human catching a bad route. There is no automatic check on the router itself, and I have not built one.

There is a sharper admission. This rebuild, the work of tearing down the old control layer and standing up the new one, ran across a long session with no task ticket opened first, which is exactly the step the new flow mandates. The system was built by violating the system's own procedure. That is not a footnote. It is the clearest evidence I have that a soft, injected instruction is skippable even by the person injecting it, and it is the first thing I need to harden.

And the largest caveat. When I say this works, I mean the injection step runs without error, the verifier agents pass, and the observed behaviour is better within a session. I do not mean I measured it. There is no before-and-after metric on decay rate or ungrounded-claim rate, old design versus new. The four-failure account is operational experience of running the thing daily, not instrumented data. I would reverse the whole approach if the token and latency cost of two verifier passes on every task ever exceeds the decay it prevents, and that is not proven in either direction yet. What I have is a control layer that makes the discipline automatic rather than hopeful, and a fairly exact map of where it can still lie to me.
