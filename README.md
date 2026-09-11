# pthreads-win32 — pinned for the NVP DSP

Third-party. This repo exists so the NVP DSP's pthreads dependency has an identified, pinned
source instead of a build-time download — the same liability ENG-268 fixed for sofia-sip.

**Nothing here is Noetica code.** `src/` is upstream pthreads-win32 under LGPL 2.1;
`prebuilt/` holds the binaries the DSP already ships.

Licence files, and an upstream inconsistency relevant to **ENG-114**: `src/COPYING` states the
LGPL 2.1 terms and refers the reader to `COPYING.LIB` — but this tarball does **not** contain
`COPYING.LIB`. The licence text ships as `src/COPYING.FSF` instead ("GNU LESSER GENERAL PUBLIC
LICENSE, Version 2.1, February 1999"). So the notice is present but misfiled upstream; anyone
assembling LGPL notices should cite `COPYING` plus `COPYING.FSF`, not the name `COPYING` points at.

## The pin

| | |
| --- | --- |
| Source | `http://files.freeswitch.org/downloads/libs/pthreads-w32-2-9-1.tar.gz` |
| Tarball SHA256 | `efcc3901f889edc946352dd5ae5e4eef5e7a2d9f3af04f0ce88725abe7f7660d` |
| Declared version | `PTW32_VERSION 2,10,0,0` (`src/pthread.h:41`) |
| Build flavour | `__CLEANUP_C` |
| Retrieved | 2026-09-11 (ENG-268) |

### The directory name is a label, not the version

The tarball and its directory are named `pthreads-w32-2-9-1`, but the source inside declares
**2.10.0.0** — a post-2.9.1 CVS snapshot repackaged under the old filename. Upstream 2.9.1 proper
declares `2,9,1,0`.

This is the same trap that cost time on the sofia-sip pin, where a *generated* `SOFIA_SIP_VERSION`
was mistaken for a tag. Believe the source, not the label. If you are matching a binary, match on
`PTW32_VERSION` and the export surface.

### Why this tarball is the right pin

It is where the shipped binary actually came from:

- FreeSWITCH fetches exactly this URL (`w32/download_pthreads.props` in `signalwire/freeswitch`),
  and `libs/win32/pthread/pthread.2017.vcxproj` builds it with `__CLEANUP_C` to a DLL named
  `pthread.dll` — which is what `nub_nvp_dsp` ships.
- `nvp_freeswitch` gitignores `/pthreads-w32-2-9-1/`, so the 2023 checkout was lost from version
  control exactly as the sofia source was. That is why this repo exists.
- The shipped `pthread.dll` version resource reads `2,10,0,0`, matching this source.

Export fingerprint against the shipped `VoiceDsp/libRelease/pthread.dll`:

- 135 exports in the DLL; 124 declared across `pthread.h`, `sched.h`, `semaphore.h`.
- Exactly one declared symbol is absent — `pthread_win32_set_terminate_np`, which exists only under
  `__CLEANUP_CXX`. Its absence is what establishes the `__CLEANUP_C` flavour.
- The 12 DLL-only symbols are internals not in the public headers: `_sched_affinitycpu*` (9) plus
  `ptw32_get_exception_services_code`, `ptw32_pop_cleanup`, `ptw32_push_cleanup`.

## Layout

```
src/        the pinned upstream tree, unmodified (headers sofia compiles against)
prebuilt/   the binaries nub_nvp_dsp ships today
```

| File | SHA256 | Note |
| --- | --- | --- |
| `prebuilt/pthread.lib` | `67c30a4ab19211439e1e748fd507809b305291ca76e87de567444900cbac2c80` | import lib; **identical** in the DSP's `libDebug` and `libRelease` |
| `prebuilt/pthread.Release.dll` | `5d47d83f277a9d08fa88a199c5a7c70baf388d6a048c2fc33fde602e2553e206` | `VoiceDsp/libRelease/pthread.dll` |
| `prebuilt/pthread.Debug.dll` | `319f408ec3790a286ca24b3eacec88cda33f97e5d21d3f9e26be0fc3563e9a32` | `VoiceDsp/libDebug/pthread.dll` |

One import lib serves both configurations: it only describes the DLL's export surface, and both
DLLs export the same 135 symbols.

Note that `nub_nvp_dsp` also carries a **third**, older `VoiceDsp/DLLs/pthread.dll` — 49152 bytes,
SHA256 `b70eade6400b9b38292d986de3a55f081fd17012c484206582130a5bd6723db3` — which differs from both
of the above. Same stale-artifact pattern as the superseded sofia DLL. It is not mirrored here;
which `pthread.dll` actually deploys is tracked on ENG-268.

## Why the binaries are pinned rather than rebuilt

`pthread.dll` is a **shipped** DSP dependency. ENG-268's spec excludes "any DSP behaviour change
beyond the `nta_msg_ackbye` patch", and its unpatched-DLL regression gate already carries merged
source-pin, toolset and linkage risk. Rebuilding pthreads would add a fourth variable to that gate
for no benefit, so we pin what we ship.

`src/` is here for provenance and for the headers, not because anything builds it. If pthreads ever
does need rebuilding, this is the source to do it from — and it would need its own regression gate.

## Consumers

- **`Noetica/sofia-sip`** — its DLL build compiles against `src/` (for `pthread.h`, reached via
  `su_port.h` because the pinned sofia config sets `SU_HAVE_PTHREADS (1)`) and links
  `prebuilt/pthread.lib`.
- **`Noetica/nub_nvp_dsp`** — ships `pthread.dll` beside the DSP executable.

Do **not** define `PTW32_STATIC_LIB` against this pin. The DSP consumes pthreads dynamically —
`pthread.lib`'s archive members are `pthread.dll/`, i.e. it is an import library.
