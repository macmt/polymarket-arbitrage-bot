# Contributing

Maintainers coordinate in the [knit-finances](https://github.com/knit-finances) org; see `CONTRIBUTORS.md` for the active roster.

## Development flow

1. Install dependencies with `npm install`.
2. Copy `.env.example` to `.env` and use simulation defaults.
3. Run `npm run typecheck` before opening a PR.
4. For runnable checks, use `npm run sim`.
5. Rebase feature branches on `develop` and resolve conflicts before requesting review.

## Commit style

- Keep commits small and single-purpose.
- Prefer messages that explain intent and operational impact.
- Update docs alongside behavioral changes.
