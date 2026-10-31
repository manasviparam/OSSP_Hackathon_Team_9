# Linux Command Execution Time Profiler

A systems-programming project that serves a local web dashboard for measuring how Linux commands use CPU, memory, I/O, and kernel resources. The backend is pure C (hand-written HTTP server, SQLite storage) and the frontend is plain HTML, CSS, and JavaScript with no frameworks.

Built for **Ubuntu 24.04 LTS** (bare metal or WSL2).

---

## Table of Contents

- [Features](#features)
- [Technology Stack](#technology-stack)
- [OS Concepts Demonstrated](#os-concepts-demonstrated)
- [Requirements](#requirements)
- [Build and Run](#build-and-run)
- [Dashboard Tabs](#dashboard-tabs)
- [API Reference](#api-reference)
- [Architecture](#architecture)
- [Security Model](#security-model)
- [Testing](#testing)
- [Project Structure](#project-structure)
- [Troubleshooting](#troubleshooting)
- [Make Targets](#make-targets)

---

## Features

| Feature | Implementation |
|---------|----------------|
| Command execution | `fork()` + `execvp()` + `wait4()`; never `system()`, `popen()`, or `sh -c` |
| Wall-clock timing | `clock_gettime(CLOCK_MONOTONIC)` |
| User / system CPU time | `struct rusage` from `wait4()` |
| Peak memory | `ru_maxrss` (KiB) |
| Page faults | `ru_minflt` (minor), `ru_majflt` (major) |
| Context switches | `ru_nvcsw` (voluntary), `ru_nivcsw` (involuntary) |
| stdout / stderr capture | `pipe()` + `dup2()` with a non-blocking drain loop |
| Timeouts | Configurable 100 to 60000 ms, then `SIGTERM`, 150 ms grace, `SIGKILL` |
| Syscall tracing | `ptrace(PTRACE_SYSCALL)`: per-syscall counts and elapsed time (x86_64) |
| Process monitor | Read-only snapshots of `/proc/<pid>/status` and `/proc/<pid>/stat` |
| History | Every run stored in SQLite; latest 100 shown in the dashboard |
| Per-command stats | Min / avg / max wall time, avg CPU, avg RSS, computed in SQL |
| Repeat analysis | Run a command N times (2 to 20): min / mean / max / stddev |
| Command comparison | Two commands side by side with charts |
| Live RSS chart | Canvas timeline polled every second for any PID |
| Allowlist | 20 pre-approved commands only; shell metacharacters rejected |
| Local only | Binds to `127.0.0.1`; no path traversal in static file serving |

**Allowed commands:** `ls pwd echo cat head tail wc sort uniq grep find sleep gzip date whoami uname df du uptime free`

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| Backend / profiler | C11 (`gcc`, `-Wall -Wextra -Wpedantic`) |
| HTTP server | Custom, in `src/server.c` and `src/http.c` |
| Database | SQLite 3 in WAL mode (`src/db.c`) |
| Frontend | HTML5, CSS3, vanilla JavaScript |
| Environment | Ubuntu 24.04 LTS, WSL2 or bare Linux |

---

## OS Concepts Demonstrated

- `fork()` / `execvp()`: spawn a child process without a shell
- `wait4()` / `getrusage()`: reap the child and collect resource usage
- `pipe()` / `dup2()` / `poll()`: capture stdout and stderr
- `clock_gettime(CLOCK_MONOTONIC)`: high-resolution timing
- `setpgid()` / `kill(-pgid, SIG)`: terminate a whole process group on timeout
- `sigaction()` / `SIGCHLD` / `SA_RESTART`: zombie-free child reaping
- `SIGTERM` then `SIGKILL`: graceful, then forced, termination
- `ptrace(PTRACE_TRACEME)` / `ptrace(PTRACE_SYSCALL)`: syscall tracing
- `/proc/<pid>/status` and `/proc/<pid>/stat`: live process inspection
- Socket programming: `socket`, `bind`, `listen`, `accept`, with one forked child per request
- SQLite WAL: concurrent-safe persistent storage

---

## Requirements

```bash
sudo apt update
sudo apt install -y build-essential libsqlite3-dev curl
```

Verify:

```bash
gcc --version      # 13 or newer recommended
sqlite3 --version  # 3.37 or newer recommended
```

---

## Build and Run

```bash
cd Linux_Command_Execution_Time_Profiler

make                      # builds ./profiler_server
./profiler_server         # default port 8080
./profiler_server 8090    # custom port
```

Open **http://127.0.0.1:8080** in your browser. Under WSL2 this also works from a Windows browser, since WSL2 forwards loopback. Stop the server with `Ctrl+C`.

The `db/` directory holds `profiler.db`, which is created automatically at startup.

---

## Dashboard Tabs

| Tab | Description |
|-----|-------------|
| **Run** | Pick a command, set arguments and timeout, view all metrics, stdout/stderr, and a time-breakdown chart |
| **Repeat** | Run the same command 2 to 20 times; min / mean / max / stddev and a per-run bar chart |
| **Compare** | Run two commands with the same timeout; side-by-side cards and a multi-metric chart |
| **Syscall Trace** | `ptrace` pass: total syscalls, per-syscall count and elapsed time, bar chart |
| **Monitor PID** | Snapshot of `/proc/<pid>`; "Start Live Watch" draws a 1-second RSS timeline |
| **History** | Latest 100 runs from SQLite |
| **Stats** | Per-command aggregates with bar chart |

Repeat and Compare are driven by the frontend, which issues repeated `/api/run` calls.

---

## API Reference

All endpoints are served from `http://127.0.0.1:<port>`.

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/api/allowlist` | List runnable command names |
| POST | `/api/run` | Execute a command and return metrics |
| POST | `/api/trace` | Execute under `ptrace` and return syscall stats |
| GET | `/api/history` | Latest 100 stored runs |
| GET | `/api/stats` | Per-command aggregates |
| GET | `/api/monitor?pid=<N>` | Snapshot of `/proc/<N>` |

### `POST /api/run`

Request:

```json
{"command": "echo", "args": "hello world", "timeout_ms": 5000}
```

Response:

```json
{
  "status": "completed",
  "wall_ms": 2.341,
  "user_ms": 0.873,
  "system_ms": 0.411,
  "max_rss_kb": 1024,
  "minor_faults": 42,
  "major_faults": 0,
  "voluntary_cs": 1,
  "involuntary_cs": 0,
  "exit_code": 0,
  "term_signal": 0,
  "timed_out": false,
  "stdout": "hello world\n",
  "stderr": ""
}
```

`status` is one of `completed`, `failed`, `timed_out`, or `signal_terminated`.

### `POST /api/trace`

Same request body as `/api/run`.

```json
{
  "total_syscalls": 47,
  "timed_out": false,
  "exit_code": 0,
  "summary": [
    {"number": 231, "name": "exit_group", "count": 1,
     "total_elapsed_ms": 0.0012, "average_elapsed_ms": 0.0012}
  ]
}
```

### Error responses

| Code | Meaning |
|------|---------|
| 400 | Invalid request, or shell metacharacters in arguments |
| 403 | Command is not on the allowlist |

---

## Architecture

```
Browser (HTML / CSS / JS)
        |  HTTP + JSON, loopback only (127.0.0.1)
        v
+---------------------------------------------------+
| server.c : accept() loop, SIGCHLD reaping         |
|   fork() per connection -> route handler          |
|     /api/run      -> profiler_run()               |
|     /api/trace    -> syscall_trace_run()          |
|     /api/monitor  -> /proc/<pid> reader           |
|     /api/history  -> db_history_json()            |
|     /api/stats    -> db_stats_json()              |
|                                                   |
| profiler.c                                        |
|   fork -> execvp -> pipes -> wait4                |
|   clock_gettime, rusage, poll, timeout kill       |
|                                                   |
| syscall_trace.c                                   |
|   fork -> PTRACE_TRACEME -> execvp                |
|   PTRACE_SYSCALL loop, entry/exit timing          |
|                                                   |
| db.c : SQLite (WAL)  ->  db/profiler.db           |
+---------------------------------------------------+
```

The parent process only accepts connections; each request runs in a forked child so a slow command never blocks the listening socket.

### Database schema

Table `runs`: `id`, `created_at`, `command`, `args`, `wall_ms`, `user_ms`, `system_ms`, `max_rss_kb`, `minor_faults`, `major_faults`, `voluntary_cs`, `involuntary_cs`, `exit_code`, `term_signal`, `status`, `stdout_text`, `stderr_text`.

---

## Security Model

- Listens on `127.0.0.1` only, never on a public interface.
- Only the 20 allowlisted commands can run.
- Arguments are validated token by token; `;`, `|`, `` ` ``, `$(`, `&`, `>`, `<` cause a 400.
- `system()`, `popen()`, and `sh -c` are never called.
- `..` path traversal is rejected when serving static files.
- Timeouts are bounded (100 to 60000 ms) and enforced by killing the process group.

This tool is intended for local, single-user use. Do not expose it to a network.

---

## Testing

```bash
make test
```

**`tests/smoke_test.sh`** (10 checks, runs its own server on port 18080): allowlist endpoint, `echo` and `pwd` runs, history persistence, `/api/stats`, rejection of `sh`, `/api/monitor`, syscall tracing (skipped gracefully if ptrace is restricted), timeout enforcement, static file serving.

**`tests/scenario_test.sh`** (12 scenarios): `cat`, `sort`, `wc -l`, failing `grep` (`status: failed`), `echo`, `sleep` timeout (`timed_out`), `bash` rejected (403), semicolon in args rejected (400), history and stats contents, `uname -a` mentions Linux, sequential runs accumulate in history.

### ptrace on WSL

Syscall tracing may be blocked by Yama or by an older WSL seccomp profile:

```bash
echo 0 | sudo tee /proc/sys/kernel/yama/ptrace_scope
wsl --update    # run from Windows PowerShell
```

If ptrace is unavailable, `/api/trace` returns a descriptive JSON error and every other feature keeps working.

---

## Project Structure

```
Linux_Command_Execution_Time_Profiler/
├── Makefile
├── README.md
├── .gitignore
├── include/
│   ├── db.h                # SQLite API
│   ├── http.h              # HTTP helpers
│   ├── profiler.h          # profile_result_t and profiler API
│   └── syscall_trace.h     # ptrace API
├── src/
│   ├── server.c            # socket server, router, /proc monitor
│   ├── http.c              # HTTP/1.1 read/write helpers
│   ├── db.c                # open, insert, history, stats
│   ├── profiler.c          # fork/exec/wait4/rusage/pipes
│   └── syscall_trace.c     # ptrace syscall counter and timer
├── public/
│   ├── index.html          # tabbed dashboard
│   ├── style.css           # responsive styles
│   └── app.js              # all frontend logic
├── tests/
│   ├── smoke_test.sh
│   └── scenario_test.sh
└── db/                     # profiler.db created at startup
```

---

## Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| `bind/listen: Address already in use` | Port taken | `./profiler_server 8090` |
| `Cannot initialize database` | `db/` not writable | `mkdir -p db && chmod 755 db` |
| `command is not on the allowlist` | Command not permitted | Use one of the 20 allowed commands |
| ptrace permission error | Yama scope or WSL restriction | See [ptrace on WSL](#ptrace-on-wsl) |
| Browser "Connection refused" | Server not running | Run `./profiler_server` and keep the terminal open |
| Page won't load from Windows | WSL1 has no port forwarding | Use WSL2, or `curl` from inside WSL |
| `sqlite3.h` not found | Dev package missing | `sudo apt install -y libsqlite3-dev` |

---

## Make Targets

| Target | Action |
|--------|--------|
| `make` | Build `profiler_server` |
| `make run` | Build, create `db/`, start the server |
| `make test` | Run smoke and scenario tests |
| `make clean` | Remove objects, binary, and database files |
| `make zip` | Clean and package to `../Linux_Command_Execution_Time_Profiler.zip` (needs `zip`) |
