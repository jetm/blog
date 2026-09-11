---
title: "What It Actually Takes to Build Yocto Properly on Arch Linux"
date: 2026-09-11T10:02:47-06:00
draft: false
description: "A Yocto build on Arch Linux fails with glibc mismatches that no recipe explains. The cause: uninative.bbclass catches its own ceiling check and downgrades it to one bb.warn line, after which native sstate silently stops being shareable. Closing it took buildtools-extended plus a uninative tarball rebuilt from Arch's own glibc commit."
ShowToc: true
ShowReadingTime: true
outputs:
  - "HTML"
tags:
  - "yocto"
  - "bitbake"
  - "arch-linux"
  - "kas"
  - "build-systems"
  - "tooling"
  - "glibc"
  - "uninative"
categories:
  - "systems"
  - "tooling"
---

**TL;DR**: Building natively on a Yocto workspace on Arch Linux hits two separate failures. The host's rolling toolchain is not pinned, so `bitbake`'s own host-side tooling drifts under you between builds. And `uninative`, the prebuilt glibc that keeps `-native` binaries loadable across hosts, disables itself silently once the glibc it sees outruns the ceiling baked into its own payload: one `bb.warn` line, and native sstate quietly stops being shareable. `buildtools-extended`, the Yocto Project's own documented recommendation for unsupported hosts, fixes the first problem. Nothing off the shelf fixed the second, so I built [`yocto-uninative-tarball`](https://github.com/jetm/aur-packages/tree/main/packages/yocto-uninative-tarball), an AUR (Arch User Repository) package that rebuilds the payload from the exact glibc commit Arch itself ships, so host and payload glibc are the same build by construction instead of racing each other.

Arch Linux is not on Yocto's list of supported host distributions, which is the root of both failures, and the standard workaround, `kas-container`, only trades them for a worse dev loop: docker or podman setup layered on top of Yocto's own failure modes, and constant context switching between host and container.

---

For years the same three problems kept turning up on every Yocto build I ran on Arch Linux: a glibc version mismatch, native tools crashing partway through, and `gcc` failures that made no sense and never reproduced the same way twice. None of it was a bug in any one recipe. It was what happens when the host's glibc is newer than anything Yocto's build machinery was designed to see, and on a rolling release it never stops happening, because the gap only grows. I knew that much the whole time, and for years my answer was the same one: build inside a container against a distro Yocto actually supports, and let the container absorb the mismatch.

## Arch is not on the list

Yocto's own [system requirements documentation](https://docs.yoctoproject.org/ref-manual/system-requirements.html) names the supported host distributions explicitly: AlmaLinux, CentOS Stream, Debian, Fedora, OpenSUSE Leap, Rocky Linux, and Ubuntu, each pinned to specific releases. No rolling release is on that list, and none ever will be, since the whole point of the list is a fixed target the build system was actually tested against. The same page names the official mitigation for exactly this case: "if your Linux distribution is not in the above list, we recommend to get the buildtools or buildtools-extended tarballs containing the host tools required by your Yocto Project release." The page actually documents three of these tarballs (`buildtools`, `buildtools-extended`, and `buildtools-make`), and describes `buildtools-extended` as the equivalent of Debian or Ubuntu's `build-essential` package.

In my own words: it is a toolchain you install on any unsupported host so `bitbake` is not left depending on whatever the system happens to ship. oe-core tracks the same distinction internally as the `SANITY_TESTED_DISTROS` variable, which `sanity.bbclass` checks against the running host.

This was never a case of Arch being broken for Yocto. It is a documented gap, with a documented partial fix, that nobody had closed all the way for a rolling-release host.

Closing it meant treating it as two separate failures rather than one:

| Axis | Orchestration | uninative ceiling |
|---|---|---|
| What drifts | `bitbake`'s host-side tooling, since Arch's rolling release changes what's on `PATH` between builds | The tarball's `UNINATIVE_MAXGLIBCVERSION`, fixed at whatever glibc the Yocto Project built it against |
| Symptom | Builds behave differently machine to machine, or run to run, with no recipe change | `uninative` silently disables itself: one `bb.warn` line, then native sstate stops being shareable |
| Fix | `buildtools-extended`, enforced so a build refuses to start without it | `yocto-uninative-tarball`, rebuilt from Arch's own glibc commit |

## Living with the container

`kas-container` works, and that is why I never filed this as a bug anywhere. It is also not how I want to spend a working day.

The compute cost of the container itself is smaller than I expected when I went looking for real numbers. A university thesis that benchmarked Docker, LXD, Podman, and Buildah against bare metal on a real workload, compiling Firefox end to end, found Docker and LXD finishing within a few seconds of bare metal on a roughly 50-minute build: a 4.92-second delta, about 0.165% ([diva-portal](https://www.diva-portal.org/smash/get/diva2:1450777/FULLTEXT01.pdf)). Podman and Buildah both ran about 7% slower (7.47% and 7.43%), and the same study measured CPU and RAM utilization as nearly identical across every tool. Whatever costs that extra 7%, the thesis itself does not say, and I am not going to fill that gap with a guess of my own.

A separate High Performance Computing (HPC) study measured similarly small CPU-bound overhead: 2.89% for Docker on an HPL-LINPACK compute benchmark, which the paper attributes to the daemon's default CPU-use restrictions rather than to containerization itself ([arXiv:1709.10140](https://arxiv.org/pdf/1709.10140)). The same paper also measured a much larger I/O penalty for Docker under IOzone, 37% write and 65% read, but that number is specific to Docker's AUFS storage driver copying a file up through its own layered filesystem, and the paper ran it on "a totally contained filesystem (without any bind or mount volume)." `kas-container` mounts the build tree in from the host, which is the opposite setup, so I am not carrying that number over to a Yocto build. What the evidence actually supports is that container overhead for a workload shaped like a Yocto build, real compute plus real disk I/O, looks close to what the compile benchmark measured: small.

The real reason I stopped using it needs no benchmark at all. Docker and podman each bring their own daemon or socket setup on top of the failure modes Yocto already has. And the constant switch between host tools and container tools, editor on one side, shell on the other, is the kind of friction that does not show up in any benchmark but drains a session anyway. The container was never broken. It just was not worth the friction once a real fix existed.

## buildtools-extended closes half the gap

`buildtools-extended` solves the half of the problem that is about orchestration. `bitbake` is a Python process that shells out to host tools throughout a build, and without a pinned toolchain those tools are whatever Arch's rolling release currently ships, which can change under you between builds.

[bakar](https://github.com/jetm/bakar), the build orchestrator I maintain, enforces this rather than hoping for it. Host builds detect a `buildtools-extended` install two ways: an already-sourced environment (`OECORE_NATIVE_SYSROOT` with a reachable `gcc`), or an unsourced install directory named by `BAKAR_BUILDTOOLS_DIR` containing an `environment-setup-*` script. When it finds the script form, it sources it in a subshell and merges the resulting `PATH` and `OECORE_*` exports into the build's environment. When it finds neither, it raises `BuildtoolsMissingError` before `bitbake` ever starts, rather than let the build quietly fall back to the system compiler and fail somewhere down the pipeline where the cause is no longer obvious ([`b4a2bcb`](https://github.com/jetm/bakar/commit/b4a2bcb), 2026-06-28).

That closes the orchestration side. It took about a month after wiring it into bakar before I found out it was only half the fix. Builds under `buildtools-extended` still failed the same way: glibc mismatches, native tools refusing to start, and this time there was no host toolchain drift to blame, because the toolchain was pinned. The only trace was a single `bb.warn` line from `uninative.bbclass`, saying it had disabled itself and moved on without stopping the build. Reading the class made the fix obvious: the comparison that decides this sits inside a `raise`, and the `try` wrapped around it has a handler that only warns. Nothing about that shape can ever stop a build, only quietly turn a feature off, so the only durable fix was making sure the comparison never fails in the first place. I had the tarball built and wired into bakar within two days ([`e2f13e7`](https://github.com/jetm/aur-packages/commit/e2f13e7), [`bb52dbd`](https://github.com/jetm/bakar/commit/bb52dbd), 2026-07-27 and 2026-07-28).

## The other half: uninative's glibc ceiling

`uninative` is oe-core's mechanism for keeping `-native` binaries loadable regardless of which glibc the host happens to run: a prebuilt loader and libc that every native artifact gets relocated against, so an object built on one machine's glibc stays runnable on another's. The check it runs, as of the Wrynose release (commit `552e037b`, line numbers below pinned to it since a floating branch tip renumbers this file), is small and unforgiving:

```python
        # openembedded-core/meta/classes-global/uninative.bbclass:98-100
        glibcver = subprocess.check_output(["ldd", "--version"]).decode('utf-8').split('\n')[0].split()[-1]
        if bb.utils.vercmp_string(d.getVar("UNINATIVE_MAXGLIBCVERSION"), glibcver) < 0:
            raise RuntimeError("Your host glibc version (%s) is newer than that in uninative (%s). "
                                "Disabling uninative so that sstate is not corrupted." % (
                                    glibcver, d.getVar("UNINATIVE_MAXGLIBCVERSION")))

        # ...lines 101-118 omitted: fetch and unpack the tarball, relocate the loader,
        # still inside the same try: block that starts well before line 98...

    # uninative.bbclass:119-120, the except clause of that same try:
    except RuntimeError as e:
        bb.warn(str(e))
```

That comparison does not run on every build either. A few lines earlier (`uninative.bbclass:38-44`), the function returns as soon as the loader is already on disk with a matching checksum, before it ever reaches the `glibcver` check. So the ceiling only gets tested when the loader is fetched fresh, which is probably part of why the failures at the top of this post never reproduced the same way twice: whether a build hit the check at all depended on what was already cached.

When it does run, it shells out to whatever `ldd` resolves first on `PATH` and compares its version against `UNINATIVE_MAXGLIBCVERSION`, the ceiling baked into the tarball at build time. This is exactly why `buildtools-extended` matters here too: with its environment sourced, its own pinned `ldd` sits ahead of the system one, so `glibcver` reflects a fixed SDK (Software Development Kit) version instead of racing Arch's rolling one. But the ceiling is fixed at whatever glibc the *tarball itself* was built against, and that predates the pinned toolchain you are using it with. If the SDK's own glibc is already past it, which happens whenever the tarball predates the `buildtools-extended` release you're running, the comparison fails anyway.

That remaining exposure closes in practice because of what a rolling release actually is. `buildtools-extended`'s own glibc is fixed at whatever the Yocto Project built it against, and only moves when you adopt a newer release. Arch's does not sit still between those releases. In the gap between any two `buildtools-extended` versions, Arch's rolling glibc reliably pulls ahead of the SDK's pinned one, the same dynamic that keeps Arch off the supported list in the first place.

bakar's own `uninative-glibc` doctor check makes this explicit: it compares the tarball's ceiling against the buildtools sysroot's glibc specifically, not the raw system glibc, because that sysroot's `gcc` is what actually links native binaries once `buildtools-extended` is active. A tarball rebuilt from Arch's own glibc commit clears that comparison with headroom to spare, not because it targets the SDK's exact number, but because Arch's glibc is almost always ahead of it.

And when the comparison fails, nothing stops the build. The class's own handler catches the exception and downgrades it to a warning: **`uninative` quietly turns itself off for the rest of the build**. `NATIVELSBSTRING` stops being `"universal"` (`enable_uninative()`, `uninative.bbclass:145`, a function the exception path never reaches), native artifacts stop being cross-host shareable, and **the only evidence is one warning line in a log nobody reads to the end**.

Mirroring the upstream tarball cannot fix this. The payload's glibc is whatever the Yocto Project happened to build it against, never the host's, and never the pinned toolchain's either. The only fix that holds is building the payload's glibc from the exact source the toolchain you are actually running is built from, so the two numbers can never drift apart. That is what [`yocto-uninative-tarball`](https://github.com/jetm/aur-packages/tree/main/packages/yocto-uninative-tarball) does: it builds glibc from the precise commit Arch itself packages, pins `pkgver` to Arch's own versioning, and regenerates the `UNINATIVE_MAXGLIBCVERSION` fragment from that build rather than from a number someone wrote down once ([`e2f13e7`](https://github.com/jetm/aur-packages/commit/e2f13e7), 2026-07-28).

## What building the tarball actually taught me

None of the three problems below showed up until I tried to build glibc under CachyOS's own `makepkg.conf`, which differs from stock Arch's in ways that only surfaced by building and reading the failure. All three are documented in the same commit that builds glibc from Arch's own source ([`e2f13e7`](https://github.com/jetm/aur-packages/commit/e2f13e7)).

| Divergence | What happened | Fix |
|---|---|---|
| `-O3` optimization | `misc/syslog.c` fails to compile: `inlining failed in call to always_inline 'syslog'`. This build only tolerated `-O2`. | Pin `-O2`, overriding whatever the invoking `makepkg.conf` set |
| `-march=native` | Bakes this machine's instruction set into a libc shared across hosts through sstate; surfaces as `SIGILL` on any host with an older CPU | Pin a portable `-march`/`-mtune` instead |
| `mold` linker | Ignores `-z nomark-plt` (a warning, not an error), leaving `_PROCEDURE_LINKAGE_TABLE_` undefined in `ld.so`, which trips glibc's own check that its loader carries no undefined symbols | Force `-fuse-ld=bfd`, which is also what stock Arch's own `makepkg.conf` uses |

None of these are Yocto problems. They are the ordinary cost of building system libc on a distro whose default toolchain flags assume you are building applications, not the library everything else loads through.

The commit is not a one-off snapshot either. As of this writing, Arch's own `glibc` package is at `2.44+r24+g16be1518495f`, the exact `pkgver` the tarball is pinned to right now. Same build, same version string, on both sides.

## Keeping it honest

A fixed ceiling that never gets revisited just becomes the next version of the same bug, one glibc upgrade later. `yocto-uninative-tarball` carries a pacman hook that fires on `glibc` itself, not on this package, since the event that invalidates the tarball is the other package changing. It reads the running version straight out of `libc.so.6` rather than asking pacman, because it executes inside a pacman transaction, and it compares only `MAJOR.MINOR`, matching what `uninative.bbclass` itself compares, so an Arch rebuild (`+r3` to `+r5`) stays silent and only a real version bump speaks up ([`bfa9c14`](https://github.com/jetm/aur-packages/commit/bfa9c14), 2026-07-31).

It warns rather than blocks. A hard dependency constraint would make the failure impossible instead of merely visible, but it would also turn a routine `pacman -Syu` into a dependency conflict on a package the AUR cannot rebuild automatically, which is a disproportionate response to a consequence that is a disabled optimization, not a broken system.

bakar backs this with seven `doctor` checks that verify the wiring actually took, gated behind `[build] uninative = true`:

| Check | Severity | Asserts |
|---|---|---|
| `uninative-fragment` | BLOCK | The fragment is installed and parses |
| `uninative-glibc` | BLOCK | The fragment's ceiling is at least the buildtools sysroot's glibc |
| `uninative-checksum` | BLOCK | The fragment's declared checksum matches its own mirrored payload |
| `uninative-dldir-links` | BLOCK | No cached entry holds a dangling payload link |
| `uninative-mirror-hit` | WARN | The payload came from the local mirror, not a network fetch |
| `uninative-cluster-ceiling` | BLOCK | Every node in a cluster resolved the same ceiling |
| `uninative-leak` | BLOCK | No native artifact in a finished build requires a glibc node above the ceiling |

`uninative-fragment` exists because the failure mode without it is silent in the worst way: the overlay that wires `uninative` in ([`bb52dbd`](https://github.com/jetm/bakar/commit/bb52dbd)) only activates when the fragment file exists, so a host missing the package does not get a parse error or a warning, it just quietly falls back to oe-core's own default ceiling. Asking for the feature and not getting it, with nothing telling you so, is the exact shape of bug this whole investigation started from. That check exists to make sure it never happens twice.

## Where this stands

This is a personal itch, scratched for my own machine and now maintained as an AUR package and a wired-in bakar feature. I have not gone looking through the Yocto Project mailing lists or existing AUR packages for someone who already solved this the same way, and I would rather say that plainly than imply I checked. If someone has, I would genuinely like to know.

What I do know is that the shape of the fix, build the payload from the same source as the toolchain instead of pinning a number that will eventually be wrong, is not specific to Arch. Any rolling-release host hits the same ceiling on its own schedule. If this turns out to be a problem the wider Yocto community has also been quietly working around, I would be glad to help get something like it upstream. For now it works, on my machine, and that is as far as I have taken it.

## Acknowledgments

Andrew Murray's writeup on uninative and native sstate reuse is where I went to double check my own understanding of the mechanism before writing this. And the fix here only works because Arch's own `glibc` package tracks upstream as closely as it does, which is the work of its maintainers, Giancarlo Razzolini and Frederik Schwan.

## Open claims

1. The Arch/uninative glibc mismatch is structural, not a one-time gap that a future oe-core release closes on its own. **Falsified by**: oe-core decoupling `UNINATIVE_MAXGLIBCVERSION` from a fixed upstream build, or Arch being added to `SANITY_TESTED_DISTROS`.
2. `buildtools-extended` alone does not eliminate exposure to glibc drift: it pins which glibc `uninative.bbclass` sees on `PATH`, but the tarball's ceiling is fixed at whatever glibc the Yocto Project happened to build it against, and a pinned SDK's own glibc can still be newer than that ceiling. **Falsified by**: tracing an actual build where `buildtools-extended` is active and the comparison still passes against an unrebuilt stock tarball, showing the SDK's glibc never exceeds the stock ceiling in practice.
3. No existing AUR package or Yocto Project patch solves this the same way, by rebuilding the payload from the host's own glibc commit rather than pinning a static ceiling. **Falsified by**: finding one. I have not done an exhaustive search of the AUR or the openembedded-core mailing list, and this claim rests on not having found one, not on having ruled one out.

## References

- [Yocto Project system requirements: sanity-tested distributions and buildtools-extended](https://docs.yoctoproject.org/ref-manual/system-requirements.html)
- [Container performance benchmark between Docker, LXD, Podman and Buildah (diva-portal thesis, Firefox compile benchmark)](https://www.diva-portal.org/smash/get/diva2:1450777/FULLTEXT01.pdf)
- [Performance Evaluation of Container-based Virtualization for High Performance Computing Environments (arXiv:1709.10140)](https://arxiv.org/pdf/1709.10140)
- [`uninative.bbclass`, openembedded-core, Wrynose release, commit `552e037b`](https://git.openembedded.org/openembedded-core/tree/meta/classes-global/uninative.bbclass?id=552e037bf598ac523f35b69d2dafc99e5ba59c5f) ([GitHub mirror](https://github.com/openembedded/openembedded-core/blob/552e037bf598ac523f35b69d2dafc99e5ba59c5f/meta/classes-global/uninative.bbclass), if the cgit link hits a bot check)
- [How uninative keeps native sstate reusable across hosts, Andrew Murray](https://www.thegoodpenguin.co.uk/blog/improving-yocto-build-time/)
- [`yocto-uninative-tarball`, AUR package source](https://github.com/jetm/aur-packages/tree/main/packages/yocto-uninative-tarball)
- [bakar, commit `b4a2bcb`: enforce pinned buildtools-extended for host builds](https://github.com/jetm/bakar/commit/b4a2bcb)
- [bakar, commit `bb52dbd`: add host-provided uninative overlay for Arch-family hosts](https://github.com/jetm/bakar/commit/bb52dbd)
- [aur-packages, commit `e2f13e7`: build glibc from the commit Arch ships](https://github.com/jetm/aur-packages/commit/e2f13e7)
- [aur-packages, commit `bfa9c14`: warn on the glibc upgrade that invalidates the tarball](https://github.com/jetm/aur-packages/commit/bfa9c14)
- [bakar doctor documentation: uninative wiring checks](https://github.com/jetm/bakar/blob/main/docs/doctor.md)
- [Introducing bakar]({{< ref "posts/introducing-bakar" >}})
