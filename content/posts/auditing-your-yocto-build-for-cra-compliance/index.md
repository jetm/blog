---
title: "Auditing your Yocto build for CRA compliance"
date: 2026-04-24T20:00:00Z
draft: false
description: "The CRA evidence layer nobody wants to assemble, and how shipcheck drafts your Annex VII technical file from a Yocto build."
ShowToc: true
ShowReadingTime: true
tags:
  - "yocto"
  - "cra"
  - "compliance"
  - "sbom"
  - "security"
  - "embedded"
  - "tooling"
  - "linux"
categories:
  - "linux"
  - "tooling"
---

**TL;DR**

- CRA is a process and design regulation; the risk analysis is the central document and the technical file is the evidence the regulator audits, not a scanner-selection problem.
- Yocto already emits the build-derivable half: SBOM (`create-spdx`), CVE scans, license manifests, signing posture.
- The vendor-committed half - CVD policy, support period, update mechanism, Declaration of Conformity - has to be written by hand.
- [shipcheck](https://github.com/jetm/shipcheck) reads a Yocto build plus `product.yaml`, pivots findings by CRA Annex, and drafts your Annex VII technical file and DoC.

## The received wisdom is wrong

Read any CRA compliance article from a security vendor and you will see the same shape of pitch: run a scanner, triage the CVEs, generate an SBOM, ship. The regulation becomes a scanner-selection problem, and whichever product the vendor sells happens to be the right scanner.

That framing is wrong, and if you ship embedded Linux you already know it. The Cyber Resilience Act (Regulation (EU) 2024/2847) is a process and design regulation. The risk analysis is the document everything else flows from; the technical file (Annex VII), the Declaration of Conformity, and the sustained record of vulnerability handling are the evidence the regulator audits. Scanning is one input. The hard part is the process around it: putting in place and keeping current the coordinated vulnerability disclosure policy, the single point of contact, the support-period commitment, the update distribution mechanism, the Annex VII technical file, the Declaration of Conformity, and the sustained record of vulnerability-handling activity required by Article 13 and 14.

Experienced Yocto maintainers have been pointing this out on the mailing lists for months. SPDX is for machines. The real daily tooling - the stuff you read while triaging a cvelistV5 feed at 11pm - is `license.manifest`, `cve-check.bbclass`, and hand-curated audits. Compliance is sustained activity, not a snapshot.

This post is about the evidence-layer problem, what Yocto already solves for you, and how [shipcheck](https://github.com/jetm/shipcheck) drafts the rest.

## What CRA actually requires, by Annex

Forget the marketing buzzwords and read the regulation by its annex structure. Three artefact classes are mandatory for products with digital elements:

**Annex I Part I (13 items a-m)** - product cybersecurity properties. Secure by default, no known exploitable vulnerabilities, secure update mechanism, confidentiality and integrity of stored and transmitted data, minimal attack surface, boot protections. These are engineering properties; most are configuration decisions your Yocto distro already made.

**Annex I Part II (8 items)** - vulnerability-handling process. SBOM (§1), addressing vulnerabilities promptly (§2-3), testing (§4), public disclosure policy (§5), coordinated disclosure cooperation (§6), secure update distribution (§7), public notification of fixed vulnerabilities (§8). These are process commitments, not code.

**Annex II (9 items)** - user-facing information. Name, type, intended purpose, support-period end date, single point of contact for vulnerability reports, update instructions. Product metadata that must accompany the product.

**Annex VII (8 items)** - the technical file. General description, design and development, risk assessment, harmonised-standards list, conformity-test results, Declaration of Conformity, labels, and any additional evidence. This is the binder regulators audit if your product is challenged.

The word "scanner" does not appear anywhere in this list. "Test" appears once (Annex I Part II §4), and the test is not the kind a scanner runs - it is an organisational commitment.

## What Yocto already gives you

Yocto Project with Poky on Scarthgap or newer ships a lot of this for free if you turn on the right classes:

| You already have | From |
|---|---|
| SPDX 2.x SBOM per image | `INHERIT += "create-spdx"` (Scarthgap emits 2.2) |
| Per-package CVE scan | `INHERIT += "cve-check"` (`cve-check.bbclass`) |
| Per-package license manifest | `tmp/deploy/licenses/<image>/license.manifest` |
| Secure Boot plumbing | `IMAGE_CLASSES += "image-uefi-sign"` (or `sbsign`), UEFI signing keys |
| FIT image signing | `UBOOT_SIGN_ENABLE = "1"`, `UBOOT_SIGN_KEYDIR` for U-Boot FIT images |
| dm-verity / IMA | `meta-security` layer |
| Update mechanism options | `swupdate`, `RAUC`, `mender`, `capsule-update` |

If you are an experienced Yocto engineer you already know this. The tooling is mature. The bitbake output lands in predictable places. `cve-check.bbclass` writes `tmp/log/cve/cve-summary.json`, `license.manifest` files land under `tmp/deploy/licenses/<arch>/<image-or-package>/`, SPDX documents under `tmp/deploy/spdx/`.

What you do not have is a tool that walks those artefacts, pivots the findings by CRA annex item, flags the gaps in the evidence layer, and emits the drafts you hand to your compliance team.

## The real gap: evidence, not scanning

Here is the mental model. Split your CRA readiness into two halves:

```text
From the build (derivable):            From the vendor (commitment):
  - SBOM summary                         - Manufacturer legal entity
  - CVE reconciliation                   - Registered postal address
  - License manifest                     - CVD policy URL
  - Secure Boot posture                  - Security disclosure contact
  - Image signing posture                - Support-period end date
  - Update mechanism detection           - Update distribution mechanism
  - Scan cadence (history)               - Declaration of Conformity
```

The left column is what bitbake emits. A tool can read those artefacts and render findings against Annex I Part I items a-m.

The right column is what the vendor must write down. No scanner in the world will tell you what your CVD policy URL is, because it is a policy commitment, not a file on disk. You have to put it in writing.

That is where the evidence-layer gap lives. Every vendor I have seen new to CRA tries to substitute the left column for the right column - "we run cve-check, so we have vulnerability handling covered". That is not what Annex I Part II §5 asks for. It asks for a published coordinated vulnerability disclosure policy with a single point of contact, a support-period commitment, and an update distribution mechanism. Those are sentences you write, not binaries you build.

## What shipcheck does about it

`shipcheck` is an open-source CLI tool ([jetm/shipcheck](https://github.com/jetm/shipcheck), Apache-2.0, Python 3.13+) that reads a Yocto build directory, cross-checks it against a `product.yaml` with the vendor commitments, and emits an evidence report plus a draft technical file and Declaration of Conformity.

Seven checks registered in v0.0.4:

- `sbom-generation` - validates SPDX 2.x against BSI TR-03183-2 field requirements (detects SPDX 3.0 and CycloneDX without field validation)
- `cve-tracking` - consumes Yocto `cve-check`, `vex.bbclass`, and [`sbom-cve-check`](https://github.com/bootlin/sbom-cve-check) JSON output (the last is preferred when present)
- `yocto-cve-check` - reads `tmp/log/cve/cve-summary.json` directly
- `license-audit` - parses per-arch `license.manifest` files
- `secure-boot` - detects signing class configuration
- `image-signing` - detects FIT signatures and dm-verity
- `vuln-reporting` - validates the vendor-commitment half from `product.yaml`

Run it like this against a Yocto build directory:

```bash
uv tool install shipcheck
cd path/to/yocto/build
shipcheck init                                 # writes .shipcheck.yaml
shipcheck check --build-dir . --format evidence --out dossier/
```

The evidence report pivots findings by CRA annex item instead of by check. That is deliberate: when a regulator asks "where is your evidence for Annex I Part II §5?", you want the answer filed under `I.P2.5`, not under "cve-tracking found 57 CVEs".

## A worked example: the VENDOR dossier

To make this concrete, here is what shipcheck produces when you feed it a deliberately-unfilled `product.yaml`. Every required vendor-commitment field is the literal string `VENDOR`:

```yaml
# product-vendor.yaml
schema_version: 1

product:
  name: "VENDOR"
  type: "VENDOR"
  version: "VENDOR"

manufacturer:
  name: "VENDOR"
  address: "VENDOR"
  contact: "VENDOR"

support_period:
  end_date: "VENDOR"

cvd:
  policy_url: "VENDOR"
  contact: "VENDOR"

update_distribution:
  mechanism: "VENDOR"
```

Run against a real `core-image-minimal` build on poky Scarthgap:

```bash
shipcheck check \
  --build-dir tests/fixtures/pilot_real/build \
  --product-config product-vendor.yaml \
  --format evidence \
  --out dossier/
```

The evidence report calls out four vendor-committed gaps by Annex item:

```text
## I.P2.5 - Coordinated vulnerability disclosure policy
- [high] (vuln-reporting) product.yaml cvd.policy_url is a placeholder token
  'VENDOR' (Annex I Part II §5): a real coordinated vulnerability disclosure
  policy URL must be declared

## I.P2.7 - Secure update distribution mechanisms
- [medium] (vuln-reporting) product.yaml update_distribution.mechanism is a
  placeholder token 'VENDOR' (Annex I Part II §7): a real secure update
  distribution mechanism must be declared

## II.2 - Single point of contact for vulnerability reporting
- [high] (vuln-reporting) product.yaml cvd.contact is a placeholder token
  'VENDOR' (Annex II §2): a real single point of contact for vulnerability
  reports is required

## II.7 - Technical security support and support period end-date
- [high] (vuln-reporting) product.yaml support_period.end_date is a
  placeholder token 'VENDOR' (Annex II §7): a real ISO 8601 YYYY-MM-DD
  support period end date must be declared
```

Meanwhile, the SBOM, CVE, license, and signing sections of the same report are populated from bitbake output exactly as you would expect. That is the demonstration: shipcheck fills in what the build emits, refuses to paper over what the vendor owes, and labels each gap with its Annex reference so your compliance officer can find it.

The same run emits a `technical-documentation.md` - the Annex VII draft - with a prominent `DRAFT - FOR MANUFACTURER REVIEW` header and every unfilled field marked `[TO BE FILLED BY MANUFACTURER: <field>]`. A Declaration of Conformity draft (Annex V full + Annex VI simplified, combined) lands in the same dossier. You hand the whole folder to your compliance team and the conversation becomes "here is what we have, here is what you need to write", instead of "where do we even start?".

The full dossier for this example is committed to the repository under [audits/0002-blog-demo/](https://github.com/jetm/shipcheck/tree/main/audits/0002-blog-demo) if you want to read the generated Annex VII and Declaration of Conformity end-to-end without running shipcheck yourself.

## Competitive landscape

If you are shopping for CRA tooling today, here is how the three common options compare:

**[EMBA](https://github.com/e-m-b-a/emba)** is an automated firmware pentester. It unpacks a binary image, runs a battery of static and dynamic analysis, and reports findings. It is excellent at what it does. What it does is not the CRA evidence layer - it is post-release firmware analysis for red teams.

**[FOSSology](https://www.fossology.org/)** is a license-compliance workbench. It scans binaries and sources for license text, tracks audits, and reviews manually. Use it to clear licenses, not to file your Annex VII.

**[shipcheck](https://github.com/jetm/shipcheck)** is the purpose-built option for the evidence layer: it consumes Yocto build output, pivots findings by Annex, and drafts the technical file. Use EMBA for pentesting, FOSSology for license review, and shipcheck for the job neither of them does - assembling the CRA evidence dossier from a Yocto build.

## Readiness is not compliance

shipcheck produces a **readiness** report, not a compliance certificate. No tool can certify you as CRA-compliant - conformity is a manufacturer's self-declaration (Annex V full, or the Annex VI simplified form) or, for the important and critical products listed in Annex III and Annex IV, a notified-body assessment. Either path requires a human signature against a completed Annex VII technical file.

Concretely:

- A **passing** shipcheck run means your Yocto build surfaces the expected artefacts and your `product.yaml` has no placeholder tokens. It does not mean your artefacts are correct, your vendor commitments are truthful, or that your product actually satisfies Annex I Part I's engineering properties.
- A **failing** run means there are gaps a regulator or notified body would ask about. Close them before you sign the Declaration of Conformity.
- The `technical-documentation.md` shipcheck emits is a **draft** with a `FOR MANUFACTURER REVIEW` banner and every unfilled field tagged `[TO BE FILLED BY MANUFACTURER: <field>]`. Each tag is a human decision, not a tooling oversight.
- The readiness score is your **team's progress signal**, not a regulator's verdict. It tells you how much of what shipcheck can see is in shape, and lets you watch the number move across builds as you close gaps. A healthy score is evidence you have resolved what the tool surfaces; signing the Declaration of Conformity remains a human act.

One-line rule: compliance is what you declare on the Declaration of Conformity. Readiness is what shipcheck can see in your build tree. They overlap, but they are not the same - and if you treat one as the other, you will sign a DoC that does not hold up to scrutiny.

## A note on the SBOM baseline

shipcheck's `sbom-generation` check validates SPDX 2.x against [BSI TR-03183-2](https://www.bsi.bund.de/SharedDocs/Downloads/EN/BSI/Publications/TechGuidelines/TR03183/BSI-TR-03183-2.pdf). BSI TR-03183-2 is the de-facto SBOM-content baseline European regulators point at when they want a concrete field list - minimum package fields, checksum requirement, supplier, license. If you are asking "but what fields are mandatory in my SBOM for CRA?", that document is the answer.

SPDX 3.0 is detected but not field-validated in v0.0.4 (validation is on the roadmap). CycloneDX is detected but not field-validated either.

## A note on CVE sources

Yocto ships `cve-check.bbclass` out of the box, which compares package versions against the NVD feed at bitbake time. It is serviceable but its matching is coarse - package name and version only - and the default `cve-summary.json` format is legacy. Two better options exist:

- `vex.bbclass` (in-tree, newer) emits per-image VEX-style JSON with the same data laid out more usefully.
- [`sbom-cve-check`](https://github.com/bootlin/sbom-cve-check) is Bootlin's out-of-tree tool that consumes your SPDX SBOM and cross-references it against NVD. Because it reads the SBOM rather than bitbake's package list, it picks up transitively included components that `cve-check.bbclass` misses, and its JSON is the richest of the three.

When shipcheck finds more than one of these in `tmp/deploy/images/`, it prefers `sbom-cve-check` output, then `vex.bbclass`, then legacy `cve-check`. If you are setting up CVE scanning from scratch, start with `sbom-cve-check` - it gives shipcheck the best input to reason over.

## Where this goes next

The v0.1 line is the first publishable release. Remaining gate items:

- Broader pilot coverage across additional Yocto image profiles
- Wider Annex I Part I coverage across the engineering-properties items
- Community outreach and maintainer feedback

v0.0.4 is usable today for the evidence-dossier workflow described in this post - an alpha line on the way to v0.1 with the gate items above rolling in next. If you try it on your own Yocto build and something breaks, [file an issue](https://github.com/jetm/shipcheck/issues) - pilot data on real-world builds is the best input to the v0.1 cut.

## Closing: CRA is sustained activity

The single most useful mental reframe I can offer is this: CRA compliance is not a one-time audit you pass and forget. It is a sustained process. Vulnerabilities appear, support periods tick down, disclosure policies need refreshing, SBOMs need regenerating after every release. The reason Article 14 and Annex I Part II are written the way they are is that the regulator is not looking for a binder; they are looking for evidence of ongoing activity.

shipcheck records every scan to a local history store so your dossier can show scan cadence and trends over time. That is the real Annex VII item 5 ("Test reports") - not a single test run, but a record of sustained testing.

If you are building embedded Linux products into 2027 and beyond, the scanner-first framing will burn a lot of engineering hours for a compliance artefact that does not actually satisfy the regulation. Start from the Annex structure instead. Figure out what is build-derivable and what is vendor-committed. Write down the vendor commitments. Let the tool draft the rest.
