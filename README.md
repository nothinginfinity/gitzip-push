# gitzip-push

Push large files (HTML, assets, anything) to any GitHub repo via the **`.gitzip` maildrop pattern** — a simple, reliable alternative to the GitHub Contents API size limit.

---

## Why

The GitHub Contents API (`PUT /repos/{owner}/{repo}/contents/{path}`) base64-encodes the payload in the request body. Large files (>1–2 MB) hit HTTP body size limits in browsers and some API gateways. `gitzip-push` works around this by:

1. **Packing** one or more files into a `.zip` archive
2. **Pushing** the zip to a dedicated maildrop path in the target repo (e.g. `_gitzip/drop-<timestamp>.zip`)
3. **Unpacking** via a GitHub Actions workflow that extracts the zip and commits the real files to their target paths

This means the only thing hitting the Contents API is a zip blob — and the heavy lifting happens server-side in Actions.

---

## Envelope format

Every zip pushed by `gitzip-pack.html` includes a manifest file at the root: **`gitzip-manifest.json`**.

```json
{
  "version": 1,
  "created": "2026-05-06T04:00:00.000Z",
  "target_branch": "main",
  "commit_message": "feat: update app",
  "files": [
    { "src": "repo-copilot.html", "dest": "repo-copilot.html" }
  ]
}
```

| Field | Description |
|---|---|
| `version` | Protocol version (currently `1`) |
| `created` | ISO timestamp of pack time |
| `target_branch` | Branch to commit unpacked files onto |
| `commit_message` | Commit message used by the unpack Action |
| `files` | Array of `{ src, dest }` — `src` is the path inside the zip, `dest` is the repo path to write |

---

## Maildrop path convention

Zips are pushed to:
```
_gitzip/drop-<unix-ms>.zip
```

The unpack Action watches `_gitzip/**.zip` and auto-triggers on push.

---

## Tools in this repo

| File | Purpose |
|---|---|
| [`gitzip-pack.html`](./gitzip-pack.html) | Single-file browser UI — drag files in, set target, push zip |
| [`gitzip-unpack.yml`](./gitzip-unpack.yml) | Reference GitHub Action — install in any repo to enable unpack |

---

## Quick start

### 1. Install the unpack Action in your target repo

Copy [`gitzip-unpack.yml`](./gitzip-unpack.yml) to `.github/workflows/gitzip-unpack.yml` in your target repo.

Add a repo secret named `GITZIP_TOKEN` — a PAT with `contents: write` on that repo.

### 2. Open `gitzip-pack.html` in your browser

No install, no build step. Open the file locally or serve it from any static host.

### 3. Drag files in, fill in target repo + PAT, push

The packer will:
- Build a zip containing your files + `gitzip-manifest.json`
- Push it to `_gitzip/drop-<timestamp>.zip` in your target repo
- The unpack Action fires automatically, extracts files to their `dest` paths, and commits

---

## Size limits

- GitHub Contents API: **~100 MB** per file (GitHub's hard limit)
- Practical browser limit: ~**50 MB** zip (base64 encoding overhead)
- For files >50 MB: use `git push` directly

---

## Protocol versioning

The `version` field in `gitzip-manifest.json` allows the unpack Action to handle future breaking changes. The current version is `1`.
