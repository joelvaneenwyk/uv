# Fork Changes: Windows GUI Subsystem Fixes for Python Launchers

This document describes the changes made in this fork
([joelvaneenwyk/uv](https://github.com/joelvaneenwyk/uv)) relative to the upstream repository
([astral-sh/uv](https://github.com/astral-sh/uv)).

## Summary

These changes fix a long-standing Windows-specific bug where `pythonw.exe` and GUI script
entrypoints incorrectly use the console subsystem, causing unwanted console windows to appear
when running GUI applications, background tasks, or scheduled jobs.

## Commits

| Commit | Description |
|--------|-------------|
| `4cb58d3` | Fix Windows `pythonw.exe` venv launcher using console instead of GUI subsystem |
| `3a1e894` | Fix GUI script console window: use `CREATE_NO_WINDOW` in trampoline for GUI launchers |
| `eb922d3` | Add taskfile for rebuilding the trampolines |
| `a941e51` | Formatting cleanup of taskfile |

## Technical Details

### 1. Venv launcher subsystem fix (`crates/uv-virtualenv/src/virtualenv.rs`)

**Problem:** `replace_link_to_executable` was called with a hardcoded `is_gui=false` for all
Windows trampoline launchers, including `pythonw.exe`. This caused `pythonw.exe` in virtual
environments to use the console PE subsystem instead of the GUI subsystem.

**Fix:**

- Added `WindowsExecutable::is_gui()` method that returns `true` for GUI-subsystem variants
  (`Pythonw`, `PythonwMajorMinort`, `PyPyw`, `PyPyMajorMinorw`).
- Added `gui_executable()` helper that resolves `python.exe` to `pythonw.exe` if it exists, so
  the GUI trampoline targets the correct GUI-subsystem base executable.
- Updated all `replace_link_to_executable` call sites to pass the correct `is_gui` value from
  `WindowsExecutable::is_gui()`.
- GUI-variant launchers (`pythonw.exe`, `pythonwXYt.exe`) now target the base `pythonw.exe`
  instead of `python.exe`, ensuring the child process is also GUI-subsystem.
- Added unit tests for `is_gui()` and `gui_executable()`.

### 2. Managed installation link functions (`crates/uv-python/src/managed.rs`)

**Problem:** `create_link_to_executable` and `replace_link_to_executable` hardcoded `is_gui=false`
when calling `windows_python_launcher`, and had a `TODO` comment acknowledging the missing GUI
launcher support.

**Fix:**

- Added `is_gui: bool` parameter to both `create_link_to_executable` and
  `replace_link_to_executable`.
- Replaced the hardcoded `false` with the new parameter when calling `windows_python_launcher`.
- Updated call sites in `crates/uv/src/commands/python/install.rs` to pass `false` (console
  executables for `uv python install` bin links).
- Removed the now-resolved `TODO(zanieb)` comment.
- Simplified `ManagedPythonInstallation::try_from_interpreter` by using `dunce::canonicalize`
  directly on the root path instead of the intermediate `absolute_root()` method, and removed
  the now-unused `absolute_root()` method and its test.

### 3. Trampoline `CREATE_NO_WINDOW` flag (`crates/uv-trampoline/src/bounce.rs`)

**Problem:** When a GUI trampoline spawns a child process, it called `CreateProcessA` with empty
creation flags. If the child process is a console-subsystem binary (e.g., `python.exe` as a
fallback when `pythonw.exe` is unavailable), Windows allocates a new console window.

**Fix:**

- Added `is_gui: bool` parameter to the `spawn_child` function.
- When `is_gui` is `true`, passes `CREATE_NO_WINDOW` as the creation flag to `CreateProcessA`,
  preventing console window allocation for the child process.
- Updated the call site in `bounce()` to forward the existing `is_gui` parameter.

### 4. Rebuilt trampoline binaries (`crates/uv-trampoline-builder/trampolines/`)

All six prebuilt trampoline executables were rebuilt to include the `CREATE_NO_WINDOW` change:

- `uv-trampoline-x86_64-console.exe` / `uv-trampoline-x86_64-gui.exe`
- `uv-trampoline-i686-console.exe` / `uv-trampoline-i686-gui.exe`
- `uv-trampoline-aarch64-console.exe` / `uv-trampoline-aarch64-gui.exe`

### 5. PE subsystem validation test (`crates/uv-trampoline-builder/src/lib.rs`)

Added a test (`prebuilt_trampolines_have_correct_subsystem`) that reads the PE headers of all
six prebuilt trampoline binaries and asserts that GUI trampolines have `IMAGE_SUBSYSTEM_WINDOWS_GUI`
(2) and console trampolines have `IMAGE_SUBSYSTEM_WINDOWS_CUI` (3). This prevents regressions
where trampolines are accidentally rebuilt with the wrong subsystem.

### 6. `pythonw.exe` fallback warning (`crates/uv-install-wheel/src/wheel.rs`)

When installing a GUI script entrypoint and `pythonw.exe` is not found next to `python.exe`, the
code now emits a `warn_user_once!` message before falling back to `python.exe`. Previously the
fallback was silent.

### 7. Documentation (`docs/concepts/python-versions.md`)

Added a new "Windows Python launchers" section documenting:

- The difference between console (`python.exe`) and GUI (`pythonw.exe`) launchers.
- How `uv venv` produces both launcher types.
- Exceptions for Pyodide and GraalPy.
- How GUI script entrypoints work.

Also updated the PyPy note to reflect that PyPy is no longer actively developed.

### 8. Taskfile for trampoline builds (`taskfile.yaml`)

Added a [Taskfile](https://taskfile.dev) that automates the trampoline rebuild workflow:

- `setup` — installs the nightly Rust toolchain, `rust-src` component, cross-compilation targets,
  and `cargo-xwin`.
- `trampoline:build` — builds trampolines for all three architectures (i686, x86_64, aarch64)
  using native builds on Windows or `cargo xwin` cross-compilation on Linux/macOS.
- `trampoline:copy` — copies the built binaries to the `uv-trampoline-builder/trampolines/`
  directory.

---

## Proposed GitHub PR

### Title

Fix `pythonw.exe` venv launcher using console subsystem on Windows

### Description

## Summary

- Fix `pythonw.exe` in virtual environments using the console PE subsystem instead of the GUI
  (Windows) subsystem, which causes unwanted console windows when running GUI applications
- Pass `CREATE_NO_WINDOW` in the GUI trampoline's `CreateProcessA` call to suppress console window
  allocation when the child process is a console binary (fallback scenario)
- Add PE subsystem validation test, `pythonw.exe` fallback warning, and documentation for Windows
  launcher behavior

## Problem

On Windows, `pythonw.exe` inside a venv created by `uv venv` has the wrong PE subsystem. It uses
the console subsystem (`IMAGE_SUBSYSTEM_WINDOWS_CUI`) instead of the GUI subsystem
(`IMAGE_SUBSYSTEM_WINDOWS_GUI`). This causes a visible console window to flash or persist when
running GUI applications, background tasks, or scheduled jobs.

**uv version:** 0.10.4 (and earlier)
**OS:** Windows 11

### Minimal reproduction

```console
> uv venv .venv
> python -c "import struct; f=open('.venv/Scripts/pythonw.exe','rb'); f.seek(0x3C); pe=struct.unpack('<I',f.read(4))[0]; f.seek(pe+0x5C); sub=struct.unpack('<H',f.read(2))[0]; print(f'Subsystem: {sub} (expected 2=GUI, got {sub})')"
Subsystem: 3 (expected 2=GUI, got 3)
```

The `pythonw.exe` trampoline reports subsystem 3 (console) instead of 2 (GUI). This can also be
observed by running a simple tkinter script through the venv's `pythonw.exe` — an empty console
window appears alongside the GUI:

```console
> uv venv .venv
> echo "import tkinter; tkinter.Tk().mainloop()" > gui_test.py
> .venv\Scripts\pythonw.exe gui_test.py
# A console window flashes on screen alongside the tkinter window
```

### Root cause

1. `replace_link_to_executable` in `virtualenv.rs` hardcodes `is_gui=false` for all trampoline
   launchers, so `pythonw.exe` gets the console-subsystem trampoline. The trampoline also targets
   `python.exe` as the child process instead of the base `pythonw.exe`.
2. The trampoline's `spawn_child` in `bounce.rs` calls `CreateProcessA` with zero creation flags.
   When the child is a console binary (e.g., `python.exe` fallback), Windows allocates a console
   window even though the parent is a GUI process.

## Related issues

- astral-sh/uv#6801 — Provide an option to start `uv` on Windows without a console window
- astral-sh/uv#6805 — Add an option to `uv run` to run a `.py` script with `pythonw.exe`

These issues focus on `uv.exe` itself or `uv run` flags; this PR fixes the underlying venv
launcher and trampoline subsystem, which is a prerequisite for correct GUI behavior.

## Test plan

- [ ] Unit tests for `WindowsExecutable::is_gui()` and `gui_executable()`
- [ ] PE subsystem validation test for all six prebuilt trampoline binaries
  (`prebuilt_trampolines_have_correct_subsystem`)
- [ ] Manual verification: `uv venv` + `pythonw.exe` + tkinter script shows no console window
- [ ] Verify no behavioral change for console (`python.exe`) usage
