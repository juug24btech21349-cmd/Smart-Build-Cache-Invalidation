# Smart Build-Cache Invalidation — Assignment Learnings

Repository: https://github.com/juug24btech21349-cmd/Smart-Build-Cache-Invalidation

This document records the setup and learning steps for Labs 0–2. These labs establish a small Python project and a baseline CI pipeline; they do not yet implement cache invalidation.

## Lab 0: Setup and project folder

### 1. Install the tools

Install Visual Studio Code, Python 3.12, and Git. In VS Code, install the Python and GitHub Copilot extensions and sign in to Copilot from the account control.

### 2. Create and open the project folder

Create a folder named `smartci` and open it in VS Code. This repository folder is the project root. Opening the root folder lets VS Code, Git, Python, and pytest use the same project context.

### 3. Create and activate a virtual environment

Open the VS Code terminal with <kbd>Ctrl</kbd>+<kbd>`</kbd>, then run:

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

On macOS, Linux, or WSL, activate it with `source .venv/bin/activate`. A virtual environment isolates this project's Python packages from other projects. If PowerShell blocks activation, use the terminal profile approved for your machine or invoke `.venv\Scripts\python.exe` directly.

### 4. Select the project interpreter

Press <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>P</kbd>, choose **Python: Select Interpreter**, and select the interpreter inside `.venv`. This makes VS Code run and diagnose code with the same environment used by the terminal.

### 5. Create the project structure

The repository includes these folders:

```text
src/                  Application code
tests/                pytest tests
scripts/              Helper scripts (reserved for later labs)
ml/                   Machine-learning work (reserved for later labs)
infra/                Infrastructure files (reserved for later labs)
ansible/              Ansible files (reserved for later labs)
.github/workflows/    GitHub Actions workflows
```

### 6. Initialize Git and ignore generated files

The repository uses Git for version history. Its `.gitignore` excludes `.venv/`, `__pycache__/`, `.pytest_cache/`, and `build.log`; these are local environment or test-output files rather than source files.

For a new local project, initialize and commit it with:

```powershell
git init
git add .
git commit -m "Lab 0: project skeleton"
git branch -M main
git remote add origin https://github.com/YOUR-USER/smartci.git
git push -u origin main
```

For this assignment, use the repository link at the top of this document as `origin` if you have permission to push to it.

### 7. Checkpoint and Copilot basics

Confirm that the repository appears on GitHub and that the terminal prompt shows the active virtual environment. Press <kbd>Tab</kbd> to accept a Copilot inline suggestion. Open Copilot Chat with <kbd>Ctrl</kbd>+<kbd>Alt</kbd>+<kbd>I</kbd>. To investigate an error, select it and ask Copilot to explain it and suggest a fix; review suggestions rather than accepting them blindly.

## Lab 1: The app and its tests

### 1. Implement pricing functions

`src/pricing.py` provides `apply_discount` and `add_tax`. The discount function accepts percentages from 0 through 100 inclusive, raises `ValueError` outside that range, and rounds the discounted price to two decimal places. Tax defaults to 18 percent and is also rounded to two decimal places.

### 2. Implement the cart

`src/cart.py` stores `(name, price, quantity)` tuples. `Cart.total()` sums each item's extended price, applies the requested discount to the subtotal, then adds tax. It imports the pricing functions from `src/pricing.py`; this dependency is important when deciding whether a build cache is still valid after a source file changes.

### 3. Implement inventory operations

`src/inventory.py` stores stock quantities by item name. Adding stock increments the existing quantity. Removing more than the available quantity raises `ValueError` instead of allowing stock to become negative.

### 4. Write and review tests

The test files cover normal behavior, boundary conditions, and invalid input:

- `tests/test_pricing.py` checks a basic discount, 0% and 100% boundaries, an invalid percentage, and default tax.
- `tests/test_cart.py` checks a discounted cart total and the empty-cart total.
- `tests/test_inventory.py` checks stock addition, successful removal, and an error when removing too much.

Tests turn expected behavior into repeatable checks. Each expected value was checked against the functions' arithmetic; `pytest.raises` verifies that invalid operations fail with the intended exception.

### 5. Install dependencies and run the test suite

```powershell
python -m pip install -r requirements.txt
pytest -v
```

`requirements.txt` lists pytest for testing and requests for the project environment. `pytest.ini` adds the project root to Python's import path so tests can import `src.*`.

### 6. Checkpoint

All tests should pass before moving on. Save the work in Git and add the `lab1` tag:

```powershell
git add .
git commit -m "Lab 1: app and tests"
git tag lab1
git push --follow-tags
```

## Lab 2: Baseline CI pipeline

### 1. Understand the workflow

`.github/workflows/ci.yml` defines a workflow named **CI** that runs on pushes and pull requests. Its `build` job runs on a fresh `ubuntu-latest` runner. The steps check out the repository with full Git history, install Python 3.12, install the requirements, then run pytest.

Each workflow run starts on a clean runner, so it cannot rely on packages installed on a developer's computer. This is the baseline that later cache work can optimize. `fetch-depth: 0` fetches full Git history for future change comparisons; the baseline test job does not itself implement cache invalidation.

The test command pipes output through `tee` to both display it and write `build.log`. `set -o pipefail` ensures that pytest's nonzero exit code still fails the step even though its output is piped.

### 2. Push and inspect a passing run

Push the workflow and open the repository's **Actions** tab. Select the latest **CI** run, then open the `build` job. Confirm that **Run tests** is green and note the total run time and the duration of dependency installation.

### 3. Perform the intentional red-run experiment

Temporarily change the expected result in a test—for example, change `assert apply_discount(100, 10) == 90.0` to `== 91.0`—then commit and push that temporary change. The workflow should fail because actual program behavior no longer matches the intentionally incorrect expectation. After confirming the failure and recording evidence, restore the correct expected value, commit, and push again. The final source tree should contain the passing assertion.

This experiment was performed in this repository. The temporary incorrect assertion caused [CI run #2 to fail](https://github.com/juug24btech21349-cmd/Smart-Build-Cache-Invalidation/actions/runs/44798849580); the correct assertion has since been restored.

### 4. What a red run is for and where the error appears

A red run is a controlled demonstration that CI detects a regression or a failed assertion. It gives the team evidence that the automated test gate is active; it is not a successful build and should not be mistaken for one. The example above makes the test expectation wrong on purpose—the production function is unchanged.

To find the failure on GitHub:

1. Open the repository and select **Actions** in the top navigation.
2. Select the failed **CI** run (red failure icon) from the run list.
3. Open the **build** job.
4. Expand the **Run tests** step. The pytest output shows the failing test name, the assertion's expected and actual values, and the source file/line. The step summary and job status also show failure.

The failure is therefore identified in **Actions → failed CI run → build → Run tests**. If the test process cannot start, inspect the earlier install/setup step that is marked as failed instead.

### 5. Screenshots to attach

For the submission, include screenshots from the actual Actions pages:

- **Actions run list:** show both the red intentional-failure run and the passing run after the test was restored.
- **Failed test log:** show the failed run's `build` job with **Run tests** expanded and the pytest assertion details visible.

The red run is [CI run #2](https://github.com/juug24btech21349-cmd/Smart-Build-Cache-Invalidation/actions/runs/44798849580). Its **Run tests** step reported `1 failed, 8 passed`; `tests/test_pricing.py:7` failed because the intentionally incorrect expected value was `91.0` while the function returned `90.0`. This is the pytest error to point out in the failed-test screenshot. The Actions page can be viewed at the run link; expand the `build` job and then **Run tests** to see the failing step. Do not use a mock image or a screenshot from another repository.

### 6. Checkpoint

The Actions history should contain at least one passing run and one intentionally failing run. Keep the final code corrected and ensure the latest run is green. Add the `lab2` tag:

```powershell
git add .
git commit -m "Lab 2: baseline CI"
git tag lab2
git push --follow-tags
```

## Key learning summary

- A virtual environment keeps Python dependencies project-specific.
- Unit tests encode expected behavior and catch changes that violate it.
- Imports express dependencies between modules; for example, the cart depends on pricing.
- CI repeats installation and tests on a clean runner for every push or pull request.
- A red run is useful evidence that CI catches a deliberate failure; diagnose it in the failing job step, then restore the code and verify a green run.
- A baseline CI workflow has no build cache yet. Cache invalidation must account for the inputs and dependencies that affect a build; that topic requires later assignment steps beyond Labs 0–2.
