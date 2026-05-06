---
title: "Yocto build tunables and their hidden costs"
date: 2026-05-06T04:00:00Z
draft: false
description: "The handful of local.conf knobs that make Yocto builds usable, and the failure modes each one buys you in return."
ShowToc: true
ShowReadingTime: true
tags:
  - "yocto"
  - "bitbake"
  - "ccache"
  - "build-systems"
  - "embedded"
  - "linux"
  - "tooling"
categories:
  - "linux"
  - "tooling"
outputs:
  - "HTML"
---

Every Yocto user eventually copies the same handful of tunables into `local.conf` to make builds bearable. ccache, a parallelism bump, a longer fetch timeout, a couple of PREMIRROR lines, an image-features prune. The recipe gets passed around in chat, lands on a wiki, gets forked into a layer. What rarely gets passed around is the failure mode each line buys you. Every one of these knobs swaps "slow" for "different failure mode", and the new mode shows up at the worst possible time - mid-fetch on a CI runner, or at link time when the box runs out of memory.

This is the list of knobs I keep in front of every Yocto build, and the catch each one carries. Nothing here is exotic. The point is that they're all routine, all worth turning on, and all under-documented in the failure-mode direction.

## TL;DR

- `INHERIT += "ccache"` is free until a recipe inherits `cmake.bbclass` and trips the launcher trap. Be ready to clear `CMAKE_*_COMPILER_LAUNCHER` per-recipe.
- The default `MIRRORS` table still points at hosts that have been dead for years. Replace it.
- `PREMIRRORS:prepend` for `github.com` turns transient outages into silent fallbacks, but only if you keep your eye on the mirror's currency.
- `BB_FETCH_TIMEOUT = "600"` makes hangs visible later, not less likely. Pair it with retries.
- `FETCHCMD_wget` overrides exist because some CDNs reject the default User-Agent. They're brittle by design - revisit them when bitbake bumps its fetcher.
- Fork PREMIRRORs with `protocol=file` are the fastest source you can have, until the path doesn't exist on a peer's machine.
- `IMAGE_FEATURES:remove` shrinks images by stripping the tools you'll wish you had next time you crash on the device.
- Bundle all of this in one include file and stop reinventing it per project.
- None of this matters until you've measured the delta on your own workload. Each tune earns its keep or it doesn't, and the failure modes are real either way.

## The shape of the problem

Out of the box, a clean Yocto build of `core-image-minimal` does roughly what you'd expect: clones every layer, fetches every source twice (once on first go, once after the first one fails), compiles every C library three times for three machine flavors, links everything that ever moved, and writes 30 GB of intermediate state to disk. The first build takes hours. Most of that time isn't compilation - it's fetch retries, parallel parser fork-out, and re-running the same compiler over and over because nothing is cached.

The community has a stable answer to most of this. The answer is six or seven `local.conf` lines. Each one removes a specific bottleneck. Each one introduces a specific failure mode you didn't have before.

## ccache and the launcher trap

```text
INHERIT += "ccache"
CCACHE_DIR = "/path/to/persistent/ccache"
```

The fastest way to make a re-build cheap. ccache hashes the preprocessed source and the compiler invocation; if the hash matches, it returns a cached object file. On a warm cache, the C/C++ compile phase essentially disappears.

There are two non-obvious failures.

**The cache is invalidated by anything that changes the compiler invocation.** Toolchain version bump, `-march` change, even a path-prefix change in build directories can blow it up. The cache will rebuild, but you'll be confused for a build cycle wondering why it isn't hitting.

**The launcher trap.** ccache is wired in as a *compiler launcher* - bitbake's `ccache.bbclass` sets `CC = "ccache <real-cc>"` so every compile flows through it. CMake recipes (anything inheriting `cmake.bbclass`) bake the same launcher into `CMAKE_C_COMPILER_LAUNCHER` and `CMAKE_CXX_COMPILER_LAUNCHER`. That's fine for normal compilation. It's not fine for recipes that use `add_custom_command(... COMMAND ${CMAKE_CXX_COMPILER} ...)` and treat the compiler variable as a literal shell token. The launcher prefix turns it into `"ccache g++"` quoted as a single argv[0], which `/bin/sh` cannot resolve. The recipe fails with a cryptic "no such file or directory" pointing at a path that contains a space.

`renderdoc` is the canonical example. It pulls in over `meta-virtualization` -> `meta-oe` -> `packagegroup-fsl-tools-gpu-external` chains and breaks the moment ccache is on. The fix isn't `CCACHE_DISABLED:pn-renderdoc = "1"` - that disables ccache for the recipe but leaves the CMake launcher variables baked in at parse time. You have to clear them directly:

```text
CMAKE_CXX_COMPILER_LAUNCHER:pn-renderdoc = ""
CMAKE_C_COMPILER_LAUNCHER:pn-renderdoc = ""
```

Once you've found one of these the lesson generalises: any time a recipe-specific `pn-` override is needed for ccache to coexist with cmake, you're papering over a recipe bug. File it upstream too, but ship the override.

## The fetch layer

The single biggest source of failed CI runs is the network. Bitbake's fetcher is robust on the third try, fragile on the first.

### Replacing the dead MIRRORS table

```text
MIRRORS = " \
    git://.*/.*     http://downloads.yoctoproject.org/mirror/sources/ \n \
    http://.*/.*    http://downloads.yoctoproject.org/mirror/sources/ \n \
    https://.*/.*   http://downloads.yoctoproject.org/mirror/sources/ \n \
    ftp://.*/.*     http://downloads.yoctoproject.org/mirror/sources/ \n \
"
```

The vanilla `MIRRORS` table inherited from `meta` still references hosts that haven't served files reliably in years - `sources.openembedded.org` being the headline offender. The fallback path ends up trying a dead host, waiting for it to time out, and only then giving up. Replacing the whole table with a catch-all that points at `downloads.yoctoproject.org` cuts seconds off every miss.

The catch: a catch-all means the yocto mirror is now a single point of failure. If `downloads.yoctoproject.org` is unreachable, every fetch falls back to the original URI on the recipe's own schedule. In practice this is fine - the yocto mirror is more reliable than the average upstream - but if you're operating in an air-gapped environment, swap in your own mirror host instead of removing the entry.

### `PREMIRRORS:prepend` for GitHub

```text
PREMIRRORS:prepend = " \
    git://github.com/.*/.*    http://downloads.yoctoproject.org/mirror/sources/ \n \
    http://github.com/.*/.*   http://downloads.yoctoproject.org/mirror/sources/ \n \
    https://github.com/.*/.*  http://downloads.yoctoproject.org/mirror/sources/ \n \
"
```

GitHub has weather. When it's stormy your build dies on `git fetch`. `PREMIRRORS` is consulted *before* the original URI, so prepending github routes to a mirror turns transient github outages into silent fallbacks. The mirror almost always has the SHA you need because Yocto's tarball mirroring runs nightly against published Yocto layer recipes.

The catch is small but real: you eat one round-trip on every `PREMIRRORS` miss, because bitbake tries the mirror, fails, then tries the upstream. For a recipe pinned by SRCREV that the mirror happens not to have, this is wasted time. In practice it's pennies versus the dollars you lose to a github stall.

### The `FETCHCMD_wget` User-Agent override

```text
FETCHCMD_wget = "/usr/bin/env wget --tries=2 --timeout=100 -U 'bitbake/2.0'"
```

This one only matters if you build any rust recipe in `meta-virtualization` (or anything else fetching `crate://` URIs). The Fastly endpoint that fronts crates.io rejects HTTP requests with the default `Wget/*` User-Agent on some routes. The error is `403 Forbidden`, which is opaque if you don't know to look. cargo itself works fine because its user agent isn't `Wget/*`. Other CDNs serve wget cleanly. It's a Fastly routing detail.

The override forces a known-acceptable UA. The catch: pinning the User-Agent hides any legitimate wget behaviour change between versions, and you have to revisit the line whenever bitbake's default fetcher options change. It's the kind of fix that ages badly. Leave a comment next to it pointing at the bug, so the next person knows it's a workaround and not a free knob.

### Local fork PREMIRRORs

```text
PREMIRRORS:prepend = "git://github.com/me/my-fork.* git:///path/to/local/checkout;protocol=file \n "
```

If you're iterating on a kernel fork or a u-boot fork, the fastest source on earth is the one already cloned on your disk. `git://...;protocol=file` lets bitbake clone from a local path as if it were a remote. On second build it's milliseconds.

Two catches. The path has to be reachable from inside the build environment - if you build in a container, that path needs to be a bind-mount, not a host path. And if the fork directory isn't there, the fetcher tries the file URL once, gets ENOENT, and falls through to the upstream URI. That's the "right" behaviour but it produces a confusing single-line error in the log on first encounter. Fix is to either materialize the fork directory unconditionally before building, or pre-flight a check that warns when the directory is absent.

## Parallelism, timeouts, and image weight

```text
BB_NUMBER_THREADS = "${@os.environ.get('NPROC', '16')}"
PARALLEL_MAKE = "-j ${@os.environ.get('NPROC', '16')}"
BB_FETCH_TIMEOUT = "600"
IMAGE_FEATURES:remove = "dev-pkgs dbg-pkgs tools-sdk tools-debug staticdev-pkgs"
```

**Parallelism.** Setting `BB_NUMBER_THREADS` and `PARALLEL_MAKE` from `NPROC` lets the same `local.conf` run sensibly on a workstation and a 64-core build farm. The catch is that the right number of bitbake parser threads is not the right number of make threads, and neither one scales linearly. Past a certain point you trade compile speed for memory pressure - link-stage OOMs on `chromium-x11` or `qtwebengine` are a common surprise on a 32 GB box. Cap `PARALLEL_MAKE` lower than `BB_NUMBER_THREADS` if you have memory-fat recipes.

**Fetch timeout.** The default per-URI timeout is around 45 seconds. Bumping it to 600 makes builds tolerant of slow mirrors. The catch is that a misconfigured mirror now hangs your build for ten minutes before failing. The cleaner shape is short timeout plus retries, which bitbake also supports - tune both together rather than picking the longest-possible timeout and walking away.

**Image features.** Stripping `dev-pkgs`, `dbg-pkgs`, `tools-sdk`, `tools-debug`, and `staticdev-pkgs` shrinks an image by hundreds of megabytes. Until the day a customer-facing build crashes on the device and the absence of `gdb` means you cannot read the stack trace. Pin two image variants - a slim production image and an inflated dev image - rather than stripping unconditionally.

## Bundle them once, stop forgetting

The shape of every project ends up the same. A `local.conf` that grew organically, half a dozen of these tunables scattered across it, no comments explaining which ones are workarounds and which ones are policy. New layers come along, the wrong half gets duplicated, and after the third project nobody remembers why `FETCHCMD_wget` is set.

The way out is to put the entire stack in one include file - kas overlay, layer-conf snippet, `local.conf.append` shipped from a tooling repo, your call. One file. Comments next to every line that explains the failure mode the line is buying you. Reviewable as a unit. Versioned alongside the rest of the build infrastructure.

The same file, applied identically across NXP, TI, Renesas, RZ, whatever silicon you're on - because none of these knobs are vendor-specific. The recipes that need ccache disabled vary by what you pull in, but the *shape* of the tuning - ccache wiring, fetch resilience, fork acceleration, image trim - is the same on every Yocto build I've ever seen.

## Bigger levers worth pulling next

The tunables above are the cheap end of the curve - one-line changes that pay for themselves on the next build. Past that point the wins get bigger but so does the setup cost. These are the directions I'd reach for next when a single-host build still isn't fast enough.

**Shared `SSTATE_MIRRORS`.** ccache speeds up the compile phase; sstate skips it entirely. A shared sstate cache - on NFS, on S3, on a build-server HTTP endpoint - lets a fresh checkout pull a previous machine's already-compiled task outputs instead of rebuilding them. On a CI farm the first build of a release candidate goes from hours to minutes if the sstate has already been warmed by a prior run. The catch is signature drift: if the host setups diverge (different host gcc, different distro versions, different uid/gid layouts) the hashes won't match and the cache becomes dead weight. Treat the sstate mirror as an artefact you publish from a known-good build host, not as a free-for-all.

**Hash equivalence server (`BB_HASHSERVE`).** Modern Yocto computes hash equivalence so that two recipes producing the same output get the same task hash even when their inputs differ in ways that don't matter. The local in-process server is on by default; pointing `BB_HASHSERVE` at a shared external server extends the same logic across machines. Pairs naturally with `SSTATE_MIRRORS`. The catch is that the server is now a shared piece of infrastructure that has to be available, healthy, and run a compatible Yocto version with the clients.

**Distributed compile with icecream or distcc.** Once a single host is parallelism-saturated, the next move is offloading compiles to a LAN of build slaves. `INHERIT += "icecc"` plus an icecream daemon network multiplies your effective core count. The catch is operational - the daemons have to be running, the cross-compilers have to be present on every node, and a flaky machine in the pool slows everyone down. Worth it on a dedicated build farm; usually not worth it on a small team's mixed-purpose desktops.

**`rm_work` for disk pressure.** `INHERIT += "rm_work"` deletes a recipe's `WORKDIR` once the recipe has packaged successfully. On a full-image build that's tens of gigabytes back. The catch is debugging: when a downstream recipe fails and you need to look at the upstream recipe's `${B}` to understand why, the directory is gone. The workaround is `RM_WORK_EXCLUDE` for the recipes you actively iterate on, but that's exactly the list that keeps changing.

**tmpfs `TMPDIR`.** Mount `tmp/` on tmpfs and the I/O overhead disappears. On a 64 GB machine with `core-image-minimal` this is a clean win. On a 32 GB machine building a multimedia stack, the OOM killer will visit you, and bitbake handles `kill -9` poorly - half-finished tasks leave broken state in the sstate cache. Either commit the RAM or stay on disk; the middle ground bites.

**Buildstats and `bitbake -P` profiling, before any of the above.** Running with `INHERIT += "buildstats"` and reading `buildstats/<run>/<recipe>/` tells you where time actually goes. The answer is rarely intuitive on a real workload - you'll find one specific recipe (usually the kernel, or webkit, or a clang-built C++ pile) eating disproportionate wall time, and the right next step might be `PACKAGECONFIG` pruning rather than any of the systemic levers above. Measure, then pick the lever that targets the bottleneck the measurement found.

## Validate on your hardware first

This is the step the rest of the post is useless without. Every tune in this list trades one failure mode for another, and the speedup is workload-dependent - the kernel-heavy build that wins big from ccache may be a small win for a build dominated by fetch time, and the parallelism that's a clean win on a 64-core/128 GB box is an OOM generator on a 16-core/32 GB laptop. You don't know which side of those curves you're on until you measure.

The protocol is straightforward and worth running before you adopt any of this in CI:

**1. Cold A/B, same machine, same workspace.** Wipe `tmp/`, build with no tunes, record `/usr/bin/time -v kas-container build <yaml>` (or whatever your invocation is). Save `Elapsed (wall clock) time` and `Maximum resident set size`. Then wipe `tmp/` again, enable the tuning include, repeat. The wall-clock delta is your real first-build savings. The RSS delta tells you whether parallelism is putting you near the OOM line.

**2. Warm-cache run.** Build a second time on top of the now-primed ccache and sstate. Run `ccache -s` before and after to read the hit/miss counts; the hit-rate divided by total compiles is the ccache yield on this workload. Sub-50% means something invalidated the cache (toolchain version, host gcc, path layout) and the tune is paying for storage you aren't using.

**3. Per-recipe attribution.** Add `INHERIT += "buildstats"` and read `buildstats/<run>/build_stats` plus the per-recipe directories. Sort by elapsed time. The top three recipes account for most of the build, and that's where the next optimization should target - more bitbake threads won't help a build that's 70% kernel.

**4. Failure-mode rehearsal.** Once. Deliberately stage one failure for every tune you turned on. Take the network down mid-build to confirm `BB_FETCH_TIMEOUT` doesn't hang for ten minutes per URI. Delete the fork PREMIRROR directory to confirm the build falls through to upstream cleanly. Pull a recipe known to break under ccache (`renderdoc` if you can pull it in) to verify the per-recipe override actually fires. The failure modes in this post are real; rehearsing them once costs an hour and saves a CI debugging session later.

The output of this exercise is a number. "Tuning saves 38% wall-clock on a cold build, 92% on a warm build, with ccache hit-rate of 84%." Or maybe it doesn't - maybe you find your build is fetch-bound and ccache changes nothing, in which case the right next move is `SSTATE_MIRRORS`, not the tunes here. Either way you know, and the include file you ship is one you can defend against the question every senior engineer eventually asks: did you actually measure this, or did you copy it off the internet?

## Closing

The takeaway isn't "turn these on". It's "every line in your `local.conf` is a trade you made". The defaults are conservative for a reason; the tunables that override them are conservative in the other direction. Knowing what each one breaks is the difference between debugging a confusing CI failure for an afternoon and recognising it in thirty seconds.

ccache is fast until cmake.bbclass meets a recipe that quotes the compiler. PREMIRRORs are robust until the mirror is stale. Wide parallelism is fast until the linker OOMs. None of this is news to a senior Yocto engineer. The under-documentation is the news. Each of these tradeoffs deserves a comment in the file where it lives, and the file deserves to live somewhere you'll actually find it the next time the same build breaks the same way.
