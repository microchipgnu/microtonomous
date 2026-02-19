# GitClaw

An autonomous agent that lives inside a GitHub repository.

It wakes up every 30 minutes, reads its memory, looks around, decides what matters, acts, and writes down what it did. Every cycle leaves a proof.

## Setup

### 1. Use this template

Click **"Use this template"** → **"Create a new repository"**.

### 2. Add secrets

Go to **Settings → Secrets and variables → Actions** and add:

| Secret | Description |
|--------|-------------|
| `AGENTWALLET_USERNAME` | Your AgentWallet username |
| `AGENTWALLET_API_TOKEN` | Your AgentWallet API token (`mf_...`) |

That's it. All workflows are paused until these secrets exist.

**Need an AgentWallet?**
1. Go to `https://frames.ag/connect?email=YOUR_EMAIL`
2. Save the `username` and `apiToken` as secrets
3. Fund your wallet at `https://frames.ag/u/YOUR_USERNAME`

### 3. Run

The heartbeat starts automatically every 30 minutes. To test immediately:

**Actions → GitClaw Agent → Run workflow** → use defaults.

Check `.gitclaw/memory/state.json` after the run — cycle should be `1`.

### 4. Optional: GPG signing

Run **Actions → Setup GPG** once (requires a `GH_TOKEN` secret with admin access). After that, all GitClaw commits show as "Verified".

## How It Works

Every 30 minutes (and on issue/comment events), the agent runs a 4-phase cycle:

1. **Wake** — read identity, state, focus, learnings, visitors
2. **Gather** — fetch open issues, PRs, recent commits; categorize by label
3. **Act** — execute directives, respond to visitors, work on self-tasks
4. **Reflect** — write journal, update focus, log cycle, write proof

### Priority System

Issues are auto-labeled by the gate workflow when opened:

| Label | Who | Priority |
|-------|-----|----------|
| `directive` | Repo owner, title starts with `[directive]` | Highest |
| `self` | Repo owner, normal issue | Low |
| `visitor` | Anyone else | Medium |
| `flagged` | Visitor, caught by safety scanner | Skipped |

### Safety

Visitor issues are scanned for prompt injection, phishing, financial manipulation, spam, and abuse before the agent engages. Flagged issues get a warning and are skipped.

## Files

```
.gitclaw/
├── config.json                 # Settings
├── workflows/
│   ├── heartbeat.prose         # Autonomous cycle (every 30 min)
│   ├── review.prose            # PR code review
│   └── x-daily-pulse.prose     # Twitter briefing (daily 9 AM)
├── safety/
│   └── scan.sh                 # Content safety scanner
├── memory/
│   ├── self.md                 # Identity and values
│   ├── state.json              # Cycle counter
│   ├── focus.md                # Current focus
│   ├── learnings.md            # What the agent figured out
│   ├── visitors.json           # Visitor tracking
│   ├── cycles.jsonl            # Structured cycle log
│   └── index.md                # Memory map
├── proofs/                     # Proof-of-thought per cycle
└── agents/                     # Persistent agent state

.github/workflows/
├── gitclaw.yml                 # Heartbeat + events
├── x-daily-pulse.yml           # Daily Twitter pulse
├── gate.yml                    # Auto-label issues + safety
└── setup-gpg.yml               # One-time GPG setup

opencode.json                   # Inference provider config
AGENTS.md                       # Agent identity document
```

## Customization

### Change the heartbeat frequency

Edit `cron` in `.github/workflows/gitclaw.yml`:

```yaml
schedule:
  - cron: '0,30 * * * *'   # Every 30 min (default)
  - cron: '0 * * * *'      # Every hour
  - cron: '*/15 * * * *'   # Every 15 min
```

### Change the model

Edit `opencode.json`. See available models:

```bash
curl https://registry.frames.ag/api/service/openrouter/v1/models
```

### Add a workflow

Create a `.prose` file in `.gitclaw/workflows/` and update routing in `gitclaw.yml`.

### Edit the agent's identity

Modify `.gitclaw/memory/self.md` — the agent reads this at the start of every cycle.

## Architecture

```
┌──────────────────────────────────────────────┐
│            GIT REPOSITORY                     │
│           (Persistent Layer)                  │
│                                               │
│  .gitclaw/memory/     ← identity + state     │
│  .gitclaw/proofs/     ← proof-of-thought     │
│  .gitclaw/workflows/  ← behavior             │
└─────────────────┬────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────┐
│         GITHUB ACTIONS                        │
│        (Ephemeral Compute)                    │
│                                               │
│  Checkout → OpenCode → wordspace run          │
│  → commit results → runner destroyed          │
└─────────────────┬────────────────────────────┘
                  │
┌─────────────────▼────────────────────────────┐
│         FRAMES REGISTRY                       │
│     (Inference + Services)                    │
│                                               │
│  registry.frames.ag   ← LLM inference        │
│  frames.ag            ← wallet + APIs         │
└──────────────────────────────────────────────┘
```

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `AGENTWALLET_USERNAME` | **Yes** | AgentWallet username |
| `AGENTWALLET_API_TOKEN` | **Yes** | AgentWallet API token |
| `GITHUB_TOKEN` | Auto | Provided by GitHub Actions |
| `GPG_PRIVATE_KEY` | Optional | From `setup-gpg` workflow |
| `GPG_KEY_ID` | Optional | From `setup-gpg` workflow |
