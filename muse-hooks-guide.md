# Muse Hooks: A Programmer's Guide

- **Status:** Draft
- **Date:** 2026-10-27
- **Summary:** How to write, test and run Muse hooks: Bash scripts with no AI that poll on a timer and wake the agent only when something changes.

A Muse hook is a Bash script with no AI that Muse runs on a timer. Each run ends in one of two verdicts: `silent`, which costs nothing, or `wake`, which starts the agent with whatever the script found.

This guide describes the hook protocol as Dex, a Muse agent, documented it from its own instructions in October 2026. Meta has not published a public specification, so details may differ between accounts or change over time.

## Motivation

**Polling with the agent is expensive.** A Muse recurring task wakes the full agent on every run. Each wake reloads the agent's standing context (its instructions, memory and profile files) before doing any work, whether or not anything has changed. In one measured setup that context was about 170,000 tokens per wake. A monitor that checks every few minutes therefore spends most of its weekly allowance confirming that nothing happened.

**Hooks move the waiting out of the agent.** A hook script does the cheap part, checking whether something changed, with no model involved. A run that finds nothing ends `silent` and costs zero tokens. Only a run that finds a change ends `wake`, and only then does the agent start and spend tokens on judgment.

**The savings scale with how rarely things change.** A monitor that wakes on every poll costs the same as a recurring task. A monitor that wakes once a day after polling every minute costs one agent run instead of 1,440. One published test of this pattern measured zero idle tokens and about a minute from a change to the agent waking ([SmartScope](https://smartscope.blog/en/blog/claude-code-delegate-tasks-meta-muse-using-spreadsheets-2026/)).

**The trade-off is access.** Hook scripts run without connector credentials or secrets, so they can only watch public data or local files. Watching a private source, such as your Gmail, needs a bridge that publishes a credential-free signal. The [Wake Board design](muse-wake-board.md) is one worked example of such a bridge.

## How a hook works

On every tick of its timer, Muse runs the hook script, and the script's verdict decides whether the agent starts.

![Flow: the Muse timer runs the hook script, which reads and writes its state files. If the change is conclusive, the verdict is wake: the agent starts with the worker instruction plus the reason and payload, at the cost of a full context, and notifies you only if it matters. Otherwise the verdict is silent: no agent and zero tokens.](muse-hooks-flow.png)

The script compares what it fetches with what it saw last time, using its own state files. Anything short of a conclusive change ends `silent`, at no cost. A conclusive change ends `wake`: the agent starts with the hook's worker instruction plus the script's reason and payload, checks the finding, and decides whether you need to hear about it.

## Anatomy of a hook

A hook has two parts: a polling script that decides, and a hook definition that tells Muse when to run the script and what the agent should do when it wakes.

| Part | What it is | Where it lives |
| --- | --- | --- |
| Polling script | A Bash script ending in `.sh` that checks one thing and returns a verdict | `~/hooks/scripts/` |
| State files | Whatever the script needs to remember between polls: last-seen IDs, hashes, timestamps | `~/hooks/state/` |
| Hook definition | A short, stable hook ID, the script's path, a polling interval and a worker instruction | Registered with Muse |
| Worker instruction | The prompt the agent receives when the script wakes it | Part of the hook definition |
| Invocation log | An append-only JSONL record of each poll's observations and verdict | Kept by Muse per hook |

The agent can write the script, or you can supply one. Before running a supplied script, the agent reads it and reports what it accesses: files, network and anything that looks like a credential.

## Writing the script

A hook script is ordinary Bash with three required lines at the top and exactly one verdict at the end.

**Requirements:**

- Start with `#!/usr/bin/env bash` and `set -euo pipefail`.
- Source `$HATCH_HOOK_RUNTIME`. It defines the protocol functions below.
- Run under the normal shell executor, with the ordinary environment and network proxy. No secrets and no connector credentials are available, by design.
- Fetch only public endpoints or local data, and bound every fetch, for example `curl --fail --silent --show-error --max-time 10`. Use `jq` to parse JSON.
- Keep anything that must survive between polls under `~/hooks/state/`.
- End every run with exactly one terminal call: `silent` or `wake`.

**Skeleton:**

```bash
#!/usr/bin/env bash
set -euo pipefail
source "$HATCH_HOOK_RUNTIME"

# Fetch, then compare against state in ~/hooks/state/ ...
if [ "$changed" = "1" ]; then
  wake "Advisory page changed" '{"url": "..."}'
else
  silent "no change"
fi
```

**Runtime functions** (provided by `$HATCH_HOOK_RUNTIME`):

| Function | Arguments | Effect |
| --- | --- | --- |
| `silent` | reason, optional JSON payload | Ends the poll. No agent starts and no tokens are spent. |
| `wake` | reason, optional JSON payload | Ends the poll and starts an agent with the worker instruction. The reason and payload arrive as event data. |
| `log` | observation, optional JSON | Appends an observation to the hook's JSONL invocation log, e.g. `log "observation" '{"price": 251.20}'`. Doesn't end the poll. |
| `disable_after_run` | none | Disables the hook after this poll. Call it before the final verdict. |

## The decision rule

Every poll ends with exactly one verdict, and a hook should wake only on a conclusive change.

- **`silent "reason" '{payload}'`** means nothing happens: no agent, no tokens. Use it for "no change", and for every case the script can't decide.
- **`wake "reason" '{payload}'`** launches an agent with the hook's worker instruction. The wake reason and payload arrive automatically as event data. The worker then verifies the finding and uses judgment before deciding whether to notify the user, so a wake doesn't guarantee a notification.
- **Inconclusive results stay silent.** If a fetch fails, times out or returns something garbled, the script logs the failure and calls `silent`. It never wakes on a maybe.
- **Finished hooks disable themselves.** When a conclusive result means no future poll could be useful, such as a one-time event that has now happened, the script calls `disable_after_run` before its final verdict.

Keep payloads small and factual: the values the worker needs to verify the change, such as a URL, an ID or a number. The worker instruction holds the instructions; the payload holds data.

## State and dry runs

Dry runs share real state, so the script itself must avoid writing state during a test.

**State.** The runtime keeps no state for you. The script stores what it needs under `~/hooks/state/`, usually the last value it saw, and compares each new fetch against it. Update state only after a conclusive result, so a failed fetch never overwrites a good baseline.

**Dry runs.** The environment variable `HATCH_HOOK_DRY_RUN` is `1` during a test run and `0` when live. A dry run doesn't isolate state. If a test run writes the new value, it "consumes" the change, and the live poll that should have fired never sees it. Check the flag and skip or redirect every state write during a dry run:

```bash
is_dry() { [ "${HATCH_HOOK_DRY_RUN:-0}" = "1" ]; }
save() { is_dry || printf '%s\n' "$2" > "$1"; }
```

**The test tool** runs only the script, synchronously. It reports the verdict and appends to the invocation log, but never launches an agent, so tests cost no tokens.

## Lifecycle and settings

A new hook starts disabled and goes live only after its dry runs produce the right fetch and verdict.

1. **Write** the script in `~/hooks/scripts/`.
2. **Register** the hook: its ID, script path, polling interval and worker instruction.
3. **Dry-run** it and fix the script until the fetch and verdict are right.
4. **Enable** it.
5. **After any update**, dry-run again. Updates keep the hook's enabled or disabled state.

A manual live run is also available, for testing the full path including the agent.

| Setting | Range | Default |
| --- | --- | --- |
| Polling interval | 5 seconds to 7 days | 60 seconds |
| Per-poll script timeout | up to 600 seconds | 600 seconds |
| Delivery | set per hook | the chat the hook was created in |
| Initial state | disabled | disabled |

## Example: a page-change hook

This hook watches one section of a public web page and wakes the agent only when that section changes. It has been tested against a simulated page for the cases listed below the code.

```bash
#!/usr/bin/env bash
# Page-change hook: wakes the agent when the watched part of a public page changes.
set -euo pipefail
source "$HATCH_HOOK_RUNTIME"

URL="${PAGE_URL:-https://example.com/advisories}"
STATE="${PAGE_STATE:-$HOME/hooks/state/page-change.sha256}"
mkdir -p "$(dirname "$STATE")"

is_dry() { [ "${HATCH_HOOK_DRY_RUN:-0}" = "1" ]; }
save() { is_dry || printf '%s\n' "$1" > "$STATE"; }   # dry runs never consume a change

# 1. Fetch with a time limit. Anything inconclusive: log it and stay silent.
if ! page=$(curl --fail --silent --show-error --location --max-time 10 "$URL" 2>/dev/null); then
  log "fetch_failed" "{\"url\":\"$URL\"}"
  silent "fetch failed"
  exit 0
fi
if [ -z "$page" ]; then
  log "empty_page" "{\"url\":\"$URL\"}"
  silent "empty page"
  exit 0
fi

# 2. Hash only the part that matters, so ads and timestamps don't cause false wakes.
#    Here: lines mentioning "advisory", with whitespace normalized.
relevant=$(printf '%s\n' "$page" | grep -i 'advisory' || true)
if [ -z "$relevant" ]; then
  log "no_match" "{\"url\":\"$URL\"}"    # page layout changed or wrong page: inconclusive
  silent "watched section not found"
  exit 0
fi
digest=$(printf '%s\n' "$relevant" | tr -s '[:space:]' ' ' | sha256sum | cut -d' ' -f1)

# 3. First run records a baseline and never wakes.
if [ ! -f "$STATE" ]; then
  save "$digest"
  silent "baseline recorded"
  exit 0
fi

# 4. Exactly one verdict.
if [ "$digest" != "$(cat "$STATE")" ]; then
  save "$digest"
  wake "Watched page changed" "{\"url\":\"$URL\"}"
else
  silent "no change"
fi
```

| Case | Verdict |
| --- | --- |
| First run | `silent`, baseline recorded |
| Same page | `silent` |
| Only unwatched parts changed (an ad, a timestamp) | `silent` |
| Watched section changed, dry run | `wake`, state unchanged |
| Watched section changed, live | `wake`, new baseline saved |
| Watched section missing | logged, `silent` |
| Empty page or failed fetch | logged, `silent` |

A matching worker instruction might read: "The watched page changed. Open the URL in the payload, read the advisory section, and tell me in one line only if it affects my plans this week."

## Design guidance and limits

A hook pays off only when wakes are rare compared to polls, because each wake costs a full agent run.

**Good fits:**

- **A public page that rarely changes**, such as a transit advisory, a status page or a public listing.
- **A value crossing a threshold**, such as a price from an API that needs no key.
- **A new item matching a rule** in an RSS feed or a public JSON endpoint.
- **A local file or a public signal written by something else**, such as a published spreadsheet cell. This is how a hook watches private data indirectly: a separate process with access writes a credential-free signal for the hook to read. The [Wake Board design](muse-wake-board.md) applies this to Gmail.

**Poor fits:**

- **Anything behind a login**, such as Gmail, Calendar or a school portal. Hook scripts have no credentials, and Muse's connectors offer no push events.
- **Fuzzy judgments.** The check must be a conclusive, mechanical rule: "changed", "new item matching X" or "value above Y". If deciding needs judgment, the agent has to do it.
- **Sources that change on every poll.** Every poll becomes a wake, so the hook costs as much as a recurring task. Narrow what you hash or compare until only meaningful changes register.

**Limits to plan around:**

- **It's polling, not push.** Changes between polls are seen late, and a change that reverts before the next poll is missed.
- **Wakes are the expensive step.** Each one loads the agent's full context. Several changes found in one poll should produce one wake with all of them in the payload.
- **A wake isn't a notification.** The worker decides whether the finding is worth telling you about. Write the worker instruction to say when to stay quiet.

## Sources

- Dex (a Muse agent), its description of the hook protocol from its own instructions, October 2026: script requirements, runtime functions, the decision rule, dry runs, lifecycle and settings
- [Wake Board Hook Design for Dex, a Muse agent](muse-wake-board.md): the per-wake context cost, and a worked bridge for private data
- [Delegating to Muse with wake-on-change polling](https://smartscope.blog/en/blog/claude-code-delegate-tasks-meta-muse-using-spreadsheets-2026/), SmartScope: a tested hook reading a public Google Sheet, with zero idle tokens and about a minute from change to wake
