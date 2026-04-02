# AskRally Plugin for Claude Code

Query calibrated AI personas through Rally's API. Ask a question, get authentic inner-monologue responses from a panel of synthetic personas grounded in real interview data, plus an actionable summary.

## Installation

### 1. Install the plugin

From inside Claude Code:

```
/install-plugin IrrealityLabs/askrally-plugin
```

Or from the terminal:

```bash
claude plugin marketplace add IrrealityLabs/askrally-plugin
claude plugin install askrally
```

### 2. Add your API key

Get an API key from [app.askrally.com](https://app.askrally.com), then either export it:

```bash
export ASKRALLY_API_KEY=rally_sk_your_key_here
```

Or add it to a `.env` file in whatever directory you'll run Claude from:

```
ASKRALLY_API_KEY=rally_sk_your_key_here
```

### 3. Install the plugin

```bash
claude plugin marketplace add IrrealityLabs/askrally-plugin
claude plugin install askrally
```

### Requirements

- Python 3.8+ (stdlib only, no pip install needed)
- An AskRally API key
- At least one audience created at [app.askrally.com](https://app.askrally.com)

## Usage

```
/ask-rally What do you think about electric scooters?
```

### Options

```
/ask-rally --count 8 --smart What features matter most in a city car?
/ask-rally --audience "General Population US" Would you consider a Chinese EV brand?
/ask-rally --voting Should we launch Feature A or Feature B?
/ask-rally --search "jazz enthusiasts" --filter-age-min 25 Would you pay for a music streaming service?
```

| Flag | Description | Default |
|------|-------------|---------|
| `--count N` | Number of personas to query | 5 |
| `--audience NAME` | Target a specific audience by name | auto-selected |
| `--smart` | Enhanced reasoning mode | off |
| `--voting` | Structured voting with reasoning | off |
| `--search "..."` | Semantic search for persona selection | random sample |
| `--session ID` | Continue an existing session | new session |

#### Demographic filters (require `--search`)

Filters tag personas as exact/inexact matches but don't exclude anyone. They require `--search` to take effect.

| Flag | Description |
|------|-------------|
| `--filter-age-min N` | Minimum age |
| `--filter-age-max N` | Maximum age |
| `--filter-gender "..."` | Gender filter (comma-separated) |
| `--filter-location "..."` | Location filter (partial match) |
| `--filter-occupation "..."` | Occupation filter (partial match) |
| `--filter-income "..."` | Income bracket filter (comma-separated) |

### Follow-ups

After the panel responds, ask follow-up questions naturally. The session preserves conversation history so personas remember what was discussed.

### Custom summaries

Ask for a re-summary with a specific angle:

> "Summarize focusing on the best quotes for marketing copy"

### Resample

Want more perspectives on the same question? Ask to add more personas to the panel.

## CLI

The plugin bundles a standalone CLI at `skills/ask-rally/scripts/askrally`. You can also use it directly:

```bash
python3 skills/ask-rally/scripts/askrally <command> [options]
```

| Command | Description |
|---------|-------------|
| `askrally me` | Show current user info |
| `askrally audiences [--search TEXT]` | List audiences |
| `askrally audience ID` | Show audience details and persona list |
| `askrally chat QUERY [options]` | Query personas |
| `askrally sessions` | List past sessions |
| `askrally session ID` | Show full session with message history |
| `askrally summarize ID PROMPT` | Custom re-summarization |
| `askrally resample ID -m INDEX -n COUNT` | Add more personas to a message |
| `askrally branch ID -m INDEX` | Branch a session at a message |

All commands accept `--json` for raw JSON output.

### Chat options

| Option | Description |
|--------|-------------|
| `-a, --audience ID` | Audience ID |
| `-s, --session ID` | Session ID (follow-ups) |
| `-n, --sample-size N` | Number of personas to sample |
| `--persona-ids ID,ID,...` | Select specific personas |
| `--mode fast\|smart` | Processing mode (default: fast) |
| `--smart` | Shorthand for `--mode smart` |
| `--voting` | Enable voting mode |
| `--no-summary` | Skip summary generation |
| `--search-query TEXT` | Semantic search for persona selection |
| `--search-pool audience\|genpop` | Search scope |
| `--filter-age-min N` | Min age |
| `--filter-age-max N` | Max age |
| `--filter-gender G,G,...` | Gender filter |
| `--filter-location TEXT` | Location filter (partial match) |
| `--filter-occupation TEXT` | Occupation filter (partial match) |
| `--filter-income B,B,...` | Income bracket filter |
| `--memory TEXT` | Inject context (repeatable) |
| `--system-prompt TEXT` | Override persona system prompt |
