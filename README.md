# Smart Build-Cache Invalidation

This repository contains the setup, sample Python application, tests, and baseline GitHub Actions workflow for Labs 0–2 of the assignment.

## Run locally

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install -r requirements.txt
pytest -v
```

The step-by-step setup, test explanations, CI behavior, and red-run experiment are documented in [LEARNINGS.md](LEARNINGS.md).
