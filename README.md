# ha-panel-ci

The single home of the panel-bundle workflow for Home Assistant custom integrations that
serve a custom panel (a Lit/TypeScript web component bundled with esbuild and committed
under `custom_components/<domain>/panel/`). The sections below say what is here, how a
consumer calls it, and how a release of this repository reaches it.

## What is here

| Path | What it is |
|---|---|
| `.github/workflows/panel-bundle.yml` | The workflow body. `on: workflow_call` only, no inputs, no secrets. Type-checks, unit-tests and builds the panel to prove it still compiles (the build artefact is discarded); every step after checkout is guarded, for the reason under Calling the workflow. The panel's TypeScript is reachable from nothing else in the stack — `tsc --noEmit` catches what the Python suite cannot see. What ships is decided by `release.yml` in `PineappleEmperor/ha-integration-ci`. |
| `frontend/package.json`, `frontend/tsconfig.json` | The templates a consumer copies into its own `frontend/`. |
| `.github/dependabot.yml` | Moves the action pins inside the body and the version ranges in the frontend templates; without this bump the templates, copied once, would rot in place while a consumer's own Dependabot moves ahead of them. |

## Calling the workflow

A consumer carries this caller workflow as its `.github/workflows/panel-bundle.yml`:

```yaml
# .github/workflows/panel-bundle.yml
name: Panel Bundle

on:
  push:
    branches: [main]
    paths:
      - 'frontend/**'
      - 'custom_components/*/panel/**'
      - '.github/workflows/panel-bundle.yml'
  pull_request:
    paths:
      - 'frontend/**'
      - 'custom_components/*/panel/**'
      - '.github/workflows/panel-bundle.yml'

permissions:
  contents: read

jobs:
  panel:
    uses: PineappleEmperor/ha-panel-ci/.github/workflows/panel-bundle.yml@{{sha}} # {{tag}}
```

One optional input. A consumer that would rather a stale committed bundle turned the run
red than left a warning adds it under the `uses:` line:

```yaml
    with:
      fail-on-stale-bundle: true
```

`{{tag}}` and `{{sha}}` resolve as release-flow's README says under Calling the
workflows, against this repository:

```
TAG=$(gh api repos/PineappleEmperor/ha-panel-ci/releases/latest --jq .tag_name)
SHA=$(gh api "repos/PineappleEmperor/ha-panel-ci/commits/$TAG" --jq .sha)
```

How Dependabot moves the pin afterwards is the version model in
[PineappleEmperor/ha-integration-ci](https://github.com/PineappleEmperor/ha-integration-ci)'s
README, which this repository follows exactly.

The resulting check is named `panel / Panel type-check and tests` — GitHub's naming rule
for a job that calls a reusable workflow is in that README too. **It must never be a
required status check.** The caller is path-filtered as above, so it does not report at
all on a Python-only PR, and a required context that never reports leaves that PR
unmergeable forever. Leave it advisory.

The path filter names the caller file itself, so the first run a repo ever sees is the
one that lands the caller. That is why the body guards every step on the panel's
manifest rather than assuming one exists — the first such run once died at
`setup-node`'s cache step, on a lock file that did not exist yet. Each step below is
gated individually because a job-level `if` cannot see files, but a step-level one can.

## The frontend templates

`package.json` carries two placeholders the consumer substitutes; nothing else in either
template is edited:

| Placeholder | Meaning | Where it appears |
|---|---|---|
| `<domain>` | The integration's domain, as in `custom_components/<domain>/` | `"name"` and the `--outfile` of the build script |
| `<name>` | The panel module's basename, `src/<name>.ts` in, `panel/<name>.js` out | The entry and the `--outfile` of the build script |

The `scripts` in `package.json` (`check`, `test`, `build`) are the contract between the
consumer, this workflow and the release zip.

`esbuild` is pinned to the major that `vitest` itself depends on. When the two ranges
disagree npm resolves two copies, and the nested one loses the `optional` flag on
esbuild's platform packages, so an install of that lock file tries to fetch
`@esbuild/aix-ppc64` and fails `EBADPLATFORM`. Keep the two in step when either moves.

It hides from the obvious check. `npm install` writes the faulty lock file and succeeds;
only `npm ci` reads it back and fails, which is why this reached a runner having passed
three of the four commands locally.

## Proving a panel locally

Run what the workflow runs, from a clean tree, in this order — the install is the step
that catches a lock file no one can see is wrong:

```
rm -rf node_modules && npm ci
npm run check
npm test
npm run build
```

Then check `git status`: a `build` that changes the committed bundle means the bundle in
the last commit was stale.

Vitest needs no config file; its default include pattern already picks up
`frontend/test/*.test.ts`. The workflow looks for test files with `find`, not a glob,
because bash `**` does not recurse without globstar and `ls` errors on a non-matching
pattern rather than reporting none. It warns when no test file exists, because the
panel's presentation logic is then unproven, and it warns when the committed bundle is
stale against a fresh build, because leaving that silent until release meant finding out
too late. Neither warning blocks a merge: gating on the bundle's freshness once blocked
merges over a build artefact. `fail-on-stale-bundle` turns the second one into a failure
for a consumer that wants it — the job goes red, and since this check is never a required
context the merge is still the author's call.

## The release zip must agree with the build step

`release.yml` in `PineappleEmperor/ha-integration-ci` builds the panel too, and says why
there. Its build step and this workflow's must run the same `package.json` scripts, in
`frontend/`, with the same `--outfile`. If either side changes the command, the
working directory or the output path, the other must change with it, or the PR check
proves a build the release never performs.

## Version model

The version model is ha-integration-ci's, linked under Calling the workflow.

## This repo's own PR gate

As "This repository's own PR gate" in ha-integration-ci's README: this repository is a
consumer of release-flow like any other, with no CI of its own beyond that.
