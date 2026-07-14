# Perpl Docs

Source of truth for the Perpl documentation published at **https://docs.perpl.xyz**, synced to GitBook via **Git Sync**. Edit the Markdown here, open a PR, and merged changes publish automatically.

- **Content** lives in [`docs/`](docs/) (GitBook-flavored Markdown). Navigation is [`docs/SUMMARY.md`](docs/SUMMARY.md).
- **Contributing:** see [CONTRIBUTING.md](CONTRIBUTING.md).
- **One-time setup / GitBook connection:** see [SETUP.md](SETUP.md).

## Why this repo exists

The docs were previously edited only in the GitBook UI. A content migration left the **For Developers** section riddled with broken cross-links — unresolved page references (`broken://pages/…`) and leaked local paths (`file:///…`) — with no review gate to catch them.

Putting the docs under version control means:

- **Every change is reviewed via PR** before it publishes.
- An **automated link check** ([`.github/workflows/link-check.yml`](.github/workflows/link-check.yml)) fails the build on broken internal or external links, so the migration problem can't recur.
- **Anyone can contribute** a fix or an addition with a pull request.
