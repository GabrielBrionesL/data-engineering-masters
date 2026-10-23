# 01 · Environment Setup: WSL2 + VS Code + uv

> **Course:** Master's in Data Engineering, Python module, environment setup session
> **Date:** 23 October 2026
> **My setup:** Windows 11 → WSL2 (Ubuntu 22.04.5 LTS) → VS Code (Remote - WSL) → uv → Python 3.14
> **Course setup:** Windows 11 → Command Prompt → PyCharm → uv → Python 3.14

The course is taught on native Windows with PyCharm. I use **WSL2 Ubuntu with VS Code** at work, so I'm following the course in the same environment. These notes record each course step, its WSL/VS Code equivalent, and the problems I ran into along the way.

> **Placeholders:** `<windows-user>` is your Windows username and `<linux-user>` is your Ubuntu username (run `whoami` in Ubuntu to see it). Replace anything in `<angle brackets>` with your own values.

---

## Table of contents

1. [Mental model: two computers in one](#1-mental-model-two-computers-in-one)
2. [Where to run commands](#2-where-to-run-commands)
3. [Prerequisites inside Ubuntu](#3-prerequisites-inside-ubuntu)
4. [Creating the project with uv](#4-creating-the-project-with-uv)
5. [Pinning the right Python version](#5-pinning-the-right-python-version)
6. [Virtual environments: `uv venv` vs `uv run`](#6-virtual-environments-uv-venv-vs-uv-run)
7. [Selecting the interpreter in VS Code](#7-selecting-the-interpreter-in-vs-code)
8. [Writing and running a first script](#8-writing-and-running-a-first-script)
9. [Run configurations (`launch.json`)](#9-run-configurations-launchjson)
10. [Debugging](#10-debugging)
11. [Running modules inside the `src` package](#11-running-modules-inside-the-src-package)
12. [Copying course materials from Windows](#12-copying-course-materials-from-windows)
13. [Jupyter Lab](#13-jupyter-lab)
14. [Ubuntu maintenance](#14-ubuntu-maintenance)
15. [Cheat sheets](#15-cheat-sheets)
16. [Gotchas I hit today](#16-gotchas-i-hit-today)

---

## 1. Mental model: two computers in one

WSL2 runs a **real Linux machine inside Windows**, with its own filesystem. This is what confused me first: running `code .` from my Ubuntu terminal opened a folder full of `.cache`, `.ssh`, `.vscode-server`... nothing like the professor's `C:\Users\...` folder.

| | Windows | Linux (WSL) |
|---|---|---|
| Home folder | `C:\Users\<windows-user>` | `/home/<linux-user>` (shortcut: `~`) |
| Hidden config | `AppData\` | Files/folders starting with `.` (dotfiles) |
| Path separator | `\` | `/` |

**Getting between the two:**

```bash
# Windows C: drive, seen from Linux
cd /mnt/c/Users/<windows-user>/

# Open the current Linux folder in Windows File Explorer
explorer.exe .
```

Linux files also appear in File Explorer's sidebar under **Linux**.

**Rule of thumb:** keep projects on the Linux side (`~/projects/...`), not under `/mnt/c`. Python, git and uv run much faster there, and it mirrors how WSL is used in industry.

**Dotfiles worth knowing:**

- `~/.bashrc`: runs every time a terminal opens (PATH, aliases).
- `~/.ssh/`: SSH keys (GitHub, servers).
- `~/.vscode-server/`: the VS Code backend that runs inside Linux. Leave it alone.

**Don't open VS Code on your whole home folder.** Make a project folder and open that instead:

```bash
mkdir -p ~/projects/masters/curso
cd ~/projects/masters/curso
```

---

## 2. Where to run commands

Everything runs in the **Ubuntu (bash) terminal**, not PowerShell or CMD.

- ✅ The VS Code integrated terminal, when the bottom-left badge says **WSL: Ubuntu** and the terminal says `bash`
- ✅ The Ubuntu app / Ubuntu tab in Windows Terminal
- ❌ Any prompt that looks like `C:\...>` or `PS C:\...>`

**Exception:** `wsl ...` commands (e.g. `wsl -u root`, `wsl --update`) are **Windows** commands and must be run in PowerShell. Inside Ubuntu they fail with `Command 'wsl' not found`.

---

## 3. Prerequisites inside Ubuntu

Windows installs don't count. WSL needs its own copy of each tool.

### git

`uv init` creates a git repo for you, but only if git is installed.

```bash
git --version
# if missing:
sudo apt update && sudo apt install -y git

git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### uv

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc     # or close and reopen the terminal
uv --version
```

---

## 4. Creating the project with uv

| Course (CMD) | Me (bash) |
|---|---|
| `cd Documents\curso` | `cd ~/projects/masters/curso` |
| `dir` | `ls` |
| `uv init --package paquete_inicial` | `uv init --package paquete_inicial` (identical) |

```bash
cd ~/projects/masters/curso
uv init --package paquete_inicial
cd paquete_inicial
ls -la          # -a shows hidden files like .git and .gitignore
code .          # open THIS folder in VS Code
```

Resulting layout:

```
paquete_inicial/
├── .git/
├── .gitignore
├── .python-version
├── pyproject.toml
├── README.md
└── src/
    └── paquete_inicial/
        └── __init__.py
```

`--package` creates the **src layout**: your code lives in `src/paquete_inicial/`, and uv installs it into the virtual environment as a real, importable package. That's what makes `uv run -m paquete_inicial.xxx` work later (section 11).

Sanity check: `--package` also creates a console script with the same name as the project.

```bash
uv run paquete-inicial
# Hello from paquete-inicial!
```

---

## 5. Pinning the right Python version

**Problem:** my first run said `Using CPython 3.10.12 interpreter at: /usr/bin/python3.10`, while the professor's said `CPython 3.14.7`. uv had picked up Ubuntu's built-in system Python.

**Fix:** let uv download and manage its own Python. Don't touch the system Python.

```bash
uv python install 3.14     # downloads a uv-managed Python 3.14
uv python pin 3.14         # writes 3.14 into .python-version
rm -rf .venv               # remove the old 3.10 environment
uv sync                    # rebuild .venv with 3.14
uv run python --version    # → Python 3.14.x
```

Also update `pyproject.toml` so the project declares it:

```toml
requires-python = ">=3.14"
```

> The version uv picks depends on what's installed. Always check the `Using CPython ...` line the first time.

---

## 6. Virtual environments: `uv venv` vs `uv run`

| Command | What it does |
|---|---|
| `uv venv` (course) | Creates an empty `.venv` |
| `uv run <something>` | Creates `.venv` if needed, installs the project and dependencies, writes `uv.lock`, then runs the command |
| `uv sync` | Makes `.venv` match `pyproject.toml` / `uv.lock` exactly |

### Activating

| Windows (course) | Linux (me) |
|---|---|
| `.venv\Scripts\activate` | `source .venv/bin/activate` |

Leave with `deactivate`.

With uv, activation is often unnecessary: `uv run ...` always uses the project's `.venv`. Activate when you want to type `python`, `pytest`, etc. directly.

List installed packages (equivalent to PyCharm's package table):

```bash
uv pip list
```

---

## 7. Selecting the interpreter in VS Code

**Course (PyCharm):** Settings → Python → Interpreter → pick `.venv`
**Me (VS Code):** `Ctrl+Shift+P` → **Python: Select Interpreter** → `./.venv/bin/python`

PyCharm's **Add Interpreter → On WSL...** option isn't needed. That's how a Windows editor reaches into WSL, but VS Code already runs *inside* WSL.

### Error I hit

> `Default interpreter path 'C:\Users\<windows-user>\AppData\Local\Programs\Python\Python312\python.exe' could not be resolved`

**Cause:** my VS Code user settings pointed at a **Windows** Python, and that setting follows me into WSL, where `C:\...` paths don't exist.
**Fix:** select the project's `.venv` interpreter as above. It's harmless, but it stops the warning.

**Extensions are installed per environment.** When VS Code offers the Python extension, choose **Install in WSL: Ubuntu**.

---

## 8. Writing and running a first script

`main.py` at the project root:

```python
def add(num1: int, num2):
    result = num1 + num2
    return result


def subtract(num1: int, num2):
    return num1 - num2


if __name__ == '__main__':
    print(add(1, 2))
    print(subtract(1, 2))
```

```bash
uv run main.py
# 3
# -1
```

### Gotcha: PyCharm's gray parameter labels aren't code

In the professor's screen, the calls looked like `suma(num1: 1, num2: 2)`. The `num1:` / `num2:` parts are **inlay hints** that PyCharm draws, not typed text. I first typed `add(num1, num2)`, which raises `NameError` because those names only exist *inside* the function.

Valid ways to call it:

```python
add(1, 2)              # positional
add(num1=1, num2=2)    # keyword arguments: same result, more explicit
```

To get the same hints in VS Code: Settings → search **inlay hints call argument names** → `all`.

**Notes:**

- `num1: int` is a **type hint**. It's documentation for humans and tools, and isn't enforced at runtime.
- PEP 8 recommends **two blank lines** between top-level functions.

---

## 9. Run configurations (`launch.json`)

**Course (PyCharm):** Run → Edit Configurations...
**Me (VS Code):** `.vscode/launch.json`, created from Run and Debug sidebar → *create a launch.json file* → Python Debugger → Python File.

**Not strictly needed** to run code (`uv run ...` or the ▷ button works), but very useful for debugging, arguments and environment variables.

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "main",
            "type": "debugpy",
            "request": "launch",
            "program": "${workspaceFolder}/main.py",
            "console": "integratedTerminal",
            "env": { "PYTHONUNBUFFERED": "1" }
        },
        {
            "name": "functions",
            "type": "debugpy",
            "request": "launch",
            "module": "paquete_inicial.functions",
            "console": "integratedTerminal",
            "env": { "PYTHONUNBUFFERED": "1" }
        },
        {
            "name": "use_function",
            "type": "debugpy",
            "request": "launch",
            "module": "paquete_inicial.use_function",
            "console": "integratedTerminal",
            "env": { "PYTHONUNBUFFERED": "1" }
        }
    ]
}
```

| PyCharm field | VS Code equivalent |
|---|---|
| Name | `"name"` |
| Interpreter | The interpreter selected in section 7 |
| Script path | `"program": "${workspaceFolder}/file.py"` |
| Module name | `"module": "package.module"` |
| Environment variables | `"env": { ... }` |
| Add content/source roots to PYTHONPATH | Not needed: uv installs the `src` package into `.venv` |
| Run with `uv run` | Not needed: running with the `.venv` Python is equivalent |

`PYTHONUNBUFFERED=1` makes `print` output appear immediately instead of in batches.

Pick a configuration from the dropdown at the top of Run and Debug, then press `F5`.

---

## 10. Debugging

Click in the gutter left of a line number to set a **breakpoint**, then start debugging (`F5`). Execution pauses *before* the highlighted line runs.

| Action | PyCharm | VS Code | Shortcut |
|---|---|---|---|
| Resume to next breakpoint | Resume ▷▷ | Continue | `F5` |
| Run line, don't enter functions | Step Over | Step Over | `F10` |
| Enter function call on this line | Step Into | Step Into | `F11` |
| Finish current function | Step Out | Step Out | `Shift+F11` |
| Restart / Stop | ⟳ / ■ | ⟳ / □ | `Ctrl+Shift+F5` / `Shift+F5` |

| PyCharm panel | VS Code panel |
|---|---|
| Threads & Variables | Variables → Locals |
| Frames list | Call Stack |
| Evaluate expression | Debug Console tab (or Watch panel to keep expressions visible) |

"No Configuration" in the Run and Debug dropdown is fine. It just debugs the open file.

---

## 11. Running modules inside the `src` package

Files: `src/paquete_inicial/functions.py` and `src/paquete_inicial/use_function.py`.

### Error 1: `.py` in a module name

```bash
uv run -m paquete_inicial.functions.py
# ModuleNotFoundError: __path__ attribute not found on 'paquete_inicial.functions'
#   while trying to find 'paquete_inicial.functions.py'
```

`-m` takes a **dotted module name, not a file path.** With `.py`, Python looks for a submodule called `py`. (The professor hit the same error on purpose.)

```bash
uv run -m paquete_inicial.functions   # ✅
```

| Command | Takes | Example |
|---|---|---|
| `uv run file.py` | A file path, with `.py` | `uv run src/paquete_inicial/functions.py` |
| `uv run -m module` | A dotted name, no `.py` | `uv run -m paquete_inicial.functions` |

### Error 2: relative imports

```python
# use_function.py
from .functions import add

if __name__ == '__main__':
    print('The result is: ', add(num1=10, num2=20))
```

Running it as a script (the ▷ button, or `python path/to/use_function.py`):

```
ImportError: attempted relative import with no known parent package
```

The leading `.` means "from **my** package." Run as a plain script, Python doesn't know which package the file belongs to. Run it as a module instead:

```bash
uv run -m paquete_inicial.use_function
# The result is:  30
```

In VS Code, use a launch configuration with `"module"` instead of `"program"` (section 9). This is the same switch as script → module in PyCharm's Edit Configurations. The ▷ button always runs files as scripts, so it will keep failing for files with relative imports.

`__pycache__/` folders appear automatically. They hold compiled bytecode, and uv's `.gitignore` already excludes them.

---

## 12. Copying course materials from Windows

Copying from Windows (OneDrive) into the Linux filesystem showed this repeatedly:

> *Are you sure you want to copy this file without its properties?*

**Why:** Windows attaches hidden metadata (the "downloaded from the internet" mark, OneDrive sync info) that Linux can't store. **Clicking Yes, with "Do this for all current items" ticked, is safe.** File contents copy intact.

**Before copying from OneDrive:** right-click the source folder → **Always keep on this device**, and wait for the green checkmarks. Cloud-only files may fail to copy.

**Terminal alternative (no pop-ups):**

```bash
cp -r "/mnt/c/Users/<windows-user>/<path>/course_materials" ~/projects/masters/curso/paquete_inicial/
```

### Cleanup: `:Zone.Identifier` and `:mshield` junk files

Some metadata came through as separate files next to every real file. Preview them first, then delete:

```bash
cd ~/projects/masters/curso/paquete_inicial
find course_materials \( -name "*:Zone.Identifier" -o -name "*:mshield" \)          # preview
find course_materials \( -name "*:Zone.Identifier" -o -name "*:mshield" \) -delete  # delete
```

(The course names this folder `Temario`; mine is `course_materials`.)

---

## 13. Jupyter Lab

With Jupyter installed in the project (e.g. `uv add jupyterlab`):

```bash
uv run jupyter lab
```

### Why I got a login page and the professor didn't

On Windows, Jupyter opens the browser automatically with a secret **token** in the URL. WSL has no browser to open, so opening plain `http://127.0.0.1:8888` asks for a token or password.

**Fix:** **Ctrl+click** the full URL printed in the terminal (`http://localhost:8888/lab?token=...`). In my case it opened inside VS Code's built-in browser tab.

| | Token (default) | Password |
|---|---|---|
| Security | Random each launch; very strong | Only as strong as what you choose |
| Convenience | Ctrl+click the URL each time | Set once, reuse |

Since the server is only reachable from my machine, both are fine. I'm sticking with the token. Never disable authentication.

Other notes:

- Stop the server with `Ctrl+C` in its terminal. Run other commands in a **second** terminal.
- `Could not determine jupyterlab build status without nodejs` is harmless unless you build extensions.
- VS Code alternative: open any `.ipynb` → **Select Kernel** → `.venv` Python, and run notebooks without a browser.

### Gotcha: `.html` vs `.ipynb`

The course materials ship both versions of each lesson. I opened `s1_elementos_basicos.html` and couldn't run anything (no ▶, no kernel; only "Distrust HTML").

- **`.ipynb`** = live notebook, runnable (`Shift+Enter` runs a cell)
- **`.html`** = static snapshot for reading only

---

## 14. Ubuntu maintenance

### Version check

I'm on **Ubuntu 22.04.5 LTS**: still supported (standard support until April 2027) but two LTS releases behind 24.04 and 26.04. WSL doesn't reliably prompt for upgrades; at most a login message says `New release '24.04.5 LTS' available`.

Plan: check what work uses (`lsb_release -a`) and match it. The two upgrade routes:

- **In place:** `sudo do-release-upgrade`. Goes one LTS at a time (22.04 → 24.04 → 26.04).
- **Fresh distro alongside:** from PowerShell, `wsl --list --online`, then `wsl --install <name>`.

uv-managed Python works the same on any of them.

### Routine updates

Not automatic. Run every week or two:

```bash
sudo apt update && sudo apt upgrade -y
```

The WSL kernel is updated separately, from PowerShell: `wsl --update`.

### Forgot the Linux password

From **PowerShell**:

```powershell
wsl -u root
```

Then, at the root prompt (`#`):

```bash
passwd <linux-user>
exit
```

Password prompts show nothing as you type. That's normal.

---

## 15. Cheat sheets

### Windows CMD ↔ bash

| CMD | bash |
|---|---|
| `dir` | `ls` (`ls -la` for hidden files and details) |
| `cd Documents\curso` | `cd ~/projects/masters/curso` |
| `.venv\Scripts\activate` | `source .venv/bin/activate` |
| `mkdir a\b\c` | `mkdir -p a/b/c` |
| `copy` / `xcopy` | `cp` / `cp -r` |
| `del` | `rm` (no recycle bin!) |
| `start .` | `explorer.exe .` |

### PyCharm ↔ VS Code

| PyCharm | VS Code |
|---|---|
| Settings → Python → Interpreter | `Ctrl+Shift+P` → Python: Select Interpreter |
| Run/Debug Configurations | `.vscode/launch.json` |
| Script vs Module run mode | `"program"` vs `"module"` |
| Inlay parameter hints (on by default) | Setting: inlay hints → call argument names |
| Evaluate Expression | Debug Console / Watch |
| "Store as project file" | Commit `.vscode/` to git (optional) |

### uv essentials

```bash
uv init --package <name>     # new project, src layout
uv python install 3.14       # get a managed Python
uv python pin 3.14           # set the project's version
uv venv                      # create an empty .venv
uv sync                      # make .venv match the lockfile
uv add <pkg>                 # add a dependency
uv pip list                  # list installed packages
uv run <file.py>             # run a file
uv run -m <pkg.module>       # run a module (no .py!)
```

---

## 16. Gotchas I hit today

1. **`code .` in `~`** opened my whole Linux home → make a project folder first.
2. **Windows Python path in VS Code settings** → select the `.venv` interpreter.
3. **uv chose system Python 3.10** → `uv python install` + `pin` + rebuild.
4. **Forgot the WSL password** → `wsl -u root` from PowerShell, then `passwd`.
5. **Ran `wsl` inside Ubuntu** → `wsl` is a Windows command.
6. **Typed PyCharm's inlay hints as code** → `NameError`.
7. **`-m module.py`** → drop the `.py`.
8. **Relative import as a script** → run with `-m` or a `"module"` launch config.
9. **`:Zone.Identifier` / `:mshield` files** after copying from Windows → `find ... -delete`.
10. **Jupyter login page** → Ctrl+click the URL with the token.
11. **Opened `.html` instead of `.ipynb`** → the HTML version is read-only.
