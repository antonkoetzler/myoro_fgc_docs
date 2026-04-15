# CI proxy — not the source

> **This repository is not the source of truth.**
> The authoritative MFGC repository lives at
> **https://git.myoro.com.br/flqn/myoro_fgc_docs**.
> File issues, read docs, and browse code there.

This GitHub repository exists for one reason: to borrow GitHub Actions' free **macOS** and **Windows** runner minutes. MFGC is hosted on a self-managed Forgejo instance; Forgejo's runner is Linux-only, so macOS and Windows desktop builds have nowhere to run without this shim.

## What lives here

```
.
├── .github/
│   └── workflows/
│       └── desktop-crossbuild.yml   ← the entire reason this repo exists
└── README.md                         ← this file
```

No source code. No crates. No Rust, no TypeScript, no docs — the build clones those fresh from Forgejo at the moment it runs.

## How it works

```
Forgejo                                              GitHub (this repo)
───────                                              ──────────────────
release.yml builds Linux + tags + publishes
  ↓ POST /dispatches (repository_dispatch)
                                                     desktop-crossbuild.yml fires
                                                       ↓ git clone Forgejo at tag
                                                       ↓ build macOS arm64 / x86_64 / Windows x64
                                                       ↓ upload to Forgejo Release via REST
```

1. A developer bumps the version on Forgejo, commits, pushes.
2. Forgejo's `release.yml` builds the Linux artifacts, creates the Git tag, publishes the Forgejo Release.
3. As its last step, Forgejo calls GitHub's `POST /repos/…/dispatches` endpoint with `{event_type: "build-release", client_payload: {tag: "v0.1.0-…"}}`.
4. GitHub Actions wakes up, runs `desktop-crossbuild.yml`, which:
   - Clones Forgejo at the given tag (public, unauthenticated).
   - Builds the three Tauri bundles.
   - Uploads each one to the matching Forgejo Release with an authenticated `POST` to `/api/v1/.../releases/<id>/assets`.

Nothing in this GitHub repo is authoritative. The workflow yaml itself is maintained on the Forgejo side and copied here by a Forgejo CI job (`sync-github-workflows.yml`) whenever it changes.

## What happens if somebody opens a PR here

The maintainer will close it and point them at Forgejo. Issues are the same. This repo is not a fork, not a mirror, not a community surface — it's a compute broker.

## Secrets that need to exist

- **`FORGEJO_RELEASE_TOKEN`** — GitHub repo secret. Forgejo access token with `write:repository` scope, used to upload the built bundles to the Forgejo Release. Without it the workflow still builds and publishes artifacts as GitHub workflow artifacts, but nothing lands on the public Release page.

## Maintenance

- If the workflow file drifts from the Forgejo copy, run the `sync-github-workflows.yml` workflow on Forgejo manually. No edits here will stick — the next sync overwrites them.
- If you need to revoke access, rotate the PAT on both sides (Forgejo's `GH_TOKEN` Actions secret and this repo's `FORGEJO_RELEASE_TOKEN` Actions secret).

---

*License and copyright: see the Forgejo repo. This proxy repo publishes no content of its own.*
