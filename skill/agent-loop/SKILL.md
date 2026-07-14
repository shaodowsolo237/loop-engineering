metadata:
  name: agent-loop
  version: 0.8.0
  description: 'Use whenever the user wants Claude to keep working on its own until a goal holds: "run a loop", "loop until the tests pass", "keep going until the build is green", "fix all of these until the suite is clean", "babysit this until it''s done", "run this autonomously", "set up a self-verifying loop", "iterate until X", or references agent loops / loop engineering / Boris Cherny''s "I write loops" methodology. Trigger even when the user never says the word "loop" — any "keep doing X until condition Y holds, then stop" request is a loop. Also use for a large multi-stage objective — "create user manuals", "break this objective into steps", "turn this into a pipeline of loops", or any deliverable whose stages fan out over many items (one loop per page, screen, or endpoint). Also use when the user asks which loop type or primitive fits a task — "/goal or /loop?", "should this be a schedule/routine?", "do I even need a loop for this?".'
  argument-hint: '[goal, e.g. "all tests in api/ pass and lint is clean"]'
content:
  title: Agent Loop
  introduction: 'Turn a goal into a loop that runs itself. Instead of prompting turn by turn, you

    define a goal with a real verification gate and let Claude run **act → verify →

    re-prompt** until the gate passes or a budget ceiling stops it. This is the

    practical companion to the loop-engineering knowledge base (see end of file).'
  sections:
  - title: 'The one rule: verification is the engine'
    text: 'A loop without a real verification gate is just repeated guessing. The model will

      happily declare victory while the build is red. What makes a loop trustworthy is a

      gate that lets *reality* — a test suite, a build, the app actually running — decide

      whether a pass made progress. Cherny''s line: a verification feedback loop "2-3x"

      the quality of the result. So the first question is never "what should the agent

      do" — it''s **"how will the loop know it''s done?"**


      If the project has no way to verify the goal (no tests, no build, no runnable

      check), stop and say so. Propose adding a check first. Looping without one is the

      single most common way these go wrong.


      **A gate proves only what it asserts — so validate the gate, don''t just run it.**

      Two traps hide here: a gate green *for the wrong reason* (a typo''d test path, a check

      that never exercises the bug — 100% coverage with mocks has shipped real auth bugs

      past review), and a gate too shallow to catch a behavioral break. Two habits close

      both: **prove it red-first** — run the gate on the *unfixed* code and confirm it fails

      *for the right reason* before looping (`verify-loop.sh` does this by default and

      refuses a green start unless `--allow-green-start`); and for correctness- or

      security-critical goals make the gate **behavioral / live-data** (drive the real

      endpoint, assert the real contract), not coverage.


      **When tests can''t see the bug, put a judge in the gate.** Even a behavioral,

      red-first test only checks what it asserts — it can''t catch a requirement wired in one

      place but missed at another call-site, a scope/permission leak, or a loop that quietly

      weakened its own tests. For non-trivial or correctness-/security-critical stages, make

      the gate **script AND judge**: the objective tests PLUS an independent reviewer

      (`scripts/judge-check.sh`) that adversarially reads the diff against a rubric and fails

      with feedback the loop then acts on. Drop a `rubric.md` into the stage folder and the

      scaffolded `verify.sh` runs it automatically (`run-tests && judge-check.sh --rubric

      rubric.md`); the judge only fires once the tests pass, so it costs ~one model call per

      green attempt. It MUST be a separate run from the one that wrote the code — the author

      judges its own work poorly. (Real case: a loop''s tests passed but it billed the wrong

      API key on one un-tested code path; only an independent review caught it.)


      **Visual / subjective deliverables — render, then let the judge SEE it.** Have the gate

      render the result to a PNG and point the rubric at the image — `judge-check.sh`''s reviewer

      uses Read, which VIEWS images, so it can rule on look-and-feel. Fold the project''s OWN

      conventions (lint, file-size cap, type-check) into the gate, and give judge stages a

      higher `effort` than mechanical ones. Full recipe + evidence: `references/gates.md`.'
  - title: Before you loop — the 60-second setup
    text: "Walk these five with the user (or infer and state your assumptions). Don't start\nthe loop until the gate and ceiling exist.\n\n1. **Goal** — a *checkable* condition, not a vibe. \"All tests in `api/` pass and\n   `ruff` is clean,\" not \"make the API better.\" Then name the goal's **forks** —\n   the readings you'd otherwise resolve silently (\"fix the failing tests\": fix the\n   code, or fix wrong tests? \"migrate\": exact behavior, or clean up too?). State\n   the 1–2 forks that change what the loop builds, pick a default, and bake it\n   into the goal prompt — an unattended loop resolves ambiguity alone, one\n   iteration at a time.\n2. **Verify command** — the shell command whose exit code is the gate. Discover it:\n   inspect `package.json` scripts, `Makefile`, `pyproject.toml`/`pytest`, `gradlew`,\n   `go test`, `cargo test`, or the CI workflow. Prefer \"can the agent actually run\n   the thing\" (tests, a smoke run, a headless browser) over lint-only — lint passing\n\
      \   says nothing about whether the code works.\n3. **Budget ceiling** — a max iteration count and, for unattended runs, a dollar\n   cap (`verify-loop.sh --max-cost USD` sums each iteration's reported cost).\n   This is what makes a loop safe to leave unattended. No ceiling, no unattended loop.\n4. **Isolation** — if the loop runs alongside other work, give it its own git\n   worktree so parallel changes don't collide (`claude --worktree <name>`).\n5. **Supervision** — attended (watch it) or background (notify on done/stuck).\n   Decide up front; it changes which primitive you pick. If background: also\n   decide what the loop may do alone — scope `--allowedTools`/permissions to the\n   minimum the goal needs (an overnight loop rarely needs push, network, or rm)."
  - title: Pick the loop type, then the primitive
    text: 'Decide **which piece of the work you''re handing off** — that picks the loop type,

      and the primitive follows. **If the work doesn''t recur and one attempt — with the

      gate run once at the end — would plausibly reach the goal, don''t build a loop:**

      run the turn, run the gate, hand back the result. Reach for the bundled `/verify`

      skill (v2.1.145+) or a project verification skill that encodes the manual check —

      not loop machinery.


      | You hand off | Type | Use | Why |

      |---|---|---|---|

      | The stop condition | Goal-based | **`/goal <condition>, stop after N tries`** | Claude loops turn after turn until a small fast model confirms the condition; takes an explicit turn cap. Works headless too. Start here. |

      | … with your own control flow | Goal-based | **`scripts/verify-loop.sh`** (a `claude -p` while-loop with a verify gate) | You own the ceiling, stall/reset/escalation handling — headless / CI. |

      | … with custom stop logic | Goal-based | **Stop hook** (command hook: exit 2 blocks the stop; prompt/agent hook: return `{"ok": false}`) | The same mechanism `/goal` wraps. |

      | The trigger | Time-based | **`/loop <interval>`** (session-scoped) or a cloud routine via **`/schedule`** (survives your machine being off) | "Every 30m, draft fix PRs for new bug issues." Polls or schedules instead of running once. |

      | The prompt itself | Proactive | **Compose:** `/schedule` trigger + `/goal` per-run done + skills to verify + workflows for fan-out | A recurring stream of well-defined work (reports, triage, migrations) with no human in real time. |

      | A step with NO objective check | any | **`scripts/judge-loop.sh`** (LLM-judge gate) | A *separate* Claude scores the result against a rubric — independent verification when no shell command can decide. Orthogonal to loop type. |


      Read `references/choosing.md` for the full taxonomy (trigger, stop criteria, and

      token levers per type) and `references/primitives.md` for the exact flags,

      caveats, and doc links. Confirm flags against current Claude Code docs — they

      change between versions.'
  - title: Run the loop
    text: "**In-session (default):** state the goal as a condition and hand it to `/goal`:\n\n```\n/goal all tests in test/auth pass and the lint step is clean, stop after 10 tries\n```\n\n**Headless / scriptable:** use the bundled script. It runs the verify command,\nbreaks the moment it exits zero, and otherwise feeds the failure back into a\nresumed `claude -p` session until the ceiling:\n\n```bash\nscripts/verify-loop.sh \\\n  --goal \"Fix the failing auth tests. Find and fix the root cause, don't skip tests.\" \\\n  --verify \"npm test -- test/auth\" \\\n  --max 10 \\\n  --tools \"Read,Edit,Bash\"\n```\n\nThe cycle each iteration: **act** (Claude edits) → **verify** (run the gate) →\nfeed the result back → **check budget** → stop or continue. The verify command is\nthe brake and the steering wheel.\n\nSafety flags worth knowing (`--help` lists all): `--stall N` bails after N no-progress\nrounds (compared by *normalized signature*, not exact output, so it still catches a\nloop that fails\
      \ differently each round); `--reset-every N` drops the session for fresh\neyes when an approach entrenches; `--escalate-model M` makes a last-ditch stronger-model\nattempt before a stall bail; `--worktree PATH` runs the loop on a throwaway branch;\n`--log DIR` writes each iteration's verify output + diff as an audit trail;\n`--allow-green-start` skips the red-first guard; `--max-cost USD` bails once the\nsummed per-iteration cost (claude's reported `total_cost_usd`) crosses the cap."
  - title: Stay in the judgment seat
    text: 'The loop produces *candidates*, not merged truth. Your job doesn''t disappear, it

      moves up a level: review the diff or the PRs, kill runaway loops, and never let a

      loop auto-merge work you haven''t looked at. Cherny: "if the code sucks, we''re not

      gonna merge it." Set the gate, set the ceiling, then judge the output.


      Make the loop hand you *decisions*, not just diffs: have the goal prompt say "log

      anything you resolve that the goal doesn''t specify, and why, to `decisions.md`". Review

      that log first — the judge-script edit (anti-patterns) was caught only by diff archaeology.'
  - title: Compound — make the loop smarter over time
    text: 'The highest-leverage habit: every time the loop makes the *same* mistake twice,

      don''t just fix it in-session — write the lesson into `CLAUDE.md` or turn it into a

      skill. Each durable correction means the next loop starts smarter and can run

      longer unattended. This is what lets a loop "just run forever" instead of needing a

      babysitter. Treat recurring corrections as a signal to update memory, not to re-explain.'
  - title: Anti-patterns (how loops go wrong)
    text: "- **No verify gate** — looping on the model's self-assessment. It will lie to itself.\n- **Lint-only verification** — green lint, broken code. Run the actual thing.\n- **An expensive gate as the only instrument** — when the gate is a full pipeline\n  (end-to-end run, film take, deploy, batch job), every hypothesis costs the whole\n  pipeline. Split the instruments: a cheap probe (API call, shell query, unit-level\n  reproducer) falsifies \"is the system right?\" in minutes; the expensive gate confirms\n  \"is the deliverable right?\" once, at the end. Seen for real: six masked defects\n  peeled at ~35 min per iteration that a 2-minute probe could each have falsified.\n- **Misreading a correct gate** — the gate can be right while its *reading* is wrong:\n  a count with an unexpected filter, a status that lags aggregation, a log line whose\n  name promises more than it measures. When a confident fix \"didn't work\", re-derive\n  the failing signal's semantics at its source before\
      \ iterating — a misread signal\n  falsifies every fix the same way, and the loop thrashes on a phantom.\n- **No budget ceiling** — an unattended loop with no max burns the whole budget on a\n  stuck problem. Always cap iterations; consider bailing after N identical failures.\n- **A flaky gate** — a nondeterministic check makes the loop thrash, and worse, tempts\n  it to \"fix\" the *symptom* of a flake instead of a bug. Stabilize it *first* — measure\n  the flake rate, split infra-flake from real bug; diagnosis playbook in `references/gates.md`.\n- **Verifying with the context that wrote the code** — a fresh check (a separate run,\n  a Stop hook, a real command) catches what the author missed.\n- **A gate the loop can edit** — if the loop has write access to its own gate (the verify\n  script, the rubric, or a skill script referenced by absolute path), it can make a red\n  gate green by *weakening the check* instead of doing the work. Keep gate + skill scripts\n  OUTSIDE the loop's\
      \ writable scope (read-only, or a path its tools can't reach), and diff\n  them after a run. Seen for real: a design loop whose judge gate gave false negatives\n  edited the judge script itself — that time a legitimate fix, but the *capability* is the\n  risk, and you only know which by reviewing the diff."
  - title: Decompose a big objective into a loop chain
    text: "One loop fixes one thing. A big objective (\"create user manuals\", \"migrate every\nendpoint\") needs several loops, some fanning out over many items. When that's the\ncase, build a **loop chain** instead of a single loop:\n\n1. **Plan it** — restate the objective, then run an ultracode/Workflow pass to\n   decide the ordered stages, each with a goal, a verification gate, declared\n   inputs/outputs, and a `next`. Mark stages that fan out (one sub-loop per page /\n   screen / endpoint), and mark non-trivial / correctness-critical stages to get a\n   `rubric.md` (a compound script+judge gate — see \"put a judge in the gate\" above).\n   The planner fixes the stage skeleton; fan-out counts are discovered at runtime.\n2. **Show the plan and get approval** before building anything.\n3. **Build it** — write `chain.json` and instantiate the backbone from the\n   template library with `scripts/scaffold-loop.sh`.\n4. **Run it** — `scripts/run-chain.sh <workspace>` drives the chain (one\
      \ loop at a\n   time via `scripts/loop-engine.sh`, whose gate can be **script** | **judge** |\n   **human**): linear stages self-verify, fan-out stages run their sub-loops in\n   parallel and join, and the terminal stage pauses for your sign-off. Resumable;\n   start mid-chain with `--from <stage>`.\n\nEach loop lives in its own folder with its own files and is reusable; loops share\ndata only through `state/`. **Read `references/chains.md` for the schemas, runtime,\nplanner procedure, and a worked user-manual example before building a chain.**"
  - title: Reference
    text: "- `references/choosing.md` — the four loop types (turn/goal/time/proactive): what you hand off, trigger, stop criteria, token levers.\n- `references/gates.md` — gate hygiene: the visual-judge recipe + flaky-gate diagnosis playbook.\n- `references/chains.md` — loop-chain schemas, runtime, planner procedure, example.\n- `references/loop-chains-design.md` — the approved design spec for loop chains.\n- `references/primitives.md` — the documented Claude Code primitives, with flags and caveats.\n- Knowledge base (the \"why\" behind all of this): `/home/cocodedk/0-projects/loop-engineering`\n  · online at https://cocodedk.github.io/loop-engineering/ · repo\n  https://github.com/cocodedk/loop-engineering. Start with `docs/04-loop-anatomy.md`,\n  `docs/05-verification-and-memory.md`, and `docs/09-example-loops.md`."
