# Agent evals factory: spec (v0)

Written Nov 8, 2026. Personal side project.

## What it is
A pipeline that turns "here is an agent" into "here is its scorecard, and whether
this change made it better or worse". It generalizes voice-evals v0 (nirajjayant/hand-of-the-king#189)
from one voice harness into a factory that works for voice, chat and browser
agents behind one interface.

```
scenarios  ->  runner + adapters  ->  scorers / judge  ->  scorecard  ->  regression gate
 (personas,     (voice, chat,          (latency, task      (report.json,   (pass/fail on
  edge cases)    browser agents)        success, judge)     report.md)      every change)
                                                                            -> tuning loop (later)
```

## Where the code goes
This repo (nirajjayant/agent-evals), code at the repo root. Python package
`agent_evals`, CLI `agent-evals`. Python 3.11+, MIT licence, pytest.

voice-evals v0 (hand-of-the-king#189, still unbuilt as of Nov 8) stays its own job: audio
rendering, recording, WER and barge-in metrics. The factory does not rebuild any of
that. Its voice adapter wraps voice-evals' adapter once hand-of-the-king#189 merges; until then it
is a stub. Generic pieces (scenario format, runner, latency, judge, report,
compare) live in the factory, so voice-evals can import them later instead of
duplicating them.

## Shared contract
Every module builds against these types so the modules can be built in parallel.
`agent_evals/core/types.py` is owned by the scenarios module (#A); other modules
may copy it verbatim into their branch and rebase onto #A's version when it merges.

```python
@dataclass
class Persona:      id: str; description: str; traits: dict[str, str]
@dataclass
class Scenario:
    id: str; modality: Literal["voice", "chat", "browser"]
    persona: Persona; goal: str
    turns: list[str] | None          # scripted user lines, or None for goal-driven
    success_criteria: list[str]      # plain-language, graded by rules or judge
    max_turns: int = 10
    tags: list[str] = field(default_factory=list)   # e.g. "edge:interrupt"
    variant_of: str | None = None
@dataclass
class Turn:         role: Literal["user", "agent"]; text: str; started_at: float; ended_at: float
                    meta: dict = field(default_factory=dict)  # audio paths, tool calls, urls
@dataclass
class Transcript:   scenario_id: str; repeat: int; turns: list[Turn]; error: str | None = None
@dataclass
class Score:        name: str; value: float; passed: bool | None; detail: dict
@dataclass
class RunResult:    run_id: str; agent: str; transcripts: list[Transcript]; scores: dict[str, list[Score]]

class AgentAdapter(Protocol):
    def start_session(self, scenario: Scenario) -> None: ...
    def send(self, user_turn: Turn) -> Turn: ...     # returns the agent turn, timed
    def end_session(self) -> None: ...
class Scorer(Protocol):
    name: str
    def score(self, scenario: Scenario, transcript: Transcript) -> Score: ...
class Judge(Protocol):   # LLM judge; a deterministic FakeJudge ships for tests
    def grade(self, scenario: Scenario, transcript: Transcript, rubric: str) -> Score: ...
```

Run folder: `runs/<run_id>/` with `transcripts/*.json`, `report.json`, `report.md`.

## Modules (one A2A issue each)

Tracked as GitHub issues in this repo: one epic and one issue per module.

- **A. Scenario builder:** YAML scenario format, persona library, variant generator
  that expands a base scenario across personas and edge cases (interruption,
  ambiguous ask, off-topic, adversarial, repeat-yourself, silence or noise marker),
  `agent-evals scenarios validate` and `expand`. Owns `core/types.py`.
- **B. Adapters and runner:** `AgentAdapter` implementations: local echo bot,
  scripted rule bot (a tiny fake support agent with a known bug), OpenAI-compatible
  HTTP chat adapter (key from env, never required for tests), browser adapter
  interface plus a stub, voice adapter stub that wraps voice-evals once it lands. A
  runner that drives scripted or goal-driven simulated users, with repeats,
  timeouts and per-turn timing. `agent-evals run`.
- **C. Scorers and judge:** latency (time to first response, per-turn p50/p95),
  turns used, rule checks, LLM-judged task success with saved reasoning, a
  pluggable judge provider (FakeJudge for tests, real provider via env), judge
  calibration against a small hand-labelled set, variance across repeats with an
  "unstable" flag.
- **D. Scorecard and regression gate:** `report.json` and `report.md`, terminal
  scorecard, `agent-evals compare A B`, `agent-evals gate --baseline A --candidate
  B --policy gate.yaml` that exits non-zero on regression, plus a GitHub Actions
  workflow for this repo that runs the gate against the sample bot.

## Later (not filed yet)
- Tuning loop: propose a change to the agent (prompt, model, config), run the
  suite, keep it only if the score improves .
- Hosted dashboard, phone-number adapters, real browser runs against live sites.

## Rules for every build
- Each module stops at an open PR for review. Nothing merged, no deploys.
- No paid API keys or accounts created. Tests and the demo run with no keys, using
  the echo bot, the rule bot and FakeJudge. Keys come from env vars only.
- No work systems or data. Synthetic scenarios only.
- Run `pytest` before opening the PR and paste the result in it.

## Build order
A, B, C and D can be built in parallel against the contract above. Merge order:
A first (owns the types), then B and C, then D. hand-of-the-king#189 is independent; the voice
adapter in B finishes once hand-of-the-king#189 merges.
