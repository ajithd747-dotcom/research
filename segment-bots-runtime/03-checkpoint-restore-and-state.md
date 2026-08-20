# Checkpoint/restore, or externalise state? — CRIU, pickle, and crash-only software

Researched 2026-08-20 by a `sonnet` agent (Rule 1). Report only; nothing
installed. Several claims verified empirically on this VM rather than from docs.

## Verdict

**CRIU is off the table here — two independent, verified blockers.** For ~300
parts with modest state, **externalise state from the start** (crash-only design)
beats both CRIU and any serialization library on total engineering cost, and it
is the only option that does not fight the zygote-fork architecture.

**This removes the checkpointing obligation from the runtime design entirely,
rather than making it cheaper.**

## 1. CRIU cannot run on this box

Verified on this VM, not inferred:

- Kernel `7.0.0-1008-gcp` — new enough; version is not the problem.
- `CAP_CHECKPOINT_RESTORE` (Linux 5.9) grants **exactly three things**: update
  `/proc/sys/kernel/ns_last_pid`, use `clone3()`'s `set_tid`, and read other
  processes' `/proc/pid/map_files` symlinks. It does **not** by itself allow
  dumping another process's memory.
- Real requirement is a *bundle*: `CAP_CHECKPOINT_RESTORE` (or `CAP_SYS_ADMIN`)
  **plus `CAP_SYS_PTRACE`** (or `yama/ptrace_scope=0`), plus `CAP_NET_ADMIN`,
  `CAP_SYS_CHROOT`, `CAP_SYS_RESOURCE`, `CAP_SETUID` depending on features used.
  (LPC 2022 talk "Unprivileged CRIU"; criu PR #1155.) A 2022 conference talk
  titled *Unprivileged CRIU* is itself the signal that this is a frontier
  feature, not a commodity.

Measured gate state on this machine:

| Setting | Value |
|---|---|
| `kernel.unprivileged_userns_clone` | 1 (enabled) |
| `kernel.apparmor_restrict_unprivileged_userns` | **1 — Ubuntu post-23.10 lockdown, on** |
| `kernel.yama.ptrace_scope` | 1 (restricted) |
| `unshare -U` | succeeds |
| `unshare --user --map-root-user` | **fails** — `write failed /proc/self/uid_map: Operation not permitted` |

The audit log names the cause: AppArmor transitions the new namespace into a
profile called `unprivileged_userns`, whose body is `audit deny capability,`
with no matching allow line. **A blanket deny of every capability inside an
unprivileged user namespace** — `CAP_CHECKPOINT_RESTORE` included.

Independently: no sudo and no apt means CRIU cannot be installed, and
`setcap` on a binary needs `CAP_SETFCAP`, which needs root. File-capability
grants are how non-root CRIU is normally deployed — that path is closed at the
root cause, before AppArmor even applies.

## 2. CRIU would be wrong for this topology even with root

**criu issue #2386 (open, unresolved) is our exact architecture.** A process
allocates 3 GB, forks; the child never touches the memory, so COW keeps the tree
at 3 GB resident. Checkpointing it produces a **6 GB dump**, and with
`--leave-running` the *live* set doubles to 6 GB during the dump. With 19 forked
children the reporter saw **40 GB**.

Cause: CRIU reads `/proc/pid/smaps` permission bits, where forked shared-anonymous
memory carries the same `rw-p` flags as genuinely private memory, so it is
misclassified `VMA_ANON_PRIVATE`; the `vmsplice()` page transfer then **forces
the copy-on-write it was trying to avoid**.

On a 30 GB box **with no swap**, running one warm numpy zygote and many forked
parts, this is not a bug to work around — it is a memory multiplier aimed
directly at the design.

Structural, not just a bug: CRIU requires *"parent-child relations must be kept
intact... not possible to re-parent a process"*. Dumping one part independently of
its zygote, or restoring it under a different parent, is not supported without the
whole tree — the opposite of "turn one part off, leave the zygote and 299 others
running."

**TCP (`--tcp-established`)** works via `TCP_REPAIR`, but needs a live netfilter
rule dropping all peer packets across the whole dump→restore window; miss it and
the kernel sends **RST**, killing the connection. That needs `CAP_NET_ADMIN`.
And for exchange sockets: even with root, you would still need reconnect/replay
logic — at which point you have already built the thing that makes CRIU
unnecessary.

Also uncheckpointable: character/block devices, lazy-unmounted files, ptrace'd
processes, most non-TCP/UDP/UNIX socket types, `O_DIRECT` pipes, corked UDP,
**fds passed over UNIX sockets**, SysVIPC outside an IPC namespace.

## 3. Serialization libraries are a false economy

Tested directly on this box, system Python 3.14.4, stdlib `pickle`:

| Object | Result |
|---|---|
| `socket.socket()` — even unconnected | `TypeError: cannot pickle 'socket' object` |
| `threading.Lock()` | `TypeError: cannot pickle '_thread.lock' object` |
| `threading.RLock()` | `TypeError: cannot pickle '_thread.RLock' object` |
| generator | `TypeError: cannot pickle 'generator' object` |
| open file (`TextIOWrapper`) | `TypeError: cannot pickle 'TextIOWrapper' instances` |
| `threading.Thread` (unstarted) | `TypeError: cannot pickle '_thread._ThreadHandle' object` |
| `lambda` | `PicklingError` — functions pickle **by qualified name**, not by value |

- **dill** adds lambdas, closures, `cell`/`method`/`module`/`code`. Its own docs
  say it still cannot pickle `frame`, `generator`, `traceback`. Nothing about
  sockets or locks — those are not serialization gaps, they are OS-handle problems
  no pure-Python library can solve.
- **cloudpickle** is for shipping code to workers, not persistence. Its README:
  *"Using cloudpickle for long-term object storage is not supported and strongly
  discouraged"*, and it requires the **exact same Python version**.
- **joblib** is a pickle wrapper optimised for large numpy arrays. `mmap_mode` on
  load is documented as incompatible with compressed files, and *"the
  reconstructed object might no longer match exactly the originally pickled
  object"* — pickling a `numpy.memmap` does **not** preserve the memmap linkage.

**The blocking problem was never "can I serialize an array" (yes, trivially). It
is that every part holds live sockets and locks — and a socket's state lives in
the kernel and in the exchange's peer, not in the Python object.** No library
closes that; it is not a maturity gap to wait out.

Also: `pickle.load` / `joblib.load` execute arbitrary code on untrusted input.

## 4. Crash-only software — the recommended design

Candea & Fox, *Crash-Only Software*, HotOS-IX 2003 (https://dslab.epfl.ch/pubs/crashonly.pdf):

> "Crash-only programs crash safely and recover quickly. There is only one way to
> stop such software—by crashing it—and only one way to bring it up—by initiating
> recovery."

> "To make components crash-only, we require that all important non-volatile state
> be kept in **dedicated state stores**, that state stores provide applications
> with the right abstractions, and that state stores be crash-only."

Formalism: `stop = crash`, `start = recover`. The power-off switch is defined as
**external to the component** — *"kill -9... not invoking any of the component's
code"*. That is T-3 of this project's own transistor rule, arrived at
independently 23 years earlier.

The paper separates state by category — transactional, single-writer persistent,
expendable, session, **soft state reconstructable at any time**, volatile — and
argues each deserves a different store, not one universal serializer.

### Cost per component, honestly

| Approach | Per-part cost | Still hand-written regardless |
|---|---|---|
| Hand-written save/load | custom `to_dict`/`from_dict` per part, plus versioning | socket reconnect/replay |
| CRIU | not viable here; elsewhere still needs reconnect logic, plus the fork memory multiplier | socket reconnect, plus privileged-tooling risk |
| pickle/dill/joblib | must still separate picklable data from must-be-rebuilt (sockets, locks) per part — the boundary code is written anyway | socket/lock rebuild |
| **Externalised state + rebuild-on-boot** | **written once** as the T-1 part template; 300 parts inherit it by construction | reconnect *is* ordinary startup — no separate restore path exists |

Turning off = flush numeric buffers (or nothing, if already memory-mapped —
unmapping *is* the flush), let the process exit. Turning on = remap state, reopen
the store, reconnect and resubscribe. **There is no "restore" code, because
startup already does it.**

This is also the only option consistent with T-3: a checkpointed-but-off part
still owns a large on-disk blob and a resume obligation, while externalised
state means off really is off — nothing the process owns is not already durable
outside it.

## 5. Memory-mapped state — where it works, where it breaks

- **`numpy.memmap`**: gives genuine "off = don't touch, on = remap" for **pure
  numeric arrays of fixed shape and dtype** — ideal for rolling market buffers and
  flat parameter arrays. Caveat from the docs: it is "a subclass [that] doesn't
  quite fit properly", and **there is no API to explicitly close the underlying
  mmap** — cleanup is best-effort via refcounting. Test this under a real
  off/on cycle before relying on it.
- **`multiprocessing.shared_memory`**: named block addressable by any process
  that knows the name; genuine cross-process access with no serialization. But it
  hands you **raw bytes**, not objects — you wrap it with a numpy view yourself.
- **LMDB**: memory-mapped B+-tree, crash-safe by design — the "crash-only state
  store" the paper calls for. Key→bytes with ACID transactions. It is the store
  behind the part template, not a substitute for memmap's zero-copy access.

**Breaks the moment state is not a fixed-layout raw buffer:**

- Any `list`/`dict`/object graph stores pointers to heap objects at arbitrary
  addresses. You cannot mmap a `dict` and get a `dict` back — you would be
  mapping CPython's internal representation, which is not stable across a restart.
- `dtype=object` numpy arrays are explicitly excluded from memmap — an object
  array is itself an array of pointers into the Python heap.
- Anything holding a socket, lock, or thread handle — same issue as §3.

So: **learned parameters as flat numeric arrays → memmap, essentially free.**
Structured metadata → SQLite or LMDB with numpy blobs.

## UNVERIFIED

- No authoritative single kernel build option (e.g. `CONFIG_CHECKPOINT_RESTORE`)
  found gating CRIU generally; gating is via the capability model (verified) plus
  per-feature configs. `CONFIG_INET_DIAG_DESTROY` appeared in searches tied to
  socket-diag tooling but was **not confirmed** as a hard `--tcp-established`
  prerequisite in official docs.
- numpy was unavailable on system Python during the agent's run, so
  `numpy.memmap` pickling was not tested locally; the "memmap loses its mmap
  linkage on pickle" claim rests on joblib's documentation.
- CRIU's interaction with free-threaded Python 3.14 is **unverified** — nothing
  CRIU-specific found. Moot here, since CRIU is unusable anyway.

## Sources

- https://man7.org/linux/man-pages/man7/capabilities.7.html
- https://github.com/checkpoint-restore/criu/pull/1155 · LPC 2022 "Unprivileged CRIU"
- https://github.com/checkpoint-restore/criu/issues/2386 (fork/COW memory multiplication)
- https://criu.org/TCP_connection · https://criu.org/What_cannot_be_checkpointed
- https://docs.python.org/3/library/pickle.html#what-can-be-pickled-and-unpickled
- https://github.com/uqfoundation/dill · https://github.com/cloudpipe/cloudpickle
- https://joblib.readthedocs.io/en/stable/persistence.html
- https://dslab.epfl.ch/pubs/crashonly.pdf (Candea & Fox, HotOS-IX 2003)
- https://numpy.org/doc/stable/reference/generated/numpy.memmap.html
- https://docs.python.org/3/library/multiprocessing.shared_memory.html
