# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AutoPkg is a macOS-focused (with Windows/Linux support) automation framework for software packaging and distribution. It uses a **pipeline/processor pattern**: recipes (plist or YAML) define a sequence of processors that perform tasks like downloading, packaging, and importing into Munki. Written in Python 3 (targeting 3.10.11), current version 2.9.1.

## Commands

### Running Tests
```bash
# All unit tests
/usr/local/autopkg/python Scripts/run_tests.py

# Single test file
/usr/local/autopkg/python -m pytest Code/tests/test_urldownloader.py

# E2E tests (requires prefs file)
/usr/local/autopkg/python Code/autopkg list-processors --prefs tests/preferences.plist
/usr/local/autopkg/python Code/autopkg run -vv Code/tests/Test-Recipes/AutopkgCore.test.recipe.yaml
```

### Linting
Pre-commit hooks enforce formatting. The three tools are:
- **black** (code formatting)
- **isort** (import sorting, line length 88, trailing commas)
- **flake8** with **bugbear** (max-line-length 80, max-complexity 65, B950 line length ignored in tests)

```bash
# Run all linters
pre-commit run --all-files

# Individual tools
black Code/
isort Code/
flake8 Code/
```

### Install pre-commit hooks
```bash
pre-commit install --install-hooks
```

## Architecture

### Entry Point
`Code/autopkg` - main CLI script (no `.py` extension). Dispatches subcommands (`run`, `search`, `repo-add`, `info`, `audit`, `list-recipes`, `list-processors`, `make-override`, `verify-trust-info`, etc.) to handler functions.

### Core Library (`Code/autopkglib/`)
- `__init__.py` - Contains three key classes:
  - **`Processor`** (base class): All processors inherit from this. Defines `input_variables`, `output_variables`, `main()`, `process()`.
  - **`AutoPackager`**: Orchestrates recipe execution - loads processors, validates recipes, runs the pipeline. Each processor's output variables feed into the next step's environment.
  - **`Preferences`**: Cross-platform preferences abstraction (macOS CFPreferences, JSON/plist elsewhere).
- Each processor is a single `.py` file where the class name matches the filename (e.g., `URLDownloader.py` contains class `URLDownloader`).
- Processors are auto-discovered at import time via `import_processors()`. Custom processors from recipe directories are loaded dynamically via `get_processor()`.

### Key Submodules
- `Code/autopkglib/github/` - `GitHubSession` for API interactions (extends `URLGetter`)
- `Code/autopkglib/URLGetter.py` - Base class for curl-based HTTP operations, used by many processors
- `Code/autopkglib/munkirepolibs/` - Munki repository integration
- `Code/autopkgcmd/` - CLI helpers: argument parsing (`opts.py`, optparse-based) and recipe search (`searchcmd.py`)

### Privileged Helpers (`Code/autopkgserver/`)
macOS LaunchDaemons for privileged operations (package creation via `autopkgserver`, package installation via `autopkginstalld`). Communicate via Unix domain sockets with plist-encoded messages.

### Windows/Chocolatey (`Code/nuget/`)
Chocolatey install script and NuSpec generators. `Code/nuget/generated/` contains auto-generated code (excluded from pre-commit and coverage).

### Tests (`Code/tests/`)
unittest-based. Test fixtures include `preferences.json`, `preferences.plist`, and `Test-Recipes/`. CI runs on macOS, Windows, and Ubuntu.

## Code Style Notes
- `Code/nuget/generated/` is excluded from all linting and formatting.
- flake8 B950 (line length) is ignored in test files due to inline plist strings.
- isort uses profile settings: `multi_line_output=3`, `include_trailing_comma=True`, `line_length=88`.
