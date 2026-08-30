<!-- file: instructions/go.md -->
<!-- version: 1.2.0 -->
<!-- guid: 6a15e7db-6d51-493a-87a6-6f4fe14a7f84 -->
<!-- last-edited: 2026-08-30 -->

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
GOTOOLCHAIN=go1.27.0 go build ./... && GOTOOLCHAIN=go1.27.0 go vet ./...
```

### Worked example: audiobook-organizer is blocked on 1.27, deliberately

`audiobook-organizer` stays on 1.26 and the reason is a dependency, not
reluctance. Bumping `go.mod` to `go 1.27.0` fails:

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

**A second blocker in the same repo, of the opposite kind.** Once `swiss` is
unblocked, `audiobook-organizer` still will not compile on 1.27, because *our
own* code uses an API that Go removed:

```text
internal/metadata/audible.go:184:     undefined: json.DiscardUnknownMembers
internal/metadata/audible.go:218:     undefined: json.DiscardUnknownMembers
internal/metadata/googlebooks.go:121: undefined: json.DiscardUnknownMembers
```

`encoding/json/v2` declared `DiscardUnknownMembers` in 1.26 and **dropped it in
1.27** (`RejectUnknownMembers` survives in both). This is the price of the
`GOEXPERIMENT=jsonv2` opt-in that this document requires elsewhere: **an
experiment's API is explicitly not covered by the Go compatibility promise and
can change between releases.** Budget for that when you opt in, and expect the
toolchain bump — not the experiment flag — to be where you discover it.

Here the fix is to delete the option: unknown members are *ignored by default*,
so `DiscardUnknownMembers(true)` only ever requested the default behaviour.
Which is the general shape — an experiment API that disappears is usually one
that was redundant or is being renamed, so check the current docs before
reaching for a shim.

Distinguish the two, because they have different owners:

| blocker | owner | action |
| --- | --- | --- |
| a dependency gating itself off the new toolchain | upstream | wait, record it by name |
| our own use of a changed or removed API | us | fix it now, don't wait |

Two rules follow, and both matter more than the version number:

- **Never force it.** Do not set `-tags untested_go_version`, do not fork the
  dependency, do not `replace` it with a patched copy to get the toolchain bump.
  That converts a loud compile error into exactly the silent runtime corruption
  the gate exists to prevent.
- **Wait for upstream, and record why.** The blocker belongs in the repo's
  `go.mod` neighbourhood or its TODO with the dependency named, so the next
  person does not re-derive it. Re-run the probe above when the dependency
  updates; the bump is then a one-line change.

### JSON v2

A project that opts into JSON v2 must set `GOEXPERIMENT=jsonv2` on **every**
build path — local shell, CI, release builders, Docker images. A local `.envrc`
alone is not enough: a builder that misses the flag compiles different
marshalling behaviour than the one you tested.

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

### Use the isolation the standard library gives you

Reach for the `testing` helper instead of hand-rolling setup and teardown. Each
one is undone automatically at test end:

| Instead of | Use |
|---|---|
| saving and restoring `os.Setenv` | `t.Setenv` (rejects `t.Parallel()` by design) |
| `os.Chdir` with a deferred restore | `t.Chdir` |
| `os.MkdirTemp` + `defer os.RemoveAll` | `t.TempDir` |
| a path-prefix check before opening a file | `os.Root` — the escape is refused, not detected |
| `context.Background()` in a test | `t.Context()` / `b.Context()` — cancelled at cleanup |
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

- Under `GOEXPERIMENT=jsonv2`, **`omitempty` and `omitzero` are not synonyms.**
  v2 *emits* `false` and `0` for `omitempty`; only `omitzero` omits them. Every
  bool and numeric field that relied on v1 behaviour changes wire shape.
- Use `omitzero` for "leave it out when unset". Reserve `omitempty` for the
  cases where you actually mean "leave it out when empty" (len-0 slices, maps,
  strings).

## Concurrency

- Protect shared mutable state with `sync.RWMutex`; always use `Set*`/`Get*` accessors.
- Goroutines launched at startup must be joined on shutdown via a
  `sync.WaitGroup`.

- Bound the fan-out. Never start one goroutine per item over an unbounded
  collection; use `errgroup.Group` with `SetLimit`, or a semaphore channel
  sized to `runtime.NumCPU()` for CPU-bound work.

### `wg.Go(fn)`, never `Add(1)` + `defer Done()`

`sync.WaitGroup.Go` landed in Go 1.25 and is **the** way to start a tracked
goroutine. Adoption across our repos is currently near zero — one repo alone has
90 `sync.WaitGroup` declarations, 296 `.Add(1)` calls (77 of them immediately
followed by `go func`) and **not one** `wg.Go`. Treat every one of those as a
site to convert when you next touch the file.

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

The remaining honest use for bare `Add` is a count that is **not** one-per-
goroutine — adding `n` up front for work started elsewhere. That is rare. If a
call site has `Add(1)` directly above a `go func`, it is a conversion, not an
exception.

- Context cancellation must propagate — check `ctx.Err()` in loops.
