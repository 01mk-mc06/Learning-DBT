# dbt Bash Commands Reference

---

## Python & Environment Commands

| Command | What it does |
|---|---|
| `python --version` | Checks which Python version is currently active |
| `py -3.11 --version` | Specifically checks Python 3.11 via the Windows py launcher |
| `py -3.11 -m venv dbt_env` | Creates a virtual environment named `dbt_env` using Python 3.11. `-m venv` means "run the venv module" |
| `.\dbt_env\Scripts\activate` | Activates the virtual environment — all pip installs after this go into `dbt_env` only, not your global Python |
| `deactivate` | Exits the virtual environment |

---

## pip Commands

| Command | What it does |
|---|---|
| `pip install dbt-core dbt-snowflake` | Installs both dbt core engine and the Snowflake adapter |
| `pip install --prefer-binary` | Downloads a pre-compiled binary instead of building from source — avoids C++ build tool errors |
| `pip install --force-reinstall` | Uninstalls and reinstalls the package even if already installed |
| `pip install --no-cache-dir` | Ignores cached downloads and fetches fresh — useful when cached files are corrupted |
| `pip install --upgrade` | Updates the package to the latest version |

---

## dbt Commands

| Command | What it does |
|---|---|
| `dbt debug` | Tests your Snowflake connection and validates profiles.yml — your go-to first command every session |
| `dbt init dbt_learning` | Creates a new dbt project folder with all the default structure |
| `dbt seed` | Reads CSV files from your `seeds/` folder and loads them into Snowflake as tables |
| `dbt run` | Compiles and executes all your SQL models — builds tables/views in Snowflake |
| `dbt test` | Runs all tests defined in your schema.yml files |
| `dbt run --select model_name` | Runs only one specific model instead of all models |
| `dbt docs generate` | Scans your project and generates documentation |
| `dbt docs serve` | Opens the documentation in your browser |

---

## PowerShell / System Commands

| Command | What it does |
|---|---|
| `pwd` | Print Working Directory — shows you which folder you're currently in |
| `cd dbt_learning` | Change Directory — navigates into the `dbt_learning` folder |
| `code .` | Opens the current folder in VS Code |
| `where.exe dbt` | Shows the full file path of where the `dbt` executable is installed |
| `Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser` | Allows PowerShell to run local scripts (needed to activate venv) |

---

## Golden Rule

Every time you open a new VS Code terminal, run these two commands before anything else:

```bash
.\dbt_env\Scripts\activate
cd dbt_learning
```

Without this, dbt won't be found or will use the wrong Python.

---

*dbt Commands Reference | King | February 2026*
