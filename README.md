# agent-evals

An eval factory for AI agents. Give it an agent, and it tells you how good the
agent is and whether your last change made it better or worse.

```
scenarios  ->  runner + adapters  ->  scorers / judge  ->  scorecard  ->  regression gate
```

- **Scenarios:** a few test conversations, multiplied across personas and edge
  cases (interruptions, ambiguous asks, off-topic turns, prompt injection).
- **Adapters and runner:** a simulated user plays each scenario against your
  agent (voice, chat or browser) and every turn is timed.
- **Scorers and judge:** latency, task success, rule checks and an LLM judge
  with saved reasoning; results that swing between repeat runs are flagged.
- **Scorecard and gate:** a report per run, a compare of two runs, and a check
  that fails when a change makes the agent worse.

Everything runs offline against built-in test bots, so no paid API keys are
needed to try it.

Status: design stage. See [docs/spec.md](docs/spec.md) for the architecture and
the shared contract, and the issues for the build plan.

Licence: MIT.
