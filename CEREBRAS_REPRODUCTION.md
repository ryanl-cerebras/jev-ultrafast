# Cerebras integration and Google Flights benchmark

This document describes the Cerebras changes in this fork, the live browser test run on
September 21, 2026, and how to repeat the comparison.

## Repository branches

| Branch | Configuration | Benchmark revision |
| --- | --- | --- |
| `main` | Upstream TypeSafe Jev policy with OpenRouter-routed Mercury 2.5 for field text | `452c1ad` |
| `cerebras-qwen-helper` | TypeSafe Jev policy with Cerebras Qwen 3.8 27B for field text | `10184f8` |
| `cerebras-only-policy` | Cerebras Qwen 3.8 27B chooses browser actions and supplies field text | `fe776be` |

The hybrid `cerebras-qwen-helper` branch keeps Jev as the browser policy. Cerebras is called only
when Jev chooses `TYPE_TEXT`. It replaces the generic text-helper configuration with:

- `CEREBRAS_API_KEY`
- `CEREBRAS_BASE_URL`, defaulting to `https://api.cerebras.ai/v1`
- `CEREBRAS_MODEL`, defaulting to `qwen-3.8-27b`
- `CEREBRAS_REASONING_EFFORT`, defaulting to `none`
- `CEREBRAS_MAX_COMPLETION_TOKENS`, defaulting to `128`

The helper uses temperature 0 and a strict JSON schema with one nullable `text` field. It validates
the response before anything is typed into the browser.

Image input is optional and was not used in the benchmark. Passing `text_vision=True` to `Agent`
adds the current browser screenshot to a text-helper request as a base64 JPEG data URI. The helper
also accepts a base64 PNG data URI.

The experimental `cerebras-only-policy` branch sets Cerebras as the default browser policy. One
Cerebras request selects an offered action and, for `TYPE_TEXT`, returns the field value in the same
structured response. Set `BROWSER_POLICY_PROVIDER=typesafe` on that branch to use the hybrid path
instead.

## What we tested

All arms received the same goal:

> Find one-way flights from Zurich to London on October 20, 2026, for one adult in economy. Do not
> finish when only the Search results heading or filters are visible. Stop only after specific
> flight options with airline names and departure times are visible. Do not select or book a flight.

The live site was `https://www.google.com/travel/flights?hl=en`. We ran five timed attempts per arm,
15 attempts total, in this rotated order:

| Repetition | First | Second | Third |
| ---: | --- | --- | --- |
| 1 | Upstream | Hybrid | Cerebras-only |
| 2 | Hybrid | Cerebras-only | Upstream |
| 3 | Cerebras-only | Upstream | Hybrid |
| 4 | Upstream | Hybrid | Cerebras-only |
| 5 | Hybrid | Cerebras-only | Upstream |

Timing began with the first policy request and ended when the agent accepted `DONE` or `BLOCKED`.
Initial browser navigation and the independent final verification were outside the timing boundary.
The verifier required all of the following:

- Google Flights search URL
- One-way trip
- Zürich city or Zurich Airport (ZRH) as the origin
- London as the destination
- October 20, 2026 as the departure date
- Specific visible flight options with airline names and departure times
- No selected or booked flight

### Primary five-attempt results

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

These results cover one task, one browser profile, and a live website. Browser cache state, Google
Flights behavior, network conditions, TypeSafe service latency, and provider routing were not fully
controlled. The upstream Mercury result is routed OpenRouter evidence, not a direct-provider test.

## Repeat the setup

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

The published examples use `CEREBRAS_API_KEY`. The original private test harness read
`CEREBRAS_TEST_API_KEY` and exposed it to the agent process as `CEREBRAS_API_KEY`; that indirection is
not needed when repeating the test.

## Run the task

From each worktree, run the same command with its own `.env`:

```bash
uv run --env-file .env python examples/run.py \
  --url 'https://www.google.com/travel/flights?hl=en' \
  --goal 'Find one-way flights from Zurich to London on October 20, 2026, for one adult in economy. Do not finish when only the Search results heading or filters are visible. Stop only after specific flight options with airline names and departure times are visible. Do not select or book a flight.'
```

The last progress line contains the agent's elapsed time. Do not count a run as successful merely
because it reaches the search page. Verify the route, date, one-way setting, and visible flight cards.
Record failed attempts and include their elapsed time in an all-attempt mean.

For a quick checked example using the repository's original September 20 fixture, run:

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
