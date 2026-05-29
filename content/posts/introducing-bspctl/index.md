---
title: "bspctl: the wrapper Yocto teams keep writing by hand"
date: 2026-05-23T00:00:00-06:00
draft: false
description: "Every embedded team ends up writing the same Makefile around kas and repo. bspctl replaces it: pre-flight checks before the build starts, curated build tuning applied at run time without touching your YAML, structured per-run observability, vendor manifest translation for NXP and TI BSPs, and bitbake-setup workspace support for Yocto 5.3+ projects. Builds run in a kas-container when KAS_CONTAINER_IMAGE is set, or directly on the host otherwise."
ShowToc: true
ShowReadingTime: true
tags:
  - "yocto"
  - "bitbake"
  - "build-systems"
  - "tooling"
  - "open-source"
  - "embedded"
  - "kas"
  - "qemu"
categories:
  - "tooling"
---

**TL;DR**: bakar is a Python CLI that wraps `kas` and `kas-container` for Yocto Board Support Package (BSP) builds. It defaults to container builds via `kas-container` when `KAS_CONTAINER_IMAGE` is set, and falls back to plain `kas` on the host when it is not. Pass `--host` to any subcommand to force host mode. On top of kas, bakar adds pre-flight environment checks before the build starts, applies a curated tuning overlay (ccache, fetch mirrors, reproducibility knobs) without modifying your YAML on disk, writes structured per-run logs, and provides `bakar triage` to locate the failing recipe after a crash. For vendor BSPs that ship as repo manifests (NXP i.MX) or oe-layertool configs (TI Sitara), it translates those to kas YAMLs automatically. For projects initialized with bitbake-setup (the official Yocto 5.3+ workspace tool), bakar detects the workspace automatically, translates the JSON layer config to a kas YAML, and drives the same pipeline. Install: `uv tool install bakar`.


> **Note (updated):** This project has been migrated to
> [bakar](https://github.com/jetm/bakar).
> The original repository is archived at
> [github.com/jetm/bspctl](https://github.com/jetm/bspctl).

---

Being [between jobs]({{< ref "posts/when-youre-fired-your-next-job-is-finding-a-job" >}}) means having time for ideas that never made it off the backlog. I have been working on this one for the past two months.

## Every team writes the same Makefile

In the Yocto/OpenEmbedded (OE) community, the standard build tools are [kas](https://kas.readthedocs.io/) and [Google repo tool](https://android.googlesource.com/tools/repo). kas describes a build stack as a YAML file - repos, layers, machine, distro - and drives `bitbake`. `kas-container` wraps it in a Docker container for reproducibility. repo tool fetches source trees from XML manifests.

Both are useful. Both are also intentionally minimal.

So every embedded team ends up writing a `Makefile` at the root of the workspace. It sources the right env script, sets `MACHINE`, calls `repo init`, runs `kas build`. The details differ - some add ccache, some add download mirror configs, some add a disk check - but the pattern repeats across every project I have worked on.

The missing pieces were always the same: a fast way to find what failed after a bitbake crash without digging through `build/tmp/work/.../temp/log.do_*`; pre-flight checks so the build does not fail four hours in on a full disk or the wrong container Python; and consistent build tuning that is not checked into every project YAML separately.

bakar fills those gaps. It defaults to `kas-container` when `KAS_CONTAINER_IMAGE` is set, and falls back to plain `kas` on the host when it is not - so it works with or without a container setup.

## A concrete run

Install:

```bash
uv tool install bspctl
# or
pip install bspctl
```

The Bring Your Own (BYO) path works with any kas YAML - including a QEMU target:

```bash
bspctl build examples/kas-qemux86-64-wrynose.yml
```

```text
:: bspctl build  BYO examples/kas-qemux86-64-wrynose.yml
INFO     build mode=byo bsp=generic yaml=examples/kas-qemux86-64-wrynose.yml
         overlay=bspctl/overlays/bspctl-tuning-generic.yml
INFO     → doctor
                                                         Pre-flight diagnosis
 Check             ┃ Sev   ┃ Status ┃ Detail
━━━━━━━━━━━━━━━━━━━╇━━━━━━━╇━━━━━━━━╇━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
 host-tools        │ BLOCK │ PASS   │ GENERIC required binaries present (kas-container, docker, python3)
 docker-daemon     │ BLOCK │ PASS   │ server 29.5.1
 container-image   │ BLOCK │ PASS   │ jetm/kas-build-env:latest present
 container-os      │ BLOCK │ PASS   │ fedora  / Python 3.12.10
 container-bitbake │ INFO  │ SKIP   │ inspection failed
 cache-dirs        │ BLOCK │ PASS   │ SSTATE_DIR=/mnt/sstate, DL_DIR=/mnt/downloads
 sysctl            │ WARN  │ PASS   │ inotify/swappiness sane
 docker-ulimits    │ WARN  │ PASS   │ nofile soft=65536
 disk-free         │ BLOCK │ PASS   │ >= 50G free on each mount
 memory            │ WARN  │ PASS   │ available+swap=173717M
 bitbake-override  │ INFO  │ SKIP   │ poky tree absent (pre-bootstrap)
 bitbake-locks     │ BLOCK │ PASS   │ no stale locks or sockets
INFO     ✓ doctor
INFO     ↷ bitbake_override (generic mode)
INFO     → kas_build
INFO     exec: kas-container --runtime-args -v examples/ccache:/work/ccache:rw build \
         kas-qemux86-64-wrynose.yml:.bspctl/overlays/bspctl-tuning-generic.yml
kas_build ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━╸ 5190/5191 tasks core-image-minimal.bb:do_create_image_sbom_spdx live 1s  +453M 0:23:10
INFO     ✓ kas_build
build succeeded
artifacts: examples/build/tmp/deploy/images/generic
```

Every `bakar build` invocation opens with the doctor table, then hands off to `kas-container` (or plain `kas` in host mode). The progress bar tracks bitbake tasks live. If the build fails, `bakar triage` picks up from there.

## Pre-flight checks

`bakar doctor` runs before every build and exits non-zero on any BLOCK failure. The checks worth calling out:

**Container Python version.** bitbake's parser `fork()`s worker processes from a multi-threaded context. Python 3.13 tightened the rules around `fork()` after `pthread_create` ([cpython#100228](https://github.com/python/cpython/issues/100228)), causing the parser to deadlock. Python 3.14 changed the default multiprocessing start method to `forkserver` ([cpython#84559](https://github.com/python/cpython/issues/84559)), which trips a `_pickle.PicklingError` in bitbake's cache code on `CoreRecipeInfo.init_cacheData`. Neither failure is obvious at the point it occurs - both surface hours into a build as a hung or immediately-crashing parse step. The `container-os` check reads the Python version from the container image before kas starts. The example output above shows `Python 3.12.10` on Fedora - safe.

**Disk space.** Yocto builds consume 50-200G depending on the BSP and image. The doctor check gates on 50G free across the workspace, `SSTATE_DIR`, and `DL_DIR`. A full disk fails bitbake mid-recipe with a generic I/O error.

**Stale bitbake locks.** A crashed build leaves `bitbake.lock`, `bitbake.sock`, and `hashserve.sock` behind. The next run fails immediately if those files are present and the owning PID is gone. The `bitbake-locks` check auto-removes stale files when the PID is dead, and blocks when a live build holds the lock.

Running doctor standalone:

```bash
bspctl doctor
```

## Finding what broke

After a failed build:

```bash
bspctl triage
```

Every build writes structured event logs to `build/runs/<YYYYMMDD-HHMMSS>/`. The directory contains `events.jsonl` (one JSON object per pipeline step), `kas.log` (raw kas-container output), `console.log`, an environment snapshot, timing data, and a disk usage trace sampled every 30 seconds. `triage` reads the latest run, finds the `step_fail` event, tails the relevant section of `kas.log`, and locates the bitbake recipe log that triggered the failure. It also matches the combined output against a table of known error patterns - fetch failures, parser deadlocks, OOM, GitHub flakes, stale bitbake cache.

The alternative is navigating several levels of `build/tmp/work/.../temp/log.do_*` by hand. On a BSP with hundreds of recipes, finding the right log takes a few minutes every time it happens. That adds up.

To follow a build's output live without waiting for failure:

```bash
bspctl log
```

## Build tuning without touching your YAML

bakar applies a curated tuning overlay at build time using kas's colon overlay syntax. Your YAML is not modified on disk; kas merges the overlay in when it parses the build.

What the overlay unions in:

- `INHERIT += "ccache"` with the cache dir mounted at `/work/ccache` inside the container
- `BB_NUMBER_THREADS` and `PARALLEL_MAKE` set to `os.cpu_count()` by default, overridable via `$NPROC`
- `BB_FETCH_TIMEOUT = "600"` to survive transient TLS stalls during `do_fetch`
- `MIRRORS` pointing at `downloads.yoctoproject.org` - replaces `sources.openembedded.org`, which stopped resolving (DNS fails as of 2026-05) and which scarthgap's poky still references
- `PREMIRRORS:prepend` routing github.com fetches through the Yocto mirror first, so a GitHub outage becomes a silent fallback instead of a build blocker
- `FETCHCMD_wget` with a browser User-Agent to work around the crates.io 403 that bites Rust-dependent recipes
- `PYTHONMALLOC=malloc` as a mitigation for the Python 3.14 parser fork-race (reduces the failure rate from ~50% to ~5% under that Python version, per the overlay's own comment)

Each of these knobs trades one failure mode for another. The tradeoffs - what actually breaks when you turn each one on - are covered in [Yocto build tunables and their hidden costs]({{< ref "posts/yocto-build-tunables-and-their-hidden-costs" >}}).

The NXP and TI overlays add `ACCEPT_FSL_EULA`, BSP-specific vendor fork PREMIRRORs for the kernel and bootloader, and a few recipe-level workarounds on top of this baseline.

To add your own knobs without modifying the built-in overlay:

```bash
bspctl build my-build.yml:my-tuning.yml
```

kas merges the three files - your YAML, the bakar overlay, and your extra tuning - at build time.

## Vendor extensibility

The NXP i.MX and TI Sitara presets are the batteries-included path. If you work with a different board vendor, `~/.config/bspctl/vendors.toml` lets you register a custom BSP family:

```toml
[[vendors]]
name = "my-board"
family = "nxp"
manifest_regex = "my-bsp-.*\\.xml"
default_machine = "my-machine"
default_image = "core-image-minimal"
```

Vendor entries are matched against the manifest filename before the built-in patterns and overlay the base NXP or TI model, so you only specify what differs.

My goal is for bakar to be extensible enough to eventually replace the proprietary build wrappers that embedded teams maintain internally. Those tools exist - they do the same job - but they are tied to specific internal infrastructure and are not shareable. bakar is the attempt to close that gap with something open.

## bitbake-setup workspace support

As of Yocto 5.3 (Whinlatter), [bitbake-setup](https://docs.yoctoproject.org/bitbake/bitbake-user-manual/bitbake-user-manual-environment-setup.html) is the official setup tool for new Yocto builds. The Poky monorepo's master branch was deprecated in 5.3 in its favour. bitbake-setup uses a registry-based JSON config to fetch layers and write build configuration - a different model from the kas YAML approach.

bakar v0.4.0 adds support for building inside an existing bitbake-setup workspace. After `bitbake-setup init` populates the workspace, `bakar build` from that directory auto-detects it and drives the build:

```bash
# Initialize the workspace (ships in the bitbake repo)
bitbake-setup init

# Build from the initialized workspace
bspctl build
```

Detection looks for `config/config-upstream.json` with the expected layer topology and `build/init-build-env`. When both are present, bakar reads the JSON config - each source, its git remote, branch or pinned SHA, and the bitbake layers it contributes - and translates it into a kas v3 YAML written as `kas-bbsetup.yml`. That file is a build artifact regenerated each run, not for version control. kas then drives the build against that YAML plus the standard bakar tuning overlay.

Machine and distro are inferred from `oe-fragment-choices` in the config. Pass `--machine`, `--distro`, or `--image` to override them; `--image` sets the bitbake target and defaults to `core-image-minimal`.

One constraint: `bakar sync` does not apply to bitbake-setup workspaces - layer fetching is done by `bitbake-setup init`. To regenerate `kas-bbsetup.yml` after changing the workspace config without rebuilding, use `bakar gen-kas -w <workspace-dir>`.

## Current limitations

bakar is at v0.4.0 and has only been tested on Arch Linux-based distributions. Debian, Ubuntu, and Fedora host systems are untested - corner cases almost certainly exist.

**SSTATE_DIR and DL_DIR.** bakar does not configure these for you. Set them as environment variables before running `bakar build` if you want a shared sstate cache or download mirror. The `cache-dirs` doctor check validates that the paths exist and are writable when the vars are set; it does not create them.

**Opinionated defaults.** `BB_NUMBER_THREADS` and `PARALLEL_MAKE` default to `os.cpu_count()` - bakar sets `NPROC` automatically before invoking kas, so the overlay picks up the actual core count. Set `NPROC` in your environment to override it downward (useful on machines where a full-core build exhausts memory) or upward.

**QEMU.** `bakar run` - the command that boots a QEMU image - is currently limited to meta-avocado builds. Building a generic QEMU image via `bakar build my-qemu.yml` works fine; only the launcher is constrained.

## Why publish now

v0.4.0 works for me on Arch Linux, but that is a sample size of one. I am publishing now to widen that sample. The goal is to find the corner cases I have not hit - different host distros, container images I have not tested, manifest patterns outside the NXP and TI defaults, vendor TOML configs that expose gaps in the dispatch logic.

The other goal is to hear whether the problem is real for other people. My assumption is that every Yocto team maintaining a wrapper script around kas or repo has the same pain. If that is not your experience - or if your pain is different from what bakar addresses - I want to know. That feedback shapes what gets built next.

If something breaks, open an issue at [github.com/jetm/bakar](https://github.com/jetm/bakar). If the tool solves a real problem for you, or if there is a feature you need that is missing, say so there too. The more concrete the input - distro, container image, manifest filename, the workflow that does not fit - the more useful it is.

## References

- [bakar on GitHub](https://github.com/jetm/bakar)
- [kas documentation](https://kas.readthedocs.io/)
- [kas-build-env container image](https://hub.docker.com/r/jetm/kas-build-env)
- [Google repo tool](https://android.googlesource.com/tools/repo)
- [bitbake-setup documentation](https://docs.yoctoproject.org/bitbake/bitbake-user-manual/bitbake-user-manual-environment-setup.html)
- [Yocto 5.3 (Whinlatter) migration guide](https://docs.yoctoproject.org/migration-guides/migration-5.3.html)
- [Yocto Project source mirror](http://downloads.yoctoproject.org/mirror/sources/)
