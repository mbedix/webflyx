# Scripting Green Hills MULTI with Python for Multicore Requirements Verification

Oct 24, 2026 · @John

## Overview

The most portable way to script MULTI is to send it ordinary Debugger commands and read back what it prints. Anything you can type in the command pane — halt a core, print a variable, set a breakpoint — you can drive from Python. This guide builds a small harness that connects to a multicore target, loads an INTEGRITY monolith, stops chosen cores, and dumps variables into a log you can keep as verification evidence.

There are three ways to drive MULTI from Python. The harness here uses the third because it works on every MULTI release.

| Approach | How it runs | Best for |
| --- | --- | --- |
| Python inside MULTI | `python myscript.py args` in the command pane, or the Py pane | Interactive dumps while you debug |
| MULTI-Python API | Python objects that wrap a Debugger session, documented in *MULTI: Scripting* | Live control with return values |
| Generated command script | Python writes a MULTI command file, runs `multi -p file`, parses the log | Unattended, repeatable verification runs |

Command names here come from the MULTI 5 and MULTI 7 manuals. Option syntax and the Python API's class names change between releases, so check each command with `usage <command>` in the command pane and keep *MULTI: Scripting* (in the `manuals` folder of your install) open while you adapt the code.

## How the pieces fit

Your script never talks to the cores directly: it talks to the MULTI Debugger, which talks to a debug server, which reaches the silicon through the probe (freeze mode) or the running kernel over the network (run mode).

- **Target list.** MULTI shows the debug server, then one CPU node per core, then the executables, AddressSpaces and tasks under them. Commands act on whichever entry is selected, which is why scripts use `route` to name a core explicitly.
- **Freeze mode** (Green Hills Probe) stops the cores themselves. Use it to stop a core and read memory with nothing else moving.
- **Run mode** (rtserv2) stops individual tasks while the INTEGRITY kernel keeps scheduling everything else.
- **Synchronous run control**, when enabled, halts every core together as soon as any one stops.

&#91;embedded content: debug path · harness to cores\]

Core 1 is shown halted on its own, which requires synchronous run control to be off for it; with it on, all four cores would stop together.

## Environment setup

Set up a named connection once in the GUI; after that, everything can be launched from a script.

1. **Find your install.** Note the MULTI directory (for example `C:/ghs/multi_7xx`). The PDFs you need are in its `manuals` folder: *MULTI: Debugging*, *MULTI: Debugging Command Reference* and *MULTI: Scripting*.
2. **Save a Connection Method.** In the Connection Organizer, create and test a connection to your board: freeze mode through the probe, run mode through rtserv2, or both. Give it a plain name such as `Board1_Freeze`; scripts refer to it by that name.
3. **Confirm the command line works.** From a terminal run `multi -connect=Board1_Freeze path/to/monolith`. Other options you will use: `-p file` runs a command script at startup, `-norc` skips personal startup scripts (use it for repeatable runs), and `-nodisplay` runs without a GUI on Linux hosts.
4. **Check MULTI's Python.** In the Py pane, run `import sys; print(sys.version)`. MULTI ships its own interpreter for in-MULTI scripts; the external harness can use your normal Python.
5. **Learn three helper commands.** `usage <command>` prints syntax, `help <command>` opens its manual page, and `components` lists every core and alias MULTI knows about.

## One script, end to end

One command line does the whole job: Python writes a MULTI command script, MULTI runs it top to bottom, and Python reads the log afterward.

```
python test_req042.py
  1. writes  runs/<stamp>_REQ-042/session.rc
  2. runs    multi -norc -p session.rc C:/work/build/monolith
  3. reads   runs/<stamp>_REQ-042/multi.log  ->  verdicts.csv
```

Below is a complete `session.rc` of the kind the harness generates. Every line is an ordinary Debugger command, so you can also run it by hand with `multi -norc -p session.rc C:/work/build/monolith` while you develop a test.

```
// ---- capture and settings ----
output -multi C:/work/runs/REQ-042/multi.log    // everything printed goes to the log
configure ProcRelativeLines off                 // #N line numbers mean file lines

// ---- connect and load ----
connect Board1_Freeze                           // saved Connection Method
// <load step: prepare_target, setup script or flash, exactly as your lab procedure does it>
components                                      // record core names in the log

// ---- breakpoints by function name ----
b %req42 mode_switch_done                       // stop when the mode switch completes
b %boot ctrl_init {mprintf("@@BOOT@@ reached=1\n"); resume}   // log it, keep running

// ---- run to the event ----
c
wait -stop -time 60000                          // give up after 60 s instead of hanging

// ---- dump values ----
mprintf("@@REQ-042@@ state=%d errs=%d\n", g_ctrl.state, g_ctrl.err_count)
route core0 {p $pc}                             // where each core was
route core1 {p $pc}

// ---- clean up ----
d %req42
d %boot
output -multi -normal
quitall
```

The log then holds the `components` listing, the breakpoint hit messages, the tagged `@@REQ-042@@` line and each core's PC, which is everything the Python side needs to decide pass or fail. The harness builds this same file from Python calls (see "A reusable harness"), so requirement IDs, expected values and timeouts can come from your test data.

## Commands, waiting and output

Every script repeats three steps: send a command, wait for the target to reach a state, and capture what MULTI printed.

**Send.** Commands are the same text you would type in the command pane. Chain several with `;`, or group them in `{ }` command lists, which breakpoints use to run actions when they hit. C and C++ style comments are stripped, so you can annotate scripts.

**Wait.** Run-control commands return before the target actually stops, so follow them with `wait`:

| Command | Blocks until |
| --- | --- |
| `wait -stop` | The selected process halts (the default for bare `wait`) |
| `wait -stop -time 30000` | It halts or 30 s pass; always use a limit in automation |
| `wait -multiStatus Stopped` | MULTI reports the named status |
| `wait -time 500` | 500 ms pass |

**Capture.** `output -multi run.log` copies everything the command pane prints into a file; `output -multi -normal` turns that off. For values you plan to parse, print with `mprintf`, which takes a C-style format string, so each log line looks exactly the way your parser expects:

```
mprintf("@@REQ-042@@ state=%d err=%u\n", g_ctrl.state, g_ctrl.err_count)
```

**The in-MULTI API.** If you want live return values instead of log parsing, the MULTI-Python API exposes a Debugger object with a method that runs a command and returns its text. Recent releases use a `RunCmd`-style call; confirm the class and method names in *MULTI: Scripting*. Keep that call behind one function so the rest of your code never touches the API directly:

```python
def run(dbg, text, log):
    out = dbg.RunCmd(text)   # the only MULTI-API call; rename if your release differs
    log.write(f"> {text}\n{out}\n")
    return out
```

## Connecting and loading the monolith

An INTEGRITY monolith is one image holding the kernel and every AddressSpace, so you load it once and then debug its pieces from the target list.

1. **Connect.** Use `connect Board1_Freeze` in a script, or `-connect=` on the command line. In freeze mode the target list then shows one CPU node per core under the debug server.
2. **Load.** Download or flash the image the same way your lab procedure does it: the Prepare Target dialog, `prepare_target`, a `.mbs` setup script, or flash. Run `usage prepare_target` and record the exact command your setup needs, so the script matches the manual procedure step for step.
3. **Find the cores.** Run `components` and save its output. The names it prints are what you pass to `route` later; add short aliases if they are long.
4. **Pick the right context.** In a monolith, each virtual AddressSpace has its own symbols and addresses. Select the AddressSpace or task in the target list, or route to it, before printing its variables; printing from the kernel context can resolve a different symbol or fail.
5. **Wait for the run-mode partner.** If you also use rtserv2, MULTI can bring up the run-mode connection automatically after the kernel boots. Until it exists, monolith tasks show the status *Waiting*; in scripts, `wait -runmode_partner` blocks until it is ready.

## Stopping specific cores

Decide first whether the requirement needs a **core** stopped or a **task** stopped; on an SMP INTEGRITY system they are different things.

| You need | Mode | What stops | Command pattern |
| --- | --- | --- | --- |
| Whole system frozen at one instant | Freeze, synchronous on | Every core | A breakpoint, or `halt` on any core |
| One core frozen, others running | Freeze, synchronous off | That core only | `route <core> {halt}` |
| One task frozen, kernel and other tasks running | Run mode (rtserv2) | That task only | Select or route to the task, then `halt` |

**Synchronous run control.** In synchronous mode MULTI halts all cores as a unit when any one of them hits a breakpoint or is halted; the others show the status *Paused by halted\_core*. That is the safe choice for reading shared data, because nothing on another core changes it while you read. To stop a single core, synchronous debugging must be off for the cores involved; see "Synchronous Debugging on a Multi-Core Target" in *MULTI: Debugging* for where the setting lives in your release.

**Routing.** Commands act on the entry selected in the target list. In scripts, `route` aims a command at one core without changing the selection:

```
components                        // list core names and aliases
route core1 {halt}                // stop core 1 only (synchronous off)
route core1 {wait -stop -time 5000}
route core1 {p $pc}               // where it stopped
route core1 {c}                   // resume core 1
```

`core1` stands for whatever name `components` printed. Run `usage route` once to confirm the syntax in your release.

**Cautions for single-core halts:**

- Tasks move between cores unless they have affinity, so a core halt freezes whatever happened to be scheduled there, not a particular task.
- A running core may spin on a lock the halted core holds, or a watchdog may fire. Keep single-core halts short and disable watchdogs in the setup script for test builds.
- Values another core is still writing can change between two prints. Read related variables in one `mprintf` and record that the snapshot is not atomic.

## Dumping variables

Print what the requirement names, in a format your parser can read, from the context that owns the variable.

| Goal | Command |
| --- | --- |
| Value of a global or struct | `p g_ctrl` or `print g_ctrl.state` |
| Hexadecimal | `p/x g_ctrl.flags` |
| Array element | `p g_buf[3]`; loop over indexes in Python for a range |
| Pointer target | `p *g_msg_ptr` |
| Register on one core | `route core1 {p $pc}` |
| Parse-friendly line | `mprintf("@@TAG@@ a=%d b=%x\n", a, b)` |
| Raw memory | `memread`, `examine`, or `memdump` to a file (check `usage` for each) |
| A local in the stopped frame | `p name` while stopped inside that function |

Tips:

- **Halt before reading.** Some targets allow memory reads while running, but the values are not mutually consistent. Halt or hit a breakpoint, read, then resume.
- **Prefer globals.** Optimized code keeps locals in registers or removes them. If a requirement depends on a value, make sure it lives in a global or static the build keeps.
- **Qualify by file when names collide.** Address expressions accept a file qualifier such as `"ctrl.c"#name`; see "Name Resolution" in the command reference and "Expression Scope" in *MULTI: Debugging* for file statics.
- **Tag every line.** A tag such as `@@REQ-042@@` ties each value to its requirement and turns parsing into one regular expression.

## Breakpoints by function name

Function names are the right default for scripted breakpoints. They survive edits that shift line numbers, and they read clearly in a test procedure. Line numbers remain available when you need a spot inside a function.

### Forms that take a function name

| Form | Where it stops | Example |
| --- | --- | --- |
| `b func` | After the prologue, so arguments and locals read correctly | `b mode_switch_done` |
| `bi func` | The first instruction of the function | `bi isr_timer0` |
| `bx func` | The exit point every return passes through | `bx compute_limits` |
| `b "file.c"#func` | That function in that file only (statics, duplicate names) | `b "ctrl.c"#step` |
| `b "file.c"#func#N` | Line N, which must lie inside func (file line, with ProcRelativeLines off) | `b "ctrl.c"#step#215` |
| `b ns::Class::method(args)` | One C++ overload | `b ctl::Loop::step(int)` |
| `b %label func` | Same as `b func`, with a name you can delete later | `b %req42 mode_switch_done`, then `d %req42` |
| `b @N func` | From the Nth hit onward | `b @50 ctrl_step` |
| `bif func {cond}` | Only when the condition is true | `bif ctrl_step {g_ctrl.state == 3}` |
| `b func {cmds}` | Runs commands on each hit; `resume` keeps going | `b ctrl_step {mprintf("@@T@@ s=%d\n", g_ctrl.state); resume}` |
| `bt func` | Prints entry and exit, with the return value | `bt compute_limits` |
| `hardbrk execute func` | Hardware breakpoint on the function's address | `hardbrk execute ctrl_step` |
| `hardbrk write -- var` | When a variable is written | `hardbrk write -- g_ctrl.state` |

### Rules for headless runs

- **Qualify any name that could match more than one function.** An ambiguous name opens a selection dialog, which stalls an unattended run. Use `"file.c"#name` for C statics and the full signature for C++ overloads.
- **Label every breakpoint the script sets,** and delete by label (`d %req42`), so cleanup never removes breakpoints someone else set.
- **Set `configure ProcRelativeLines off` at the top.** With it off, `#215` means file line 215 everywhere, including inside `func#215`. With it on, the same number counts from the start of the function.
- **Budget hardware breakpoints.** Most cores have fewer than four. Spend them on data writes, code in flash, and the per-core cases below.
- **Version breakpoint sets** with `bpsave` and `bpload` alongside the test procedure.

### Breakpoints on a specific core

How a breakpoint behaves depends on whether the cores share code.

| Setup | What a software breakpoint does | How to stop on one core only |
| --- | --- | --- |
| AMP: each core runs its own image | Lives in that core's image only | `route core1 {b func}` |
| SMP: cores share one kernel image | It is a patch in shared code, so any core that runs it stops | A hardware breakpoint through that core's debug unit, or a condition on the core ID |

```
// AMP: core 1 has its own image and symbols
route core1 {b %c1_step ctrl_step}

// SMP: hardware execute breakpoint placed through core 1's debug unit
route core1 {hardbrk execute ctrl_step}

// SMP: software breakpoint that stops only when core 1 runs it
bif %c1_only ctrl_step {<core_id_expr> == 1}
```

`<core_id_expr>` is your architecture's core-ID register as MULTI names it in the Register View, or a per-core variable your software keeps. The condition is evaluated whenever any core reaches the breakpoint, so every core still pauses briefly; use the hardware form when timing matters. Confirm in a dry run that the hardware form fires on core 1 only, since that depends on your debug server. With synchronous run control on, a hit on core 1 still stops all cores.

### Breakpoints in a monolith

In an INTEGRITY monolith the same function name can exist in the kernel and in several AddressSpaces, so set each breakpoint from the context that owns the function.

| Where the function lives | Mode | Example |
| --- | --- | --- |
| Kernel | Freeze | Kernel entry selected, then `b %k1 kernel_func` |
| Any task in an AddressSpace | Run (rtserv2) | Attach to a task there, then `sb at %as1 ctrl_step` |
| One specific task | Run (rtserv2) | Attach to that task, then `sb tt %t1 ctrl_step` |
| Any task, hardware breakpoint | Run, INTEGRITY 10 or later | `hardbrk execute global -- ctrl_step` |
| Same name in two places | Either | Qualify by file: `b "kernel_tbl.c"#init_tables` vs `sb at "app_tbl.c"#init_tables` |

In `sb`, the first letter picks which tasks trigger it (`a` = any task in the current AddressSpace, `t` = the current task). The second picks what stops; INTEGRITY supports only `t`, which stops the task that hit it. `hardbrk` options must appear in the documented order, which is why `global` comes after `execute`.

To get into an AddressSpace's context from a script, wait for its task to exist, then attach:

```
wait -time 60000 -taskname "ctrl_task"            // task appears after boot
attach -addressSpace CtrlAS ctrl_task -nohalt     // becomes the current entry, keeps running
sb at %as1 ctrl_step                              // any task in CtrlAS that calls ctrl_step
wait -stop -time 30000
mprintf("@@REQ-077@@ mode=%d\n", g_mode)            // CtrlAS's own g_mode
```

`attach` is marked GUI only, so this works when MULTI opens its window (the normal case on Windows) but not with `-nodisplay`. In freeze mode, the target list shows the kernel (OSA master process) and OSA tasks under each CPU. If `components` lists them, `route` to them. Otherwise use run mode for AddressSpace-specific breakpoints, or qualify by file.

## A reusable harness

The harness writes a MULTI command script, runs it, and parses tagged lines from the log, so a test is Python code that queues commands and checks values. It needs only the standard library and the `multi` executable.

```python
"""multi_harness.py - script Green Hills MULTI for requirement checks."""
import csv, datetime, pathlib, re, subprocess

MULTI_EXE = "C:/ghs/multi_7xx/multi.exe"   # your install
CONNECT = "Board1_Freeze"                   # saved Connection Method
PROGRAM = "C:/work/build/monolith"          # INTEGRITY monolith image
NL = r"\n"                                  # literal backslash-n for mprintf


class MultiSession:
    def __init__(self, name, root="runs"):
        stamp = datetime.datetime.now().strftime("%Y%m%d_%H%M%S")
        self.dir = pathlib.Path(root) / f"{stamp}_{name}"
        self.dir.mkdir(parents=True)
        self.log = self.dir / "multi.log"
        self.lines = ["configure ProcRelativeLines off", f"connect {CONNECT}"]

    def cmd(self, text):
        self.lines.append(text)
        return self

    def on(self, core, text):               # aim one command at one core
        return self.cmd(f"route {core} {{{text}}}")

    def wait_stop(self, ms=30000, core=None):
        w = f"wait -stop -time {ms}"
        return self.on(core, w) if core else self.cmd(w)

    def dump(self, tag, **exprs):           # name=expression pairs, printed as %d
        fmt = " ".join(f"{k}=%d" for k in exprs)
        args = ", ".join(exprs.values())
        return self.cmd(f'mprintf("@@{tag}@@ {fmt}{NL}", {args})')

    def run(self, timeout_s=900):
        script = self.dir / "session.rc"
        body = [f"output -multi {self.log.as_posix()}", *self.lines, "quitall"]
        script.write_text("\n".join(body) + "\n")
        subprocess.run([MULTI_EXE, "-norc", "-p", str(script), PROGRAM],
                       timeout=timeout_s, check=False)
        return Results(self.log.read_text(errors="replace"), self.dir)


class Results:
    def __init__(self, text, folder):
        self.text, self.dir, self.rows = text, folder, []

    def values(self, tag):                  # last tagged line -> {name: int}
        hits = re.findall(rf"@@{re.escape(tag)}@@ (.*)", self.text)
        if not hits:
            return None
        return {k: int(v, 0) for k, v in re.findall(r"(\w+)=(\S+)", hits[-1])}

    def check(self, req, tag, name, expected):
        got = (self.values(tag) or {}).get(name)
        verdict = "PASS" if got == expected else "FAIL"
        self.rows.append([req, name, expected, got, verdict])
        return verdict == "PASS"

    def report(self):
        with open(self.dir / "verdicts.csv", "w", newline="") as f:
            w = csv.writer(f)
            w.writerow(["requirement", "variable", "expected", "actual", "verdict"])
            w.writerows(self.rows)
```

`dump` prints every value as `%d`; add a format argument if you need hex, unsigned or floating-point values.

**Sample test: whole system stopped at an event.** The requirement says a state must be 3 with no errors once a mode switch completes:

```python
from multi_harness import MultiSession

s = MultiSession("REQ-042")
# s.cmd("<your load / prepare_target command>")
s.cmd("components")                       # record core names in the log
s.cmd("b %req42 mode_switch_done")        # the event the requirement names
s.cmd("c").wait_stop(ms=60000)            # run until it fires; all cores stop if synchronous
s.dump("REQ-042", state="g_ctrl.state", errs="g_ctrl.err_count")
s.on("core1", "p $pc")                    # extra context for the reviewer
s.cmd("d %req42")

r = s.run()
r.check("REQ-042", "REQ-042", "state", 3)
r.check("REQ-042", "REQ-042", "errs", 0)
r.report()
```

**Variant: one core stopped while the rest run** (synchronous off for that core):

```python
s.cmd("c").cmd("wait -time 2000")         # let the system run for 2 s
s.on("core1", "halt").wait_stop(ms=5000, core="core1")
s.dump("REQ-051", beat="g_core1_heartbeat")
s.on("core1", "c")
```

Each run folder ends up holding the generated command script, MULTI's full log and a verdict CSV: the three things a reviewer needs to reproduce the result. If a command returns before the target is ready (for example right after `connect`), add a `wait` after it.

## Recipes for cores and monoliths

Each recipe gives the MULTI commands, then the same steps as harness calls. Replace `core0`–`core2`, `CtrlAS`, `ctrl_task` and the variable names with yours, and take the real core names from one `components` run. Register names such as `$sp` follow your architecture's Register View.

Resuming depends on the run-control mode. With synchronous run control on, `c` resumes every core. With it off, `c` resumes only the selected entry, so resume each core you need running with `route coreN {c}`.

### 1. Freeze the whole system at a function

Freeze mode, synchronous on. Every core stops at the instant the breakpoint hits.

```
b %r1 sched_switch_done
c
wait -stop -time 30000
mprintf("@@R1@@ ticks=%d ready=%d\n", g_ticks, g_ready_count)
route core0 {p $pc}
route core1 {p $pc}
d %r1
```

```python
s = MultiSession("R1")
s.cmd("b %r1 sched_switch_done").cmd("c").wait_stop(ms=30000)
s.dump("R1", ticks="g_ticks", ready="g_ready_count")
s.on("core0", "p $pc").on("core1", "p $pc").cmd("d %r1")
r = s.run()
r.check("REQ-R1", "R1", "ready", 4)
r.report()
```

### 2. Stop when core 1 runs a shared function

Freeze mode, SMP. A hardware breakpoint goes through core 1's debug unit, so the other cores can run the same code without stopping. With synchronous off, only core 1 halts.

```
route core1 {hardbrk execute ctrl_step}
c
route core1 {wait -stop -time 30000}
route core1 {p $pc}
mprintf("@@R2@@ state=%d\n", g_ctrl.state)
route core1 {hardbrk delete=*}
```

```python
s = MultiSession("R2")
s.on("core1", "hardbrk execute ctrl_step").cmd("c")
s.wait_stop(ms=30000, core="core1")
s.on("core1", "p $pc")
s.dump("R2", state="g_ctrl.state")
s.on("core1", "hardbrk delete=*")
```

### 3. Halt one core on demand and dump its state

Freeze mode, synchronous off for core 2. Use this when the requirement is about what one core holds at an arbitrary moment while the others keep working.

```
c                                    // system running
wait -time 2000                      // let it run for 2 s
route core2 {halt}
route core2 {wait -stop -time 5000}
route core2 {p $pc}
route core2 {p $sp}
mprintf("@@R3@@ beat=%d\n", g_heartbeat[2])
route core2 {c}
```

```python
s = MultiSession("R3")
s.cmd("c").cmd("wait -time 2000")
s.on("core2", "halt").wait_stop(ms=5000, core="core2")
s.on("core2", "p $pc").on("core2", "p $sp")
s.dump("R3", beat="g_heartbeat[2]")
s.on("core2", "c")
```

### 4. Stop one task in an AddressSpace while everything else runs

Run mode. Start from a run-mode Connection Method, or from a freeze-mode connection followed by `wait -runmode_partner`. Only `ctrl_task` stops; the kernel and every other task keep running.

```
wait -time 60000 -taskname "ctrl_task"
attach -addressSpace CtrlAS ctrl_task -nohalt
sb tt %r4 ctrl_step
wait -stop -time 30000
mprintf("@@R4@@ mode=%d cmd=%d\n", g_mode, g_last_cmd)
d %r4
c
```

```python
s = MultiSession("R4")
s.cmd('wait -time 60000 -taskname "ctrl_task"')
s.cmd("attach -addressSpace CtrlAS ctrl_task -nohalt")
s.cmd("sb tt %r4 ctrl_step").wait_stop(ms=30000)
s.dump("R4", mode="g_mode", cmd="g_last_cmd")
s.cmd("d %r4").cmd("c")
```

### 5. Log every call in an AddressSpace without stopping

Run mode. Each hit prints a tagged line and resumes at once. That suits checks over many cycles, such as a value never exceeding a limit.

```
attach -addressSpace CtrlAS ctrl_task -nohalt
sb at %r5 ctrl_step {mprintf("@@R5@@ state=%d\n", g_ctrl.state); resume}
wait -time 10000                     // collect for 10 s
d %r5
```

```python
import re

s = MultiSession("R5")
s.cmd("attach -addressSpace CtrlAS ctrl_task -nohalt")
s.cmd('sb at %r5 ctrl_step {mprintf("@@R5@@ state=%d' + NL + '", g_ctrl.state); resume}')
s.cmd("wait -time 10000").cmd("d %r5")
r = s.run()
states = [int(v) for v in re.findall(r"@@R5@@ state=(-?\d+)", r.text)]
assert states and max(states) <= 7, f"state exceeded limit: {max(states, default=None)}"
```

`NL` is the constant from `multi_harness.py`; import it alongside `MultiSession`.

### 6. Same function name in the kernel and an AddressSpace

Pin each breakpoint by file so neither resolves to the wrong copy, and so no selection dialog opens.

```
b %k6 "kernel_tbl.c"#init_tables                  // kernel copy
attach -addressSpace CtrlAS ctrl_task -nohalt
sb at %a6 "app_tbl.c"#init_tables                 // CtrlAS copy
```

```python
s.cmd('b %k6 "kernel_tbl.c"#init_tables')
s.cmd("attach -addressSpace CtrlAS ctrl_task -nohalt")
s.cmd('sb at %a6 "app_tbl.c"#init_tables')
```

## Pitfalls and checks

- **Syntax drift between releases.** Run `usage` once for every command in your scripts on the MULTI version your lab uses, and record that version in each run's log.
- **Hung runs.** Give every `wait` a `-time` limit and the Python `subprocess.run` a timeout, so a target that never stops fails the test instead of stalling the batch.
- **Synchronous surprises.** If stopping "only core 1" keeps stopping everything, synchronous debugging is still on for that core.
- **Wrong symbol context.** In a monolith the same name can exist in several AddressSpaces. Route to or select the owning AddressSpace before printing.
- **Optimized-out values.** Build verification images with debug information, and read requirement values from globals the optimizer keeps.
- **Probe reads versus cached data.** On some cores, memory read through the probe can differ from what the core sees in its cache. Check how your debug server handles caches before trusting a value in shared memory.
- **Intrusiveness.** Every halt changes timing. Requirements about deadlines or rates need trace or tracepoints, not breakpoints.
- **Evidence rules.** If these runs count as formal verification, your program's process may require the harness, the generated scripts and the tool version to be under configuration management, and the harness itself to be validated. Ask your verification lead before the first formal run.

The manuals in your own install are the authority; the sources below are public copies of older releases (MULTI 5 and MULTI 7) used to check command names.

## Sources

- [MULTI: Debugging Command Reference, MULTI 5 (public copy)](https://dn720701.ca.archive.org/0/items/NintendoDeveloperDocs/NintendoDeveloperDocs/WiiU_Docs/debug_cmd.pdf): syntax for `wait`, `output`, `new -alias`, `b`, `bif`, `hardbrk`, `sb`, and the command index (`route`, `components`, `python`, `setsync`, `memdump`)
- [MULTI: Debugging, MULTI 7 (public copy)](https://studylib.net/doc/27446691/debug): target list and statuses, Py pane, `-connect`, `-p`, `-nodisplay`, synchronous debugging, INTEGRITY monolith debugging
- [Green Hills MULTI IDE product page](https://www.ghs.com/products/MULTI_IDE.html): synchronous run control across cores
- [Green Hills training courses](https://www.ghs.com/training.html): MULTI-Python integration and scripted breakpoints
- [Green Hills automotive platform page](https://www.ghs.com/products/auto_powertrain.html): external control of MULTI through Python
- [Infineon community example](https://community.infineon.com/t5/Code-Examples/How-to-use-GHS-debugger-to-take-work-flash-data-dump-in-TRAVEO-T2G/td-p/1063547): running a Python script with arguments from the MULTI command pane
