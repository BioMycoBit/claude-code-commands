# Cross-Platform Risk Registry

Claude reads this before every `/implement`. Never write platform-specific code without a cross-platform alternative.

**Gold Standard:** Use `IS_LINUX = platform.system() == 'Linux'` / `IS_MAC` / `IS_WINDOWS` with fallback chains per OS.

---

## Anti-Pattern Table

| # | Pattern | Risk | Cross-Platform Alternative |
|---|---------|------|---------------------------|
| 1 | `import fcntl` | Linux-only module | `msvcrt` on Windows or `portalocker` |
| 2 | `signal.SIGKILL` / `signal.SIGSTOP` | Unix-only signals | `process.terminate()` |
| 3 | `os.fork()` | No Windows support | `multiprocessing.Process()` |
| 4 | `os.getuid()` / `os.getgid()` | Unix-only | `ctypes.windll` admin check |
| 5 | `import grp` / `import pwd` | Unix-only modules | Conditional import with try/except |
| 6 | `"/tmp/"` hardcoded | Wrong on Windows | `tempfile.gettempdir()` |
| 7 | `"/dev/null"` hardcoded | Unix-only | `os.devnull` |
| 8 | `select.poll()` | Not on Windows | `select.select()` or `selectors` module |
| 9 | `subprocess.*"bash "` | No bash on Windows | `subprocess([sys.executable, ...])` |
| 10 | `os.symlink(` without guard | Needs admin on Windows | Platform check or skip |
| 11 | `os.chmod(.*0o` | No-op on Windows | Document limitation or skip |
| 12 | `"ss "` / `"lsof "` without fallback | Linux/Mac only | Fallback chain: ss → lsof → netstat |

---

## Known-Risk Dependencies

Add your project's dependencies here. Common cross-platform gotchas:

| Dependency | Platform Gotcha |
|-----------|----------------|
| **asyncio** | Windows `ProactorEventLoop` default; `sleep()` has ~15ms resolution vs 1ms on Linux |
| **signal** | Windows only has `SIGINT` + `SIGTERM`; no `SIGKILL`/`SIGSTOP`/`SIGUSR1` |
| **subprocess** | `shell=True` → bash (Unix) vs cmd.exe (Windows); different quoting rules |
| **os.symlink** | Windows requires admin or Developer Mode enabled |
| **pathlib/os.path** | Case-sensitive (Linux) vs case-insensitive (Mac/Windows); path separators differ |
| **sqlite3** | File locking behavior differs: advisory (Unix) vs mandatory (Windows) |
| **uvicorn/gunicorn** | gunicorn is Unix-only; uvicorn works cross-platform |

---

## Platform Behavior Matrix

| Behavior | Linux | Mac | Windows |
|----------|-------|-----|---------|
| File locking | `fcntl` (advisory) | `fcntl` (advisory) | `LockFileEx` (mandatory) |
| Signals | Full POSIX set | Full POSIX set | `SIGINT` + `SIGTERM` only |
| Symlinks | No special perms | No special perms | Admin or Developer Mode |
| Temp dir | `/tmp` | `/tmp` or `/var/folders/` | `C:\Users\X\AppData\Local\Temp` |
| Process spawn | `fork()` + `exec()` | `fork()` + `exec()` | `CreateProcess()` only |
| Shell | bash/sh | bash/zsh | cmd.exe/PowerShell |
| asyncio loop | `SelectorEventLoop` | `SelectorEventLoop` | `ProactorEventLoop` |
| Timer resolution | ~1ms | ~1ms | ~15ms |
| Path case | Sensitive | Insensitive (default) | Insensitive |
| `os.chmod` | Full POSIX perms | Full POSIX perms | Read-only bit only |

---

## When Writing New Code

1. **Check this table first** — if your code uses any pattern from column 2, use column 4
2. **New dependencies** — check if the library has platform-specific behavior before adopting
3. **Shell commands** — always use fallback chains, never hardcode one tool
4. **File paths** — use `pathlib.Path` or `os.path.join`, never string concatenation with `/`
5. **Temp files** — `tempfile.gettempdir()`, never `"/tmp/"`
