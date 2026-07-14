# One-time setup: repo + GitBook Git Sync

Do these once. After that, everything is PR-driven. Steps 1 and 2 need GitHub + GitBook admin access (they can't be automated from outside).

## 1. Create the GitHub repo and push

From `dev-repos/perpl-docs/` (this directory):

```bash
git init -b main
git add .
git commit -m "chore: scaffold Perpl docs repo (GitBook Git Sync + link-check CI)"

# Create the remote and push (adjust org / visibility as needed):
gh repo create PerplFoundation/perpl-docs --private --source=. --remote=origin --push
```

Or create the repo in the GitHub UI, then:

```bash
git remote add origin git@github.com:PerplFoundation/perpl-docs.git
git push -u origin main
```

## 2. Connect GitBook Git Sync

In GitBook, open the space that backs `docs.perpl.xyz`:

1. **Space → Configure → Git Sync** (or Integrations → GitHub).
2. Install / authorize the **GitBook GitHub app**, then select the repo (`PerplFoundation/perpl-docs`) and branch (`main`).
3. GitBook reads [`.gitbook.yaml`](.gitbook.yaml) — the content root is `docs/`.
4. **Choose the initial sync direction — this matters:**
   - ✅ **GitBook → GitHub (do this first):** exports the *current live content* into `docs/`, giving the repo the complete, authoritative pages. Nothing is lost; you then fix the broken links via PRs.
   - GitHub → GitBook: makes this repo the source and **overwrites** the space. Only use once the repo already holds the full, correct content.
5. After the initial export, **protect `main`** (GitHub → Settings → Branches): require a PR and a passing **Link Check** before merge.

Reference: GitBook Git Sync — https://docs.gitbook.com/getting-started/git-sync

## 3. From then on

Contributors follow [CONTRIBUTING.md](CONTRIBUTING.md): branch → edit `docs/` → open PR → Link Check passes → merge → auto-publish.

## 4. Fixing the existing broken links

Once step 2's export has populated `docs/`, the broken For Developers cross-links (`broken://…`, `file:///…`) will be visible in the Markdown and flagged by the Link Check. Fix them by replacing each with the correct **relative path** to the target page (see CONTRIBUTING.md). These make good first PRs.
