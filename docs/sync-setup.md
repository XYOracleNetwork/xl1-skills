# Skills sync setup: xl1-skills → xyo-skills

This repo auto-publishes its skills to the public mirror [`XYOracleNetwork/xyo-skills`](https://github.com/XYOracleNetwork/xyo-skills) so they're installable via the `skills.sh` CLI (`npx skills add XYOracleNetwork/xyo-skills`). The sync is driven by three workflows:

| Workflow | Trigger | Purpose |
| --- | --- | --- |
| `.github/workflows/sync-skills.yml` | Push to `main` touching `plugins/xl1-skills/skills/**`, or manual dispatch | Validate, mirror, push to `xyo-skills` |
| `.github/workflows/validate-skills.yml` | PR touching the skills tree | Catch malformed frontmatter before merge |
| `.github/workflows/pat-health.yml` | Monthly cron + manual dispatch | Detect expired/revoked sync PAT and open a tracking issue |

The sync uses plain `git` + `rsync` — no third-party action holds the write-credentialed PAT.

## One-time setup

### 1. Create the fine-grained PAT

Use a fine-grained PAT scoped to only the target repo. Classic `repo`-scoped tokens grant access to every private repo the owner can see and are rejected here for blast-radius reasons.

1. Go to https://github.com/settings/personal-access-tokens/new (Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token).
2. **Token name**: `xl1-skills sync to xyo-skills`.
3. **Resource owner**: `XYOracleNetwork`.
4. **Repository access**: "Only select repositories" → pick `xyo-skills` only.
5. **Repository permissions**: set **Contents: Read and write**. Leave everything else as "No access".
6. **Expiration**: 1 year (the maximum). Record the expiry date in a shared calendar — `pat-health.yml` is the safety net, but the calendar reminder is the primary signal.
7. Generate, copy the token (you only see it once).

### 2. Store the PAT as a repo secret

In **this** repo (`xl1-skills`):

1. Settings → Secrets and variables → Actions → New repository secret.
2. **Name**: `PUBLIC_REPO_SYNC_TOKEN` (exactly).
3. **Secret**: paste the PAT from step 1.
4. Save.

### 3. Seed the target repo

The target repo (`XYOracleNetwork/xyo-skills`) already exists. The sync workflow only writes inside `skills/`, so `README.md` and `LICENSE` at the target root need a one-time manual seed.

**`README.md`** — paste this verbatim at the target repo root via the GitHub web UI:

````markdown
# xyo-skills

Auto-synced mirror of agent skills from [xl1-skills](https://github.com/XYOracleNetwork/xl1-skills), packaged for installation via the [skills.sh](https://skills.sh) CLI.

## Install

All skills:

```sh
npx skills add XYOracleNetwork/xyo-skills
```

A single skill:

```sh
npx skills add XYOracleNetwork/xyo-skills --skill xl1-scaffold
```

Available skills: `xl1-scaffold`, `xl1-patterns`, `xl1-knowledge`, `xyo-knowledge`, `xy-toolchain`, `xy-development`.

## Don't open PRs here

This repo is auto-generated from [xl1-skills](https://github.com/XYOracleNetwork/xl1-skills) by a GitHub Action. PRs against `xyo-skills` will be closed — open them against the source repo instead.

## License

LGPL v3 (inherited from the source repo).
````

**`LICENSE`** — copy `LICENSE.txt` from this repo into `xyo-skills` as `LICENSE`. The shell command from inside `xyo-skills` working copy:

```sh
cp ../xl1-skills/LICENSE.txt LICENSE
git add LICENSE && git commit -m "chore: add LGPL v3 license (matches xl1-skills source)"
```

## End-to-end first-run test

Run these in order before relying on the auto-sync. Each step confirms a specific behavior — don't skip.

1. **Confirm secret + target.** From this repo's Actions tab, manually trigger `PAT health check` (`workflow_dispatch`). It should report `PAT OK (HTTP 200)`. If it opens an issue, the PAT/secret is misconfigured — fix before continuing.

2. **Confirm validator locally.** From this repo root:
   ```sh
   node scripts/validate-skills.mjs plugins/xl1-skills/skills
   ```
   Expect: `validated 6 skill(s) in ...`.

3. **Confirm validator catches bad input.** Temporarily create `plugins/xl1-skills/skills/bad/SKILL.md` with frontmatter missing `description`. Re-run the validator. Expect non-zero exit with a `::error file=...,line=N::frontmatter missing required field: description` annotation. Delete the bad skill.

4. **Dry-run the sync.** From this repo's Actions tab → `Sync skills to xyo-skills` → "Run workflow", check `dry_run: true`, run against branch `main`. Expect logs ending in `dry_run=true — would push the staged diff above to XYOracleNetwork/xyo-skills, but skipping.` Confirm the diff stat shows all six skills being added (the target's `skills/` is currently empty).

5. **First real sync.** Re-run the same workflow with `dry_run: false`. Verify:
   - One new commit appears on `XYOracleNetwork/xyo-skills` main, authored by `github-actions[bot]`, message `chore: sync skills from xl1-skills@<sha>`.
   - The target's `skills/` now contains all six skill directories.
   - `README.md` and `LICENSE` at the target root are unchanged.

6. **Confirm CLI discovery.**
   ```sh
   npx skills add XYOracleNetwork/xyo-skills --skill xl1-scaffold
   ```
   The CLI should resolve the repo, find the skill, and install it.

7. **Confirm idempotency.** Re-run the sync workflow (with `dry_run: false`) without changing anything in the source. Expect logs ending in `No changes to sync.` and no new commit on the target.

8. **Confirm path-filter precision.** Push a commit to `main` that only touches a path outside the filter (e.g. `packages/xl1-scaffold/src/...`). Verify `Sync skills to xyo-skills` does **not** run.

9. **Confirm deletion sync.** Rename a skill locally on a throwaway branch — e.g. `git mv plugins/xl1-skills/skills/xy-toolchain plugins/xl1-skills/skills/xy-toolchain-renamed` (be sure to update the frontmatter `name:` to match the new directory). PR → merge through your normal flow → push to `main`. The next sync should remove `skills/xy-toolchain/` and add `skills/xy-toolchain-renamed/` in a single commit on the target. **Revert the rename** before continuing if this was a throwaway test.

## PAT rotation

Fine-grained PATs cap at 1 year. The flow:

1. `pat-health.yml` runs monthly. When the PAT is within ~30 days of expiry GitHub starts sending email warnings to the token owner; rotate then. If the PAT actually expires or is revoked, `pat-health.yml` opens an issue on its next run.
2. Repeat steps 1 and 2 of "One-time setup" above to mint and store a new PAT.
3. From the Actions tab, re-run `PAT health check`. Confirm green.
4. Close the tracking issue (if one was opened).

## Curating the mirror

By default all six skill directories under `plugins/xl1-skills/skills/` get mirrored. To exclude one (for example, keep `xy-toolchain` internal while publishing the others), add an `--exclude` flag to the `rsync` line in `.github/workflows/sync-skills.yml`:

```yaml
rsync -a --delete --safe-links \
  --exclude='.git' \
  --exclude='.github' \
  --exclude='node_modules' \
  --exclude='xy-toolchain' \
  "${GITHUB_WORKSPACE}/${SOURCE_DIR}/" skills/
```

`--exclude` patterns are relative to the rsync source, so `xy-toolchain` matches the top-level skill directory regardless of nesting elsewhere.

## Upgrade path: GitHub App

When team size grows past one maintainer, replace the PAT with a [GitHub App](https://docs.github.com/en/apps/creating-github-apps) installed on `XYOracleNetwork/xyo-skills` with `Contents: Write`. Mint a per-run installation token with [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token) and pass it as the `PAT` env in the sync workflow. Benefits: tokens are short-lived (1 hour), tied to the App's installation rather than a user account, and survive personnel changes.

For a single-maintainer flow the fine-grained PAT is fine.
