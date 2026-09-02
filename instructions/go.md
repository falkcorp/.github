<!-- file: instructions/go.md -->
<!-- version: 1.4.0 -->
<!-- guid: 6a15e7db-6d51-493a-87a6-6f4fe14a7f84 -->
<!-- last-edited: 2026-09-01 -->

# Go Coding Standards

See [file-headers.md](file-headers.md) for mandatory version header rules.

## Go version policy

**Go 1.26 is the mandatory minimum and the default.** Every repo declares
`go 1.26` in `go.mod` and pins `go-version: '1.26'` in CI. No repo may sit
below it, and the idioms in this document assume it — the pre-1.24 forms they
replace are not acceptable in new code.

**Prefer Go 1.27 wherever every third-party library and tool supports it.** If
a repo's full dependency graph builds, vets and tests clean on 1.27, move it:
bump `go.mod`, bump every `go-version` in CI together, and say so in the PR. A
repo that *can* be on 1.27 and is sitting on 1.26 for no stated reason is drift,
not a decision.

**Do not bump a repo whose dependencies are not ready.** The check is a build,
not a guess:

```sh
GOTOOLCHAIN=go1.27.1 go build ./... && GOTOOLCHAIN=go1.27.1 go vet ./... && GOTOOLCHAIN=go1.27.1 go test -short ./...
```

Use the latest patch release, and include the tests: `go test` on 1.27 runs the
`stdversion` vet check by default, so a file that uses a symbol newer than the
`go` directive in force for it fails there and nowhere else.

### Pin the toolchain; do not just declare it

The `go` directive is a *minimum*. `GOTOOLCHAIN` defaults to `auto`, which
**never steps down**: a developer with a newer local Go builds with that Go,
and the binary they ship differs from the one CI and the Dockerfile produce.
That is how a deploy broke on 2026-08-24 — same source, different toolchain,
nothing in the diff. So every repo carries one explicit pin and mirrors it:

| where | form | why this one too |
| --- | --- | --- |
| `Makefile` | `export GOTOOLCHAIN := go1.27.1` (`:=`, so a stale shell value cannot override it) | the only pin `make` sees |
| `.envrc` | `export GOTOOLCHAIN=go1.27.1` | bare `go` commands and editors under direnv |
| `Dockerfile*` | `FROM golang:1.27.1-alpine@sha256:…` | builders do not read the Makefile |
| CI | `go-version: '1.27'` on every job | the runner's `setup-go` does not read `.envrc` |
| `go.mod` | `go 1.27.0`, **no** `toolchain` line below it | see the next section |

The literal is deliberate: `go.mod` may legally say `go 1.27`, and `go1.27` is
not a valid toolchain name, so deriving the pin from the directive is fragile.
Drift in the dangerous direction is self-detecting — if `go.mod` ever requires
more than the pin, Go refuses with `go.mod requires go >= X (running go1.27.1;
GOTOOLCHAIN=go1.27.1)`. Drift in the *other* direction is not, so when you bump,
find every copy first:

```sh
grep -rn --exclude-dir=node_modules --exclude-dir=.git -E 'go1\.2[0-9]|golang:1\.2[0-9]|go-version' \
  Makefile* .envrc Dockerfile* .github/workflows .vscode .github/codeql 2>/dev/null
```

Editor settings (`.vscode/settings.json` `go.toolsEnvVars`) count as a build
path for this purpose: gopls building with a different toolchain than `make`
reports errors the build does not have, or misses ones it does.

### Worked example: audiobook-organizer was blocked on 1.27, deliberately — and moved when the blocker moved

Through August 2026 `audiobook-organizer` stayed on 1.26 and the reason was a
dependency, not reluctance. Bumping `go.mod` to `go 1.27.0` failed:

```text
# github.com/cockroachdb/swiss
map.go:286:7:  undefined: hashFn
map.go:337:14: undefined: getRuntimeHasher
map.go:338:22: undefined: fastrand64
```

`github.com/cockroachdb/swiss` arrives transitively via
`github.com/cockroachdb/pebble/v2/internal/cache`. It reaches into Go runtime
internals by `//go:linkname`, and it gates that file on an explicit **upper
bound**:

```go
//go:build (go1.20 && !go1.27) || untested_go_version
```

with the comment: *"This file introspects into Go runtime internals. In order to
prevent accidental breakage when a new version of Go is released we require
manual [verification]."*

So this is **not a bug and not an oversight — it is the library protecting
you.** Runtime internals are unstable by definition, and the maintainer has
chosen to fail the build loudly rather than let a silently-changed hash function
corrupt a cache at runtime.

**A second blocker in the same repo, of the opposite kind.** With `swiss`
unblocked, `audiobook-organizer` still did not compile on 1.27, because *our
own* code used an API that Go removed:

```text
internal/metadata/audible.go:184:     undefined: json.DiscardUnknownMembers
internal/metadata/audible.go:218:     undefined: json.DiscardUnknownMembers
internal/metadata/googlebooks.go:121: undefined: json.DiscardUnknownMembers
```

`encoding/json/v2` declared `DiscardUnknownMembers` in 1.26 and **dropped it in
1.27** (`RejectUnknownMembers` survives in both). This is the price of the
`GOEXPERIMENT=jsonv2` opt-in that this document required on 1.26: **an
experiment's API is explicitly not covered by the Go compatibility promise and
can change between releases.** Budget for that when you opt in, and expect the
toolchain bump — not the experiment flag — to be where you discover it.

Here the fix is to delete the option: unknown members are *ignored by default*,
so `DiscardUnknownMembers(true)` only ever requested the default behaviour.
Which is the general shape — an experiment API that disappears is usually one
that was redundant or is being renamed, so check the current docs before
reaching for a shim.

**How it ended (2026-09-01).** The upstream rule did what it promises. pebble
v2.1.7 pulled a `swiss` that no longer reaches into the runtime; re-running the
probe passed; and the bump was the one-line `go.mod` change the rule predicts —
plus deleting `GOEXPERIMENT=jsonv2` from **thirteen** places (Makefile, two
Dockerfiles, `.envrc`, editor settings, a CodeQL README, and nine workflows,
two of them via a release action's `go-experiment` input). That second number
is the real cost of an experiment opt-in: not setting the flag, but finding
every copy of it later. The `DiscardUnknownMembers` calls had been deleted
weeks earlier, as the table below says to.

Distinguish the two, because they have different owners:

| blocker | owner | action |
| --- | --- | --- |
| a dependency gating itself off the new toolchain | upstream | wait, record it by name |
| our own use of a changed or removed API | us | fix it now, don't wait |
| a stale `toolchain` directive below the `go` line | us | fix it now — not a blocker, a bug |

Three rules follow, and all of them matter more than the version number:

- **Never force it.** Do not set `-tags untested_go_version`, do not fork the
  dependency, do not `replace` it with a patched copy to get the toolchain bump.
  That converts a loud compile error into exactly the silent runtime corruption
  the gate exists to prevent.
- **Wait for upstream, and record why.** The blocker belongs in the repo's
  `go.mod` neighbourhood or its TODO with the dependency named, so the next
  person does not re-derive it. Re-run the probe above when the dependency
  updates; the bump is then a one-line change.
- **Confirm the blocker against a working control before you name it.** Find a
  repo that already does the same thing successfully on the same toolchain, and
  diff the two. A blocker recorded here is load-bearing — every later repo cites
  it to justify staying behind — so "the build failed and the cause looks
  external" is not enough. With no control available, write the blocker down as
  *unconfirmed*.

### Worked example: the blocker that was ours all along

`overnight-burndown` failed CodeQL's `Analyze (go)` on its 1.27 bump:

```text
Autobuilder was built with go1.27.0, environment has go1.26.2
go: go.mod requires go >= 1.27.0 (running go 1.26.2; GOTOOLCHAIN=local)
make: *** [Makefile:30: build] Error 1
Extraction failed for all discovered Go projects.
```

The repo has no CodeQL workflow — code scanning runs through GitHub **default
setup**, so nothing in the repo can pin its toolchain. That reads as an airtight
"a third-party tool is not ready yet," and very nearly got written down as one.

The control disproved it. `subtitle-manager` was also on `go 1.27.0`, under the
same CodeQL build, the same runner and the same build mode — and passed. One
line differed:

| repo | go.mod `toolchain` | CodeQL `Setup Go` | result |
| --- | --- | --- | --- |
| subtitle-manager | *absent* | `go1.27.0` | pass |
| overnight-burndown | `go1.26.2` | `go1.26.2` | fail |

The bump had written `go 1.27.0` but left `toolchain go1.26.2` in place.
Deleting that one line turned 1 failure into 6/6 green.

### Keep `toolchain` at or above the `go` line, or omit it

A `toolchain` older than the `go` directive is self-contradictory, and it is
**invisible on a developer machine**: `GOTOOLCHAIN` defaults to `auto`, so Go
quietly downloads a newer toolchain and moves on. Only a `GOTOOLCHAIN=local`
environment — CodeQL, and most sandboxed CI — turns it into a hard error.

Worse, a green check does not clear a repo. `magnet-handler` carries the same
inversion (`go 1.26.0` with `toolchain go1.24.2`) and its `Analyze (go)` passes,
because it has no root Makefile: autobuild fell through to `go get ./...`, which
self-healed (`go: downloading go1.26.0`, `go: removed toolchain go1.24.2`). A
repo *with* a build script gets `make build` under `local` instead, and dies. So
the check passing means only that nothing invoked the pinned toolchain.

Prefer omitting `toolchain` entirely when the `go` directive already names the
version you build with. Audit every module, not just the one that failed:

```sh
for f in $(git ls-files '*go.mod'); do
  printf '%s\tgo=%s\ttoolchain=%s\n' "$f" \
    "$(awk '/^go /{print $2; exit}' "$f")" \
    "$(awk '/^toolchain /{print $2; exit}' "$f")"
done
```

### JSON v2 — the rule depends on the toolchain

**On Go 1.26** a project that opts into JSON v2 must set `GOEXPERIMENT=jsonv2`
on **every** build path — local shell, CI, release builders, Docker images. A
local `.envrc` alone is not enough: a builder that misses the flag compiles
different marshalling behaviour than the one you tested.

**On Go 1.27 the flag comes out everywhere, in the same PR as the bump.**
`encoding/json/v2` and `encoding/json/jsontext` are GA and on by default;
`encoding/json` itself is now implemented on top of v2 (same behaviour, error
message text may differ — a test that matches error strings is the thing that
breaks). A leftover `GOEXPERIMENT=jsonv2` is dead weight that tells the next
reader a requirement exists, and a leftover `go-run-linters: false` whose only
reason was "the lint step didn't see the experiment" is a decision that has lost
its reason — re-examine it, and say in the PR whether you flipped it. The
opt-out is `GOEXPERIMENT=nojsonv2`, it is documented as temporary, and using it
is an issue to file upstream, not a fix.

Moving from the 1.26 experiment to the 1.27 GA API, check for these — each was
removed or renamed on the way and fails to compile:

| 1.26 experiment | 1.27 |
| --- | --- |
| `format` tag option | removed |
| `unknown` tag option | removed |
| `inline` tag option | renamed `embed` |
| `DiscardUnknownMembers` | removed — it requested the default; delete the call |
| `SkipFunc` sentinel | removed |
| `jsontext` numeric `Token` accessors | now also return an error |

`grep -rn 'json:".*,\(format\|unknown\|inline\)' --include='*.go'` finds the
tag options; the compiler finds the rest.

## Deprecated standard library — do not use

These have had drop-in replacements for years. New code must not use them, and
touching a file that still does means fixing it in the same change.

| Deprecated | Use instead |
|---|---|
| `io/ioutil.ReadFile` | `os.ReadFile` |
| `io/ioutil.WriteFile` | `os.WriteFile` |
| `io/ioutil.ReadAll` | `io.ReadAll` |
| `io/ioutil.ReadDir` | `os.ReadDir` (returns `[]os.DirEntry`, not `[]os.FileInfo`) |
| `io/ioutil.TempFile` / `TempDir` | `os.CreateTemp` / `os.MkdirTemp` — in tests, `t.TempDir()` |
| `io/ioutil.NopCloser` | `io.NopCloser` |
| `io/ioutil.Discard` | `io.Discard` |

**The whole package is deprecated**, so the import itself is the smell — grep
for `"io/ioutil"`, not for individual functions. `os.ReadDir` is the one that is
not a pure rename: it returns `[]os.DirEntry`, which is cheaper because it does
not `stat` every entry, so call `.Info()` only where you actually need it.

## Modernize with `go fix`, then review by class

Run `go fix ./...` **at the toolchain the module targets** (the analyzers key
off the `go` directive) after every version bump and periodically in between.
On 1.27.1 it carries these modernizers: `any`, `atomictypes`, `buildtag`,
`embedlit`, `errorsastype`, `forvar`, `hostport`, `inline`, `mapsloop`,
`minmax`, `newexpr`, `omitzero`, `plusbuild`, `rangeint`, `reflecttypefor`,
`slicesbackward`, `slicescontains`, `slicessort`, `stditerators`,
`stringsbuilder`, `stringscut`, `stringscutprefix`, `stringsseq`,
`testingcontext`, `unsafefuncs`, `waitgroupgo` (`fmtappendf` was dropped in
1.27 and `waitgroup` was renamed `waitgroupgo` — a script that names analyzers
must be updated). Land the result as **one commit per top-level directory**,
not one commit for the tree: a 500-file diff cannot be bisected, reviewed, or
partially reverted, and every file it touches needs its header bumped anyway,
which is a per-file script either way.

Most hunks are behaviour-preserving by construction. These classes are **not**,
or are only under a precondition, and each hunk in them gets read, not skimmed:

| analyzer | what to check before accepting |
| --- | --- |
| `waitgroupgo` | correct **only** when the enclosing loop has per-iteration variables — `go 1.22` or later in the `go.mod` in force for that file. Under an older directive the identical source is a data race. The precondition lives in one `go.mod` line, not in the diff; say so in the commit. Skip sites where `Add` runs under a mutex or `Add(n)` counts work started elsewhere (see the WaitGroup section). |
| `omitzero` | a wire-shape change for any struct that is (or will be) marshalled through `encoding/json/v2`; see the JSON section. |
| `atomictypes` | changes a field's **type** (`int64` → `atomic.Int64`). On an exported struct that is an API change; the field also stops being copyable, so a struct that was passed by value now needs a pointer. |
| `testingcontext` | `t.Context()` is cancelled **before** `Cleanup` functions run. A cleanup that still needs the context (closing a client, flushing) breaks; keep `context.Background()` there or capture a fresh one. |
| `slicessort` | `sort.Slice` → `slices.SortFunc` is unstable → unstable, so nothing changes — but if the *tests* only pass because ties happened to fall one way, they were already wrong. Tie-free fixtures prove nothing about ties. |
| `minmax` | for floats, the `min`/`max` builtins propagate NaN and treat `-0 < +0`; an `if a < b` rewrite did neither. Read float hunks. |
| `stditerators` | the iterator form is lazy: a body that **mutates the collection** while ranging over it changes meaning. (`stringsseq` has no such hazard — strings are immutable — but note the census hand-rewrite `strings.Split(s, "\n")` → `strings.Lines` is *not* this analyzer and is *not* equivalent: `Lines` keeps the terminator on each line.) |

Two things `go fix` will **not** do for you and that a bump PR should:
`go mod tidy` under `go 1.27` merges duplicate `require` blocks into at most two
(comments preserved) — expect a one-time large `go.mod` diff and do not fight
it; and `go test` now runs the `stdversion` vet check, so a file using a symbol
newer than its effective `go` version fails at test time. That is the check
that catches "works on my machine" API use.

## Prefer the current idiom

Not analyzers, or not fully — these are the rewrites a hand pass finds at scale
and that a reviewer should ask for on new code:

- **`new(expr)`** (1.26) for a pointer to a value: `new("x")`, `new(true)`,
  `new(int64(n))`. Delete the `stringPtr`/`boolPtr`/`intPtr` helpers — one repo
  had eight copies of `stringPtr` and 461 call sites.
- **`slices` and `maps`** over `sort` and hand loops: `slices.Sort`,
  `slices.SortFunc`, `slices.Contains`, `slices.Index`, `slices.Collect(maps.Keys(m))`,
  `slices.Sorted(maps.Keys(m))`. Note `sort.Slice` and `slices.SortFunc` are
  both unstable; use `slices.SortStableFunc` when order among equals matters.
- **`min`/`max`** builtins, **`for i := range n`**, **`for x := range seq`**.
- **`sync.OnceValue` / `OnceValues`** instead of a `sync.Once` plus a package
  variable it fills in.
- **`strings.CutLast` / `bytes.CutLast`** (1.27) for the "split on the last
  separator" shape. Check what the old code did on **index 0**: many
  `idx := strings.LastIndex(s, "/"); if idx > 0` sites deliberately treated a
  leading separator as "no split", and `CutLast` returns `found == true` there.
- **`url.URL.Clone` / `url.Values.Clone`** (1.27) instead of a hand copy or a
  `*u` dereference that shares the `User` pointer.
- **`errors.AsType[T]`** (see Error handling) — `go fix errorsastype` does the
  simple form; the sites it leaves are the ones where the target was reused.
- **Generic methods** (1.27): a method may now declare its own type parameters,
  which removes the "generic function taking the receiver as its first
  argument" workaround. Interface methods still cannot be generic, and a generic
  method cannot satisfy an interface, so this is for concrete types only.
- **Struct-literal keys may be any field selector** (1.27): `T{A.B: 1}` sets a
  promoted field directly instead of `T{A: A{B: 1}}`; `go fix embedlit`
  proposes it.
- **Standard-library `uuid`** (1.27) for new UUIDs instead of a third-party
  module. Do **not** convert `oklog/ulid` sites — ULIDs are chosen for being
  time-sortable, which a UUID is not.
- **`httptest.NewTestServer`** (1.27) when a test uses `testing/synctest`: it
  runs over an in-memory `net` so the bubble owns every goroutine.
  `synctest.Sleep` is `time.Sleep` followed by `synctest.Wait`.
- **`http.Server.MaxHeaderValueCount`** (1.27; `DefaultMaxHeaderValueCount`
  applies when unset) on every server we expose, alongside the `ReadHeaderTimeout`/`ReadTimeout`/`IdleTimeout` trio
  that every listener already needs.
- **`crypto/tls.Config.Rand` is deprecated** (1.27): tests that stubbed it use
  `testing/cryptotest.SetGlobalRandom` instead.

Two 1.27 changes that are not idioms but bite an upgrade: `compress/flate` is
faster and produces **different bytes** — a golden file for anything
zip/gzip/zlib/png-encoded will fail and needs regenerating, not a code fix; and
the `go` command now accepts a removed `GODEBUG` setting (e.g. `asynctimerchan`)
only at its final default value, so a `godebug` line in `go.mod` that was
pinning old behaviour now fails to build.

## File headers

Every `.go` file starts with:

```go
// file: internal/package/filename.go
// version: 1.2.3
// guid: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
// last-edited: YYYY-MM-DD
```

Bump version and update `last-edited` on every change.

## General rules

- Use `context.Context` as the first parameter of any function that does I/O or calls other services.
- Return errors explicitly; never panic in library code.
- Prefer table-driven tests; name subtests with descriptive strings.
- Use `slog` for structured logging — no `fmt.Println`, no `log.Printf`.
- Call `slog.SetDefault` **once**, in `main` (or the one function `main` calls
  to set up logging), before anything else runs. A second call anywhere else
  silently replaces the first: one repo configured a rotating file handler in
  `cmd/` and then had `server.New` install a stderr handler, so the file log
  went quiet the moment the server started and nothing reported it. If a
  package needs a differently-configured logger, it takes a `*slog.Logger`
  parameter; it does not touch the process default.
- All exported types and functions must have doc comments.

## Imports

Group imports in three blocks separated by blank lines:

1. Standard library
2. External packages
3. Internal packages (`github.com/falkcorp/...`)

## Error handling

- Wrap errors with context: `fmt.Errorf("doing X: %w", err)`
- Define sentinel errors as `var ErrFoo = errors.New("foo")` — not inline strings.
- Use `errors.Is` for sentinels and `errors.AsType[T]` for typed errors, never
  string matching. `errors.AsType[T]` replaces the `errors.As` two-step:

  ```go
  // Old — declare a target, pass its address, then use it.
  var perr *fs.PathError
  if errors.As(err, &perr) { ... perr.Path ... }

  // Current — one expression, and no zero-valued target left in scope on the
  // false path.
  if perr, ok := errors.AsType[*fs.PathError](err); ok { ... perr.Path ... }
  ```

## Testing

- Test files live alongside the code they test (`*_test.go`, same package or `_test` suffix).
- Use `testify/require` for fatal assertions, `testify/assert` for non-fatal.
- Integration tests that hit the DB or network use `testing.Short()` gate:

  ```go
  if testing.Short() {
      t.Skip("skipping integration test in short mode")
  }
  ```

- Mock interfaces generated by mockery; do not hand-write mocks.
- `-short` is a **budget**, not a label. A package's `go test -short` run should
  finish in well under a minute; the point of the gate is that `make ci` gives
  an answer while you are still looking at it. One repo's database package took
  257 s under `-short` (and timed out at 10 minutes when run alongside a
  coverage job), which means the gate no longer means anything there. When a
  package crosses the minute, move tests behind `testing.Short()` or into a
  build-tagged integration package; do not raise the timeout.
- A test must not depend on **what else is running on the machine**. Two real
  cases: a "daemon is down" test that dialled `127.0.0.1:11434` and failed on
  any developer box with that daemon actually running — point it at an
  `httptest.Server` that returns the failure you mean; and a fixture that
  pinned an external tool's output to a literal (`ffprobe`'s MP3 duration
  estimate, to 0.001 s) and failed on every ffprobe release that estimated
  differently — derive the expected value from the same tool at test time, or
  loosen the tolerance to what the tool actually guarantees.

### Use the isolation the standard library gives you

Reach for the `testing` helper instead of hand-rolling setup and teardown. Each
one is undone automatically at test end:

| Instead of | Use |
|---|---|
| saving and restoring `os.Setenv` | `t.Setenv` (rejects `t.Parallel()` by design) |
| `os.Chdir` with a deferred restore | `t.Chdir` |
| `os.MkdirTemp` + `defer os.RemoveAll` | `t.TempDir` |
| a path-prefix check before opening a file | `os.Root` — the escape is refused, not detected. Applies when there **is** a fixed root; a handler that receives an arbitrary absolute path from a request has nothing to root at and needs validation, not `os.Root` (2 of 18 CodeQL path-injection sites in one repo qualified) |
| `context.Background()` in a test | `t.Context()` / `b.Context()` — cancelled at cleanup, **before** `t.Cleanup` functions run, so not for use inside them |
| `for i := 0; i < b.N; i++` | `for b.Loop()` — no timer juggling, args stay alive |
| keeping a file around to inspect after a failure | `t.ArtifactDir()` |

### Concurrency: `testing/synctest`, not `time.Sleep`

A test that sleeps to let a background goroutine finish is asserting on wall
clock time, which is not the thing it means to assert on. Wrap it in
`synctest.Test` instead: inside the bubble, time is virtual and advances only
once every bubbled goroutine is durably blocked, and `synctest.Wait()` returns
exactly when the others have gone quiet.

```go
synctest.Test(t, func(t *testing.T) {
    got := make(chan string, 1)
    go func() { time.Sleep(time.Hour); got <- "done" }()
    synctest.Wait()          // returns when the goroutine is durably blocked
    time.Sleep(time.Hour)    // virtual — completes instantly
    require.Equal(t, "done", <-got)
})
```

It is a full isolation bubble, not just a fake clock: every goroutine started
inside is tracked, and one that escapes the bubble panics rather than leaking
quietly into the next test.

The failure this prevents is not a red test. It is a timeout constant that
creeps upward over the years because a *correct* implementation keeps losing a
race with the assertion on a loaded CI machine. **A timeout that has been
raised more than once is a defect report, not a tuning parameter.**

**Do not** convert a sleep that stands in for real I/O, a subprocess, or a
database fsync. A virtual clock does not make those faster, and the conversion
will hang instead of passing.

### Globals: the one thing Go will not isolate for you

The helpers above isolate the environment, the working directory, the
filesystem and the context. There is **no** isolator for package-level state.
A function that reads a global cannot be handed a different one by a test, and
the failure is silent rather than loud:

```go
// Hard to test: reads a global. A caller that forgets to configure it gets
// zero values — and a guard that quietly does nothing.
func Cleanup(root string) { app := appconfig.Current(); ... }

// Testable: the dependency is a required parameter. A caller that omits it is
// a COMPILE ERROR, not a misbehaviour discovered in production.
func Cleanup(root string, app AppDirs) { ... }
```

Prefer the second shape for anything a test needs to steer. If a global is
genuinely unavoidable, the only real isolation left is a subprocess re-exec.

### Assert on behaviour, not on rendered output

Assert on the returned value or the stored state, not on a formatted string. A
test that matches log text or a rendered response body breaks when someone
rewords a message — and, worse, keeps passing when an idempotent formatter
renders the broken state and the correct state identically.

## JSON

- **`omitempty` and `omitzero` are not synonyms, and which one a field gets is
  decided by the package that marshals it — the import path, not the
  toolchain.** A struct marshalled through `encoding/json` keeps v1 semantics on
  1.27 (`omitempty` still drops `false` and `0`). The same struct marshalled
  through `encoding/json/v2` *emits* `false` and `0` for `omitempty`; only
  `omitzero` omits them. So switching a file's import from `encoding/json` to
  `encoding/json/v2` is a **wire-shape change** for every bool and numeric field
  tagged `omitempty` (one repo counted 153), and needs the field audit in the
  same PR — not a GOEXPERIMENT toggle somewhere else.
- Use `omitzero` for "leave it out when unset" on scalars; it means the same
  thing under both packages, which is the point. Reserve `omitempty` for the
  cases where you actually mean "leave it out when empty" (len-0 slices, maps,
  strings).
- `go fix`'s `omitzero` analyzer proposes exactly this rewrite. It is the one
  modernizer whose hunks are **not** behaviour-preserving for v2 callers — read
  each one against the consumer of the JSON.

## Concurrency

- Protect shared mutable state behind accessors (`Set*`/`Get*`), and pick the
  primitive by the state's shape, not by habit:
  - a single word — a flag, a counter, a phase — is a **typed atomic**
    (`atomic.Bool`, `atomic.Int64`, `atomic.Pointer[T]`); `go fix atomictypes`
    converts the untyped `atomic.AddInt64(&x, 1)` form;
  - anything wider is a **`sync.Mutex`** by default;
  - `sync.RWMutex` only when a profile shows reads dominating *and* the critical
    section is long enough for reader parallelism to matter. It is not a free
    upgrade: `RLock` is slower than `Lock` on the uncontended path, and a writer
    can be starved by a stream of readers.
- Goroutines launched at startup must be joined on shutdown via a
  `sync.WaitGroup`.

- Bound the fan-out. Never start one goroutine per item over an unbounded
  collection; use `errgroup.Group` with `SetLimit`, or a semaphore channel
  sized to `runtime.NumCPU()` for CPU-bound work.

### `wg.Go(fn)`, never `Add(1)` + `defer Done()`

`sync.WaitGroup.Go` landed in Go 1.25 and is **the** way to start a tracked
goroutine. Adoption across our repos is currently near zero — one repo alone
had 90 `sync.WaitGroup` declarations and **not one** `wg.Go`. (An earlier
version of this section quoted "296 `.Add(1)` calls"; a census showed a raw
`.Add(1)` grep overstates WaitGroup work about **4×** — 121 of 142 sampled hits
were `atomic.Int64.Add(1)`. Count the `go func` that follows, not the `Add`.)
Treat every `Add(1)`-above-`go func` as a site to convert when you next touch
the file; `go fix waitgroupgo` now does the simple shape for you, under the
precondition in the `go fix` section.

```go
// Old — three separate things that must agree, and nothing checks that they do.
wg.Add(1)
go func() {
    defer wg.Done()
    work()
}()

// Current — the counter and the defer are the method's job, not yours.
wg.Go(func() {
    work()
})
```

It is not only shorter. `Add`/`Done` can desynchronise in ways the compiler
cannot see, and both failure modes are bad:

- A **missed `Done()`** on an early `return` or a panicking path hangs `Wait()`
  **forever**. This is the one that shows up as a CI job that times out at 6
  hours instead of failing.
- A **missed `Add(1)`**, or an `Add` placed *inside* the goroutine, lets `Wait()`
  return before the work has run. That one is silent: the test passes, and the
  race only appears under load.

`wg.Go` makes both unrepresentable — there is no counter to get wrong.

**Do not confuse it with `errgroup.Group.Go`,** which has existed for years and
is a different tool: `errgroup` collects the first error and can cancel its
context, and `SetLimit` bounds concurrency. Use `errgroup` when the work can
fail or must be bounded; use `sync.WaitGroup.Go` when it cannot fail and you
only need to join.

Bare `Add` keeps an honest job in more places than "rare" suggests. The census
above found four shapes that must **not** be converted, and a reviewer should
be able to name which one a surviving `Add` is:

- **`Add(n)` up front** for `n` workers started in a loop or elsewhere
  (`wg.Add(workers)`, `wg.Add(2)` for a bidirectional bridge) — the count is
  the point.
- **`Add` under a mutex that `Wait`'s caller also takes**, with the `go`
  statement after `Unlock` — the ordering "counter incremented before anyone
  can observe `Wait`" is the point, and `wg.Go` fuses `Add` and `go` into one
  call so the separation cannot be expressed.
- **`Done` fired from a different goroutine or a callback** — the work is not
  the function body, so there is nothing to hand `wg.Go`.
- **A `Done` on a path `wg.Go` cannot see**, e.g. the goroutine hands the token
  to a pool that signals completion later.

If a call site has `Add(1)` directly above a `go func` whose first line is
`defer wg.Done()`, it is none of these: it is a conversion, not an exception.

- Context cancellation must propagate — check `ctx.Err()` in loops.
