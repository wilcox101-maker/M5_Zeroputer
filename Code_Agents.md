# Code_Agents.md

Read this file before every task. Chat instructions override this file.

This file is the contract for coding agents working in this repository.
Goal: smallest correct change that solves the requested task. Plausibility is not correctness.

---

## Non-negotiables

1. Never invent files, APIs, flags, test results, commit hashes, or library functions. Read the code or run the command.
2. Never claim tests, lint, or typecheck passed unless you actually ran them and captured the output.
3. Never commit secrets, credentials, private keys, `.env` files, tokens, or generated dumps.
4. Never rewrite unrelated files "while you are here."
5. Never delete or overwrite user data, migrations already applied, lockfiles wholesale, or production config without an explicit ask.
6. If requirements are ambiguous, stop and ask. Do not pick a path silently.
7. If a change expands into a redesign, stop and ask before continuing.

---

## Defaults

- Project needs determine the codebase. Language, layout, and toolchain come from the task plus what is already in the tree — not from this file's examples.
- Stay inside the requested scope.
- Match existing style, layout, and libraries. Do not introduce a new pattern unless the task is a redesign.
- Prefer the simplest working fix over a framework, abstraction, or new dependency.
- Ask before adding a dependency, changing package manager, or altering CI.
- Restate the task in one sentence before large edits.
- Show what changed and why. Keep diffs reviewable.
- Flag low-confidence conclusions. Prefer "I need to verify X" over guessing.

---

## Project commands

Project needs determine the codebase. Do not assume Python, Rust, or C because those recipes appear below.

Before the first edit:

1. Read the request: target hardware, runtime, latency, memory, safety, and deploy environment.
2. Inspect the tree: lockfiles, build files, CI, existing packages. Those win over this document.
3. Use only the language and build system the need + tree already imply.
4. If the tree is empty or greenfield, pick the smallest stack that satisfies the need, state the choice, and ask if more than one stack is reasonable.
5. Do not add a second language, package manager, or build system to an existing tree unless the need cannot be met without it — and then ask first.

### How to choose (existing tree wins)

| Evidence in the repo or the need | Use | Do not |
|---|---|---|
| `pyproject.toml`, `uv.lock`, `*.py` | Python + the package manager already present | Introduce `pip` + `requirements.txt` next to `uv`, or add Rust/C "for speed" unasked |
| `Cargo.toml`, `rust-toolchain.toml`, `*.rs` | Rust, crate-scoped cargo | Full-workspace builds, editing a generated root `Cargo.toml` |
| `CMakeLists.txt`, `*.c` / `*.h` / `*.cpp` and no embedded project file | C/C++ + existing CMake/Make/Meson | Invent a second build root |
| `platformio.ini`, `idf_component.yml`, `west.yml`, vendor HAL | That embedded toolchain | Drop a host CMakeLists on top of it |
| `package.json` + lockfile | The package manager of the lockfile (`pnpm` / `npm` / `yarn` / `bun`) | A different JS package manager |
| `go.mod` | Go modules | Vendoring a second Go tree |
| Need is a host library / SDK / CLI with no tree yet | Prefer the stack the user named; if they did not, ask | Default to Python because xAI SDK examples use it |
| Need is firmware, ISR, GPIO, tight RAM/flash | C or the vendor SDK already chosen | Pull in Python or a RTOS the hardware does not use |
| Need is a concurrent agent harness / systems service | Rust if the tree is Rust; otherwise the tree's language | Rewrite a working C or Python service in Rust unasked |
| Mixed tree (firmware + host tool) | Edit the package the task names. Keep boundaries. | "Unify" the monorepo onto one language |

Command recipes below are the gate to run **when that language is in scope**. They are not a mandate to create that language in the repo.

xAI public-repo convention, used only when starting or extending a matching tree: Python → `uv` + Ruff + Pyright + pytest (`xai-org/xai-sdk-python`). Rust → crate-scoped cargo + clippy + rustfmt (`xai-org/grok-build`). C/C++ has no published xAI CI recipe; use the repo's build file, or CMake + Clang if the tree has none.

Do not invent a different package manager than the one the tree already uses.

### Python (when the tree or need is Python)

Python ≥ 3.10. Tooling: `uv`, Ruff, Pyright, pytest, pre-commit.

```text
# Install / sync lockfile
uv sync --locked

# Hooks (once per clone)
uv run pre-commit install

# Add a dependency (ask first)
uv add <package>
uv add --dev <package>

# Dev / run a module or example
uv run python -m <package>
uv run <script>

# Format
uv run ruff format
uv run ruff format --check
uv run ruff format path/to/file.py

# Lint
uv run ruff check
uv run ruff check --fix
uv run ruff check --output-format=github
uv run ruff check path/to/file.py

# Typecheck
uv run pyright

# Tests — single file / node
uv run pytest -n auto -v path/to/test_file.py
uv run pytest -n auto -v path/to/test_file.py::test_name

# Tests — full suite (CI equivalent)
uv run pytest -n auto -v
```

Ruff line length in xAI Python repos is 120. Generated proto / vendor trees are excluded from Ruff and Pyright — do not format or typecheck them by hand unless asked.

### Rust (when the tree or need is Rust)

Toolchain is pinned by `rust-toolchain.toml`. Target a crate. Do not run full-workspace builds unless asked. Do not edit a generated root `Cargo.toml`.

```text
# Check
cargo check -p <crate>

# Dev / run a binary crate
cargo run -p <crate>
cargo run -p <crate> --release

# Format
cargo fmt --all
cargo fmt -- --check

# Lint
cargo clippy -p <crate> --all-targets -- -D warnings

# Tests — one crate
cargo test -p <crate>
cargo test -p <crate> -- <test_name>

# Tests — one crate, quieter
cargo test -p <crate> -- --nocapture
```

Clippy config lives in repo-root `clippy.toml`. rustfmt config lives in `rustfmt.toml`.

### C / C++ (when the tree or need is C or C++)

Use the build system already in the tree. If there is none and the need is a host C/C++ program, then Clang + CMake + Ninja. C11 / C++17 minimum unless the repo pins otherwise. Out-of-source builds only. Never compile in the source tree.

```text
# Install toolchain (Debian/Ubuntu example — do not run if already present)
sudo apt-get update
sudo apt-get install -y clang clang-tidy clang-format cmake ninja-build lld \
    libc++-dev libc++abi-dev libgtest-dev

# Configure (Debug + sanitizers for agent work)
cmake -S . -B build -G Ninja \
  -DCMAKE_BUILD_TYPE=Debug \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++ \
  -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
  -DCMAKE_C_FLAGS="-Wall -Wextra -Werror -Wconversion -Wshadow -fno-omit-frame-pointer" \
  -DCMAKE_CXX_FLAGS="-Wall -Wextra -Werror -Wconversion -Wshadow -fno-omit-frame-pointer" \
  -DCMAKE_C_FLAGS_DEBUG="-O1 -g -fsanitize=address,undefined" \
  -DCMAKE_CXX_FLAGS_DEBUG="-O1 -g -fsanitize=address,undefined"

# Dev / build
cmake --build build
cmake --build build --target <target>

# Run a binary
./build/<target>
ctest --test-dir build --output-on-failure -R <test_name>

# Format (repo .clang-format if present)
clang-format -i path/to/file.c path/to/file.h
clang-format -i path/to/file.cpp path/to/file.hpp
clang-format --dry-run -Werror path/to/file.c

# Lint / static analysis (needs compile_commands.json from the configure step)
clang-tidy -p build path/to/file.c --quiet
clang-tidy -p build path/to/file.cpp --quiet --fix
cppcheck --enable=warning,style,performance,portability --error-exitcode=1 \
  --project=build/compile_commands.json

# Tests
ctest --test-dir build --output-on-failure
ctest --test-dir build --output-on-failure -R <test_name>

# Release configure (no sanitizers; do not use as the default agent gate)
cmake -S . -B build-rel -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DCMAKE_C_COMPILER=clang \
  -DCMAKE_CXX_COMPILER=clang++
```

Do not disable `-Werror` or sanitizers to make a gate green. Fix the diagnostic.

If the tree is embedded (ESP-IDF, Zephyr, PlatformIO, vendor HAL) and already has a project file, use that instead of inventing a CMake root:

```text
# PlatformIO
pio run
pio test
pio check

# ESP-IDF
idf.py build
idf.py test
idf.py clang-check

# Zephyr
west build -b <board> app
west twister -T tests/<suite>
```

C/C++ error-proofing that the linter will not catch for you:

- Check every libc / HAL / SDK return. Do not ignore `errno`, `esp_err_t`, `HAL_StatusTypeDef`.
- No unbounded `strcpy`, `sprintf`, `gets`, `scanf("%s")`. Use `snprintf`, `memcpy_s` / explicit lengths, or `strlcpy` if the libc has it.
- No VLAs sized from untrusted input. No `alloca` of attacker-controlled size.
- Initialize every local and every struct. `-Wuninitialized` is not optional.
- Pair every `malloc`/`new`/`fopen`/`open` with a single release path. Prefer a cleanup label or RAII.
- Integer sizes and overflows: `size_t` for lengths, checked add before buffer math.
- Do not block in ISRs. Do not call malloc/printf from ISRs.
- Compile and run the sanitizer build before claiming done. A green `-O0` warning-free build without ASan/UBSan is not done.

### Proto / generated code

```text
buf generate
```

Do not hand-edit files under generated proto trees (`src/*/proto`, `gen/`). Change the `.proto` and regenerate.

### Local CI gate (run the row that matches the files you touched)

Python change (only if you changed Python):

1. `uv run ruff format path/to/changed.py`
2. `uv run ruff check path/to/changed.py`
3. `uv run pyright`
4. `uv run pytest -n auto -v path/to/matching_test.py`

Rust change (only if you changed Rust):

1. `cargo fmt --all`
2. `cargo clippy -p <crate> --all-targets -- -D warnings`
3. `cargo test -p <crate>`

C / C++ change (only if you changed C or C++):

1. `clang-format -i path/to/changed.c path/to/changed.h`
2. `clang-tidy -p build path/to/changed.c`
3. `cmake --build build --target <target>` (or `pio run` / `idf.py build` / `west build` if that is the tree)
4. `ctest --test-dir build --output-on-failure -R <related_test>`

A change is not done until those gates pass on the files you touched. Never report green from a dry read of the diff. A C/C++ host change is also not done until the sanitizer-enabled Debug build has run.

---

## Error-proofing (required)

Treat every boundary as hostile: user input, network, disk, hardware, clocks, concurrency, and other processes.

### Fail loud, fail early

- Validate at the edge. Reject bad input before it enters domain logic.
- Do not swallow exceptions. Catch only what you can handle; rethrow or wrap the rest with context.
- Never use empty `catch`, `except: pass`, or a bare `On Error Resume Next`.
- Never return success, `null`, `0`, or `{}` to hide a failure.
- Prefer Result / Either / explicit error types over thrown exceptions for expected failures. Use exceptions for unexpected ones.
- Every I/O call has a timeout. Every lock has a bound. Every retry has a cap and jitter.
- Check return codes. In C/C++/embedded, check every HAL/SDK call. Do not ignore `esp_err_t`, `HAL_StatusTypeDef`, or `errno`.

### Types and contracts

- Public functions have explicit types (or the language's equivalent) on parameters and return values.
- Do not use `any`, untyped `Object`, or `void*` at API boundaries unless FFI requires it — and then isolate it.
- Parse, don't validate twice. Once data is parsed into a typed model, trust the model inside the module.
- Make illegal states unrepresentable: enums over magic strings/ints; newtypes over raw IDs; discriminated unions over boolean flags that collide.
- Function arguments beyond two become a named options object / struct.
- No default-true "force" or "skip validation" flags in production paths.

### Defensive structure

- Early return. Flatten nesting. No arrow anti-pattern.
- Initialize everything. No uninitialized reads. No use-after-free. No dangling references.
- Bounds-check every index and buffer write. Prefer length-aware APIs (`snprintf`, slices, `span`, `string_view`) over raw pointer arithmetic.
- Integer math: check overflow on size, index, money, and time calculations. Do not mix signed/unsigned casually.
- Concurrency: no shared mutable state without a documented lock or actor. No data races. Document which mutex guards which fields.
- Resources: RAII, `with`, `defer`, or `try/finally`. Every open has a close on all paths, including error paths.
- Idempotent writes where the caller may retry. Safe to run twice.

### Logging and observability

- Log the failure with enough context to reproduce: operation, identifier, error code. Do not log secrets or PII.
- Do not log inside a tight hot loop.
- Errors that leave the process get a stable error code or typed error the caller can switch on.
- Add a test that would have caught the bug you just fixed.

### Decision table — errors

| Situation | Do | Do not |
|---|---|---|
| Expected domain failure (not found, conflict, invalid input) | Return a typed error / Result | Throw a generic Exception and hope |
| Unexpected invariant break | Fail fast, include context | Log and continue with a default |
| Transient I/O (network, bus, disk) | Retry with cap + backoff + jitter | Retry forever or retry on 4xx / validation errors |
| Partial multi-step write | Use a transaction, outbox, or compensating action | Leave half-written state |
| Missing config | Fail at startup | Invent a default that changes behavior in prod |
| Optional feature unavailable | Disable that path and report | Crash the whole service |

---

## Coding practices

### Shape of the change

- One concern per function, per file, per commit when practical.
- Names describe behavior, not implementation (`parseConfig`, not `handleData2`).
- Delete dead code you just made dead. Do not comment it out.
- No speculative abstraction. Three similar lines beat a premature helper used once.
- Comments explain *why* or a non-obvious invariant. Do not narrate what the next line does.
- Magic numbers that appear more than once, or come from a spec, become named constants. One-off obvious values (HTTP 200, `i = 0`) stay inline.

### Style (only where it is not already enforced by the formatter)

- Match the repo formatter and linter. Do not bikeshed quotes, tabs, or line length in this file — put those in the tool config.
- Always use braces for `if` / `else` / `for` / `while`, even one-liners.
- Prefer immutability (`const`, `final`, `readonly`) until mutation is required.
- No wildcard imports.
- Keep modules small. New code goes next to the code it changes, not in a new top-level grab-bag.

### Dependencies and APIs

- Reuse what the repo already uses. Decision table:

| Need | Do | Do not |
|---|---|---|
| HTTP | Existing client in the repo | Add axios/got/requests if fetch/`httpx` is already there |
| Dates | Existing date lib | Mix moment, dayjs, and raw Date |
| Validation | Existing schema lib (zod/pydantic/etc.) | Hand-rolled if/throw chains at every handler |
| DB access | Existing repository / query layer | Raw SQL sprinkled in handlers |
| Config | Existing env loader | `os.getenv` with silent defaults in business code |

- Pin versions the way the repo already pins them. Update the lockfile when you add a dependency — and only after asking.

### Testing

- Tests prove behavior, not implementation details.
- Name tests: `method_whenCondition_expectedResult` or the repo's existing pattern.
- Cover: happy path, one invalid input, one failure from a collaborator, one boundary (empty, max, off-by-one).
- Do not hit real networks, devices, or paid APIs in unit tests. Fake the edge.
- Do not assert on log text or exact error message wording unless that string is a contract.
- If you change behavior, update or add a test in the same change.
- Never delete a failing test to make the suite green.

### Security

- Treat all external input as untrusted: HTTP, files, serial, BLE, MQTT, query strings, headers, env pulled from the user.
- Parameterize queries. No string-built SQL, shell, or firmware command lines.
- Authz is checked on every path, not only the UI path.
- Default deny. Fail closed.
- Do not disable TLS, cert checks, or auth "just for now."
- Do not put secrets in source, comments, test fixtures committed to git, or client bundles.

### Git and review

- Do not commit unless asked.
- Do not force-push shared branches.
- Do not amend published history.
- Commit messages: imperative, scoped, explain why if the diff is not obvious.
  Example: `fix(parser): reject truncated frames instead of returning zeros`
- Before you call a task done:

  1. Diff is limited to the task.
  2. Format / lint / types ran on touched files.
  3. Relevant tests ran.
  4. No secrets in the diff.
  5. You can describe the failure mode you closed.

---

## Workflow — implement a change

1. Confirm language and build system from the need + the tree. Do not switch stacks.
2. Read the files you will touch. Quote the current contract (types, error codes, pin map, API) before editing.
3. State assumptions in one short list. Ask if any assumption changes behavior.
4. Write the smallest patch. Keep unrelated cleanup out.
5. Add or update tests for the behavior you changed.
6. Run file-scoped format, lint, typecheck, and tests for the language you actually edited.
7. If any gate fails, fix the cause. Do not weaken the gate.
8. Summarize: files, behavior change, tests run, residual risk.

## Workflow — debug a failure

1. Reproduce. Capture the exact command, input, and error.
2. Find the first point the invariant breaks. Do not patch a symptom two layers up.
3. Write a failing test or a minimal repro if one does not exist.
4. Fix the root cause. Handle the error on all paths.
5. Re-run the repro and the surrounding tests.
6. Note the class of bug (bounds, timeout, race, bad default) so the next change cannot repeat it.

## Workflow — stop conditions

Stop and report instead of continuing when:

- More than one plausible design would change public behavior.
- The need does not uniquely determine the language or build system.
- A required service, toolchain, or hardware target is unreachable.
- A test run exceeds a reasonable bound or hangs.
- The task requires credentials you do not have.
- The next edit would touch generated code, vendored code, or applied migrations.

---

## Language notes (apply only the row for files this task actually touches)

| Language | Do | Do not |
|---|---|---|
| TypeScript / JS | Strict types, no `any` at boundaries, `unknown` + narrow | Implicit any, unhandled promises, `==` |
| Python | `uv` only when the tree uses `uv`, type hints on public fns, `pathlib`, Ruff + Pyright clean | `pip` / `requirements.txt` beside `uv`, bare `except`, mutable default args |
| Go | Check every `err`, context on I/O, no discarded `_` on errors | Panic for expected failures, ignoring `err` |
| Rust | `Result`/`Option` at edges, no `unwrap` in library paths | `unwrap`/`expect` in production code without an invariant comment |
| C / C++ / embedded | Check every HAL return, bounded buffers, init order, watchdog-friendly loops | Ignore status codes, VLAs of untrusted size, blocking in ISRs |
| C# / Java | Nullable annotations, specific exceptions, dispose/using | Empty catch, swallowing `Exception` |
| Shell | `set -euo pipefail`, quoted expansions, explicit exit codes | Unquoted `$var`, parsing `ls` |

---

## What "done" means

- The requested behavior exists in the stack the need and tree already required.
- Invalid and failure paths are explicit, tested or demonstrated.
- Gates on touched files are green because you ran them.
- Diff contains nothing extra and no unsolicited language or build-system change.
- Residual risk is stated in one or two lines (what you did not run, what hardware you could not exercise).

Do not report done from a diff that merely looks right.
