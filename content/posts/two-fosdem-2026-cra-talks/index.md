---
title: "Two FOSDEM talks on the CRA, from opposite sides of the table"
date: 2026-05-15T20:00:00Z
draft: false
description: "What I gathered from watching two FOSDEM 2026 talks on the Cyber
  Resilience Act: one from German IT lawyers explaining what manufacturers should
  do, one from the EU Commission and BSI explaining what the regulation actually
  requires - and what it does not require from non-commercial FOSS developers."
ShowToc: true
ShowReadingTime: true
tags:
  - "cra"
  - "fosdem"
  - "foss"
  - "compliance"
  - "security"
  - "embedded"
  - "linux"
categories:
  - "linux"
  - "regulation"
---

Two FOSDEM 2026 talks on the Cyber Resilience Act caught my attention when I
watched the recordings. One came from two German IT lawyers - Anika Niemann and
Florian Hackel - who spent twenty minutes walking an audience of open source
developers through how to think about CRA compliance from a manufacturer's
perspective. The other came from the European Commission itself, joined on stage
by representatives from CEN-CENELEC, ETSI, and the German BSI.

The regulators have been attending FOSDEM since at least 2024, when they ran a
session titled "the regulators are coming." This year, Philippe Mourão from DG
Connect opened with a deliberate callback: two years later, the regulators are
here.

The two talks approached the same regulation from opposite directions. One
explained what manufacturers need to do. The other explained what the regulation
actually requires - and, pointedly, what it does not require from
non-commercial open source developers. Together they filled in gaps that neither
covered alone.

## Non-commercial FOSS developers have no obligations under the CRA

Both talks addressed this, but Carl-Daniel Hailfinger from the BSI was the
most direct. Hailfinger is unusual in that he showed up as a representative of
Germany's cybersecurity authority and also as a self-described free and open
source software developer and maintainer of 24 years. He spent the last portion
of the BSI segment making clear that non-commercial FOSS developers have zero
obligations under the CRA: no SBOM requirements, no bug-fix obligations, no
reaction-time commitments. He cited CRA Article 2(1) and Recital 18 as the
basis and added that this holds even if someone is using your project in their
commercial product - that is their problem, not yours. On the question of
developers who claim otherwise, he was brief: they can please go away and read
the law.

Mourão confirmed this from the policy side. Only directly monetized open source
products are subject to the CRA. He distinguished carefully: charging for your
time or resources as a developer may not even constitute commercialization under
the regulation. Providing support services, branding, and placing a product on
the market under a trademark does. Guidance spelling out the boundary in detail
is still pending; Mourão said the commission treats it as a living document that
will be updated as questions arise.

The open source steward concept, which generated considerable anxiety in the
open source community during the CRA's drafting, also got a clear explanation.
An open source steward under the CRA is a specific legal category: a non-profit
legal entity that provides sustained support for open source software development
without directly monetizing it. For stewards, the obligation is light - publish
a security policy for the community. It is not a full manufacturer obligation,
and not every project maintainer qualifies. The definition requires being an
actual legal entity.

## Manufacturers have obligations toward upstream FOSS developers

This part of the regulation tends to get lost in coverage that focuses on what
the CRA demands of manufacturers. CRA Article 13(6) requires manufacturers that
integrate open source components and discover or develop security fixes to report
those vulnerabilities to the upstream maintainers and share the fixes with them,
free of charge. This is not a best-practice recommendation.

Hailfinger walked through what this means in practice. Manufacturers who use
your project in a product with digital elements must report all vulnerabilities
they find to you. They must give you their security fixes. The burden has been
deliberately shifted from FOSS developers to manufacturers.

The FOSDEM session materials included a handout - titled "Helpful replies for
FOSS developers (Not legal advice)" - that listed pre-written responses to
common manufacturer demands, each citing the relevant CRA article. One entry:

> Q: We are a manufacturer affected by a security bug in your FOSS code
> contained in our product. Fix it now!
>
> A: As the upstream non-commercial FOSS developer, I have no obligations under
> the CRA. See CRA Article 2(1) and CRA Recital 18. Oh btw, if you develop a
> fix you have to share it with me for free. See CRA Article 13(6) and
> Recital 34.

For embedded Linux teams that consume large amounts of upstream FOSS, the
implication runs in both directions. As a manufacturer, you now have a legal
obligation to disclose vulnerabilities you find in your components back to the
upstream projects that maintain them. That obligation has market surveillance
authority oversight behind it.

## What manufacturers should actually be doing

The lawyers' talk was structured as a dialogue between a manager with compliance
anxiety and a lawyer walking them through the terrain. Their framework had three
tiers of effort, tied to product risk.

Level one covers products going to external customers. These warrant the full
treatment: SBOM generation, vulnerability assessment, license evaluation,
documentation.

Level two covers products shared internally across sister or parent companies.
Moderate effort.

Level three covers internal-only software. Automated tooling and lightweight
documentation are sufficient.

Within each tier, they described three layers: the risk classification itself, a
policy document specifying who does what, and concrete role assignments
connecting each step to a person. Their example: the SBOM gets generated by the
development team in the CI/CD pipeline, the security team cross-references it
against CVE databases, and the legal team evaluates licenses. The same SBOM, two
separate evaluation paths. This framing - SBOM as the fork point between the
license track and the vulnerability track - came up independently in both talks
and seems to be becoming the standard mental model in this space.

The documentation prescription was blunt: write down everything, in one system.
If a market surveillance authority asks for it, you want a single location with
every decision, every finding, and every evaluation. The lawyers mentioned the
OpenChain security assurance specification and ISO 5230 as audit-ready templates
that provide structure without requiring you to invent the format from scratch.

## An OS standard is being written right now

One detail from the regulators' talk that is easy to miss in the broader
discussion: ETSI is developing 18 product-category-specific standards under the
CRA standardization request. EN 304 626 covers operating systems. EN 304 623
covers boot managers.

Hailfinger mentioned he is personally involved in developing both. Draft versions
are publicly accessible in a GitLab instance at labs.etsi.org/rep/stan4cra/,
and as of the FOSDEM talk in February, 13 mature drafts had already been posted.
The formal open consultation window closed at the end of March 2026; comments
submitted through national standardization body delegations are still accepted
through roughly the middle of summer.

These vertical standards matter because they translate the CRA's essential
requirements into technical specifications with assessment criteria. When a
market surveillance authority evaluates an embedded Linux product, EN 304 626
will define what "no known exploitable vulnerabilities in the default
configuration" means for an operating system in practice. The deadline for the
OS standard is October 2026.

## The timeline is no longer abstract

The CRA entered into force in late 2024. The transition period runs three years,
ending in December 2027, but not all obligations wait that long. Reporting
obligations - the requirement for manufacturers to actively report exploited
vulnerabilities and severe incidents through a Single Reporting Platform being
built by ENISA - apply from September 2026. The implementing act defining
important and critical product categories was adopted in December 2025. The
EN 304 626 deadline is October 2026.

For embedded Linux teams treating CRA readiness as a future problem, the
calendar no longer supports that framing. Hailfinger put it plainly from the
perspective of someone who is simultaneously writing the OS standard and working
on security features in upstream FOSS projects: solving technical problems is
easy; getting paid or taken seriously is harder. His hope is that the CRA becomes
the helping hand for those boring tasks. Whether it does will depend on
manufacturers taking the obligations seriously before the deadlines, not after.

The recordings of both talks are available from the FOSDEM 2026 archive:

- [CRA Integration: How FOSS compliance measures support CRA obligations](https://fosdem.org/2026/schedule/event/R98AQQ-cra_foss_compliance/)
- [Implementing the Cyber Resilience Act - Engaging with Open Source](https://fosdem.org/2026/schedule/event/EERURR-implementing_the_cyber_resilience_act_-_engaging_with_open_source/)
