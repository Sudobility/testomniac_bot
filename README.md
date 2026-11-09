# @sudobility/testomniac-bot

A GitHub App that automatically reviews pull requests using Claude, acting as a team member. When a PR is opened or updated, the bot fetches the diff, sends it to Claude for review, and posts the result as a PR review comment.

## Tech stack

- **Runtime**: [Bun](https://bun.sh)
- **HTTP server**: [Hono](https://hono.dev)
- **GitHub integration**: `@octokit/app` + `@octokit/webhooks`
- **AI**: `@anthropic-ai/sdk` (model `claude-sonnet-4-6`)
- **Language**: TypeScript (ESM, strict mode)

## Project layout

```
src/
├── index.ts      # Hono server + /webhook and /health routes
├── config.ts     # Loads and validates required env vars
├── github.ts     # GitHub App auth, fetch PR diff, submit review
├── reviewer.ts   # Claude API call + review system prompt
└── webhook.ts    # pull_request.opened / .synchronize handlers
```

## Setup

### 1. Create a GitHub App

1. Go to **GitHub Settings > Developer settings > GitHub Apps > New GitHub App**.
2. Set the webhook URL to `https://your-domain.com/webhook`.
3. Permissions:
   - **Pull requests**: Read & Write
   - **Contents**: Read
4. Subscribe to events: **Pull request**.
5. Generate a private key and download the `.pem` file.

### 2. Configure environment

```bash
cp .env.example .env
```

Fill in (all required unless noted):

- `GITHUB_APP_ID` — from the app settings page
- `GITHUB_PRIVATE_KEY` — base64-encoded `.pem` contents: `base64 -i your-app.pem`
- `GITHUB_WEBHOOK_SECRET` — the secret you set when creating the app
- `ANTHROPIC_API_KEY` — your Anthropic API key
- `PORT` — HTTP port (optional, defaults to `8022`)

The process exits on startup if any required var is missing.

### 3. Install dependencies

```bash
bun install
```

> This package and its `@sudobility/*` scope resolve from GitHub Packages (see `.npmrc`). Installing requires a GitHub token with `read:packages` configured for the `@sudobility` scope.

### 4. Run

```bash
bun dev      # start with hot reload (src/index.ts)
bun start    # production start
```

For local development, use [smee.io](https://smee.io) or ngrok to tunnel GitHub webhooks to `http://localhost:8022/webhook`.

## Scripts

| Command              | Description                     |
| -------------------- | ------------------------------- |
| `bun dev`            | Start with hot reload           |
| `bun start`          | Production start                |
| `bun run typecheck`  | Type-check with `tsc --noEmit`  |

There is no automated test suite; `bun run typecheck` is the available check.

## HTTP endpoints

- `GET /health` — returns `{ "status": "ok" }`
- `POST /webhook` — GitHub webhook receiver (verifies `x-hub-signature-256`; returns `400` on invalid signature)

## How it works

When a PR is opened or synchronized, the bot:

1. Resolves an installation Octokit client from the webhook payload.
2. Fetches the PR diff via the GitHub API.
3. Truncates the diff to 100k characters to stay within token limits.
4. Sends it to Claude for review.
5. Posts the review as a PR review comment (`COMMENT` event — never approve/request-changes).

Errors during review are logged and swallowed so a single failing PR does not crash the server.

## Docker

```bash
docker build --build-arg NPM_TOKEN=<github-packages-token> -t testomniac_bot .
docker run -p 8022:8022 --env-file .env testomniac_bot
```

The image runs as a non-root user under `dumb-init` and has a `/health` healthcheck.

## CI/CD

`.github/workflows/ci-cd.yml` delegates to the reusable `johnqh/workflows` unified CI/CD pipeline on pushes/PRs to `main` and `develop`. npm publishing is skipped; the Docker image is published with `publish-target: github`.

## License

BUSL-1.1
