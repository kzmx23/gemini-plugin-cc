# Gemini plugin for Claude Code

Use Google's [Gemini CLI](https://github.com/google-gemini/gemini-cli) from inside [Claude Code](https://docs.anthropic.com/en/docs/claude-code) to review code or delegate tasks.

## What You Get

- `/gemini:review` for a normal read-only Gemini review
- `/gemini:adversarial-review` for a steerable challenge review
- `/gemini:rescue`, `/gemini:status`, `/gemini:result`, and `/gemini:cancel` to delegate work and manage background jobs

## Requirements

- **Google account or Gemini API key.**
  - Sign in with Google (free tier: 60 req/min, 1,000 req/day) or set `GEMINI_API_KEY` from [AI Studio](https://aistudio.google.com/apikey).
- **Node.js 18.18 or later**

## Install

### 1. Add the marketplace

```bash
claude /plugin marketplace add sakibsadmanshajib/gemini-plugin-cc
```

### 2. Install the plugin

```bash
claude /plugin install gemini@google-gemini
```

### 3. Reload plugins

In an active Claude Code session:

```
/reload-plugins
```

### 4. Run setup

```
/gemini:setup
```

If Gemini CLI is not installed, the plugin will offer to install it for you (`npm install -g @google/gemini-cli`).

If Gemini CLI is installed but not authenticated, run `!gemini` in Claude Code to authenticate interactively, or set `GEMINI_API_KEY` in your environment.

## Usage

### `/gemini:review`

Runs a Gemini review on your current work.

> [!NOTE]
> Code review especially for multi-file changes might take a while. It's generally recommended to run it in the background.

Use it when you want:

- a review of your current uncommitted changes
- a review of your branch compared to a base branch like `main`

Use `--base <ref>` for branch review. It also supports `--wait` and `--background`. It is not steerable and does not take custom focus text. Use [`/gemini:adversarial-review`](#geminiadversarial-review) when you want to challenge a specific decision or risk area.

Examples:

```bash
/gemini:review
/gemini:review --base main
/gemini:review --background
```

This command is read-only and will not perform any changes. When run in the background you can use [`/gemini:status`](#geministatus) to check on the progress and [`/gemini:cancel`](#geminicancel) to cancel the ongoing task.

### `/gemini:adversarial-review`

Runs a **steerable** review that questions the chosen implementation and design.

It can be used to pressure-test assumptions, tradeoffs, failure modes, and whether a different approach would have been safer or simpler.

It uses the same review target selection as `/gemini:review`, including `--base <ref>` for branch review.
It also supports `--wait` and `--background`. Unlike `/gemini:review`, it can take extra focus text after the flags.

Use it when you want:

- a review before shipping that challenges the direction, not just the code details
- review focused on design choices, tradeoffs, hidden assumptions, and alternative approaches
- pressure-testing around specific risk areas like auth, data loss, rollback, race conditions, or reliability

Examples:

```bash
/gemini:adversarial-review
/gemini:adversarial-review --base main challenge whether this was the right caching and retry design
/gemini:adversarial-review --background look for race conditions and question the chosen approach
```

This command is read-only. It does not fix code.

### `/gemini:rescue`

Hands a task to Gemini through the `gemini:gemini-rescue` subagent.

Use it when you want Gemini to:

- investigate a bug
- try a fix
- continue a previous Gemini task
- take a faster or cheaper pass with a smaller model

> [!NOTE]
> Depending on the task and the model you choose these tasks might take a long time and it's generally recommended to force the task to be in the background or move the agent to the background.

It supports `--background`, `--wait`, `--resume`, and `--fresh`. If you omit `--resume` and `--fresh`, the plugin can offer to continue the latest rescue thread for this repo.

Examples:

```bash
/gemini:rescue investigate why the tests started failing
/gemini:rescue fix the failing test with the smallest safe patch
/gemini:rescue --resume apply the top fix from the last run
/gemini:rescue --model pro investigate the flaky integration test
/gemini:rescue --model flash fix the issue quickly
/gemini:rescue --background investigate the regression
```

You can also just ask for a task to be delegated to Gemini:

```text
Ask Gemini to redesign the database connection to be more resilient.
```

**Notes:**

- if you do not pass `--model`, Gemini chooses its own defaults
- model aliases: `pro` (gemini-2.5-pro), `flash` (gemini-2.5-flash), `flash-lite` (gemini-2.5-flash-lite)
- you can also pass concrete model names like `gemini-3-pro-preview`
- follow-up rescue requests can continue the latest Gemini task in the repo

### `/gemini:status`

Lists active and recent Gemini jobs for this repository.

```bash
/gemini:status
/gemini:status <job-id>
/gemini:status --wait
```

### `/gemini:result`

Shows the full stored output for a finished job.

```bash
/gemini:result
/gemini:result <job-id>
```

### `/gemini:cancel`

Cancels an active background job.

```bash
/gemini:cancel
/gemini:cancel <job-id>
```

### `/gemini:setup`

Checks whether Gemini is installed and authenticated.
If Gemini is missing and npm is available, it can offer to install Gemini for you.

You can also use `/gemini:setup` to manage the optional review gate.

#### Enabling review gate

```bash
/gemini:setup --enable-review-gate
/gemini:setup --disable-review-gate
```

When the review gate is enabled, the plugin uses a `Stop` hook to run a targeted Gemini review based on Claude's response. If that review finds issues, the stop is blocked so Claude can address them first.

> [!WARNING]
> The review gate can create a long-running Claude/Gemini loop and may drain usage limits quickly. Only enable it when you plan to actively monitor the session.

## Typical Flows

### Review Before Shipping

```bash
/gemini:review
```

### Hand A Problem To Gemini

```bash
/gemini:rescue investigate why the build is failing in CI
```

### Start Something Long-Running

```bash
/gemini:rescue --background redesign the error handling across the API layer
/gemini:status
```

## Gemini Integration

The plugin communicates with Gemini CLI via **ACP** (Agent Client Protocol) — a JSON-RPC 2.0 interface over stdio. A persistent broker process keeps the connection alive across multiple commands within a Claude Code session.

### Common Configurations

If you want to change the default model or settings, configure them in your Gemini settings file:

**User-level:** `~/.gemini/settings.json`

```jsonc
{
  "modelConfigs": {
    "customAliases": {
      "precise-mode": {
        "extends": "chat-base",
        "modelConfig": {
          "generateContentConfig": { "temperature": 0.0 }
        }
      }
    }
  }
}
```

**Project-level:** `.gemini/settings.json` (overrides user settings)

Your configuration will be picked up based on:

- user-level config in `~/.gemini/settings.json`
- project-level overrides in `.gemini/settings.json`

Check out the [Gemini CLI docs](https://github.com/google-gemini/gemini-cli) for more configuration options.

### Authentication Methods

| Method | Setup | Best For |
|--------|-------|----------|
| Sign in with Google | `gemini` (interactive) | Desktop use |
| Gemini API Key | `export GEMINI_API_KEY=...` | CI/headless |
| Vertex AI | `export GOOGLE_CLOUD_PROJECT=...` | Enterprise |

### Moving The Work Over To Gemini

Delegated tasks and any review gate runs can be directly resumed inside Gemini by running `gemini --resume` with the session ID from `/gemini:result` or `/gemini:status`.

## FAQ

### Do I need a separate Gemini account for this plugin?

If you are already signed into Gemini on this machine, that account should work immediately. This plugin uses your local Gemini CLI authentication.

If you only use Claude Code today and have not used Gemini yet, you will also need to authenticate. The free tier (Sign in with Google) gives you 60 requests per minute and 1,000 per day. Set `GEMINI_API_KEY` for headless use, or run `!gemini` inside Claude Code to authenticate interactively.

### Does the plugin use a separate Gemini runtime?

The plugin starts Gemini in ACP mode (`gemini --acp`) and communicates via JSON-RPC. A broker process keeps the connection alive for the duration of your Claude Code session and is automatically cleaned up when the session ends.

### Will it use the same Gemini config I already have?

Yes. The plugin inherits your `~/.gemini/settings.json` and any project-level `.gemini/settings.json` overrides.

### Can I keep using my current API key or Vertex AI setup?

Yes. Because the plugin uses your local Gemini CLI, your existing authentication method and config still apply. If you use Vertex AI, ensure `GOOGLE_CLOUD_PROJECT` and `GOOGLE_CLOUD_LOCATION` are set.

## Architecture

```
Claude Code ──[Bash]──> gemini-companion.mjs ──[Unix socket]──> ACP Broker
                                                                    |
                                                              gemini --acp
                                                              (persistent)
```

- **gemini-companion.mjs** — Main CLI handling all subcommands
- **acp-broker.mjs** — Persistent daemon multiplexing JSON-RPC requests via Unix socket
- **acp-client.mjs** — Client with broker-first, direct-spawn fallback
- **15 lib modules** — Git context, state persistence, job tracking, rendering

## License

[MIT](LICENSE)
