# ai-portfolio

Code and tools by Jess Gray — a working portfolio of AI-native projects.

## Projects

- [**JSCC — Job Search Command Center**](https://github.com/jgray314/jscc) — A job-search pipeline tracker built around eval-backed LLM stages (extraction and fit scoring), with structural safeguards keeping real personal data out of git and LLM calls.

  Start with these:
  - [An eval result reported as a band, not a point](https://github.com/jgray314/jscc/blob/main/evals/README.md) — 76% and 82% on the same prompt, with the case count sized against the noise.
  - [A review finding that overturned an earlier call](https://github.com/jgray314/jscc/blob/main/jscc/fetcher.py) — a DNS-rebinding gap in the URL fetcher, fixed in code after an earlier review had rated it low-risk.
  - [Safety by construction](https://github.com/jgray314/jscc/blob/main/jscc/sanitizer.py) — one definition of "personal data" enforced at both git and every LLM call.
  - [Decisions with rejected alternatives](https://github.com/jgray314/jscc/tree/main/decisions) — ADRs and design principles, including what was dropped and why.
  - [Time and cost reported with their limits](https://github.com/jgray314/jscc/blob/main/scripts/active_time.py) — an active-time estimate from commit history that prints its own bias.
  - [Working an AI coding agent like a junior engineer, not a ghostwriter](https://github.com/jgray314/jscc/blob/main/docs/lessons-learned.md) — slice-sized handoffs, a false impression about where the time went that got checked rather than published, and a list of the specific ways AI-written tests can pass while proving nothing.

  The [JSCC README](https://github.com/jgray314/jscc#whats-worth-reading) has the same list with more detail.

## How these are built

- [**The harness behind these projects**](docs/harness.md) — the rules, hooks, feature list and review gates around the AI coding agent, organized by five harness-engineering concerns, with what is enforced by tools versus asked of the agent, and the known gaps.

More coming soon.
