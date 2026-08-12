# Droga Satis — Private Composer Package Registry

Self-hosted static index (via [composer/satis](https://github.com/composer/satis)) that serves Droga's private Composer packages, starting with `drogatechnology/postman-generator`. Hosted for free on GitHub Pages, rebuilt automatically on every tagged release.

## How it works

1. `satis.json` lists every private Droga package repo.
2. A GitHub Actions workflow in **this** repo (`.github/workflows/build.yml`) runs Satis, which clones each listed repo, re-packages tagged releases as zip archives, and writes a static `packages.json` index into `public/`.
3. That `public/` folder is pushed to the `gh-pages` branch and served by GitHub Pages.
4. Consuming Laravel projects add **one** repository URL (this Pages site) to their `composer.json` — no per-package VCS entries, no per-developer GitHub tokens.

Because Satis re-packages releases as zip archives, developers pulling packages only need read access to this Pages site (which can be left public, since the archives contain no more than what's tagged/released) — they don't need direct GitHub access to `postman-generator` or any other private source repo.

## One-time setup

### 1. Create this repo

Push this folder to `github.com/drogatechnology/satis` (adjust `satis.json`'s `homepage` field and this README if you name it differently).

### 2. Enable GitHub Pages

Repo Settings → Pages → Source: **Deploy from a branch** → Branch: `gh-pages` / `/ (root)`. The first workflow run creates that branch; Pages activation may need to happen right after the first successful build.

### 3. Add the `SATIS_GITHUB_TOKEN` secret

Repo Settings → Secrets and variables → Actions → New repository secret:
- Name: `SATIS_GITHUB_TOKEN`
- Value: a fine-grained PAT scoped to **read-only Contents** access on every private package repo listed in `satis.json` (just `postman-generator` for now). This is what lets the build step clone private source during the build — it's never exposed to consumers, only used inside the Action.

### 4. (Optional but recommended) Wire up instant rebuilds

By default this repo rebuilds every 6 hours via `schedule`, which is a fine safety net but slow. For instant updates:

1. Create the `SATIS_DISPATCH_TOKEN` — a fine-grained GitHub PAT with **Contents: read/write** + **Actions: read/write** scoped to `drogatechnology/satis` only.
2. Set it as a **repo-level secret on each package repo** (required on GitHub Free, where org-level secrets are only delivered to public repositories):

   ```bash
   gh secret set SATIS_DISPATCH_TOKEN --repo drogatechnology/postman-generator --body <token>
   ```

   > If the org is upgraded to GitHub Team or Enterprise, this can be stored once as an org-level secret (`gh secret set SATIS_DISPATCH_TOKEN --org drogatechnology --visibility all`) and every package repo inherits it automatically.

3. Copy `package-notify-example.yml` into `.github/workflows/notify-satis.yml` in `postman-generator` (and any future private package repo).
4. Now `git push origin v1.1.0` in that package repo triggers an immediate Satis rebuild instead of waiting on the schedule.

## Adding a new package later

Edit `satis.json`:
```json
"repositories": [
    { "type": "vcs", "url": "https://github.com/drogatechnology/postman-generator" },
    { "type": "vcs", "url": "https://github.com/drogatechnology/your-new-package" }
],
"require": {
    "drogatechnology/postman-generator": "*",
    "drogatechnology/your-new-package": "*"
}
```
Commit to `main` — the build workflow runs on every push to `main` as well, so the new package appears in the index right away. Then drop `notify-satis.yml` into that new package's repo too, and set the `SATIS_DISPATCH_TOKEN` repo-level secret on it (see step 4 in the setup section).

## Consumer setup (in any Droga Laravel project)

Add the Satis feed as a repository in `composer.json`:

```json
"repositories": [
    {
        "type": "composer",
        "url": "https://drogatechnology.github.io/satis"
    }
]
```

Then install normally:

```bash
composer require drogatechnology/postman-generator --dev
```

No GitHub token needed on the consumer side — Composer pulls the pre-packaged zip archive straight from the static Pages index.

## Troubleshooting

- **Package doesn't show up**: check the Actions tab on this repo for a failed build — usually an expired/under-scoped `SATIS_GITHUB_TOKEN`.
- **404 on the Pages URL**: confirm Pages is set to serve from `gh-pages` branch, and that at least one workflow run has completed successfully (the branch doesn't exist until then).
- **Consumer gets "could not find package"**: version constraint mismatch — Satis only republishes tags that exist on the source repo (`v1.0.0` etc.), so make sure the package repo has an actual git tag, not just commits on `main`.
