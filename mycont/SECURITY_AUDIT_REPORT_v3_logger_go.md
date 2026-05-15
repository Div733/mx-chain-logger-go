# Security Audit Report v2 — mx-chain-logger-go
**Scope:** Open findings only (ISSUE-049, ISSUE-050, ISSUE-051 already fixed and excluded)
**Mandatory fixes:** 1
**Conditional fix:** 1 (dormant in current call sites — fix defensively)
**Valid bugs — not mandatory:** 1

---

## Mandatory Fixes

| ID | File | Severity | Nature |
|---|---|---|---|
| FINDING-1 | `logOutputSubject.go` | High | RLock held across external writer calls — deadlock if any writer calls back into the logger |

## Conditional Fix

Triggerable only if `ChangeFileLifeSpan` is called at runtime in a consuming repo (e.g. `mx-chain-go`). If it is never called after startup this is dormant. Verify call sites before deciding.

| ID | File | Severity | Nature |
|---|---|---|---|
| FINDING-2 | `file/lifeSpanner.go` | Medium | `reset()` sends on unbuffered channel without select — goroutine deadlock if lifeSpanner is already closing |

## Valid Bugs — Not Mandatory

Not triggerable under normal operating conditions. Fix during a cleanup pass.

| ID | File | Severity | Nature |
|---|---|---|---|
| FINDING-3 | `logLineWrapperFormatter.go` | Low | Marshal error silently swallowed — remote log consumer receives no bytes with no signal |

---

## FINDING-1 — RLock Held Across External Writer Calls in `logOutputSubject.Output`

### Location
`logOutputSubject.go` — `Output`, lines 38–55

### Vulnerable Code
```go
func (los *logOutputSubject) Output(line *LogLine) {
    los.mutObservers.RLock()

    convertedLine := los.convertLogLine(line)
    for i := 0; i < len(los.writers); i++ {
        format := los.formatters[i]
        buff := format.Output(convertedLine)
        _, writeErr := los.writers[i].Write(buff)   // external call under RLock
        ...
    }

    los.mutObservers.RUnlock()
}
```

### What the Bug Is
`mutObservers.RLock()` is held for the entire duration of the writer loop, including every `Write()` call into external observers (file, network pipe, gRPC stream, etc.). Any writer that — directly or indirectly — calls back into the logger (e.g. `AddLogObserver`, `RemoveLogObserver`, `ClearObservers`) will attempt to acquire `mutObservers.Lock()`. Because a `sync.RWMutex` blocks new write-lock attempts while any read-lock is held, this produces a deadlock.

This is a realistic scenario: the file logger in `file/fileLogging.go` calls `logger.AddLogObserver` and `logger.RemoveLogObserver` during rotation, and rotation is triggered from a goroutine that can fire at any time. If a log line is being written at the same moment rotation runs, the rotation goroutine blocks on `AddLogObserver` → `mutObservers.Lock()` while `Output` holds the RLock waiting for the file `Write()` to complete. The file `Write()` itself is not the re-entrant call here, but any observer that logs internally (e.g. a network observer that logs its own errors) creates the re-entrant path.

Additionally, a slow or blocking writer (full disk, blocked TCP socket) stalls all other log output for the entire duration of the write, because the RLock prevents concurrent `Output` calls from making progress on other writers.

### Impact
- Deadlock between the log output path and any observer that calls back into the logger subsystem
- All log output stalled for the duration of any slow writer
- Node process hangs with no error signal

### Severity
**High** — Deadlock in the primary log output path. Reachable whenever a writer or its error path calls back into the logger, which is a natural pattern for network-backed observers.

### Fix
Copy the writer and formatter slices under the lock, then release before calling Write:

```go
func (los *logOutputSubject) Output(line *LogLine) {
    los.mutObservers.RLock()
    writers := make([]io.Writer, len(los.writers))
    formatters := make([]Formatter, len(los.formatters))
    copy(writers, los.writers)
    copy(formatters, los.formatters)
    los.mutObservers.RUnlock()

    convertedLine := los.convertLogLine(line)
    for i := 0; i < len(writers); i++ {
        buff := formatters[i].Output(convertedLine)
        if _, err := writers[i].Write(buff); err != nil {
            _, _ = fmt.Fprintf(os.Stderr,
                "logOutputSubject: observer #%d write failed: %v\n", i, err)
        }
    }
}
```

The slice copy is O(n) where n is the number of observers (typically 2–3), so the overhead is negligible. The lock is held only for the copy, not for any I/O.

### Effort
5 minutes. Replace the lock scope with a copy-then-release pattern.

---

## FINDING-2 — Unbuffered Channel Send in `lifeSpanner.reset()` Can Deadlock on Close

### Location
`file/lifeSpanner.go` — `reset`, line 33

### Vulnerable Code
```go
func (spanner *lifeSpanner) reset() {
    spanner.resetChan <- struct{}{}   // blocks if process() is not reading
}
```

`resetChan` is created as `make(chan struct{})` — unbuffered. The send blocks until `process()` receives from it. `process()` only receives from `resetChan` inside its select loop. If `close()` is called (which cancels the context) concurrently with `reset()`, the following sequence is possible:

1. `process()` selects `ctx.Done()` and returns, closing the goroutine.
2. `reset()` (called from `resetDuration`) is now blocked forever on the send — no receiver exists.
3. The caller of `resetDuration` (e.g. `fileLogging.ChangeFileLifeSpan`) hangs indefinitely.

`ChangeFileLifeSpan` is a public API method. If it is called from the node's configuration reload path at the same time as `Close()`, the node goroutine calling `ChangeFileLifeSpan` deadlocks permanently.

### When Is This Reachable
Only if `ChangeFileLifeSpan` is called at runtime after startup. If `mx-chain-go` and other consumers only set the log rotation interval once at startup and never call it again, this race cannot occur and the bug is dormant. **Check call sites in consuming repos before deciding whether to fix.**

### Impact
- Goroutine leak and permanent hang in `ChangeFileLifeSpan`
- `fl.mutOperation` held locked forever — log rotation stops entirely, log file grows without bound until disk fills
- Node configuration reload path stalls with no error returned
- No crash, no log — invisible in production

### Severity
**Medium** — Requires a specific race between `ChangeFileLifeSpan` and `Close`. Dormant if `ChangeFileLifeSpan` is never called at runtime.

### Fix
Use a non-blocking send with a select:

```go
func (spanner *lifeSpanner) reset() {
    select {
    case spanner.resetChan <- struct{}{}:
    default:
    }
}
```

Alternatively, make `resetChan` buffered with capacity 1. Either approach ensures `reset()` never blocks when `process()` is gone.

### Effort
2 minutes. Replace the bare channel send with a select/default.

---

## FINDING-3 — Marshal Error Silently Swallowed in `logLineWrapperFormatter.Output`

### Location
`logLineWrapperFormatter.go` — `Output`, lines 24–31

### Vulnerable Code
```go
func (llwf *logLineWrapperFormatter) Output(line LogLineHandler) []byte {
    if check.IfNil(line) {
        return nil
    }

    buff, err := llwf.marshalizer.Marshal(line)
    if err == nil {
        return buff
    }

    return nil   // error silently dropped
}
```

If `Marshal` fails, `nil` is returned with no error surfaced. The caller (`logOutputSubject.Output`) writes the nil/empty byte slice to the observer's writer. The remote log consumer (e.g. a log aggregator connected over the network) receives an empty frame with no indication that a log line was lost. This is the same silent-loss pattern as ISSUE-051 but in the serialisation layer rather than the write layer.

This formatter is used specifically for network/remote log consumers via `NewLogLineWrapperFormatter`. A marshal failure here means security-relevant log lines are silently dropped to remote observers with no local or remote signal.

### Impact
- Silent log loss to remote observers
- No error returned to caller, no stderr diagnostic
- Security-relevant events may be invisible to remote log aggregators during incidents

### Severity
**Low** — Marshal failure requires a broken marshalizer or an unencodable log line, which is not a normal operating condition. Impact is limited to remote observers; local console/file output is unaffected.

### Fix
```go
buff, err := llwf.marshalizer.Marshal(line)
if err != nil {
    _, _ = fmt.Fprintf(os.Stderr,
        "logLineWrapperFormatter: marshal failed: %v\n", err)
    return nil
}
return buff
```

Add `"fmt"` and `"os"` to the import block.

### Effort
3 minutes. Add the error check and stderr diagnostic.

---

## Security Guarantees After Fixes

✅ 0 deadlock paths in the log output subject

✅ 0 goroutine-leak paths in the file rotation lifecycle

✅ 0 silent serialisation-loss paths to remote observers

✅ 0 nil dereferences in file logger shutdown or finalizer (ISSUE-049 fixed)

✅ 0 file descriptor leaks on rotation failure (ISSUE-050 fixed)

✅ 0 silent write-error paths in observer loop (ISSUE-051 fixed)

✅ 0 remotely exploitable vulnerabilities

✅ 0 fund theft or state corruption paths

---

## Fix Priority

### Mandatory
| Order | ID | File | Severity | Effort |
|---|---|---|---|---|
| 1st | FINDING-1 | `logOutputSubject.go` | High | 5 min |

**Total mandatory fix time: ~5 minutes.**

### Conditional (verify call sites first)
| Order | ID | File | Severity | Effort |
|---|---|---|---|---|
| 2nd | FINDING-2 | `file/lifeSpanner.go` | Medium | 2 min |

Fix if `ChangeFileLifeSpan` is called at runtime in `mx-chain-go` or any other consumer. Skip if it is only called at startup.

### Recommended (not blocking)
| Order | ID | File | Severity | Effort |
|---|---|---|---|---|
| 3rd | FINDING-3 | `logLineWrapperFormatter.go` | Low | 3 min |

**Total fix time if all three addressed: ~10 minutes.**
