# Tutorial — Your First Marketplace Mod

Hands-on walkthrough from zero to a live, signed, installable mod in the Maker Studio Marketplace. Should take about 30 minutes the first time. Subsequent mods take 10.

We'll pretend you're an author called "Alex" publishing a mod called **Hello Streak** (`com.alex.hello-streak`) that tracks how many days in a row you've opened the editor.

Substitute your own GitHub handle, mod id, and name as you go.

---

## Before you start

You'll need:

- A **public** GitHub account.
- Maker Studio installed locally (to test the mod).
- Git on your machine. PowerShell on Windows, or bash on macOS/Linux.
- (Optional but recommended) minisign — for signing your releases. The registry's `scripts/` folder bundles it for Windows; mac/Linux: `brew install minisign` / `apt install minisign`.

Clone the registry repo locally — you'll need its scripts:

```powershell
gh repo clone Toskan4134/maker-studio-mods
cd maker-studio-mods
```

(No `gh`? `git clone https://github.com/Toskan4134/maker-studio-mods.git` works the same.)

---

## Step 1 — Scaffold your mod folder

The registry ships a scaffold script that fills in `manifest.json` + an `activate(ctx)` skeleton + a starter `README.md`.

```powershell
# Windows
.\scripts\new-mod.ps1

# macOS / Linux
./scripts/new-mod.sh
```

It asks four questions:

```
Mod id (reverse-DNS, e.g. com.yourname.mymod): com.alex.hello-streak
Display name (shown in Mod Manager): Hello Streak
Author (your display name): Alex
Description (one short sentence — optional): Tracks how many days in a row you've opened the editor.
```

You'll get a new `hello-streak/` folder with:

```
hello-streak/
├── manifest.json
├── index.js
└── README.md
```

Move it out of the registry clone — your mod needs its own repo. Drop it on your Desktop or anywhere you keep code:

```powershell
Move-Item .\hello-streak C:\Users\Alex\Desktop\ms-hello-streak
cd C:\Users\Alex\Desktop\ms-hello-streak
```

---

## Step 2 — Write your mod logic

Open `index.js`. The scaffold gives you:

```js
export function activate(ctx) {
  ctx.log.info("Hello Streak activated");
  ctx.ui.showToast({ message: "Hello from Hello Streak!", level: "info" });
  // ...
}
```

Replace it with your actual feature. For Hello Streak it might look like:

```js
const STORAGE_KEY = "streak";

export async function activate(ctx) {
  const today = new Date().toISOString().slice(0, 10);
  const prev = await ctx.storage.get(STORAGE_KEY, { lastDay: null, count: 0 });

  let count = prev.count;
  if (prev.lastDay !== today) {
    const yesterday = new Date(Date.now() - 86_400_000).toISOString().slice(0, 10);
    count = prev.lastDay === yesterday ? prev.count + 1 : 1;
    await ctx.storage.set(STORAGE_KEY, { lastDay: today, count });
  }

  ctx.ui.showToast({
    message: `🔥 ${count}-day streak — keep going!`,
    level: "info",
    durationMs: 4000,
  });
}
```

The full Mod API surface (every method on `ctx`) is in the registry's `docs/` folder — see `docs/api-reference.md` for the narrative reference, `docs/quick-reference.md` for the one-page cheat sheet, and `docs/mod-api.d.ts` for TypeScript types you can drop into your editor for autocomplete.

Open `manifest.json` and declare the permissions you actually use. Hello Streak uses `ctx.storage` (no permission needed — it's mod-local) and `ctx.ui.showToast` (needs `ui.toasts`):

```json
"permissions": ["ui.toasts"]
```

---

## Step 3 — Test locally

Drop your mod folder into the global mods directory and restart the editor.

**Windows:**
```powershell
Copy-Item -Recurse C:\Users\Alex\Desktop\ms-hello-streak "$env:APPDATA\maker-studio\Mods\hello-streak"
```
**macOS:** `~/Library/Application Support/maker-studio/Mods/hello-streak/`
**Linux:** `~/.local/share/maker-studio/Mods/hello-streak/`

Boot Maker Studio → **Mods → Mod Manager**. Your mod appears in the **Installed** tab with the **global** badge. Open any project — the toast should fire.

If the mod fails to load, expand its row in Mod Manager to see error logs. Fix, then click **Reload** on the row (no full restart needed).

Repeat edit → reload until it works.

When done testing, delete the test copy:
```powershell
Remove-Item -Recurse "$env:APPDATA\maker-studio\Mods\hello-streak"
```

---

## Step 4 — Generate a signing key (recommended)

Signed mods get a green **Verified** chip in the Marketplace and skip the "I understand this is unverified code" confirmation step. Users are much more likely to install them.

From your scaffolded mod folder, point the keypair script at the registry clone:

```powershell
# Windows — uses bundled minisign.exe, no install needed
C:\path\to\maker-studio-mods\scripts\generate-keypair.ps1

# macOS / Linux
/path/to/maker-studio-mods/scripts/generate-keypair.sh
```

When prompted for the mod id, enter `com.alex.hello-streak`. The script produces two files in the current directory:

```
com.alex.hello-streak.key   ← SECRET. Never push. Back up offline.
com.alex.hello-streak.pub   ← public. Line 2 (starts with RWQ) goes in your registry entry.
```

**Back up `.key` to two offline locations now** (USB + encrypted cloud). Lose it = lose this identity forever; new key means a registry PR + every installed user gets sig-failure errors until they reinstall.

Copy line 2 of `.pub` — you'll paste it into `index.json` in Step 7.

If you don't want to sign yet, skip this step. Your mod will install with an "Unverified" warning. You can add signing later by republishing.

---

## Step 5 — Create the GitHub repo for your mod

Web: <https://github.com/new>

- Owner: your account
- Repository name: `ms-hello-streak` (or anything you like)
- Visibility: **Public**
- Don't initialize with README / .gitignore / license — you're pushing your own.

Click **Create repository**. Copy the repo URL (e.g. `https://github.com/alex/ms-hello-streak.git`).

---

## Step 6 — Push, tag, build, sign, release

Back in your mod's working directory:

```powershell
cd C:\Users\Alex\Desktop\ms-hello-streak

# 1. Init + push
git init
git add manifest.json index.js README.md
git commit -m "v1.0.0 — initial release"
git branch -M main
git remote add origin https://github.com/alex/ms-hello-streak.git
git push -u origin main

# 2. Tag the release (tag must match manifest version exactly — without 'v')
git tag v1.0.0
git push --tags

# 3. Build the release zip
$zip = "com.alex.hello-streak-1.0.0.zip"
Compress-Archive -Path manifest.json,index.js,README.md -DestinationPath $zip

# 4. Sign (skip if you didn't generate a key)
$key      = ".\com.alex.hello-streak.key"
$minisign = "C:\path\to\maker-studio-mods\scripts\bin\win\minisign.exe"
& $minisign -S -s $key -m $zip
```

You should now have two files in the directory: `<modId>-1.0.0.zip` and `<modId>-1.0.0.zip.minisig`.

### Create the GitHub Release

Web UI: <https://github.com/alex/ms-hello-streak/releases/new>

- **Choose a tag:** `v1.0.0` (already exists from the push above — pick it)
- **Release title:** `v1.0.0`
- **Describe this release:** write the changelog. Becomes the body of the card in the Marketplace:
  ```
  Initial release. Tracks how many days in a row you've opened the editor and
  shows a streak toast on every project load.
  ```
- **Attach binaries:** drag both `<modId>-1.0.0.zip` AND `<modId>-1.0.0.zip.minisig` into the assets area.
- Click **Publish release**.

Or via `gh` CLI:
```powershell
gh release create v1.0.0 $zip "$zip.minisig" `
  --title "v1.0.0" `
  --notes "Initial release. Tracks daily editor-open streak."
```

Verify the release page shows both assets and the green "Latest" badge.

---

## Step 7 — Submit to the registry

This is the only step that touches the maker-studio-mods repo. Two paths.

### Path A — GitHub web UI (zero local git)

1. Go to <https://github.com/Toskan4134/maker-studio-mods/blob/main/index.json>
2. Click the **pencil icon** (top right of the file view) — **Edit this file**.
3. GitHub auto-forks the registry to your account behind the scenes. No `git` commands.
4. Add your mod entry inside the `"mods": [ ... ]` array. If the array is empty, paste:
   ```json
   "mods": [
     {
       "id": "com.alex.hello-streak",
       "name": "Hello Streak",
       "author": "Alex",
       "repo": "alex/ms-hello-streak",
       "description": "Tracks how many days in a row you've opened the editor.",
       "tags": ["stats", "motivation"],
       "homepage": "https://github.com/alex/ms-hello-streak",
       "apiVersion": "1.0.0",
       "pubkey": "RWQ...your-pubkey-line-from-step-4...",
       "permissions": ["ui.toasts"]
     }
   ]
   ```
   If there are already entries, add yours as the last item (don't forget the comma after the previous one).
5. Also update `updatedAt` to the current ISO-8601 timestamp.
6. Scroll down to **Propose changes**. Commit message can be anything — `add: com.alex.hello-streak` works.
7. **Create pull request** → fill in the PR template (it auto-loads). Submit.

### Path B — Local fork (for power users / multiple mods)

```powershell
gh repo fork Toskan4134/maker-studio-mods --clone --remote
cd maker-studio-mods
git checkout -b add-hello-streak
# edit index.json same as above
git add index.json
git commit -m "add: com.alex.hello-streak"
git push origin add-hello-streak
gh pr create --fill
```

---

## Step 8 — Wait for CI, respond to review

When the PR opens, the registry's CI workflow runs three checks:

| Check | What it does |
|-------|--------------|
| **Schema validation** | `index.json` matches `schema/index.schema.json` (required fields, valid id pattern, semver shape, pubkey format) |
| **Duplicate id check** | No two entries share the same `id` |
| **Release asset check** | The repo you listed actually has a `releases/latest` containing a `.zip` whose name includes your `id`, plus a `.minisig` if you set `pubkey` |

All green? The maintainer reviews and merges. If something's red, click the failed check for the exact error message. Common fixes:

| Error | Cause | Fix |
|-------|-------|-----|
| `should match pattern "^[a-zA-Z0-9._-]+$"` for `id` | Used spaces or unusual chars in the id | Edit to only letters/digits/`._-` |
| `repo has no published releases yet` | You forgot to publish the Release in Step 6 | Publish the release, push the PR again (it re-runs CI) |
| `no .zip asset whose name contains '<id>'` | Zip is named differently | Re-upload zip named `<modId>-<version>.zip` |
| `pubkey declared in registry but no .minisig asset` | You added `pubkey` but didn't attach the `.minisig` | Attach the `.minisig` to the release OR remove `pubkey` from the PR |

Push fixes to the same branch — CI re-runs automatically. Once merged, you're live within ~1 hour (editor caches the index for that long).

---

## Step 9 — Verify in the editor

Open Maker Studio → **Mods → Mod Manager → Marketplace** → click **Refresh** (forces an immediate fetch).

- Your card should appear with a green **Verified** chip (if you signed) or yellow **Unverified** (if you didn't).
- Click **Install**. A consent dialog lists your declared permissions. Accept.
- Wait for the "Installed Hello Streak" toast.
- Switch to the **Installed** tab — your mod is loaded and active.

That's it. You shipped a mod.

---

## Releasing updates

Every subsequent release is the simplified loop:

1. Edit code.
2. Bump `manifest.json#version` (`1.0.0` → `1.0.1`).
3. Commit, tag `v1.0.1`, push.
4. Build new zip, sign, attach both to a new GitHub Release.
5. **No registry PR.** Editor sees the new tag on its next update check (within 1 hour). Users get an "Update available" toast.

That's it — the registry only cares about pointing at your repo; release management lives entirely on your side.

---

## Common first-time pitfalls

- **Tag doesn't match manifest version.** Tag must be exactly `v1.0.0` if manifest says `"version": "1.0.0"`. Off by one character → CI rejects.
- **Forgot to flip the repo to public.** Default for new GitHub repos can be Private. Settings → General → Visibility → Change → Public.
- **Zip wraps content in a deep folder.** The installer only auto-strips a single top-level folder. `my-mod/manifest.json` inside the zip = fine. `Documents/stuff/my-mod/manifest.json` = breaks.
- **Permissions in manifest don't match code.** If you `ctx.fs.writeProjectFile(...)` but don't declare `fs.write.project`, reviewers will reject.
- **Committed your `.key` file.** Don't. The registry's `.gitignore` blocks `*.key` by default; do the same in your mod repo's `.gitignore`. If you ever do leak it: generate a new keypair and PR the new `pubkey` to the registry immediately.

---

## Where to ask for help

Open a [discussion on the registry](https://github.com/Toskan4134/maker-studio-mods/discussions) for general questions, or file an [issue](https://github.com/Toskan4134/maker-studio-mods/issues) for a specific bug in the docs, scaffolder, or schema.
