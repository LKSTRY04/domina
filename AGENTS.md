# AGENTS.md — Meridian (DLMM LP Agent)

## Repo type
Node.js 18+ autonomous Solana agent. ReAct loop with cron-based screening and management cycles.

## Entrypoints
- `index.js` — main autonomous agent (REPL + cron + Telegram polling)
- `cli.js` — direct CLI tool access (`node cli.js <cmd>`)
- `agent.js` — ReAct loop (LLM → tools → repeat)

## Developer commands

```bash
npm start          # live mode
npm run dev        # DRY_RUN=true (no on-chain txs)
npm run test       # syntax check only (no unit tests)

# discord-listener runs separately
cd discord-listener && npm start
```

**CLI tool examples:**
```bash
node cli.js screen [--dry-run]      # one screening cycle
node cli.js manage [--dry-run]      # one management cycle
node cli.js positions               # open positions
node cli.js pnl <addr>             # PnL for position
node cli.js balance                # wallet balances
node cli.js candidates --limit 5   # top pool candidates
node cli.js deploy --pool <addr> --amount <sol>
node cli.js claim --position <addr>
node cli.js close --position <addr>
node cli.js lessons
node cli.js evolve                 # threshold evolution (needs 5+ closed positions)
```

## Config
- `user-config.json` — thresholds, deploy size, models, intervals (gitignored template: `user-config.example.json`)
- `.env` — API keys, private key, RPC, Telegram tokens (NEVER commit)
- `.envrypt` + `.env.raw` — encrypted env flow (gitignored)
- env can also live in `~/.meridian/.env` and `~/.meridian/.envrypt`

## Important quirks

**DRY_RUN flag** — must appear before env loads:
```bash
DRY_RUN=true node index.js
node cli.js screen --dry-run   # flag position matters
```

**postinstall runs `node scripts/patch-anchor.js`** after every `npm install`.

**onchainos CLI** is required by the screener agent — not installed via npm. It's a separate tool. Used for smart money signals and token risk data.

**No unit tests** — only `find . -name '*.js' -exec node --check {} \;` syntax validation. Test files (`test/`) are integration-style, not framework-based.

**PM2 for production Telegram** — `pm2 start ecosystem.config.cjs`. After `git pull`, always run `npm install` then `pm2 restart` to pick up lockfile changes.

**HiveMind** — blanks in `hiveMindUrl`/`hiveMindApiKey` fall back to built-in Agent Meridian defaults. No explicit off switch.

## Effect-TS best practices

**MUST** run `npm run explore-repos` before writing Effect code — explore the `repos/effect` directory for patterns, utilities, and conventions.

## Architecture

```
index.js     — REPL + cron + Telegram polling
agent.js     — ReAct loop
config.js    — user-config.json + .env merging
state.js     — positions (state.json)
lessons.js   — learning engine
pool-memory.js — per-pool deploy history
decision-log.js — structured deploy/close/skip rationale
cli.js       — all tools as subcommands (JSON output)
tools/
  definitions.js  — OpenAI tool schemas
  executor.js    — dispatch + safety checks
  dlmm.js        — Meteora DLMM SDK wrapper
  screening.js   — pool discovery
  wallet.js      — balances + Jupiter swap
  token.js       — token info, holders, narrative
  study.js       — top LPer study via LPAgent API
discord-listener/
  index.js       — Discord selfbot
  pre-checks.js  — signal pipeline (dedup, rug check, fees)
```

## Two agents run independently

| Agent | Interval | Role |
|---|---|---|
| Screening | 30 min | Pool discovery + deploy |
| Management | 10 min | Position monitoring, claim, close |

Each runs through the ReAct harness in `agent.js`.

## Claude Code integration
`.claude/agents/screener.md` and `.claude/agents/manager.md` define sub-agents. Slash commands (`/screen`, `/manage`, etc.) live in `.claude/commands/`. These are the primary terminal interface.