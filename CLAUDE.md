# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

This is a **TUF (The Update Framework) update server repository** for the `DocuWareExporter` application, managed via the [`tufup`](https://github.com/dennisvang/tufup) Python library. It does **not** contain the application source code — it is the static update channel that DocuWareExporter clients poll for new releases.

The repository is served over HTTP/HTTPS as a static file server. Clients use `tufup` to securely download and verify updates against the signed metadata here.

## Repository structure

```
metadata/          # TUF signed metadata (JSON)
  root.json        # Root of trust — defines signing keys for all roles
  1.root.json      # Archived root v1 (required for key rotation support)
  targets.json     # Lists available packages with SHA-256 hashes and lengths
  snapshot.json    # References the current version of targets.json
  timestamp.json   # References the current snapshot; shortest expiry — refreshed most often
targets/           # Actual application packages
  DocuWareExporter-1.0.0.tar.gz
```

## TUF role hierarchy

```
root  →  signs keys for: targets, snapshot, timestamp
targets  →  lists packages (name, hash, length, tufup metadata)
snapshot  →  pins the version of targets.json
timestamp  →  pins the version of snapshot.json (shortest-lived, refreshed frequently)
```

Each metadata file is signed by an `ed25519` key. The private keys live **outside** this repository and must be available locally to sign updates.

## Expiry schedule (as of v1.0.0)

| File            | Expires            |
|-----------------|--------------------|
| root.json       | 2027-05-21         |
| targets.json    | 2026-05-28         |
| snapshot.json   | 2026-05-28         |
| timestamp.json  | 2026-05-22         |

`timestamp.json` has the shortest TTL — it must be renewed before its expiry or clients will refuse updates. When renewing, snapshot.json and targets.json also typically need refreshing.

## Adding a new release (tufup workflow)

New releases require the `tufup` dev tools and access to the signing private keys. The typical workflow (run from the **DocuWareExporter source repo**, not here):

1. Build the new application bundle/tarball.
2. Use `tufup` to add the new target — this updates `targets.json`, `snapshot.json`, and `timestamp.json` with new signatures, and places the new `.tar.gz` in `targets/`.
3. Commit and push this update repo so the static file server picks up the new files.

The `consistent_snapshot` flag is `false` in `root.json`, meaning metadata files are not version-prefixed on disk (no `2.snapshot.json` etc.) — only `targets/` filenames are versioned.

## Key management

Four ed25519 keys are defined in `root.json` — one per role (root, snapshot, targets, timestamp). The public keys are embedded in `root.json`; the corresponding private keys must be stored securely and **never committed to this repo**.
