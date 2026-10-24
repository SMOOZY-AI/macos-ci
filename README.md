# smoozy-macos-ci

CI/CD pipeline for the extracted **Smoozy macOS** app.

The actual application source lives in a **private GitLab repository**:

- `https://gitlab.com/smoozy-app/smoozy-macos`

This GitHub repository contains **only GitHub Actions workflows**. It has no application code. Its only job is to give us a public Actions repo while the real macOS source stays private on GitLab.

## How it works

```text
┌────────────────────────────────┐     ┌─────────────────────────────┐
│ GitLab (private)               │     │ GitHub (this repo, public)  │
│ gitlab.com/smoozy-app/         │     │ Smoozy-LLC/smoozy-macos-ci  │
│   smoozy-macos                 │     │                             │
│                                │     │   .github/workflows/        │
│ - all source code              │     │   release.yml               │
│ - all commits (incl. version   │     │   ci.yml                    │
│   bumps pushed back by CI)     │     │                             │
│ - issues, MRs, code review     │     │   (no application code)     │
└──────────┬─────────────────────┘     └──────────┬──────────────────┘
           │                                      │
           │  git clone with SSH deploy key       │  workflow_dispatch
           └──────────────────────────────────────┘  triggers
                                │
                                ▼
                    ┌────────────────────────┐
                    │ macos-14 runner        │
                    │  1. clone GitLab       │
                    │  2. pnpm install       │
                    │  3. tauri build        │
                    │  4. build DMG          │
                    │  5. sign + notarize    │
                    │  6. upload artifacts   │
                    │  7. publish latest.json│
                    │  8. git push bump ->   │
                    │     GitLab             │
                    └────────────────────────┘
```

## Triggering a release

1. GitHub -> `Actions` -> `Release macOS`
2. Choose `bump: patch` or `minor`
3. Run workflow
4. After the job finishes, the new updater metadata and DMG artifacts are published

## Manual sanity check

Use `Actions -> CI (manual sanity check)` when you want a quick GitHub-side confidence pass without shipping a release.

- default run: `pnpm typecheck` + `pnpm build` + `cargo check`
- optional flag `run_tests=true`: also run the inherited Vitest suite

The test suite flag is optional because the extracted app intentionally preserves the current shared/runtime behavior, and there are a few pre-existing test failures that already reproduce in the original source tree too.

## Required secrets

Add these in GitHub -> Settings -> Secrets and variables -> Actions:

| Secret | Purpose |
|---|---|
| `GITLAB_SSH_PRIVATE_KEY` | Clone source + push version bump back to GitLab over SSH |
| `GITLAB_TOKEN` | GitLab API access for release creation |
| `GITLAB_DEPLOY_TOKEN_USERNAME` | Upload artifacts to GitLab Package Registry |
| `GITLAB_DEPLOY_TOKEN_PASSWORD` | Same deploy token, token value |
| `TAURI_SIGNING_PRIVATE_KEY` | Sign updater artifacts with the existing minisign key |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD` | Minisign passphrase |
| `TAURI_SIGNING_PRIVATE_KEY_WEST` | Sign updater artifacts of the **west** build (`flavor: west`) — its own key pair (key id `3EA2495B90742A8E`, pubkey in `src-tauri/tauri.west.conf.json`), same key as in `windows-ci`. `release.mjs` checks the key id against the pubkey and stops on a mismatch |
| `TAURI_SIGNING_PRIVATE_KEY_PASSWORD_WEST` | Passphrase for the west key |
| `UPDATE_SECRET` | Bearer token for backend updater publish |
| `APPLE_CERTIFICATE_BASE64` | Developer ID `.p12` in base64 |
| `APPLE_CERTIFICATE_PASSWORD` | Password for the `.p12` certificate |
| `APPLE_API_KEY_ID` | App Store Connect API key id |
| `APPLE_API_ISSUER` | App Store Connect issuer id |
| `APPLE_API_KEY_CONTENT` | Raw `.p8` key contents |

## Notes

- Keep the updater signing key identical to the current app. Regenerating it would break auto-updates for existing users.
- This repo is intentionally public and intentionally contains no source code.
- The optional Vitest run currently mirrors the original repo's baseline, including a small set of inherited red tests.
- GitHub Actions talks to the private GitLab source repo over a dedicated SSH deploy key, not over the broken local HTTPS Git token.

## Flavors (W2)

`flavor: ru` is the release as it always ran: bump, build, notarize (if ticked), publish, push the bump commit.

`flavor: west` re-builds the last Russian release (`chore: release <version>` on GitLab `main`) as the west build — own update channel `/api/update/macos-west`, own key (`TAURI_SIGNING_PRIVATE_KEY_WEST`), package `smoozy-macos-west`, downloads record `?market=west`. No bump, no push; `bump` is ignored. It requires `notarize: true` (a west user installs from the DMG with nothing to vouch for it). Run it after the Russian release of the same version.
