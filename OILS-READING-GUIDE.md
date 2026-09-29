# Oils Reading Curriculum — for a Systems Programmer New to Shells

This is an ordered reading path through the Oils repo (`oils-for-unix/oils`),
tuned for someone who already knows OS/networking/infrastructure but has never
gone deep on shells.

Oils is two shells sharing one runtime:

- **OSH**: a cleaned-up, POSIX/bash-compatible shell. Runs your existing
  `#!/bin/sh` and `#!/bin/bash` scripts.
- **YSH**: a new shell language (Python/JS-flavored) that manipulates the same
  interpreter state more sanely.

The repo is [`bin/osh`](bin/osh) + [`bin/ysh`](bin/ysh) as dev entry points, but the deployed binary
is C++ generated from typed Python (see Phase 8).

Suggested way to use this guide: at each phase, (1) skim the doc, (2) read the
listed files in order, (3) do the 10-minute experiment. Don't try to read
top-to-bottom; shells are mutually recursive parsers + evaluators, so you want
to do 2–3 passes: skeleton first, details later.

---

## 0. Orientation: what is a shell, really?

If you only take one mental model from this guide, take this:

> A shell is a **REPL over an OS process API**, with a weird old programming
> language bolted on. Roughly: **lexer → parsers → evaluators → fork/exec**,
> plus persistent **interpreter state** (variables, functions, options, traps,
> jobs, file descriptors).

Your OS background already covers ~40% of shell semantics: `fork()`, `execve()`,
`waitpid()`, pipes, `dup2()`, signals, exit statuses, environment variables.
What will be new is:

1. **Word evaluation / expansion** — the most shell-specific and most
   bug-prone part. `echo $x`, `*.py`, `~bob`, `$((1+2))`, `$(hostname)` all
   happen *after* parsing, at runtime, per-word.
2. **Dynamic parsing** — `eval`, `source`, `trap`, aliases, `$PS1` re-enter the
   parser at runtime.
3. **POSIX compatibility baggage** — lots of behavior exists only because
   bash/POSIX does it that way. Oils documents this explicitly; see
   [`doc/known-differences.md`](doc/known-differences.md) and [`doc/warts.md`](doc/warts.md).

### Read first (30 min, docs only)

| Order | File | Why |
|---|---|---|
| 0.1 | [`README.md`](README.md) | Dev build vs release tarball distinction. You want [`bin/osh`](bin/osh) / [`bin/ysh`](bin/ysh) (Python dev build). |
| 0.2 | [`doc/repo-overview.md`](doc/repo-overview.md) | Map of [`osh/`](osh/), [`ysh/`](ysh/), [`core/`](core/), [`frontend/`](frontend/), [`builtin/`](builtin/), [`mycpp/`](mycpp/), [`spec/`](spec/), etc. |
| 0.3 | [`doc/interpreter-state.md`](doc/interpreter-state.md) | The semantic core: stack memory, no heap/pointers in OSH, two namespaces (vars vs functions), dynamic scope, values-vs-locations. |
| 0.4 | [`doc/process-model.md`](doc/process-model.md) | When does a shell `fork()`? Pipelines, command sub `$(...)`, process sub `<(...)`, `&`, explicit `( subshell )`, plus “subshells by surprise” (`echo hi \| read x`). You’ll understand the bash-vs-zsh example immediately. |
| 0.5 | [`doc/simple-word-eval.md`](doc/simple-word-eval.md) | The single best doc for “what’s wrong with shell and what YSH fixes.” No splitting/globbing/elision by default. |

### Do this now

```sh
bin/osh -c 'echo hi | read x; echo x=$x'   # OSH behaves like zsh here
bash -c 'echo hi | read x; echo x=$x'      # compare: bash prints x=

bin/osh -n -c 'echo ${x:-default}$((1+2))' # -n prints the AST, no execution
bin/osh --tool syntax-tree -c 'echo hi'    # same idea, one-pass parse tool
bin/ysh -c 'var x = {"a": 42}; echo $x["a"]'
```

If `-n` output makes sense as a tree, you’re ready for Phase 1.

---

## What’s notable about how Oils differs from bash/dash/zsh

Come back to this list as you read; each item maps to a phase.

1. **Written in typed Python, translated to C++ with [`mycpp/`](mycpp/).** Not C like
   bash/dash, not hand-rolled memory management. The deployed `oils-for-unix`
   binary has no Python dependency. Consequence you’ll see everywhere: no
   exceptions-as-control-flow in hot paths, no arbitrary Python idioms, explicit
   types, `mylib.PYTHON` branches, manual GC points (`mylib.MaybeCollect()`).
   See [`mycpp/README.md`](mycpp/README.md).
2. **ASDL schemas define the AST and runtime values.** [`frontend/syntax.asdl`](frontend/syntax.asdl)
   (commands/words/expressions), [`core/value.asdl`](core/value.asdl), [`core/runtime.asdl`](core/runtime.asdl),
   [`frontend/types.asdl`](frontend/types.asdl). Borrowed from CPython’s ASDL, heavily refactored.
   Most shells have implicit C structs; Oils has a readable grammar for its
   data structures, with generated Python + C++ code. Start any deep dive at
   the `.asdl` file, not the `.py` file.
3. **Multiple small parsers, not one yacc grammar.** OSH needs a recursive-descent
   command parser ([`osh/cmd_parse.py`](osh/cmd_parse.py)), a word parser ([`osh/word_parse.py`](osh/word_parse.py)),
   arithmetic ([`osh/tdop.py`](osh/tdop.py), [`osh/arith_parse.py`](osh/arith_parse.py)), boolean/test ([`osh/bool_parse.py`](osh/bool_parse.py)),
   plus a pgen2-based YSH expression parser ([`ysh/expr_parse.py`](ysh/expr_parse.py),
   [`ysh/grammar_gen.py`](ysh/grammar_gen.py)). Shell *cannot* be parsed like C; see
   [`doc/parser-architecture.md`](doc/parser-architecture.md) and the linked “How to Parse Shell Like a
   Programming Language” post.
4. **Regex-based modal lexer compiled with `re2c`.** [`frontend/lexer_def.py`](frontend/lexer_def.py)
   defines lexer modes; [`pyext/fastlex.c`](pyext/fastlex.c) + [`frontend/match.py`](frontend/match.py) execute them.
   Oils deliberately avoids char-by-char hand lexing. This is unusual for shells
   and worth studying if you like lexing.
5. **Lossless parsing invariant.** `osh --tool lossless-cat` + [`test/lossless.sh`](test/lossless.sh):
   tokens must reassemble to the original file. That’s the foundation for
   `--tool fmt` and `--tool ysh-ify`. Most shells throw whitespace/comments away.
6. **Spec tests as executable spec.** [`spec/*.test.sh`](spec/) run against bash, dash,
   mksh, zsh *and* OSH via [`test/sh_spec.py`](test/sh_spec.py). Compatibility is measured, not
   asserted. Contributing a failing spec test alone is a valid contribution.
7. **Simple word evaluation as opt-in sanity.** `shopt -s simple_word_eval`
   (default in YSH) collapses POSIX’s multi-stage expansion into single-step
   evaluation. Read [`doc/simple-word-eval.md`](doc/simple-word-eval.md) before [`osh/word_eval.py`](osh/word_eval.py) or you’ll
   drown.
8. **Explicit process/executor split.** [`core/executor.py`](core/executor.py) (`ShellExecutor` vs
   `PureExecutor`), [`core/process.py`](core/process.py) (`FdState`, `Waiter`, `JobControl`,
   `ExternalProgram`), [`osh/cmd_eval.py`](osh/cmd_eval.py). Fork-optimization (`noforklast`) is
   explicit and documented in [`doc/process-model.md`](doc/process-model.md).

---

## Phase 1 — The skeleton: entry point → main loop (1 hour)

Goal: be able to trace [`bin/osh`](bin/osh) `-c 'echo hi'` end to end without getting lost.

Read in this order:

1. [`bin/oils_for_unix.py`](bin/oils_for_unix.py) (~263 lines) — tiny `main()`. Locale setup, flag
   parsing, dispatch to [`core/shell.py:Main()`](core/shell.py). Note the busybox-style
   `argv[0]` behavior.
2. [`core/shell.py:Main()`](core/shell.py) (~1300 lines but skim it) — **the assembler**. This is
   where every circular dependency is wired: `Mem`, `ParseContext`, splitters,
   globbers, `CommandEvaluator`, `WordEvaluator`, `ArithEvaluator`,
   `ExprEvaluator`, builtins table, methods table, `TrapState`, `FdState`,
   `Waiter`, `Tracer`. Don’t read every initializer; read the section headers
   and the final dispatch: `--eval` → interactive (`main_loop.Interactive`) →
   `--tool` → batch (`main_loop.Batch`). This file answers “where does X live?”
3. [`core/main_loop.py`](core/main_loop.py) (~487 lines) — the read-parse-execute loop. Compare
   `Batch()` (scripts, `-c`, `source`/`eval`) vs `Interactive()` (prompt,
   history, `ParseInteractiveLine`, traps, job polling) vs `ParseWholeFile()`
   (`-n`, `--tool`). Note `arena.DiscardLines()` + `mylib.MaybeCollect()`: manual
   memory discipline for the C++ target.
4. [`frontend/parse_lib.py`](frontend/parse_lib.py) — `ParseContext`, the factory for all parsers.
   This is the key to “mutually recursive parsers share an arena, aliases, and
   options.” Skim the `Make*` methods; you’ll revisit them in Phase 3.

Focus: dependency injection done by hand. `cmd_ev` needs `word_ev` needs
`arith_ev` needs `cmd_ev` (command sub inside arithmetic inside words inside
commands). [`core/vm.py:InitCircularDeps`](core/vm.py) breaks the cycle. That pattern repeats
everywhere.

Experiment:

```sh
bin/osh --debug-file /tmp/osh.log -c 'echo hi'
bin/osh -n --ast-format text -c 'x=1; echo $x'
```

---

## Phase 2 — Lexing: characters → tokens (1–2 hours)

Goal: understand why shell needs a *modal* lexer.

1. [`frontend/lexer_def.py`](frontend/lexer_def.py) (1132 lines — skim, don’t memorize) — rules grouped
   by `lex_mode_e` (from [`frontend/types.asdl`](frontend/types.asdl)). Notice separate modes for
   command position, word parts, arithmetic, `[[ ]]`, `printf`, history, etc.
   Read the header comment about NUL-terminated lines and `\0` sentinel: that’s
   for the C++/re2c translation, and Python’s regex engine doesn’t need it.
2. [`frontend/lexer.py`](frontend/lexer.py) + [`frontend/match.py`](frontend/match.py) + [`frontend/reader.py`](frontend/reader.py) — `Reader`
   feeds lines, `LineLexer`/`Lexer` apply the mode’s regexes, `MaybeUnreadOne()`
   handles `ungetc`-like cases (see [`doc/parser-architecture.md`](doc/parser-architecture.md) § Lexer Unread).
3. [`frontend/id_kind_def.py`](frontend/id_kind_def.py) + generated [`_devbuild/gen/id_kind_asdl.py`](_devbuild/gen/id_kind_asdl.py) —
   the `Id` token type. Every token has an `Id`; every AST node carries
   locations. Grep for `Id.VSub_Name` or `Id.Lit_Chars` to see the vocabulary.

Focus: **modes switch constantly**. `echo "a $x"` lexes `echo` in command mode,
then enters double-quote mode, then `$x` enters var-sub mode, then back. The
parser drives the lexer, not the other way around. That’s the opposite of a
typical language and the reason `parse-shell`-style posts exist.

Experiment:

```sh
bin/osh --tool tokens -c 'echo "hi $USER" $(hostname)'
```

---

## Phase 3 — Parsing: tokens → AST (the biggest phase, 3–5 hours)

Goal: see how four parsers cooperate. Read the ASDL first, then the parsers.

1. [`frontend/syntax.asdl`](frontend/syntax.asdl) (721 lines) — **read this fully**. It’s the grammar of
   everything OSH/YSH can say: `command`, `CompoundWord`, `word_part`,
   `arith_expr`, `bool_expr`, YSH `expr`. If you skim one file in the whole repo,
   make it this one.
2. [`osh/cmd_parse.py`](osh/cmd_parse.py) (2889 lines) — recursive-descent command parser:
   `ParseLogicalLine`, simple commands, pipelines, `&&`/`||`, subshells, braces,
   `for`/`while`/`if`/`case`, functions, redirects, here-docs
   (`VirtualLineReader`), aliases (`SnipCodeString`). Pair with
   [`doc/parser-architecture.md`](doc/parser-architecture.md) § Re-parsing: here-docs, array l-values,
   backticks, and aliases each re-read text or tokens. That doc explains why
   error locations and `--tool fmt` are hard.
3. [`osh/word_parse.py`](osh/word_parse.py) (2378 lines) — word parser: `$x`, `${x:-default}`,
   `$((...))`, `$(...)`, backticks, `$''`, glob/brace/tilde detection. This is
   where shell *feels* different from other languages: words are mini-languages.
   Note `_ReadCommandSubPart`, `_MakeAssignPair` with `do_lossless` branches.
4. [`osh/tdop.py`](osh/tdop.py) + [`osh/arith_parse.py`](osh/arith_parse.py) + [`osh/bool_parse.py`](osh/bool_parse.py) — tiny Pratt/TDOP
   parsers for `$(( ))` and `[[ ]]`/`test`. Short, self-contained, good “first
   parser to read fully” if [`cmd_parse.py`](osh/cmd_parse.py) intimidates you.
5. YSH side: [`ysh/grammar_gen.py`](ysh/grammar_gen.py) + [`pgen2/`](pgen2/) + [`ysh/expr_parse.py`](ysh/expr_parse.py) (389 lines)
   + [`ysh/expr_to_ast.py`](ysh/expr_to_ast.py) — YSH expressions are parsed with a CPython-derived
   parser generator, then transformed to the same [`syntax.asdl`](frontend/syntax.asdl) AST. Compare
   with the hand-written OSH parsers: generated vs hand-rolled, and why both
   exist (YSH is expression-heavy; OSH is irregular enough to need hand code).

Focus areas:

- **Here-docs and aliases** are the two runtime-visible parse hacks. Trace
  `source` with an alias defined and watch `ParseLogicalLine` re-enter.
- **`a[x+1]=foo`**: bash allows spaces across word boundaries; Oils re-parses
  from tokens. `grep do_lossless osh/*.py` shows all four re-parse sites.
- Use `osh -n` constantly. It’s the fastest way to check your mental parse tree.

Experiments:

```sh
bin/osh -n -c 'a[x+1]=foo; echo ${a[$((1+2))]}'
bin/osh --tool lossless-cat spec/testdata/lossless.sh | diff - spec/testdata/lossles*.sh
grep -rn do_lossless osh/*.py frontend/*.py
```

---

## Phase 4 — Evaluation: AST → side effects (3–5 hours)

Goal: understand the two big evaluators and the two small ones.

1. [`osh/cmd_eval.py`](osh/cmd_eval.py) (2630 lines) — `CommandEvaluator:ExecuteAndCatch`.
   `case`/`if`/`for`/`while`, functions/procs, pipelines (fork or
   `SubProgramThunk`), redirects (`_RunSimpleCommand` → `DoExec` → fork/exec or
   builtin), `&&`/`||`/`!`, `errexit`/`pipefail` handling, traps. Read the
   `IsMainProgram` / `OptimizeSubshells` / `MarkLastCommands` flags in
   `main_loop.Batch2` first — they control fork elision.
2. [`osh/word_eval.py`](osh/word_eval.py) (2667 lines) — `NormalWordEvaluator`. Tilde, var/command/
   arith sub, splitting ([`osh/split.py`](osh/split.py), `$IFS` state machine), globbing
   ([`osh/glob_.py`](osh/glob_.py)), brace expansion ([`osh/braces.py`](osh/braces.py)), empty elision, quoting.
   This file *is* shell semantics. Read it alongside
   [`doc/simple-word-eval.md`](doc/simple-word-eval.md) and the wiki “OSH Word Evaluation Algorithm.”
   YSH’s `simple_word_eval` short-circuits most of it — compare
   `EvalWordSequence` paths with the option on/off.
3. [`osh/sh_expr_eval.py`](osh/sh_expr_eval.py) — arithmetic + `[[ ]]` evaluation, including bash’s
   recursive arithmetic (`a='1+2'; b='a+3'; echo $((b))` → `6`) and why Oils
   intentionally diverges (statically parsed arithmetic; see
   [`doc/known-differences.md`](doc/known-differences.md)).
4. [`ysh/expr_eval.py`](ysh/expr_eval.py) (1636 lines) + [`ysh/val_ops.py`](ysh/val_ops.py) — YSH expression
   evaluation over [`core/value.asdl`](core/value.asdl) types (Str/Int/Float/List/Dict/Obj/eggex).
   Cleaner than word eval; good palate cleanser.

Supporting cast (skim, then revisit when curious):

- [`osh/word_compile.py`](osh/word_compile.py), [`osh/split.py`](osh/split.py), [`osh/braces.py`](osh/braces.py), [`osh/glob_.py`](osh/glob_.py),
  [`osh/string_ops.py`](osh/string_ops.py) — the word-eval helpers.
- [`ysh/regex_translate.py`](ysh/regex_translate.py) — Eggex (`/ dot* '.py' /`) → ERE. Fun, self-contained.

Experiment:

```sh
shopt -s simple_word_eval  # then compare:
var pat='*.py'; argv $pat vs argv "$pat"
bin/osh -c 'x="a b"; argv $x; argv "$x"'
```

---

## Phase 5 — State: memory, variables, options (2 hours)

Goal: learn what persists between commands. This is “the OS process’s heap,”
except it’s a value store with shell quirks.

1. [`core/state.py`](core/state.py) (3056 lines — biggest file; skim by class) — `Mem` (stack of
   frames, globals at index 0, `export`/`readonly` flags, dynamic scope),
   `Procs` (functions vs variables are separate namespaces), `MakeOpts`
   (`parse_opts` vs `exec_opts` vs `mutable_opts`), `OptHook`. Read
   [`doc/interpreter-state.md`](doc/interpreter-state.md) first or this file will feel like a bag of methods.
2. [`core/value.asdl`](core/value.asdl) + [`core/runtime.asdl`](core/runtime.asdl) — `value.Str/List/Dict`,
   `cell` (value + exported/readonly flags), `cmd_value.Assign/Argv`. Oils tags
   *values* with types; bash tags *locations* (`declare -A`). That one sentence
   explains half the simplification.
3. [`core/optview.py`](core/optview.py) + [`frontend/option_def.py`](frontend/option_def.py) — every `shopt`/`set -o` option
   as a typed record. `grep shopt spec/*.test.sh | head` shows how much behavior
   hangs off options.
4. [`core/alloc.py`](core/alloc.py) — `Arena` for source lines/tokens, `ctx_SourceCode` for error
   locations. Needed to understand how errors point at the right line after
   re-parsing.

Focus: dynamic scope. `f() { echo $x; }; g() { local x=1; f; }` prints `1`.
YSH limits this; OSH keeps it for compatibility. `grep -n DynamicScope`
[`core/state.py`](core/state.py) is a good thread to pull.

Try:

```sh
bin/osh -c 'f() { echo "x=$x"; }; g() { local x=1; f; }; x=0; f; g'
bin/ysh -c 'pp (shvarGet("PATH"))'
```

---

## Phase 6 — Processes: fork, exec, redirs, jobs, signals (2–3 hours, your home turf)

Goal: connect the evaluator to syscalls. You’ll be faster here than most readers.

1. [`core/process.py`](core/process.py) (2271 lines) — `FdState` (saved fds, `dup2`/`close`
   discipline, `ctx_FileCloser`, `ctx_Descriptors`), `Waiter` (`waitpid(-1)`,
   `PollForEvents`), `JobControl`/`JobList` (process groups, terminal control),
   `ExternalProgram` (`execve` + `SearchPath` + `OILS_HIJACK_SHEBANG`),
   `ctx_TerminalControl`. Skim for `posix.` calls — that’s the syscall surface.
2. [`core/executor.py`](core/executor.py) (1064 lines) — `ShellExecutor` (may fork) vs
   `PureExecutor` (never forks; used for `--eval-pure`, `evalExpr`). This split
   doesn’t exist in bash; it’s an Oils clarity win. `RunSimpleCommand`,
   `RunBackground`, `RunPipeline`, `RunCommandSub`.
3. [`builtin/process_osh.py`](builtin/process_osh.py), [`builtin/trap_osh.py`](builtin/trap_osh.py), [`core/vm.py`](core/vm.py) — `fork`,
   `forkwait`, `wait`, `jobs`/`fg`/`bg`, `trap` on EXIT/ERR/signals. Traps run
   between commands in `main_loop.Interactive` (`RunPendingTraps`) and on exit
   (`RunTrapsOnExit`).

Focus:

- **Fd discipline bugs are shell bugs.** `ShowDescriptorState` in
  [`core/main_loop.py`](core/main_loop.py) (`ls -l /proc/PID/fd`) is a debugging superpower.
- **Fork elision** (`OptimizeSubshells`, `noforklast`): when can the shell avoid
  `fork()` for the last command in a script or pipeline? Wrong answers break
  `pipefail`, `errexit`, traps, and crash dumps — all documented as caveats in
  [`doc/process-model.md`](doc/process-model.md).
- Compare `io.captureAll()` (YSH gets stdout+stderr+status at once — other
  shells can’t) in [`builtin/method_io.py`](builtin/method_io.py).

Experiments:

```sh
bin/osh -c 'echo hi | read x; echo x=$x'          # lastpipe behavior
bin/osh -c 'd=$(date); echo $d'                   # command sub fork
bin/osh -c 'diff -u <(sort a.txt) <(sort b.txt)'  # process sub
OILS_DEBUG_DIR=/tmp bin/osh -c 'echo hi' && cat /tmp/*-osh.log | head
```

---

## Phase 7 — Builtins, functions, and the YSH data layer (2 hours)

Goal: see how much of “the shell” is builtins, not syntax.

1. [`builtin/`](builtin/) (~40 files) — each file is a builtin group: [`assign_osh.py`](builtin/assign_osh.py)
   (`declare`/`export`/`readonly`/`shift`/`unset`), [`bracket_osh.py`](builtin/bracket_osh.py)
   (`test`/`[`), [`io_osh.py`](builtin/io_osh.py)/[`io_ysh.py`](builtin/io_ysh.py) (`echo`/`printf`/`read`/`mapfile`/`pp`),
   [`dirs_osh.py`](builtin/dirs_osh.py) (`cd`/`pushd`/`popd`), [`meta_oils.py`](builtin/meta_oils.py)
   (`source`/`eval`/`use`/`type`/`command`), [`module_ysh.py`](builtin/module_ysh.py), [`json_ysh.py`](builtin/json_ysh.py),
   [`hay_ysh.py`](builtin/hay_ysh.py). [`builtin/README.md`](builtin/README.md) + [`frontend/builtin_def.py`](frontend/builtin_def.py) map names to IDs.
2. [`ysh/func_proc.py`](ysh/func_proc.py), [`builtin/func_*.py`](builtin/), [`builtin/method_*.py`](builtin/) — YSH
   `proc`/`func`, builtin functions (`len`/`type`/`join`/`split`/`toJson`),
   and methods on Str/List/Dict (`methods[value_e.Str]` in [`core/shell.py`](core/shell.py)).
   Note `M/` prefix convention for mutating methods.
3. [`data_lang/`](data_lang/) (J8/JSON/HTM8), [`builtin/json_ysh.py`](builtin/json_ysh.py) — structured data in shell.
   [`doc/j8-notation.md`](doc/j8-notation.md), [`doc/json.md`](doc/json.md). If you’ve ever piped JSON through
   `jq` in bash, this is Oils’ typed answer.
4. [`display/`](display/), [`osh/prompt.py`](osh/prompt.py), [`osh/history.py`](osh/history.py), [`core/completion*.py`](core/),
   [`builtin/completion_*.py`](builtin/), [`frontend/py_readline.py`](frontend/py_readline.py) — interactive shell:
   `$PS1` evaluation (itself a mini word-parser via
   `MakeWordParserForPlugin`), history expansion, tab completion
   (`RootCompleter`, `SpecBuilder`, `Trail` in [`frontend/parse_lib.py`](frontend/parse_lib.py)).
   Read only if you care about interactive UX; skip on first pass otherwise.

Try:

```sh
bin/ysh -c 'var d = {name: "x", nums: [1,2]}; pp (d); echo $[toJson(d)]'
bin/osh -c 'type cd; type echo; builtin echo hi; command ls /tmp | head'
```

---

## Phase 8 — Meta: how it’s built, tested, and shipped (1–2 hours)

Goal: understand why the repo looks nothing like a normal Python project.

1. **Dev build vs native build.** [`build/py.sh`](build/py.sh) `all` gives you slow CPython
   [`bin/osh`](bin/osh). [`./NINJA-config.sh`](NINJA-config.sh) `&& ninja` runs [`mycpp`](mycpp/) translation + C++
   compile to [`_bin/cxx-*/osh`](_bin/). End users get a tarball built by
   [`_build/oils.sh`](_build/oils.sh), not the repo (see [README](README.md) + wiki “Repo Is Different From
   Tarball”). [`NINJA-config.sh`](NINJA-config.sh), [`build/ninja_*.py`](build/), [`cpp/NINJA_subgraph.py`](cpp/NINJA_subgraph.py).
2. **[`mycpp/README.md`](mycpp/README.md)** (617 lines — read the first 150 + skim the rest).
   Typed-Python subset, MyPy-derived, C++ type system as checker, [`pea/`](pea/) and
   [`yaks/`](yaks/) as future directions. Constraint to internalize: if a Python idiom
   can’t map to C++11, it’s banned. That explains otherwise-odd code style
   (explicit `NewDict()`, no generators in hot paths, `tagswitch` instead of
   `isinstance` chains).
3. **Codegen pairs.** `ls */*_def.py */*_gen.py`: `lexer_def`/`lexer_gen`,
   `id_kind_def`, `option_def`/`option_gen`, `flag_def`/`flag_gen`,
   `grammar_gen`, `consts_gen`, `signal_gen`, `optview_gen`. `def` = abstract
   definition (not translated); `gen` = emits Python+C++. [`build/codegen.sh`](build/codegen.sh) +
   [`build/dev.sh`](build/dev.sh) drive them; outputs land in [`_devbuild/gen/`](_devbuild/gen/) and [`_gen/`](_gen/).
4. **Tests.** Unit: `foo_test.py` next to `foo.py`, run with [`test/unit.sh`](test/unit.sh).
   Gold: [`test/gold.sh`](test/gold.sh). Spec: [`spec/*.test.sh`](spec/) via [`test/sh_spec.py`](test/sh_spec.py) and
   [`test/spec.sh`](test/spec.sh) — run a single file against multiple shells to see the
   compatibility matrix. Stateful: [`spec/stateful/`](spec/stateful/) (pexpect). Lossless:
   [`test/lossless.sh`](test/lossless.sh). Wild: [`test/wild.sh`](test/wild.sh) (parse real-world scripts).
5. **Metrics/soil.** [`metrics/source-code.sh`](metrics/source-code.sh) `overview` (line counts by dir),
   [`soil/`](soil/) (multi-cloud CI). Good for “how big is X really?”

Commands:

```sh
metrics/source-code.sh overview
test/spec.sh osh-only spec/word-split.test.sh   # one spec file, fast loop
test/unit.sh 'osh/*' 2>&1 | tail
```

---

## Suggested schedules

- **Weekend skim (4–6h):** Phase 0 + Phase 1 + [`syntax.asdl`](frontend/syntax.asdl) + Phase 4 headers +
  Phase 8 skim. You’ll be able to navigate any file and explain the loop.
- **Full pass (15–20h):** Phases 0–8 in order, doing every experiment. Read
  [`cmd_parse.py`](osh/cmd_parse.py) and [`word_eval.py`](osh/word_eval.py) with `osh -n` open in another terminal.
- **Deep cuts (pick one):** (a) word evaluation + `simple_word_eval`,
  (b) fork/fd/job-control discipline, (c) mycpp translation constraints,
  (d) YSH expression pipeline ([pgen2](pgen2/) → [`expr_to_ast`](ysh/expr_to_ast.py) → [`expr_eval`](ysh/expr_eval.py)).

## Primary sources to keep open

- [`frontend/syntax.asdl`](frontend/syntax.asdl) — the language.
- [`core/shell.py:Main()`](core/shell.py) — where everything is wired.
- [`core/main_loop.py`](core/main_loop.py) — the execution loop.
- [`doc/parser-architecture.md`](doc/parser-architecture.md), [`doc/interpreter-state.md`](doc/interpreter-state.md),
  [`doc/process-model.md`](doc/process-model.md), [`doc/simple-word-eval.md`](doc/simple-word-eval.md) — the four docs that
  actually explain shell semantics.
- [`spec/`](spec/) — ground truth for “does it behave like bash?”
- [`mycpp/README.md`](mycpp/README.md) — why the Python looks the way it does.

## One-paragraph history to keep in mind

Oils started as “Oil” (bash-compatible shell done right), grew a second
language (YSH) on the same runtime, and then paid down the two classic shell
debts explicitly: quoting/splitting confusion (fixed by `simple_word_eval` +
  explicit `@[split()]`/`@[glob()]`) and stringly-typed state (fixed by
  value-tagged types + `pp`). The C++ translation exists so the result is still
  a small, fast `/bin/sh` replacement, not just a Python prototype. Reading with
  that arc in mind — compatibility first, sanity second, performance via
  translation — makes otherwise-puzzling choices (three `ParseContext`s! two
  executors! four re-parse sites!) click into place.
