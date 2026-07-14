# Contributing to Perpl Docs

Docs publish to https://docs.perpl.xyz from the `main` branch via GitBook Git Sync. Merged PRs go live automatically (usually within a minute or two).

## Workflow

1. Branch from `main` (or fork): `git checkout -b docs/<short-topic>`.
2. Edit Markdown under `docs/`. One page = one `.md` file; the page tree is [`docs/SUMMARY.md`](docs/SUMMARY.md) — **add new pages there** or GitBook won't show them.
3. (Optional) Preview locally with any Markdown viewer, or use GitBook's change-request preview after opening the PR.
4. Open a PR. The **Link Check** CI must pass — it fails on broken internal or external links.
5. On merge to `main`, GitBook syncs and republishes.

## Writing rules — avoid the broken-link trap

These are the exact issues this repo was created to eliminate:

- **Internal links must be relative Markdown paths** to the target file, e.g. `[Networks & Configuration](getting-started/networks.md)`.
  - ❌ Never commit `broken://pages/<id>` links (GitBook's unresolved-page marker).
  - ❌ Never commit `file:///…` links (leaked local paths from the migration).
  - ❌ Avoid absolute `https://docs.perpl.xyz/…` links for pages that live in this repo — use the relative path so the link check can verify the target exists.
- **Every new/renamed page must be in `docs/SUMMARY.md`.**
- GitBook-flavored blocks are supported and should stay intact: `{% hint %}`, `{% tabs %}`, code fences, tables.

## Check links before you push

```bash
# internal-link check (fast, offline):
npx lychee --offline docs
# full check including external URLs:
npx lychee docs
```

## Structure

- `docs/` — published content (content root per [`.gitbook.yaml`](.gitbook.yaml)).
- `docs/SUMMARY.md` — table of contents / navigation.
- `.github/workflows/link-check.yml` — CI link checker.
- `.github/pull_request_template.md` — PR checklist.
