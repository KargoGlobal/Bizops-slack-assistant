# AGENTS.md

## What this is

A Slack bot for BizOps help (Salesforce questions from sellers: opportunity setup, stage movement errors, ownership changes, splits, referrals). It answers DMs from a curated knowledge base, optionally with OpenAI, and offers a BizOps handoff message when it cannot help. It also watches one intake channel and posts a "How do you want help?" prompt on new top-level messages.

Node, ES modules (`"type": "module"`), `@slack/bolt` with `@vercel/slack-bolt`. No tests, no lint config, no CI workflows in the repo.

## Layout

- `api/slack/events.js`: Vercel-style HTTP handler. `GET` returns "Slack bot is running". Other methods are converted from a Node request to a Web `Request` and passed to the Bolt handler from `lib/slackApp.js`. Body parsing is disabled (`config.api.bodyParser = false`) and the handler reads the raw request stream itself, so do not add a body parser in front of it.
- `lib/slackApp.js`: the live Bolt `App` and `VercelReceiver`. Handlers: DM messages, intake-channel messages, and the `start_private_chat` and `bizops_help` button actions.
- `lib/ai.js`: classification, intent detection and reply building. `getBizOpsReply` uses OpenAI only when `USE_OPENAI === "true"` and `OPENAI_API_KEY` is set. Otherwise it falls back to `getMockBizOpsReply` (rule-based, no network).
- `lib/knowledge.js`: hard-coded `KNOWLEDGE_BASE` entries (keywords, summary, snippets, `source: "curated_v1"`) and `getKnowledgeContext`.
- `lib/conversationState.js`: per-channel conversation memory in a module-level `Map`, last 6 turns.
- `lib/escalation.js`: builds the BizOps handoff message text.
- `lib/logger.js`: `logInfo` / `logWarn` / `logError` with a `[bizops-bot]` prefix.
- `app.js`: see gotchas.

## Environment variables

Read in code (there is no `.env.example`; `.env` is gitignored):

- `SLACK_BOT_TOKEN`, `SLACK_SIGNING_SECRET`
- `TARGET_CHANNEL_ID`: the intake channel the bot watches
- `BIZOPS_HUB_URL`, `BIZOPS_GPT_URL`: link buttons in the intake prompt
- `USE_OPENAI`, `OPENAI_API_KEY`, `OPENAI_MODEL` (defaults to `gpt-5.2` in `lib/ai.js`)

## Commands

- `npm install` installs dependencies.
- `npm test` is the npm placeholder and exits 1 ("no test specified"). There are no tests.
- No start, build or deploy script is defined in `package.json`. The `main` field points at `index.js`, which does not exist.

## Gotchas

- `app.js` is a standalone Bolt app (`app.start(process.env.PORT || 3000)`) that duplicates the keyword classifier and the `start_private_chat` action. It uses `require(...)`, but `package.json` sets `"type": "module"`, so Node loads it as ESM and `require` is not defined. Nothing in the repo imports it. The live code path is `api/slack/events.js` into `lib/slackApp.js`. If you change bot behavior, change `lib/`, and check whether `app.js` is still wanted before editing it.
- Conversation state and the knowledge base live in process memory and source. On a serverless host, state will not reliably survive between invocations.
- `lib/slackApp.js` logs the text of incoming DMs (`dm_received`). Keep that in mind before logging more user content.
- Knowledge entries and keyword lists are plain arrays in `lib/knowledge.js` and `lib/ai.js`. Match the existing shape when adding entries.
- No CODEOWNERS file, no required-label workflow.
