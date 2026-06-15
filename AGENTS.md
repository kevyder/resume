# AGENTS.md

## What this is
Single-source LaTeX resume. Edit `src/resume.tex`, build to PDF, upload to Cloudflare R2, serve via Worker at `resume.kevyder.dev`. Auto-deploys on push to `main`.

## Commands
- Local build: `./build.sh` (requires Docker; no native LaTeX needed)
- Worker dev:  `cd worker && npm install && npm run dev`  (runs `wrangler dev`)
- Worker deploy: `cd worker && npm run deploy`
- No tests, lint, or typecheck exist. Do not invent them.

## `build.sh` gotcha
`build.sh:12` runs `sed -i "s/__VERSION__/$(git rev-parse --short=8 HEAD)/" src/resume.tex`. The substitution only fires if `resume.tex` contains the literal `__VERSION__` (it currently does not). If you add it to the tex, every build will mutate your working tree. Version string is the 8-char short SHA; must run inside a git repo with commits.

## LaTeX package sync
Any new `\usepackage{...}` in `src/resume.tex` must be appended to the `tlmgr install` line in `Dockerfile:7-16`, or the Docker build fails. Current set: `titlesec`, `marvosym`, `enumitem`, `fancyhdr`, `hyperref`, `xcolor`, `collection-fontsrecommended`, `collection-latexrecommended`.

`Dockerfile:4` overrides the default TeX Live mirror to `mirror.ctan.org` because the default is unreachable from GitHub Actions. Do not remove it.

## Dead template files in `src/`
`src/awesome-cv.cls`, `src/fontawesome.sty`, and `src/fonts/` are leftover from the upstream template. `resume.tex` uses `\documentclass{article}` and does not load them. They need xelatex + fontspec, which the Dockerfile does not install. Do not try to wire them in to "fix" a build.

## R2 / Worker wiring (hardcoded)
- R2 bucket name: `kevyder` (`wrangler.toml:7`, also hardcoded in `upload.sh:6`).
- Served object key: `latest-resume.pdf` (`worker/index.js:15`).
- Archived as: `archive/resume-<8char-sha>.pdf` (`upload.sh:16`).
- Worker route: `resume.kevyder.dev` (`wrangler.toml:10`). Domain must already be on Cloudflare.
- Cache: ETag from R2 + 304 handling in `worker/index.js:24-31`; `Cache-Control: public, max-age=3600, must-revalidate`.

## CI secrets (`.github/workflows/upload-resume-to-r2.yml`)
- `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_API_TOKEN` (consumed by `cloudflare/wrangler-action@v3`).
- `CLOUDFLARE_R2_ACCESS_KEY_ID`, `CLOUDFLARE_R2_SECRET_ACCESS_KEY`, `CLOUDFLARE_R2_ENDPOINT` (consumed by `upload.sh` via `aws s3 cp --endpoint-url`).
- `aws cli` is preinstalled on GH runners; no install step.

## CI step ordering
`build.sh` -> `upload.sh` -> `wrangler deploy` run as three separate steps. An upload failure does not skip the worker deploy. Practical impact is small for a single-author repo, but a broken build can ship a stale `latest-resume.pdf`.

## Gitignored artifacts
`*.pdf` is gitignored — `src/resume.pdf` is build output and is owned by `root` after a Docker build. LaTeX aux/log/out/toc, `node_modules/`, and `.wrangler/` are also ignored. Normal.

## Branch / PR convention
Single branch, push to `main` = deploy. No PRs, no reviews, no branch protection. Do not suggest PRs for routine edits.
