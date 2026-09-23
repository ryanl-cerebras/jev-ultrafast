# Cerebras integration and benchmark results

This document describes the Cerebras changes in this fork, two live browser benchmarks run on
September 21 and 22, 2026, and how to repeat the primary comparison.

## Summary

- Jev remains the browser policy in the hybrid configuration. Cerebras Qwen 3.8 27B is called only
  when Jev chooses `TYPE_TEXT`.
- On the matched five-run Google Flights task, Jev plus Cerebras averaged **7.563 seconds** with
  **5/5 verified**, compared with **9.958 seconds** and **4/5 verified** for upstream Jev plus
  OpenRouter-routed Mercury 2.5.
- The hybrid mean was **24.1% lower** than the upstream mean on this task.
- A Cerebras-only policy averaged **9.104 seconds** with **5/5 verified** on the simple task.
- On the longer booking-flow smoke test, only the Cerebras-only policy completed the full workflow,
  passing **3/5** attempts with a **42.938 second** mean across successful runs. Neither Jev-based
  arm completed the full workflow in five attempts.

These are small live-site experiments, not a general browser-agent benchmark.

## Workload shape

Jev appears architecturally prefill-heavy. Each decision sends the current structured browser state
and a dynamic set of actions and targets, then returns a small operation and element choice. This is
an inference from the request and response shape, not a direct measurement of accelerator
utilization. It is not the most natural workload for a system whose advantage is most visible during
substantial token generation.

The hybrid is therefore the more natural Cerebras integration tested here. Jev performs the compact
browser-policy decision, while Cerebras supplies generated text only when the selected operation is
`TYPE_TEXT`. The Cerebras-only branch is useful as an experiment, but its reliability fell on the
longer workflow.

## Published branches

| Branch | Configuration | Benchmark revision |
| --- | --- | --- |
| `main` | Upstream TypeSafe Jev policy with OpenRouter-routed Mercury 2.5 for field text | `452c1ad` |
| `cerebras-qwen-helper` | TypeSafe Jev policy with Cerebras Qwen 3.8 27B for field text | `10184f8` |
| `cerebras-only-policy` | Cerebras Qwen 3.8 27B chooses browser actions and supplies field text | `fe776be` |

The hybrid branch adds these settings:

- `CEREBRAS_API_KEY`
- `CEREBRAS_BASE_URL`, defaulting to `https://api.cerebras.ai/v1`
- `CEREBRAS_MODEL`, defaulting to `qwen-3.8-27b`
- `CEREBRAS_REASONING_EFFORT`, defaulting to `none`
- `CEREBRAS_MAX_COMPLETION_TOKENS`, defaulting to `128`

The helper uses temperature 0 and a strict JSON schema with one nullable `text` field. It validates
the response before anything is typed into the browser.

Image input is optional and was not used in either benchmark. Passing `text_vision=True` to `Agent`
adds the current browser screenshot to a text-helper request as a base64 JPEG data URI. The helper
also accepts a base64 PNG data URI.

The experimental `cerebras-only-policy` branch sets Cerebras as the default browser policy. One
Cerebras request selects an offered action and, for `TYPE_TEXT`, returns the field value in the same
structured response. Set `BROWSER_POLICY_PROVIDER=typesafe` on that branch to use the hybrid path.

## Benchmark 1: simple Google Flights search

All arms received the same goal:

> Find one-way flights from Zurich to London on October 20, 2026, for one adult in economy. Do not
> finish when only the Search results heading or filters are visible. Stop only after specific
> flight options with airline names and departure times are visible. Do not select or book a flight.

The live site was `https://www.google.com/travel/flights?hl=en`. Five attempts were run per arm, 15
attempts total, in rotated order.

Timing began with the first policy request and ended when the agent accepted `DONE` or `BLOCKED`.
Initial navigation and independent final verification were outside the timing boundary. The verifier
required the Google Flights search URL, one-way mode, the requested route and date, specific visible
flight options, and no selected or booked flight.

### Results

Arithmetic means include every timed attempt, including the failed upstream attempt.

| Configuration | Individual times (seconds) | Mean | Median | Verified |
| --- | --- | ---: | ---: | ---: |
| TypeSafe Jev + Cerebras text helper | 7.929, 7.605, 6.142, 8.364, 7.774 | **7.563** | 7.774 | 5/5 |
| Cerebras-only policy | 10.111, 8.970, 8.329, 8.425, 9.685 | 9.104 | 8.970 | 5/5 |
| Upstream Jev + routed Mercury | 8.169, 10.661, 9.256, 11.146, 10.557 | 9.958 | 10.557 | 4/5 |

The hybrid Jev plus Cerebras configuration had the lowest mean, 24.1% below the upstream mean.
Cerebras-only was 8.6% below the upstream mean.

The failed upstream attempt reached the correct search URL and filters but never exposed specific
flight options, then ended as `BLOCKED`. A user-requested sixth upstream attempt was run afterward as
a post-hoc replacement. It failed in the same way after 11.499 seconds. It is not substituted into
the primary table. A replacement view that excludes the first failure and includes the follow-up has
an upstream mean of 10.028 seconds and still verifies only 4/5 attempts.

## Benchmark 2: round trip through checkout

The second experiment asked each arm to find a Zurich to London round trip departing October 20 and
returning October 25, select outbound and return flights, continue to Expedia, select seats for both
legs, enter an authorized passenger name and email, and stop at payment. Payment data and purchase
actions were forbidden. Authorized identity values were redacted before every external model call.

Five live attempts were run per arm in rotated order. This was a smoke test of a substantially longer
workflow and required follow-up browser-control fixes beyond the three published benchmark revisions
above.

| Configuration | Verified | Mean successful time | Mean stop time | Furthest stages |
| --- | ---: | ---: | ---: | --- |
| Upstream Jev + routed Mercury | 0/5 | n/a | 6.894 s | Search form or Google results |
| TypeSafe Jev + Cerebras text helper | 0/5 | n/a | 12.162 s | Google outbound or return results |
| Cerebras-only policy | **3/5** | **42.938 s** | 44.454 s | Checkout review or payment |

Mean successful time is the performance metric. Mean stop time includes failures and must not be
interpreted as completion speed. The two Cerebras-only failures reached an Expedia booking stage but
did not satisfy the full independent verifier. The outcome was therefore **not ready** for a general
booking workflow claim.

## Repeat the primary comparison

You need:

- macOS with Google Chrome
- Python 3.12 or newer
- [`uv`](https://docs.astral.sh/uv/)
- A TypeSafe API key for the upstream and hybrid arms
- A Cerebras API key for the hybrid and Cerebras-only arms
- An OpenRouter API key for the upstream Mercury helper

Clone the fork and create one worktree per configuration:

```bash
git clone https://github.com/ryanl-cerebras/jev-ultrafast.git
cd jev-ultrafast
git fetch origin main cerebras-qwen-helper cerebras-only-policy
git worktree add ../jev-upstream origin/main
git worktree add ../jev-hybrid origin/cerebras-qwen-helper
git worktree add ../jev-cerebras-only origin/cerebras-only-policy
```

Install each environment:

```bash
for folder in ../jev-upstream ../jev-hybrid ../jev-cerebras-only; do
  (cd "$folder" && uv sync)
done
```

Run Browser Harness diagnostics from any worktree:

```bash
cd ../jev-hybrid
uv run browser-harness --doctor
```

Allow remote debugging if Chrome prompts for permission.

### Configure the upstream arm

Create `../jev-upstream/.env` and keep it out of Git:

```dotenv
TYPESAFE_API_KEY=<your TypeSafe key>
TYPESAFE_MODEL=jev-latest
TEXT_MODEL_API_KEY=<your OpenRouter key>
TEXT_MODEL_BASE_URL=https://openrouter.ai/api/v1
TEXT_MODEL=inception/mercury-2.5
TEXT_MODEL_REASONING=none
```

### Configure the hybrid arm

Create `../jev-hybrid/.env`:

```dotenv
TYPESAFE_API_KEY=<your TypeSafe key>
TYPESAFE_MODEL=jev-latest
CEREBRAS_API_KEY=<your Cerebras key>
CEREBRAS_BASE_URL=https://api.cerebras.ai/v1
CEREBRAS_MODEL=qwen-3.8-27b
CEREBRAS_REASONING_EFFORT=none
CEREBRAS_MAX_COMPLETION_TOKENS=128
```

### Configure the Cerebras-only arm

Create `../jev-cerebras-only/.env`:

```dotenv
CEREBRAS_API_KEY=<your Cerebras key>
CEREBRAS_BASE_URL=https://api.cerebras.ai/v1
CEREBRAS_MODEL=qwen-3.8-27b
CEREBRAS_REASONING_EFFORT=none
CEREBRAS_MAX_COMPLETION_TOKENS=128
BROWSER_POLICY_PROVIDER=cerebras
```

The published examples use `CEREBRAS_API_KEY`. The private test harness read
`CEREBRAS_TEST_API_KEY` and exposed it to the agent process as `CEREBRAS_API_KEY`; that indirection is
not needed when repeating the test.

### Run the task

From each worktree, run the same command with its own `.env`:

```bash
uv run --env-file .env python examples/run.py \
  --url 'https://www.google.com/travel/flights?hl=en' \
  --goal 'Find one-way flights from Zurich to London on October 20, 2026, for one adult in economy. Do not finish when only the Search results heading or filters are visible. Stop only after specific flight options with airline names and departure times are visible. Do not select or book a flight.'
```

The last progress line contains the agent's elapsed time. Do not count a run as successful merely
because it reaches the search page. Verify the route, date, one-way setting, and visible flight cards.
Record failed attempts and include their elapsed time in an all-attempt mean.

For a checked example using the repository's original September 20 fixture, run:

```bash
uv run --env-file .env python examples/flights.py --keep-open
```

That example has a built-in independent verifier, but its date differs from this benchmark.

Before live testing, run the offline checks on each branch:

```bash
uv run ruff check .
uv run pytest
node --check jev_ultrafast/static/app.js
node --check jev_ultrafast/snapshot.js
```

Live examples make paid API calls. Never commit `.env`, API keys, or raw traces containing
credentials.

## Limitations

- The primary comparison covers one task, one browser profile, and a live website.
- Browser cache state, site behavior, network conditions, TypeSafe service latency, and provider
  routing were not fully controlled.
- The upstream Mercury arm used OpenRouter, not a direct Mercury endpoint.
- Five attempts per arm are enough for an exploratory comparison, not a general reliability claim.
- The longer booking-flow smoke used additional browser-control fixes, so it should not be compared
  directly with the primary benchmark as if only the model configuration changed.
