# Submitting a Mod

Short version. Full step-by-step guide: [PUBLISHING.md](PUBLISHING.md).

## Requirements

Your mod must:

1. Live in a **public** GitHub repo you own.
2. Have a `manifest.json` at the root with at least these fields:
   - `id` (reverse-DNS, e.g. `com.yourname.modname`) — permanent identity
   - `name` (human-readable)
   - `version` (semver, must match release tag)
   - `apiVersion` (the Mod API version your mod targets, e.g. `"1.0.0"`)
   - `main` (relative path to JS entry file, e.g. `"index.js"`)
3. Tag each release `vX.Y.Z` matching `manifest.json#version` exactly (without the leading `v`).
4. Attach a zip asset named `<modId>-<version>.zip` to each GitHub Release, with `manifest.json` at the zip root (or wrapped in one top-level folder — installer auto-strips it).
5. Be loadable cleanly in the latest version of Maker Studio (test by dropping the unzipped folder into `%APPDATA%/maker-studio/Mods/` and restarting).

## Recommended

- Sign every release with [minisign](https://jedisct1.github.io/minisign/). Adds a green **Verified** chip in the Marketplace and skips the unverified-warning confirm step. See [KEYS.md](KEYS.md) for the one-shot setup script.
- Start from one of the [`examples/mods/`](examples/mods/) folders — fastest way to learn the API and avoid common mistakes.
- Provide a clear release-notes body — it becomes the changelog shown to users.
- Provide an icon (64×64 PNG) hosted in your repo or as a release asset.
- Declare every capability you use in the manifest's `permissions` array. Users see the list before installing — undeclared usage is a red flag.

## How to submit

First-timer? Follow the hands-on walkthrough in **[TUTORIAL.md](TUTORIAL.md)** — covers scaffolding the mod, signing, releasing, and submitting from scratch.

Otherwise the short version:

1. **Easiest (no local git):** open [`index.json`](index.json) on github.com → click the pencil icon → **Edit this file**. GitHub auto-forks the repo under your account. Paste your entry into the `mods` array. Click **Propose changes** → fills the commit message + opens the cross-repo PR. Done.
2. **Local fork (multiple mods or bigger edits):** `gh repo fork Toskan4134/maker-studio-mods --clone` → branch → edit `index.json` → push → `gh pr create`.
3. CI validates schema + checks your release zip + `.minisig` (if `pubkey` declared) actually exist on the listed repo.
4. Maintainer reviews. Once merged, your mod is live for users within 1 hour.

## Updating

Updates do **not** need a new PR. Just publish a new GitHub Release on your own repo — users see the new version on their next update check.

## Removal

Open a PR removing your entry from `index.json`.

## `index.json` entry template

```json
{
  "id": "com.yourname.modname",
  "name": "My Mod",
  "author": "Your Name",
  "repo": "your-handle/your-repo",
  "description": "One-sentence pitch.",
  "tags": ["tools"],
  "icon": "https://raw.githubusercontent.com/your-handle/your-repo/main/icon.png",
  "homepage": "https://your-site",
  "minStudioVersion": "2.0.0",
  "apiVersion": "1.x",
  "pubkey": "RWQ...your-minisign-pubkey..."
}
```

Drop `icon`, `homepage`, `minStudioVersion`, `apiVersion`, `pubkey`, `tags` if not applicable. `id`, `name`, `author`, `repo` are mandatory.

## Rejection reasons

PRs may be rejected for:

- Mod doesn't load on latest Maker Studio.
- Obfuscated or minified JS without source.
- Hidden network calls to unknown domains.
- Permissions in manifest don't match what the code actually uses.
- Misleading name/description.
- License conflicts (your mod repo must be openly available; the mod itself can be any OSI license).
