# stackr-manifest

Version catalog for [Stackr](https://github.com/igrdkl/stackr) — the list of
installable components (PHP, web servers, databases, caches), their download
URLs, and SHA-256 checksums. The desktop app fetches `manifest.json` at runtime,
so **new versions ship without a Stackr release**, and a bad build can be pulled
with a one-line edit (kill switch).

## Why this exists

- **Update without a release.** New PHP out? Add a line here; every client sees
  it within minutes. No installer rebuild.
- **Kill switch.** A build turns out broken → remove it from the manifest.
- **Integrity.** Each entry carries a SHA-256; the client verifies the download
  before extracting and refuses a mismatch.
- **No more scraping.** windows.php.net moves old builds from `releases/` to
  `releases/archives/`; MySQL shuffles GA vs. archive. The manifest stores the
  already-correct full URL, so the client stops guessing.

## Format (`manifest.json`)

```jsonc
{
  "schema": 1,                       // bump only on breaking format changes
  "updated": "2026-07-15",
  "components": {
    "nginx": [
      { "version": "1.27.3",
        "url": "https://nginx.org/download/nginx-1.27.3.zip",
        "sha256": null }             // null = not yet computed; client skips verify
    ]
  }
}
```

- Versions are **newest first**; `[0]` is the default the UI preselects.
- `sha256: null` means the checksum isn't filled yet — the client installs
  without verifying that entry. Fill it (see below) to enable verification.

## Adding / updating a version

1. Add the entry (newest first) with `"sha256": null`.
2. Push — the **lint** job checks the JSON parses and every URL is reachable.
3. Run the **checksums** workflow (Actions → verify-manifest → *Run workflow*).
   It downloads each archive and logs `SUGGEST <component> <version> -> <digest>`
   for null entries.
4. Paste the digests into the manifest, commit. From then on the nightly run
   fails loudly if any declared checksum stops matching or a URL dies.

## How the client consumes it (planned wiring)

1. Fetch `https://raw.githubusercontent.com/igrdkl/stackr-manifest/main/manifest.json`.
2. Cache it to `C:\Stackr\config\manifest.json` with a timestamp.
3. Offline → use the cache. No cache → fall back to the current live scraping.
4. Install: pass the entry's `sha256` to `download_and_extract_checked`, which
   verifies before extracting.
