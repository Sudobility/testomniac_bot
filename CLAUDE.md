# testomniac_bot

> **Git policy — never auto-commit or auto-push.** Leave your work in the working tree.
> Run `git commit`, `git push`, `gh pr create`, or `scripts/push_all.sh` **only when the user
> explicitly asks in that turn**. Approval for an earlier change does not carry forward, and
> finishing a task is not permission to commit it.
> Do not use feature branches for code changes — stay on the current branch.

GitHub App that auto-reviews pull requests using Claude and posts the review as a PR comment.

## Stack

- **Runtime**: Bun (not Node). Package manager: Bun (`bun.lock`).
- **Server**: Hono
- **GitHub**: `@octokit/app` + `@octokit/webhooks`
- **AI**: `@anthropic-ai/sdk`, model `claude-sonnet-4-6`
- TypeScript, ESM, strict. Source imports use `.js` extensions (ESM/bundler resolution).

## Commands (verified)

- `bun install` — install deps (resolves `@sudobility/*` from GitHub Packages per `.npmrc`)
- `bun dev` — start with hot reload (`bun run --hot src/index.ts`)
- `bun start` — production start
- `bun run typecheck` — `tsc --noEmit` (the only check; there is **no test suite**)

## Architecture

```
src/index.ts    Hono app. GET /health, POST /webhook (verifyAndReceive).
src/config.ts   Env loading; required() throws on missing var at startup.
src/github.ts   githubApp (App instance), installation octokit, getPullRequestDiff, submitReview.
src/reviewer.ts Anthropic client + SYSTEM_PROMPT; reviewDiff(diff) -> text.
src/webhook.ts  registerWebhooks() wires pull_request.opened/.synchronize -> fetch diff -> review -> post.
```

Flow: webhook event → installation Octokit → fetch diff (`mediaType: diff`) → truncate to 100k chars → `reviewDiff` → `submitReview` as `COMMENT` review.

## Conventions & gotchas

- Webhook signature is verified on every `POST /webhook` via `verifyAndReceive`; bad signature → 400.
- `GITHUB_PRIVATE_KEY` is **base64-encoded** in env; `config.ts` decodes it. Not raw PEM.
- Reviews are always posted as `event: "COMMENT"` — never APPROVE/REQUEST_CHANGES.
- Diffs over 100k chars are truncated before sending to Claude.
- Review errors are caught/logged per-PR, never thrown — a failed review must not crash the server.
- `reviewDiff` assumes the first content block is text and throws otherwise.
- Default port 8022 (`PORT` env overrides). Dockerfile EXPOSEs 8022 and healthchecks `/health`.

## Environment

Requires `.env` (see `.env.example`): `GITHUB_APP_ID`, `GITHUB_PRIVATE_KEY` (base64), `GITHUB_WEBHOOK_SECRET`, `ANTHROPIC_API_KEY`, optional `PORT`.

## CI/CD

`.github/workflows/ci-cd.yml` calls reusable `johnqh/workflows/.github/workflows/unified-cicd.yml@main`.
npm publish skipped; Docker image published with `publish-target: github`. Part of the testomniac family.
