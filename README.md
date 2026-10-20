# Smarter Testing demo: test impact analysis and auto rerun with Jest

This is the companion repo for the CircleCI tutorial **Set up CircleCI Smarter Testing: run fewer tests, retry flaky ones**. It's a TypeScript e-commerce API with two Jest test suites:

| Suite | Test files | Tests | What it tests | Runs where |
|---|---|---|---|---|
| Unit | 25 (`src/**/*.spec.ts`, `tests/workflows/`) | 684 | Services and libraries, with collaborators mocked | Anywhere, in seconds |
| Integration | 18 (`tests/integration/`) | 196 | Repositories and services against a real PostgreSQL database | Locally with Docker; in CI with a Postgres service container |

The `main` branch is the tutorial's **starting point**, where CI runs Jest directly. Over five steps, you switch it to [Smarter Testing](https://circleci.com/docs/guides/test/smarter-testing-overview/):

- **Test impact analysis** runs only the test files that executed the code a change touches, and logs why it picked each one.
- **Auto rerun of failed tests** retries a failing test file in the same job, so a flaky failure doesn't fail the build. The failed attempt stays on record.

The finished files are in [`solution/`](solution/).

> **Cost note.** Running tests through `circleci testsuite`, with static timing-based splitting, is free on every plan. Test impact analysis, auto rerun, and dynamic test splitting are paid features, counted by the test atoms (for Jest, test files) each run discovers, and every plan includes a [free monthly allowance](https://circleci.com/docs/guides/test/smarter-testing-overview/). `circleci testsuite doctor` flags each paid option before it's turned on. Commands with `--local` (including every `npm run demo:*` script) keep their data on your machine and send nothing to CircleCI. Check [pricing](https://circleci.com/pricing/) for your plan.

## Prerequisites

- Node.js 22 or later
- A [CircleCI account](https://circleci.com/signup/)
- [CircleCI CLI](https://cli.circleci.com/) v1 or later (`brew install circleci` on macOS), logged in with `circleci auth login`
- [GitHub CLI](https://cli.github.com/), for the fork command below, or fork in the GitHub UI
- Docker, to run the integration suite locally

## Steps

Each step matches a section of the tutorial, with the same name.

- [ ] **Fork and build.** Fork the repo, install, and follow it in CircleCI. Note the test counts in the first build's **Tests** tab. That build is your baseline.

  ```shell
  gh repo fork CIRCLECI-GWP/smarter-testing-demo --clone
  cd smarter-testing-demo
  npm ci
  circleci project follow
  ```

- [ ] **Step 1: Describe the test suites and run doctor.** Run `circleci extension install testsuite`, create `.circleci/test-suites.yml` with a `unit tests` and an `integration tests` suite, and run `circleci testsuite doctor` on each one. Run `npm run db:up` first, because doctor runs the integration tests against Postgres. Every check should report `success` or `skipped`.
- [ ] **Step 2: Run the tests through circleci testsuite.** In `.circleci/config.yml`, replace each Jest step and its `store_test_results` step with `circleci testsuite run "<suite>"`, and add `parallelism: 3` to the integration job. Push to `main`. The **Tests** tab should match the baseline counts.
- [ ] **Step 3: Turn on test impact analysis.** Install `@circleci/jest-circleci-coverage` and `jest-environment-node@^30`, point `jest.shared.js` at the plugin, and add `analysis`, `impact-key`, `test-impact-analysis`, and `full-test-run-paths` to each suite. Run doctor again, then push to `main`.
- [ ] **Step 4: Check test selection on a branch.** Run `npm run demo:analyze` once, then `npm run demo:leaf`, and compare the output with [`expected/leaf.txt`](expected/leaf.txt). Then run `git push -u origin demo/leaf`. In CI, the unit job runs 2 of 25 files, and an integration test catches the change.
- [ ] **Step 5: Retry flaky tests with auto rerun.** Run `npm run demo:reset`, add `max-auto-rerun: 2` to each suite's options, and push to `main`. (`solution/` sets it only on the unit suite, where the flaky demo test runs.) Then run `npm run demo:flaky` and `git push -u origin demo/flaky`. The job passes, and the **Tests** tab still records the failed attempt.

To check your work, compare it with [`solution/`](solution/):

```shell
diff .circleci/test-suites.yml solution/.circleci/test-suites.yml
diff .circleci/config.yml solution/.circleci/config.yml
diff jest.shared.js solution/jest.shared.js
```

## Scenarios

Each `npm run demo:*` script except `analyze` and `reset` commits one change on a `demo/<name>` branch, then previews which unit test files test impact analysis would select. They need the suite file from Step 3, so run them after that step, and run `npm run demo:analyze` once first to build the local impact map. The tutorial uses `leaf` and `flaky`. The others show more cases.

| Command | What it changes | What gets selected | Expected output |
|---|---|---|---|
| `npm run demo:leaf` | One error message in the user service | 2 of 25 files | [`leaf.txt`](expected/leaf.txt) |
| `npm run demo:shared` | The base error class every service uses | 17 of 25 files | [`shared.txt`](expected/shared.txt) |
| `npm run demo:new-test` | Adds a new test file | The new file only | [`new-test.txt`](expected/new-test.txt) |
| `npm run demo:config` | `jest.config.js`, which is listed in `full-test-run-paths` | All 25 files | [`config.txt`](expected/config.txt) |
| `npm run demo:coverage-gap` | Removes an enum member. Coverage can't see the change, so the type check catches it | 0 files | [`coverage-gap.txt`](expected/coverage-gap.txt) |
| `npm run demo:flaky` | Adds a simulated flaky test that fails on its first attempt and passes on retry (needs Step 5) | The new file, run twice | [`flaky.txt`](expected/flaky.txt) |
| `npm run demo:reset` | Returns to `main` and deletes the `demo/*` branches | | |

Push any scenario branch with `git push -u origin demo/<name>` to see the same selection in CI. Timings in your output will differ from `expected/`.

## Running the tests directly

```shell
npm ci
npm test                        # unit tests

npm run db:up                   # Postgres 18 on localhost:5432
npm run test:integration        # integration tests
npm run db:down
```

## Where things are

| Path | Purpose |
|---|---|
| `.circleci/config.yml` | Pipeline: lint and type check, unit tests, integration tests |
| `jest.shared.js` | Jest settings shared by both suites |
| `jest.config.js`, `jest.integration.config.js` | Per-suite Jest config |
| `solution/` | The finished `.circleci/config.yml`, `.circleci/test-suites.yml`, and `jest.shared.js` |
| `scripts/demo.sh` | The scenario scripts behind `npm run demo:*` |
| `expected/` | Expected scenario output, for comparison |

The `lint-and-typecheck` job runs on every commit because coverage only sees code that runs inside a test. The `demo:coverage-gap` scenario shows a change that selection skips and the type check catches.

## About the timings

The tutorial's results come from runs on the Docker `large` resource class (4 vCPU). The config doesn't set a resource class, so a fork uses your project's default and its timings will differ. To compare like for like, add `resource_class: large` to both executors in `.circleci/config.yml`. Larger classes use more credits per minute.

## Dynamic test splitting

The tutorial and `solution/` don't use [dynamic test splitting](https://circleci.com/docs/guides/test/use-dynamic-test-splitting/). It hands test files to parallel nodes in batches during the run, which pays off on suites with minutes of tests per node, and this demo's suites are too small to benefit. To try it, add `dynamic-test-splitting: true` to the integration suite's options. It's a paid feature.

## Turning it off

Delete `test-impact-analysis` and `max-auto-rerun` from `.circleci/test-suites.yml`. `circleci testsuite run` keeps working with the free features: static timing-based splitting and rerunning failed tests. You can also go back to calling Jest directly at any point.

## License

MIT
