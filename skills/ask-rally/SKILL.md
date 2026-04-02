---
name: ask-rally
description: Query AskRally's persona-based audience research API from the terminal. Use when the user wants to ask questions to synthetic personas, get audience feedback, test messaging, do consumer research, simulate focus groups, or anything involving AskRally. Also use when the user mentions personas, audience research, panel feedback, or wants opinions from a demographic. Even if they just say "ask the audience" or "what would people think about X" — this skill applies.
user_invocable: true
---

# AskRally API Skill

Query calibrated AI personas through Rally's API. Each persona is grounded in real interview data and responds with authentic inner-monologue thoughts. The API handles persona selection, parallel querying, and summary generation — this skill orchestrates a bundled CLI that wraps it.

## Prerequisites

Set `ASKRALLY_API_KEY` in the environment or in a `.env` file in the working directory. Get an API key from https://app.askrally.com.

## CLI

The CLI is bundled with this skill. Run it via:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/askrally <command> [options]
```

## Invocation

The user invokes this skill with `/ask-rally <query>` or `/ask-rally`. If no query is provided, ask for one.

### Flags (parsed from the user's message)

| Flag | Maps to | Default |
|------|---------|---------|
| `--count N` or `--sample-size N` | `--sample-size` on `chat` | 5 |
| `--audience NAME` | Look up audience ID, then `--audience` on `chat` | (pick for user) |
| `--smart` | `--smart` on `chat` | off |
| `--voting` | `--voting` on `chat` | off |
| `--search "..."` | `--search-query` on `chat` | (none) |
| `--filter-age-min`, `--filter-age-max`, `--filter-gender`, `--filter-location`, `--filter-occupation`, `--filter-income` | demographic filters on `chat` (require `--search-query` to take effect; tags personas as exact/inexact, does not exclude) | (none) |
| `--session ID` | `--session` on `chat` for follow-ups | (none) |

## Step-by-Step Execution

### Step 1: Parse the invocation

Extract the query and any flags from the user's message. Examples:

- `/ask-rally What do you think about electric scooters?` -> query only, defaults
- `/ask-rally --count 8 --smart How important is range?` -> sample_size=8, smart mode
- `/ask-rally --audience "General Population US" What matters in a city car?` -> specific audience
- `/ask-rally --search "jazz enthusiasts" --filter-age-min 25 --filter-age-max 45 Would you pay for a music streaming service?` -> semantic search + filters
- `/ask-rally --voting Should we launch Feature A or Feature B?` -> voting mode

### Step 2: Resolve the audience

If the user specified an audience by name, look it up:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/askrally audiences --search "search term"
```

Pick the best match from the results and use its ID. If unsure, show the user the matches and ask them to pick.

If no audience was specified, choose a sensible default:
1. If the user has mentioned a specific audience before in this conversation, reuse it.
2. Otherwise, list audiences. If the question targets a specific group (parents, tech workers, etc.), pick the most relevant audience. If it's general, prefer "General Population US" or ask the user to pick.

To create a new audience, direct the user to https://app.askrally.com.

### Step 3: Design the research

Before sending anything to the API, think about what the user actually wants to learn and how best to get a useful answer from personas. The user's question as written may not be the best thing to ask a persona directly.

**Ask yourself:**

1. **Is this a comparison question?** ("Do women like X more than men?", "How do millennials vs boomers feel about...") If so, don't ask the comparison question directly — personas can only speak for themselves. Instead, split into separate queries with demographic filters or targeted sampling, then compare the results yourself. For example, "Do women like cake more than men?" becomes two calls — one asking 5 women "How do you feel about cake?" and one asking 5 men the same — followed by your own comparative analysis.

2. **Is the question leading or biased?** ("Don't you think X is bad?") Rephrase to be neutral so personas give honest reactions rather than agreeing with a premise.

3. **Would the question make sense to a real person?** Personas respond as themselves — they don't have access to population-level data. Asking "What percentage of people prefer X?" won't work. Instead, ask about their personal experience and preferences, then you synthesize the patterns.

4. **Does it need multiple rounds?** Some questions benefit from a warm-up question followed by the real one, or from asking the same panel a sequence of probing questions within a session.

5. **Is a straight pass-through fine?** Many questions work great as-is: "What do you think about electric scooters?", "Would you pay $50/month for this?", "What's your biggest frustration with grocery shopping?" If the question is open-ended and asks for personal opinion, just send it directly.

**After designing your approach, briefly tell the user your plan before executing.** For example: "I'll ask 5 women and 5 men separately how they feel about cake, then compare their responses." This takes one sentence — don't over-explain.

### Step 4: Send the query (or queries)

Build and run the chat command:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/askrally chat "Your question here" \
  --audience AUDIENCE_ID \
  --sample-size 5 \
  [--smart] \
  [--voting] \
  [--search-query "semantic search text"] \
  [--filter-age-min N] [--filter-age-max N] \
  [--filter-gender "male,female"] \
  [--filter-location "California"] \
  [--session SESSION_ID]
```

The API returns persona responses and a summary in one call. No subagents needed.

**Timeout note:** Large panels (10+ personas) can take 30-60 seconds. Set a generous Bash timeout (120000ms).

### Step 5: Present the results and analyze

**If you ran a single query**, present each persona response, then the API-generated summary.

**If you ran multiple queries** (e.g., split by demographic), present each group's responses separately, then write your own comparative analysis. The API summaries cover each group individually — your job is the cross-group synthesis that answers the user's original question. This is where the real value is.

**Format:**
```
---
### [Name], [persona_id]
> [Their inner monologue response]
---
```

Then summary or analysis:
```
---
## Summary / Analysis
[Synthesis here]
---
```

### Step 6: Post-query actions (optional, offer when relevant)

#### 6a: Custom summary

If the user wants a different take on the responses:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/askrally summarize SESSION_ID "Pick out the best quotes for marketing copy"
```

#### 6b: Add more personas (resample)

If 5 personas wasn't enough signal:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/askrally resample SESSION_ID --message-index 0 --sample-size 5
```

#### 6c: Follow-up question

Continue the conversation with the same panel:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/askrally chat "Follow-up question" --session SESSION_ID
```

The session preserves conversation history so personas remember previous questions.

### Step 7: Offer next steps

After presenting results, briefly mention what the user can do next:

```
**Next steps:** Ask a follow-up to this panel, resample for more perspectives,
request a custom summary, or start fresh with a new audience.
```

## CLI Reference

All commands support `--json` for raw JSON output (useful for programmatic processing).

| Command | Description |
|---------|-------------|
| `askrally me` | Show current user info |
| `askrally audiences [--search TEXT]` | List audiences (with optional name filter) |
| `askrally audience AUDIENCE_ID` | Show audience details and persona list |
| `askrally chat QUERY [options]` | Query personas (main command) |
| `askrally sessions` | List past chat sessions |
| `askrally session SESSION_ID` | Show full session history |
| `askrally summarize SESSION_ID PROMPT` | Generate a custom summary |
| `askrally resample SESSION_ID -m INDEX -n COUNT` | Add more personas to a message |
| `askrally branch SESSION_ID -m INDEX` | Branch a session at a message |

### Chat options

| Option | Description |
|--------|-------------|
| `--audience, -a` | Audience ID |
| `--session, -s` | Session ID (for follow-ups) |
| `--sample-size, -n` | Number of personas (default: server decides) |
| `--persona-ids` | Comma-separated specific persona IDs |
| `--mode fast\|smart` | Processing mode |
| `--smart` | Shorthand for `--mode smart` |
| `--voting` | Enable voting mode |
| `--no-summary` | Skip summary generation |
| `--search-query` | Semantic search for persona selection |
| `--search-pool audience\|genpop` | Where to search |
| `--filter-age-min`, `--filter-age-max` | Age range |
| `--filter-gender` | Comma-separated genders |
| `--filter-location` | Location (partial match) |
| `--filter-occupation` | Occupation (partial match) |
| `--filter-income` | Comma-separated income brackets |
| `--memory` | Inject context (repeatable) |
| `--system-prompt` | Override default persona instructions |

## Behavioral Notes

- **The API does the heavy lifting.** Persona responses come from Rally's calibrated backend. The quality depends on the audience's calibration data, not on Claude's roleplay ability.
- **Semantic search is powerful.** When the user asks about a niche topic, use `--search-query` to find the most relevant personas rather than random sampling. Asking about jazz? Add `--search-query "jazz enthusiasts who appreciate live music"`.
- **Smart mode costs more but thinks deeper.** Default to fast mode. Only suggest smart mode if the user asks for detailed analysis or the question is complex.
- **Voting mode for decisions.** If the user is choosing between options (A vs B, name preferences, feature prioritization), suggest voting mode.
- **Session continuity matters.** Always capture the `session_id` from the first chat response and reuse it for follow-ups. This preserves conversation history so personas give contextually aware responses.
- **Sample size guidance.** 5 is good for focused questions. 8-10 for broader exploration. Above 15 rarely adds signal and costs more.
- **This is a simulation.** The personas are AI-generated from real interview data. Useful for generating hypotheses and pressure-testing messaging, but not a substitute for talking to real people.
