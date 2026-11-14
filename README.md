# magpie-watch

Degradation watch for an LLM gateway: catch a free model that quietly got worse
before your users do.

Three small Python tools, no dependencies beyond the `magpie` CLI:

| tool | question it answers | cadence |
|---|---|---|
| `magpie-canary` | does it still get the answers right? | daily |
| `magpie-think` | did it stop thinking? | hourly |
| `magpie-bench` | how good is it, before it takes live traffic? | on admission, refreshed weekly |

They are deliberately separate. A single "quality score" mixes things that fail
independently, and a single small question set cannot resolve the difference
between "6/8" and "8/8" — see [statistics](#why-the-numbers-are-small).

> Code comments and CLI output are in Chinese; the documentation is in English.

## Why

Free model pools rot in three different ways:

1. **It dies** — rate limits, 5xx, provider pulls the model. A probe catches this.
2. **It stops reasoning** — the model still answers, but its thinking budget
   collapses (median `reasoning_tokens` halves, zero-reasoning share jumps). No
   error, no failed request. Only a trend catches this.
3. **It gets dumber** — same model id, different weights or a different upstream
   deployment. Deterministic trick questions catch this.

Each needs its own detector with its own cadence. This repo is detectors 2 and 3,
plus the benchmark that decides whether a model may join the pool at all.

## Requirements

- A [magpie](https://usemagpie.ai) gateway on `http://127.0.0.1:3425` exposing an
  OpenAI-compatible `/v1/chat/completions`, with `magpie usage --csv` available
  (it is the source of truth for every counter below).
- Python 3.11+ (no third-party packages).
- `bwrap` (bubblewrap) for `magpie-bench`'s code sandbox.
- HumanEval questions for `magpie-bench`:

```bash
python3 -m pip download --no-deps human-eval -d /tmp/he
unzip -o /tmp/he/human_eval-*.whl -d /tmp/he/x
gunzip -c /tmp/he/x/human_eval/data/HumanEval.jsonl.gz > ~/.local/share/magpie-bench/HumanEval.jsonl
cp share/tool_cases.json ~/.local/share/magpie-bench/
```

## magpie-canary — fixed questions, retried before alarming

Eight questions with exact-match answers (two classic ratio traps, one
instruction-following case, one JSON case, counting, weekday arithmetic, string
reversal). Runs at `temperature: 0` against every model in a group.

The useful part is not the questions, it is the two rules around them:

**Rule 1 — a failure must reproduce.** When a model lands below its baseline, only
the questions it got wrong are asked again. Two rounds wrong is a verdict; a
retry that passes is jitter. Measured here: the same model scored 6/8 and then 7/8
twenty minutes apart on the same trick question.

**Rule 2 — exit 0 even when it finds a problem.** A detected regression is a
successful run; returning a non-zero code makes the scheduler render it as
`Cron failed` every day, and an alert that cries wolf daily stops being read.
Non-zero is reserved for environment faults (gateway unreachable, every request
errored).

```bash
magpie-canary                                   # all models in the group
CANARY_MODELS=provider/model magpie-canary      # one model
CANARY_GROUP=free-first CANARY_TIMEOUT=90 magpie-canary
```

History goes to `~/.local/state/magpie-canary/history.jsonl`, one line per
question plus a `__score__` line per model per run.

### Why the numbers are small

With 8 questions at p≈0.9 the standard error is `sqrt(p(1-p)/n)`:

| questions | 95% half-width |
|---|---|
| 8 | ±21 points |
| 100 | ±6 points |
| 300 | ±3 points |

Resolving a 5-point drop at 95% confidence needs roughly 277 questions per group.
So the canary is a collapse detector, not a trend detector — use `magpie-think`
for trends, where the sample is thousands of requests.

## magpie-think — thinking budget versus its own baseline

For every model: the median `reasoning_tokens` of requests older than 24 h (the
baseline) against the last 24 h (recent). Ratio thresholds: `<= 0.5` warning,
`<= 0.25` severe, plus a hard signal when 80 % of the baseline had reasoning and
80 % of recent has none.

Models whose baseline median is below 15 are skipped — a model that never really
thinks (baseline 6) is noise at half of 6.

```bash
magpie-think              # table
magpie-think --json       # machine-readable
magpie-think --notify     # send to Feishu when something degrades, 24 h cooldown per model
```

**Use `served_model`, never `requested_model`.** In a routed gateway most traffic
asks for the group id, not a model; grouping by the request collapses every member
into one fake "model" and under-counts each real model's sample by an order of
magnitude. Measured here: 62 % of a week's requests were group-addressed, and the
per-model view went from 0 samples to 1663 after switching columns.

Results are also cached to `~/.local/state/magpie-think/latest.json` so a
long-running exporter can forward them without re-running this script.

## magpie-bench — admission control

A model joins the pool only after it is measured. Two suites:

- **code**: 20 HumanEval problems, sampled evenly from 164. Generated code is
  *really executed* against the unit tests in a `bwrap` sandbox (no network, 10 s
  timeout). Pass means the tests pass.
- **tool**: 20 function-calling cases — simple selection, choosing between
  similar tools, parallel calls, cases where calling anything is wrong, and
  argument precision (enums, ISO dates, numeric types, nested objects). Scored by
  comparing the returned `tool_calls`.

`score = 0.5 * code + 0.5 * tool`.

```bash
magpie-bench run --model provider/model-name --code-n 20 --tool-n 20
magpie-bench run --next          # one model that still needs measuring (timer-friendly)
magpie-bench pending             # who is unmeasured or stale
magpie-bench report              # ranking table
magpie-bench select --current ... # the list the group sync should install
```

### Ranking is not the benchmark score

```text
composite = 0.6 * bench + 0.4 * availability - 0.02 * max(0, p50_seconds - 3)
```

- availability and p50 come from the last day of the real usage log, not from a probe.
- below `BENCH_MIN_SAMPLES` requests, availability is replaced by a neutral 0.95 —
  a model with one request and one failure otherwise reads as 100 % down.
- a record older than `BENCH_STALE_DAYS` counts as unmeasured again, so a score
  cannot be frozen at whatever the model did the day it joined.

`BENCH_SORT=bench` switches back to ranking by score alone.

### The free-model guard

`magpie-bench` refuses any id that neither contains `free` nor appears in
`~/.local/share/magpie-bench/free_extra.txt`. That one string test is the only
thing standing between a loop bug and a paid provider bill. Models from other
providers whose names do not say "free" go in that file, one id per line.

## Running on a timer

```bash
mkdir -p ~/.config/systemd/user
cp systemd/*.service systemd/*.timer ~/.config/systemd/user/
systemctl --user daemon-reload
systemctl --user enable --now magpie-think.timer
```

`magpie-canary` is meant to be driven by whatever scheduler you already use (a
cron job, Hermes, a CI pipeline). If that scheduler treats non-zero exit codes as
failures, keep rule 2 above in mind.

## Caveats we hit, so you do not have to

- **A partial window is not a score.** Require the full question counts before a
  record counts as measured, or a 3-question smoke run becomes the model's
  permanent ranking.
- **Bench code tasks need a system prompt.** Without a `tools` array some models
  answer `(The model asked for a tool this session doesn't have.)`, which scores
  as a coding failure that is really an artifact.
- **Models echo the whole function.** Build two candidates — the text alone (when
  it re-declares the entry point) and prompt + continuation — and pass if either
  runs. Strip markdown fences, and never truncate at `"""`: that cuts the
  docstring in half and every task fails with `SyntaxError`.
- **A run where half the requests errored is an error, not a score.** Retry it
  later instead of writing 0 and locking the model out.
- **Probe timeouts are not death.** Under load the gateway queue can hold a probe
  past a short timeout; a 45 s budget marked healthy models dead. Use a generous
  timeout and a consecutive-failure threshold.
- **Question sets saturate.** These questions are intentionally mundane; do not
  read a 1-question difference as capability.

## License

MIT
