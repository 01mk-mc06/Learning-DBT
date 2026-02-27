# Learning-DBT
# Setup Documentation
# dbt Core + Snowflake — Setup & Installation Documentation

**Prepared by:** King  
**Date:** February 26, 2026  
**Environment:** Windows 11 | dbt-core 1.11.6 | dbt-snowflake 1.11.2

---

## 1. Overview

This document captures the complete installation journey for dbt Core with the Snowflake adapter on a Windows machine, including all errors encountered, troubleshooting steps taken, and final resolutions. Serves as a reference for future setups and onboarding.

---

## 2. Environment & Prerequisites

| Component | Details |
|---|---|
| Operating System | Windows 11 |
| IDE | Visual Studio Code |
| Initial Python Version | Python 3.14 (caused issues) |
| Final Python Version | Python 3.11.9 (working) |
| Terminal | PowerShell (VS Code integrated) |
| dbt Core Version | 1.11.6 |
| dbt Snowflake Version | 1.11.2 |
| Cloud Platform | Snowflake Trial (Azure — Southeast Asia) |

---

## 3. Installation Steps

### Step 1: Install Python & VS Code

VS Code and Python were installed successfully. Initial Python version installed was 3.14.

- VS Code installed with integrated terminal (PowerShell)
- Python 3.14 installed and added to PATH
- dbt Power User extension installed in VS Code

---

### Step 2: Attempt dbt Installation on Python 3.14 —  Failed

Initial attempt to install dbt-core and dbt-snowflake on Python 3.14 resulted in a build failure.

**Command run:**
```bash
pip install dbt-core dbt-snowflake
```

**Error:**
```
ERROR: Failed building wheel for snowflake-connector-python
Failed to build snowflake-connector-python
error: failed-wheel-build-for-install
```

**Root Cause:**  
Python 3.14 is too new — `snowflake-connector-python` does not support it. The package requires C++ build tools to compile from source, and even after installing Microsoft C++ Build Tools, Python 3.14 remained incompatible.

**Attempted fixes (all failed on 3.14):**
```bash
pip install --upgrade pip setuptools wheel
pip install dbt-core dbt-snowflake --no-cache-dir
pip install dbt-core dbt-snowflake --prefer-binary
# Also installed: Microsoft C++ Build Tools (Desktop development with C++)
```

---

### Step 3: Install Python 3.11.9

Python 3.11.9 was downloaded and installed from python.org alongside the existing 3.14 installation. Windows supports multiple Python versions via the `py` launcher.

**Download:** https://www.python.org/downloads/release/python-3119/  
**Installer:** `Windows installer (64-bit)` `.exe`  
**Note:** "Add Python 3.11 to PATH" was checked during installation.

**Verification:**
```bash
py -3.11 --version
# Python 3.11.9
```

---

### Step 4: Create Python 3.11 Virtual Environment

A virtual environment was created using Python 3.11 to isolate dbt dependencies.

**Create the environment:**
```bash
py -3.11 -m venv dbt_env
```

**PowerShell Execution Policy Error:**
```
.\dbt_env\Scripts\activate : File cannot be loaded because running scripts is disabled on this system.
```

**Fix:**
```bash
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
# Type Y and press Enter
```

**Activate:**
```bash
.\dbt_env\Scripts\activate
# (dbt_env) should appear in the terminal prompt
```

---

### Step 5: Install dbt-core + dbt-snowflake —  Success

```bash
pip install dbt-core dbt-snowflake --prefer-binary
```

**Verification:**
```bash
dbt --version

# Core:
#   - installed: 1.11.6
#   - latest:    1.11.6 - Up to date!
# Plugins:
#   - snowflake: 1.11.2 - Up to date!
```

---

### Step 6: mashumaro Compatibility Error —  then  Fixed

**Error:**
```
mashumaro.exceptions.UnserializableField: Field "schema" of type Optional[str]
in JSONObjectSchema is not serializable
```

**Root Cause:** Installed mashumaro version was incompatible with dbt-core 1.11.6.

**Fix:**
```bash
pip install mashumaro==3.13.1 --force-reinstall
```

---

### Step 7: Install Git

`dbt debug` flagged a missing Git installation. Git is required for dbt project version control.

**Download:** https://git-scm.com/download/win

**Installation options selected:**
| Option | Selection |
|---|---|
| Default editor | Visual Studio Code |
| Initial branch name | Let Git decide |
| PATH environment | Git from command line and 3rd-party software |
| SSH executable | Use bundled OpenSSH |
| HTTPS transport | Native Windows Secure Channel library |
| Line endings | Checkout Windows-style, commit Unix-style |
| Terminal emulator | MinTTY |
| Git pull behavior | Fast-forward or merge |
| Credential helper | Git Credential Manager |
| File system caching | Enabled |

---

### Step 8: Initialize dbt Project

```bash
dbt init dbt_learning
```

**Configuration provided:**
| Prompt | Value |
|---|---|
| Database adapter | Snowflake (option 1) |
| Authentication | password (option 1) |
| Role | accountadmin |
| Warehouse | dbt_wh |
| Database | dbt_db |
| Schema | raw |
| Threads | 1 |

Profile written to: `C:\Users\King\.dbt\profiles.yml`

---

### Step 9: Configure profiles.yml

The account identifier required manual correction. Including `.snowflakecomputing.com` caused a 404 error because dbt appends it automatically.

**Broken:**
```yaml
account: <account>.snowflakecomputing.com
```

**Fixed:**
```yaml
account: <account>
```

**Final profiles.yml:**
```yaml
dbt_learning:
  outputs:
    dev:
      type: snowflake
      account: <account>
      user: <user>
      password: <your_password>
      role: accountadmin
      database: dbt_db
      schema: raw
      warehouse: dbt_wh
      threads: 1
  target: dev
```

---

### Step 10: dbt debug —  All Checks Passed

```bash
cd dbt_learning
dbt debug
# All checks passed.
```

---

## 4. Troubleshooting Summary

| Issue | Status | Resolution |
|---|---|---|
| Python 3.14 incompatible with snowflake-connector |  Failed | Installed Python 3.11.9 in parallel |
| Failed building wheel for snowflake-connector-python |  Failed | Switched to Python 3.11 virtual environment |
| PowerShell blocking venv activation |  Failed | Set-ExecutionPolicy RemoteSigned -Scope CurrentUser |
| dbt running from Python 3.14 instead of venv |  Failed | Recreated venv with py -3.11, activated correctly |
| mashumaro UnserializableField error |  Failed | pip install mashumaro==3.13.1 --force-reinstall |
| Git not found in dbt debug |  Warning | Installed Git for Windows with default settings |
| Account identifier 404 (URL appended twice) |  Failed | Removed .snowflakecomputing.com from account field |
| dbt debug — connection test |  Success | All issues resolved, Snowflake connected |

---

## 5. Key Lessons Learned

- Always use **Python 3.11.x** for dbt — avoid 3.12+ until dbt officially supports it
- Always create a **virtual environment** before installing dbt — never install globally
- On Windows PowerShell, run `Set-ExecutionPolicy RemoteSigned` before activating a venv
- Snowflake `account` in profiles.yml should **NOT** include `.snowflakecomputing.com`
- Use `--prefer-binary` flag when pip install fails due to missing build tools
- Pin `mashumaro==3.13.1` if you encounter UnserializableField errors with dbt-core 1.11.x
- Install Git before running `dbt debug` to avoid confusing warnings

---

## 6. Quick Start Reference

Use these commands every time you start a new dbt session:

```bash
# 1. Activate virtual environment
cd C:\Users\King
.\dbt_env\Scripts\activate

# 2. Navigate to project
cd dbt_learning

# 3. You're ready to work
dbt debug
```

**Common dbt commands:**

| Command | Purpose |
|---|---|
| `dbt debug` | Test Snowflake connection |
| `dbt seed` | Load CSV files into Snowflake |
| `dbt run` | Build all models |
| `dbt test` | Run all data tests |
| `dbt docs generate` | Generate project documentation |
| `dbt docs serve` | Open docs in browser |
| `dbt run --select model_name` | Run a specific model only |
| `dbt run --target prod` | Run against production target |

---

*dbt Core + Snowflake Setup Documentation | King | February 2026*
