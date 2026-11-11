# SOLUTION — Tutorial 7: Automate Tests with GitHub Actions

## 1. The task

> Create a GitHub Actions workflow that runs the provided tests on pull
> requests. Inject some bugs into the implementations of the source files and
> open a pull request to observe what happens.

The modules and tests already exist (`password_validator.py`, `triangle.py`,
`parking_fee.py` + three `test_*.py` files, 14 tests in total). The only thing
that is missing is the automation.

## 2. The deliverable

`.github/workflows/tests.yml`

```yaml
name: Run Tests

# Run the test suite automatically on every pull request (and on pushes to main),
# so a broken change is caught by CI before it is merged.
on:
  pull_request:
    branches: [main, master]
  push:
    branches: [main, master]
  workflow_dispatch:

# Least privilege: the workflow only needs to read the repository.
permissions:
  contents: read

# If a PR gets several pushes, cancel the outdated runs instead of queueing them.
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

jobs:
  test:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - name: CheckOut Your Code
        uses: actions/checkout@v4

      - name: Set Up Python
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Run teacher tests
        run: python -m unittest discover -v
```

Why each piece is there:

| Piece | Reason |
| --- | --- |
| `on: pull_request` | This is the requirement: the tests must run when a PR is opened or updated, before anything is merged. |
| `push` to `main` | Keeps the default branch green too, so a direct push cannot slip through. |
| `workflow_dispatch` | Lets you re-run the job by hand from the Actions tab. |
| `permissions: contents: read` | The job only reads the code; no write token is exposed to the PR's code. |
| `concurrency` + `cancel-in-progress` | A new push to the PR cancels the obsolete run, saving minutes. |
| `runs-on: ubuntu-latest` | Standard GitHub-hosted Linux runner. |
| `timeout-minutes: 10` | Stops a hung test run from burning the 6-hour default. |
| `actions/checkout@v4` | Without it the runner's workspace is empty. |
| `actions/setup-python@v5` | Pins a known interpreter instead of relying on whatever the image ships. |
| `python -m unittest discover -v` | Discovers `test_*.py` in the repository root, prints one line per test. It exits non-zero on failure, which is what turns the check red. |

The job is named `test` and the steps are named *CheckOut Your Code*,
*Set Up Python* and *Run teacher tests*, matching the expected result
screenshot (`expected_results.png` in the tutorial repository).

## 3. Verification performed

**a. The workflow file is valid.** It was linted with
[`actionlint`](https://github.com/rhysd/actionlint) 1.7.12 — no errors, no
warnings (`exit 0`). This catches YAML mistakes, unknown contexts and invalid
`uses:` versions before pushing.

**b. The tests pass on clean code.** `python -m unittest discover -v` in this
folder:

```
Ran 14 tests in 0.001s
OK
```

**c. The workflow was replayed locally, step by step.** `tools/run_ci_locally.py`
copies the project into an empty workspace (the "checkout"), runs the same
interpreter setup and then the same command, and prints the step list the way
the Actions UI does. On clean code: `[ok] Run teacher tests`, 14 tests, exit 0.

## 4. Bug injection — the pull request

Three single-line, realistic regressions were injected (see
`bugs/inject_bugs.patch`, or run `python tools/inject_bugs.py`):

| File | Change | Bug |
| --- | --- | --- |
| `password_validator.py` | `< 8` → `< 7` | Off-by-one: a 7-character password is accepted. |
| `triangle.py` | `<=` → `<` in the inequality test | A degenerate "triangle" such as (1, 2, 3) is accepted. |
| `parking_fee.py` | dropped `min(..., DAILY_MAX)` | The $80 daily cap disappears; 15 hours costs $140. |

Replaying the workflow on that branch (`python tools/run_ci_locally.py`) gives
the red run a PR would show:

```
[FAILED] Run teacher tests      (python -m unittest discover -v)
Ran 14 tests in 0.003s
FAILED (failures=4)

x test_daily_cap
x test_valid_fee_calculation
x test_invalid_length_returns_false
x test_invalid_triangle_inequalities_return_false
```

The failure messages are exactly the ones a reviewer needs:

```
AssertionError: 90.0 != 80.0 within 2 places (10.0 difference) : Fee for 10 hour(s) should be 80.0.
AssertionError: True is not false : Password 'A1#b2C3' has invalid length and should return False.
AssertionError: True is not false : Triangle (1, 2, 3) should be invalid.
```

Full logs: `results/clean_run.txt` (green) and `results/buggy_run.txt` (red);
machine-readable summary in `results/test_results.json`.

One subtle detail worth noticing: the triangle bug only changes `<=` to `<`,
and the parking bug only drops a `min()`. Both compile, both look plausible in
a diff, and both are caught immediately — which is the whole point of the
tutorial: CI is a safety net for changes that *look* fine.

## 5. What you see on GitHub

1. Push the branch and open the pull request (`feature/small-update` → `main`).
2. The PR shows a yellow dot next to **Run Tests / test**, then a red X.
3. **Details → Run teacher tests** prints the four `FAIL:` blocks and
   `FAILED (failures=4)`; the workflow run page shows the failed step with a red
   X, and the assertion messages appear as annotations.
4. The PR footer says *Some checks were not successful* and, with branch
   protection enabled, **Merge** stays blocked until the job is green.
5. Fix the three lines (`python tools/inject_bugs.py --revert`), push again,
   and the same check turns green — the result shown in
   `docs/expected_results.png`.

## 6. Reproduce it end to end

```bash
# from inside "Tutorial 7"
python -m unittest discover -v                 # 1. baseline: 14 tests OK
python tools/run_ci_locally.py --report        # 2. replay CI: clean PASS, buggy FAIL
python tools/inject_bugs.py                    # 3. inject the bugs for the PR branch
git checkout -b feature/small-update
git add -A && git commit -m "Small Update."
git push -u origin feature/small-update        # 4. open the PR on GitHub
python tools/inject_bugs.py --revert           # 5. undo them afterwards
```

## 7. Checklist

- [x] `.github/workflows/tests.yml` created at the repository root
- [x] Triggers on `pull_request` (and `push` to `main`)
- [x] Checks out the code and sets up Python on the runner
- [x] Runs the provided `unittest` suite; non-zero exit fails the job
- [x] Workflow validated with `actionlint`
- [x] Clean code: 14/14 tests pass
- [x] Bugs injected on a separate branch: 4 tests fail, job goes red
- [x] Local replay logs and JSON report saved under `results/`

## 8. Note about pushing

This folder was prepared locally. No GitHub token or SSH key is present on this
machine, so the branch was not pushed and no pull request was opened here yet.

Two ways to finish the task:

- `python tools/open_pr.py` — forks `eduhk-zywei/int6181-tutorial-7`, pushes
  `feature/small-update` and opens the pull request, given a token in
  `GITHUB_TOKEN` (classic scopes `repo` + `workflow`). `--dry-run` previews it,
  `--clean` opens the PR without the injected bugs.
- The manual commands in section 6.

Note: in this environment plain `git` over HTTPS fails with
`schannel: AcquireCredentialsHandle failed`, which is a TLS-backend problem, not
an auth problem — `git -c http.sslBackend=openssl ...` works, and the script
sets that automatically.
