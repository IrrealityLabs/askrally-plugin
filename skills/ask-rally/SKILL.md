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

### Step 3: Send the query

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

### Step 4: Present the results

The CLI outputs formatted text with persona responses and a summary. Present it to the user with light reformatting:

**For each persona response:**
```
---
### [Name], [persona_id]
> [Their inner monologue response]
---
```

**Then the summary:**
```
---
## Summary
[The API-generated summary]
---
```

If the summary feels thin or the user would benefit from a different angle, offer a custom summary (Step 5a). Otherwise, move to Step 6.

### Step 5: Post-query actions (optional, offer when relevant)

#### 5a: Custom summary

If the user wants a different take on the responses:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/askrally summarize SESSION_ID "Pick out the best quotes for marketing copy"
```

#### 5b: Add more personas (resample)

If 5 personas wasn't enough signal:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/askrally resample SESSION_ID --message-index 0 --sample-size 5
```

#### 5c: Follow-up question

Continue the conversation with the same panel:

```bash
python3 ${CLAUDE_SKILL_DIR}/scripts/askrally chat "Follow-up question" --session SESSION_ID
```

The session preserves conversation history so personas remember previous questions.

### Step 6: Offer next steps

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
