# Homelab Knowledge Base

Curated git-submodule collection of homelab infrastructure, monitoring, and media stack projects. Used as a knowledge base for AI agents working with the IoTBase homelab.

## Adding Repositories

When the user asks to add repositories, follow **all** of these steps. Do not skip any.

### 1. Look up latest release tags

```bash
gh api repos/ORG/REPO/releases/latest --jq '.tag_name'
```

Pin to the latest release tag. If a repo has no releases, track latest (use a bare string in `ALLOWED_REPOS`).

### 2. Add to `sync.py`

Add entries to the `ALLOWED_REPOS` list in the appropriate section. If no existing section fits, create a new one with a comment header matching the style of existing sections.

- Pinned: `("org/repo-name", "v1.0.0")`
- Latest: `"org/repo-name"`
- Custom local name: `("org/repo-name", "v1.0.0", "local-name")` or `("org/repo-name", None, "local-name")`

### 3. Add git submodules

Run `git submodule add <url> repos/<name>` for each repo. This can be done in parallel.

### 4. Run `uv run sync.py`

This verifies the allowlist matches reality and checks out pinned versions. Confirm all repos show green in the output.

### 5. Update `AGENTS.md`

Three places must be updated:

1. **Directory tree** — Add entries under the matching section in the `repos/` tree diagram.
2. **Repository summaries** — Add a summary block for each repo following the existing format:
   - `ls` the repo to discover key paths
   - Include: Language, Version (or "Tracking: Latest"), Org, one-liner description, Key paths, Relevant for
3. **Common Tasks table** — Add rows for key use cases if applicable.

### 6. Verify

Run `uv run sync.py --dry-run` to confirm the allowlist is in sync. Check `git status` to see all expected changes.

## Updating Repository Versions

```bash
gh api repos/ORG/REPO/releases/latest --jq '.tag_name'
```

Update the version in `sync.py`, run `uv run sync.py`, and update the version in `AGENTS.md` summaries.

## Removing Repositories

Remove from `ALLOWED_REPOS` in `sync.py`, run `uv run sync.py` (handles submodule cleanup), and remove from all three places in `AGENTS.md` (tree, summary, common tasks).
