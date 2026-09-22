# flutter-lean-ci

Cut GitHub Actions minutes on a Flutter repo without shipping less-tested
code: a **pre-push hook gates the PR**, and **CI runs once per merge** instead
of on every push.

Built for apps using [Very Good Workflows](https://github.com/VeryGoodOpenSource/very_good_workflows)
(`flutter_package.yml`) and local packages under `packages/`. Extracted from
the [Vibe Fit](https://appvibefit.com) app, where it is in daily use.

## Why

On a private repo, every Actions minute comes out of a shared quota. Our app's
CI ran on every push to every PR. Over 30 days:

| | |
|---|---|
| PRs opened | 144 |
| CI runs on those PRs | 311 (≈ 2.2 per PR) |
| Minutes, app suite | 2,058 |
| Minutes, local packages suite | 1,223 |
| PRs that touched `packages/` | 57 of 144 |

Most of those runs re-checked what the author had already run locally, and the
packages suite ran on PRs that could not have broken it.

## How it works

```
 your machine                         GitHub Actions
 ────────────                         ──────────────
 git push ──► .githooks/pre-push      PR push ─────────► skipped (0 min)
              deps · format ·         PR + `run-ci` ───► ci + packages*
              analyze · bloc lint ·   Dependabot PR ───► ci + packages*
              tests + coverage ·      merge to main ───► ci + packages*
              packages*               tag v* ──────────► ci + packages (always)
                                      * packages: only if packages/ changed
```

- **The hook is the PR gate.** It runs the same checks as `flutter_package.yml`
  (recursive `pub get`, `dart format`, `flutter analyze`, bloc lint, tests with
  a coverage floor) plus the local packages' tests when `packages/` changed.
  Docs-only pushes skip it.
- **CI runs after merge**, so anything that slipped past a hook shows up the
  same day, not at release time. Release tags always run everything.
- **Dependabot PRs always run CI** — they never pass through anyone's hook.
- **`run-ci` label** forces CI on a PR: risky changes, or a push made with
  `--no-verify`.
- **`packages/` has its own workflow**, filtered by path. Local packages don't
  import the app's `lib/`, so a PR that doesn't touch them can't break them.

## Setup

1. Copy the contents of [`template/`](template) into your repo root
   (`.githooks/pre-push` and `.github/workflows/*`). If you already have a CI
   workflow for PRs, replace it with `ci.yaml`. The files live under
   `template/` so they don't run in this repo, which has no Flutter project.
   From your repo root:
   ```bash
   git clone --depth 1 https://github.com/AppVibeFit/flutter-lean-ci /tmp/flutter-lean-ci
   cp -R /tmp/flutter-lean-ci/template/. .
   ```
2. Adjust the config block at the top of the hook (coverage floor, excludes,
   `packages/` dir, bloc lint) and match `ci.yaml`.
3. Make the hook executable and point git at it — once per clone; worktrees
   share it:
   ```bash
   chmod +x .githooks/pre-push
   git config core.hooksPath .githooks
   ```
4. Create the label:
   ```bash
   gh label create run-ci --description "Run the full CI on this PR"
   ```
5. If a CI check is **required** by branch protection, a skipped PR run
   blocks merging — drop the requirement, or require only on `main` pushes.

The hook needs [`very_good_cli`](https://pub.dev/packages/very_good_cli) and,
for bloc lint, `bloc_tools` in `dev_dependencies`.

## Gotchas we hit

**A skipped run can cancel a real one.** With `concurrency: cancel-in-progress`
grouped only by ref, pushing a commit and adding `run-ci` at the same time
fires two events. If the unlabeled one lands last, it cancels the labeled
run and then skips itself — the PR ends up with no CI at all. The workflows
here add a `run`/`skip` suffix to the group, computed with the same condition
as the job's `if`. **If you change one, change the other.**

**`flutter analyze` fails on a fresh clone or worktree.** At the repo root it
also analyzes `packages/*/test`, which imports each package's dev
dependencies. The Very Good workflow runs `very_good packages get --recursive`
first; the hook does the same. A hook that only runs `flutter pub get` passes
on your main checkout and breaks everywhere else.

**Path filters don't apply to tag pushes.** That's a feature here: a `v*` tag
always runs the packages suite, even though its commit touched nothing under
`packages/`.

**`*.md` in `paths-ignore` only matches the repo root.** `*` doesn't cross
`/`. Good if nested Markdown is read by code (prompt files, content); use
`**/*.md` if it isn't.

## Trade-offs

- A bug the hook misses reaches `main` before CI catches it. The hook runs
  the same suite, so this mostly means flaky tests or `--no-verify` pushes.
- Pushing takes as long as your test suite (≈ 4 min on a ~6,000-test app).
  That's time you'd otherwise spend waiting for CI anyway.
- The hook runs on the author's machine, so toolchain drift between laptops
  and CI can hide. Pinning the Flutter version in both places helps.

## License

MIT — see [LICENSE](LICENSE).
