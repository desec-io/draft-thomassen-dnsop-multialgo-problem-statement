---
title: DNSSEC Multi-Algorithm Problem Statement
abbrev: DNSSEC Multi-Algorithm Problem Statement
docname: draft-thomassen-dnsop-multialgo-problem-statement-latest
date: {DATE}
category: info

ipr: trust200902
area: "Operations and Management"
workgroup: "Domain Name System Operations"
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: P. Thomassen
    name: Peter Thomassen
    org: deSEC, Secure Systems Engineering
    email: peter@desec.io
 -
    ins: D. Wessels
    name: Duane Wessels
    org: Verisign
    email: dwessels@verisign.com
 -
    ins: S. Huque
    name: Shumon Huque
    org: Salesforce
    email: shuque@gmail.com

normative:

informative:
  RolloverCZ:
    target: https://indico.dns-oarc.net/event/29/contributions/651/attachments/638/1033/ecdsa-in-cz.pdf
    title: "ECDSA in .CZ: Story of first usage of ECDSA in TLD space"
    author:
      - name: Jaromír Talíř
    date: 2018
  RolloverBR:
    target: https://ftp.registro.br/pub/gts/gts32/01-br-algorithm-rollover.pdf
    title: Algorithm Rollover on .br
    author:
      - name: Cesar Kuroiwa
    date: 2018
  RolloverNL:
    target: https://www.sidn.nl/en/news-and-blogs/looking-back-at-nls-algorithm-rollover
    title: Looking back at .nl's algorithm rollover
    author:
      - name: Stefan Ubbink
      - name: Jeroen Bulten
      - name: Niek Willems
    date: 2023
  RolloverVerisign:
    target: https://indico.dns-oarc.net/event/47/contributions/1012/attachments/973/1821/verisign-tld-updates.pdf
    title: Transitioning Verisign’s TLDs to Elliptic Curve DNSSEC
    author:
      - name: Duane Wessels
    date: 2023
  RolloverDesignTeam:
    target: https://mailarchive.ietf.org/arch/msg/dnsop/KFI0bOgeaGj509HkW-1tUxJVEuY/
    title: "[DNSOP] Meet the Root Zone Algorithm Rollover Design Team @ IETF 116"
    author:
      - name: James Mitchell
    date: 2023
  IANAproposal:
    target: https://www.icann.org/en/system/files/files/proposal-future-rz-ksk-rollovers-01nov19-en.pdf
    title: Proposal for Future Root Zone KSK Rollovers
    author:
      - org: IANA
    date: 2019
  DSHack:
    target: https://archive.icann.org/meetings/icann73/files/content=t_attachment,f_%223.7%20Huque%20RFC%20Adjustments%20for%20Multi-Signer.pdf%22/3.7%20Huque%20RFC%20Adjustments%20for%20Multi-Signer.pdf
    title: RFC Adjustments for Multi-Signer
    author:
      - name: Shumon Huque
    date: 2022

--- abstract

There is an increasing awareness for use cases that rely on signed DNS responses
to contain RRSIGs for only some (not all) of the algorithms advertised in a
domain's DS RRset or trust anchor set. This document looks at these use cases in
detail, and describes how validating resolvers are expected to react today when
encountering these situations. Mostly, such setups work fine, but they are not
compliant with specifications. To address this problem, the document walks
through a number of solution approaches, each based on a different idea, and
points out each one's benefits and disadvantages.

TO BE REMOVED: This document is being collaborated on in Github at:
[https://github.com/peterthomassen/draft-thomassen-dnsop-multialgo-problem-statement](https://github.com/peterthomassen/draft-thomassen-dnsop-multialgo-problem-statement).
The most recent working version of the document, open issues, etc. should all be
available there.  The authors (gratefully) accept pull requests.

--- middle

# Introduction

Although DNSSEC has been designed with algorithm agility in mind, *most* zones
today use only one signing algorithm at a time, especially when in a
steady-state configuration. Multiple signing algorithms are usually in use only
for algorithm rollovers and for some multi-provider configurations, both of
which are relatively rare. Nevertheless, a number of notable zones have
performed algorithm rollovers (such as .cz in 2010 and 2018 {{RolloverCZ}},
followed by .br {{RolloverBR}}, and most recently .nl {{RolloverNL}} and
.edu/.net/.com {{RolloverVerisign}}). Such algorithm rollovers come with a
temporary transition period where both the old and the new algorithm are in use
for the zone in question.

There also is an increasing demand for DNSSEC-enabled multi-provider setups (as
evidenced by existing support at Cloudflare and deSEC, plus implementation work
going on at UltraDNS and others). According to the specs, all providers involved
in such a setup would use the same set of signing algorithms. However, for a
variety of reasons (including those given below), this is not a realistic
premise. As a result, configurations are expected to occur in the wild where
each provider's DNS response contains an RRSIG for only one algorithm, although
several are advertised in the zone's DS RRset. The ongoing uncertainty about how
to deal with such situations might become an impediment to predictable behavior
in multi-signer DNSSEC setups, and thus to multi-signer DNSSEC itself.

Further, during an algorithm rollover, when both the old and the new DS record
have been deployed, there may be benefits (such as smaller response sizes, see
{{rollovers}}) in not always requiring nameservers to serve signatures of either
algorithm. This desire has been expressed in a presentation by the Root Zone
Algorithm Rollover Study Design Team at IETF 116 {{RolloverDesignTeam}}.

TODO: actual presentation not publicly available; should we make it available?

Looking closer, it turns out that these two problems are two sides of the same
coin; they are actually the same problem.

This document looks at these use cases in more detail, and describes how
validating resolvers react today when encountering these situations. It then
walks through a number of ways to address the problem, each based on a different
idea, and points out each one's benefits and disadvantages.

# Use cases for serving RRSIGs for only a subset of advertised algorithms {#usecases}

Current specifications {{!RFC4035}}{{!RFC6840}}} require that a zone be signed with
each signing algorithm listed in a zone's DS RRset or appearing via its trust
anchors (TAs).

This section explores a number of use cases where this poses a problem.

## Algorithm Rollovers {#rollovers}

When performing an algorithm rollover, current specifications effectively
require double-signing the zone with both the old and the new algorithm before
adding the new DS record. (Because the old DS record may be cached, this also
applies when *replacing* the DS record, which is not an instantaneous
operation.)

Double-signing a zone leads to larger zone files, larger responses, higher risk
of amplification attacks, larger resource usage and increased latency due to
more frequent TCP fallback. For online signers, it causes additional real-time
computational overhead.

In addition to these (and perhaps other) operational risks, double-signing may
also introduce financial burden (of course, depending on the size and/or qps
rate of the zone).

Those problems could be avoided if a (safe) way was found to allow responses
that have signatures of only a subset of advertised algorithms.

### Root Zone Algorithm Rollover

This problem also appears when rolling the root zone algorithm, which requires
updating resolvers' trust anchors. It is expected that signatures for the old
algorithm need to be in place for a significant amount of time, while resolver
operators are adding the new trust anchor to their configuration.

This problem is exacerbated for a root zone algorithm rollover, where updating
the trust anchor data used by resolvers is a slow and deliberate process.
{{!RFC5011}} stipulates a new key must be pre-published for at least 30 days
before it can be considered trusted. Additionally, IANA's documented process for
future KSK rollovers pre-publishes a new key in the DNS for a year and a half
(phase D of {{IANAproposal}}), and pre-publishes it in IANA’s trust anchor file
two years (phases B through D). This applies both with or without a change to
the signing algorithm, so that resolver operators can update their trust anchor
configuration before the new key is actually used for signing.

Current specifications require that, in this case, the zone be double-signed
during that period of time ({{!RFC6840}} Section 5.11). For the root zone, this
could lead to a potentially rather long phase of double-signing (on the order of
a year) during which the above disadvantages would manifest.

It thus seems desirable to find a way for publishing the new trust anchor
without introducing the new algorithm into the zone just yet, as emphasized by
the Root Zone Algorithm Rollover Study Design Team at IETF 116
{{RolloverDesignTeam}}.

## DNSSEC-Enabled Multi-Provider Setups

### Permanent (for Redundancy)

Some sites use several independent DNS providers, allowing to compensate for a
full outage of one of the providers. One such site is github.com, which uses
services by both Amazon Route 53 and NS1. Such highly redundant setups lead to
very high DNS availability.

When combined with DNSSEC, with each provider holding their own signing key(s),
this is called a "multi-signer setup" {{?RFC8901}}. This allows multiple
providers using distinct DNSSEC keys to cooperatively serve the same DNS zone.

Under current specifications, these methods are illegal if the providers
involved employ KSKs (or CSKs) with different DNSSEC algorithms. However, for
reasons explained below, it is not a realistic expectation to have all providers
use the same set of signing algorithms at all times; in fact, requiring this
prevents any one provider from changing the algorithm. As a result,
cryptographic hygiene effectively stalls.

### Temporary (for Provider Change)

Multi-signer setups can also be employed temporariliy in order to
non-disruptively transfer a signed zone from one provider to another, each of
which uses their own signing key. It is conceivable that this process be fully
automated in the future.

If the old and the new provider do not use the same signing KSK (or CSK)
algorithms, such transfer cannot be done legally under current specifications.
However, it is not a realistic expectation that both providers support the same
set of algorithms (in particular when the reason for the transfer is outdated
operations at the former provider).

# Resolver Behavior

While the current specifications don't permit the situations described in
{{usecases}}, resolvers usually deal with them just fine as long as they find a
signature that allows verifying the response.

Still, there is uncertainty about what precisely would work and what would not.

This section describes how validators behave in these and related scenarios,
such as when encountering unsupported algorithms.

## Expected Behavior {#expectedbehavior}

Assume a delegation with a DS RRset advertising two algorithms A and B. If a
resolver supporting only algorithm A receives a response that carries an RRSIG
for algorithm B (but not for algorithm A), the response is considered bogus and
SERVFAIL results.

Consequently, it is not safe to omit signatures for algorithms announced in
the DS RRset: the result is dependent on the set of algorithms supported by the
resolver -- which the authoritative server cannot predict in the general case.
An authoritative nameserver therefore runs the risk of provoking a SERVFAIL
response to the resolver's client unless it serves signatures for all algorithms
advertised in the DS RRset.

Note that there's no risk of SERVFAIL when the resolver supports none of the
advertised algorithms: in this case, the zone is treated as insecure.

For a summary, see the following table.

Resolvers Supports | Algorithms in DS | Algorithms in RRSIG | Expected Result
------------------ | ---------------- | ------------------- | ---------------
   A,B             |    A,B           |    A                |    valid
   A,B             |    A,B           |    B                |    valid
   A,B             |    A,B           |    A,B              |    valid
   A               |    A,B           |    A                |    valid
   A               |    A,B           |    A,B              |    valid
   A               |    A,B           |    B                |    bogus
   A,B             |    C             |    C                |    insecure

## Actual (Testbed) Behavior

TODO

[Duane: Maybe we also want to set up some testbed zones and see how implementations actually behave.]

[Shumon: I have personally in the past setup such testbed configurations, and have confirmed that the big 4 validators (BIND, Unbound, powerdns, knot) can in fact validate them fine. But I don't currently have one available publicly. I could probably set something up again though if needed (when I have some free time).]


# Challenge

Given the increasing demand for DNSSEC-enabled multi-provider setups, it seems
like such setups *will* sooner or later appear in the wild (or perhaps already
have appeared). In fact, they *have* appeared for KSK algorithm rollovers, such
as at .cz in 2010 {{RolloverCZ}} and at .br in 2018 {{RolloverBR}}.

From a signer's perspective, just "going ahead" is a reasonable thing to do, as
for most configurations that one would practically deploy, serving just one
signature *will* work. For example, when a zone's DS RRset advertises algorithms
8 and 13, resolvers are going to accept responses that carry signatures for only
one of these algorithms. In fact, pre-rollover testing by registro.br had shown
that adverse effects are expected to be negligible {{RolloverBR}}.

However, things break down as soon as one of the involved algorithms is no
longer expected to be supported everywhere. For example, if the DS RRset
advertises algorithms 5 and 8 (either permanently, or during a provider
transfer), this might have worked a year or two ago. Today, resolvers running on
a new Red Hat Linux release that has since appeared do no longer support
algorithm 5 (due to a system-wide algorithm policy). When receiving a response
with an algo-5 signature only, they will respond with SERVFAIL.

As observed in {{expectedbehavior}}, this is related to the fact that an
authoritative nameserver cannot reliably predict which algorithms a resolver
might support. It's even harder to predict developments in algorithm support
over time.

Leaving this problem space without guidance seems like a dangerous path. With
some serious use cases, it is time to start thinking about solutions that can
make them work predicably and reliably, both now and in the future.


# Solution Approaches

This section describes ideas for accommodating the use cases described above,
and points out aspects of them that are particularly favorable or problematic.

There's no expectation that this is a complete or gapless analysis. Rather, it
is meant to structure the solution space, and rule out some options that look
promising at first sight, but turn out problematic upon closer inspection.

## Just loosen the rules, "any RRSIG is fine"

The proposal is to allow serving only one RRSIG for any single algorithm
advertised in the DS RRset. If that algorithm isn't understood by the resolver,
return the response as insecure (don't SERVFAIL).

This would work well if only done with algorithms that are known to be supported
everywhere. For example, with algorithms 8 and 13 in the DS RRset, resolution
and validation are expected to work smoothly under the modified specification.

However, this minimal approach does not offer any safety nets to make sure that
this is really only done with generally supported algorithms. Codifying this
approach without any further guidance on the algorithm constellations for which
this is safe to do seems to miss the core of the problem, namely that algorithm
support in resolvers is not everywhere and always the same. This becomes a
significant problem when considering the time dependence of algorithm support,
as follows:

In 2020, this approach would have worked with a provider using algorithm 5 or 7
(which use SHA-1 for input hashing) and another provider using 8 or 13: any
algorithm combination from these sets (such as 5 and 8) was supported virtually
everywhere.

With the new Red Hat Linux release that has since appeared, resolvers running on
it no longer support algorithms 5 and 7. As a result, when receiving an algo-5
or algo-7 RRSIG for a zone whose DS RRset also advertises 8 or 13, such
resolvers will treat the response as insecure under the proposed rules. This
allows an on-path attacker to downgrade the zone to insecure simply by switching
the RRSIG algorithm number fields in all responses to 5 or 7.

This implication opens up a trivial vector for downgrade-to-insecure attacks,
and drastically changes the security model.

This downgrade attack vector also impacts a DNS operator's capability to
configure a new algorithm in addition to a very common one. This even applies to
a single-provider setup where the operator intends to always serve RRSIGs for
two algorithms. An operator could want to do this for at least two reasons:

- To gain experience with some new algorithm, for experimental purposes; or
- When a new class of algorithms is created and the operator desires dual-mode
  operation as long as that new algorithm class is not universally supported
  (e.g., some future PQC algorithm combined with a conventional algorithm for
  legacy clients).

The proposed specification update thwarts this mode of operation, because any
resolver that does not support the additionally deployed algorithm can be
trivially caused to return unvalidated responses, despite the other algorithm
being supported. All that is needed is for the attacker to change the RRSIG's
algorithm number to the unsupported one.

This makes it extremely unattractive for DNS operators to deploy additional
algorithms, even when known to deliver additional security benefits, or when
their deployment would be beneficial for other reasons (such as gaining
experience). Experiments and early-adopter deployments with new algorithms would
be stalled, and as a result, cryptographic agility is severely hindered by this
approach.

## Serve all RRSIGs

As this violates the goal of loosening the requirement somehow, it is not really
a solution idea, but rather invites rethinking what are (or aren't) the problems
when we keep doing what the specs say.

### Root zone (+ other zones with trust anchors)

For the reasons laid out by the Root Zone Algorithm Rollover Study Design Team
{{RolloverDesignTeam}}, the root will have difficulty following this practice.
The preferred approach would be to publish a trust anchor for the new algorithm
and allow it to be configured on resolvers, without yet introducing the
corresponding RRSIGs into the zone.

Note that unlike in the DS case, the situation where all trust anchor algorithms
are unsupported cannot plausibly occur, as it does not seem reasonable to
configure an unsupported trust anchor in the first place. Put differently, the
set of configured trust anchors is tailored for the validator at hand (unlike
the DS RRset, where the signer cannot assume that all configured algorithms are
supported).

The addition of a trust anchor with a new algorithm (which may not yet be
accompanied by corresponding RRSIGs) is thus *completely inconsequential, as
long as an existing trust anchor continues to work* (i.e., corresponding DNSKEYs
and RRSIGs are served). This can be expected to be the case if the existing
trust anchor’s algorithm is one that validators MUST support (see {{Section 3 of
!RFC8624}}).

Thus, there seems to be no good reason to insist on serving the new RRSIGs when
publishing a trust anchor with a new algorithm; the signer only MUST ensure that
a pre-existing, mandatory-to-support algorithm is available and works throughout
the pre-publication phase.

Based on this analysis, it seems conceivable to loosen the rules for the root
(or, more generally, zones with trust anchors), without loosening the rules for
RRSIG presence in the context of DS-advertised algorithms.

### Zones with DS RRset

As described further up, there are multiple scenarios in multi-provider setups
where providers might want to use different algorithms for their SEP keys.

Enforcing them to use the same algorithm is not practical. There may be
practical reasons preventing that (e.g., two providers using HSMs whose sets of
supported algorithms doesn't overlap; an operator may be uncooperative).

Further, requiring all providers to use the same algorithms stalls all attempts
for major providers to transition to future algorithm, unless all introduce
support at the same time -- which seems an unrealistic premise.

For these reasons, we have to expect live setups where RRSIGs are not present
for all algorithms. Reiterating the specs is denying reality; we need to think
harder.


## Retry elsewhere when an expected signature is missing {#retry}

The idea here is to ask another nameserver when the response does not contain an
RRSIG with a supported algorithm from the DS RRset. For example, if the DS RRset
advertises algorithms 5 and 8, with only the latter supported by the resolver,
the resolver would ask another nameserver if the response only
contained an RRSIG with the unsupported algorithm (5).

This reflects the notion that *some* provider in a multi-provider setup will
have requested inclusion of that DS record, and will have nameserver in the NS
RRset. Thus, if you keep asking another one, you will eventually get a response
with an RRSIG corresponding to your favorite DS-advertised algorithm.

However, there are multiple problems here:

  - Retries add extra latency, bandwidth usage, and other resource consumption
    such as for encrypted tranpsort.

  - More critically, this approach undermines the very guarantees a
    multi-provider setup is supposed to give. This is because if one provider's
    RRSIG is not supported, resolution depends on the other provider. As a
    result, providers share fate, which is exactly what multi-provider should
    defend against. (Otherwise, why have multiple providers?)

## DELEG

The DELEG record is envisioned to modernize the way DNS delegations work, by
providing a signed service binding of the delegation owner name to a nameserver
{{?I-D.dnsop-deleg}}. Multiple DELEG records can be published by the parent (just
like multiple NS records), and each of them can carry key-value parameters
pertaining to the nameserver it points at.

This mechanism is expected to be capable of signaling DNSSEC parameters on a
per-nameserver basis, exposing which SEP algorithms are in use at each
nameserver (provider). When the algorithm sets vary across nameservers, a
validating resolver may selectively query only those that are expected to return
supported RRSIGs. This would get rid of the undesirable cases where the resolver
happens to ask a "wrong" nameserver that only serves an unsupported signature,
and also removes the extra latency of the "retry" approach ({{retry}}).

However, such selective querying undermines multi-provider guarantees: When one
provider's set of nameservers is excluded from querying, the setup is
effectively reduced to a single-provider setup (just like in the "retry"
proposal). In other words, although it may seem tempting at first sight,
nameserver-specific signaling is not fit for purpose in a multi-provider
context. (DELEG certainly has great potential, but *this* is not amongst the use
cases it enables.)

Further, resolvers that do not support DELEG will be around for a very long
time; backwards-compatible delegations without DELEG will thus have to work for
the foreseeable (and unforeseeable) future. Even if DELEG *did* help with the
problem at hand, the requirement of a working classical fallback would leave
legacy resolvers in the dark. Indeed, this is not only the case for permament
multi-provider setups, but in all of the use cases mentioned above: root trust
anchor pre-publishing, uncooperative provider transfer etc. work no better when
DELEG is available.

## DS hacks to signal RRISG presence rules

It has been proposed that special-purpose DS records could signal the manner in
which the domain owner wishes the set of DS-advertised algorithms to be use for
validation (such as, "all of them", "any of them", "opportunistic", ...)
{{DSHack}}.

This approach is problematic for several reasons:

  - It is not backward-compatible (requires all resolvers to upgrade).

  - It does not work with registries that consume DNSKEY-style records and
    compute DS records themselves.

  - It is entirely unclear how this kind of signaling would be compatible with
    CDS provisioning (RFC 7344) with multiple providers.

## What else ...?

There may be other appraoches that no one has thought of.


# Summary

The above solution approaches turn out to be insufficient to address the problem
space:

  - Structural modifications (DS hack etc.) in general are problematic, as they
    consistute a significant change to the protocol framework.

  - Ignoring certain nameservers is not pratical (retry/DELEG),

  - Serving all RRSIGs is not pratical.

  - Just relaxing the rules is not practical.

These proposals quite distinctly take sides either with availability (e.g.,
"just relax the rules") or security ("ignore certain nameservers"). As such,
they are rather uncompromising.

However, it is conceivable to design a more measured solution by combining
aspects from these approaches.

Perhaps, it's possible to relax requirements for some algorithms that are
generally understood (so that a signer might choose "one from the pool"), while
keeping tighter rules for algorithms where relaxed rules would pose risks.

Perhaps, it's also possible to combine this with a well-considered retry
mechanism.

And perhaps there are directions nobody has thought of. In any case, it seems
worthwhile to think hard about a proper solution.


# Acknowledgements

Edward Lewis

--- back


# Security Properties in Multi-Algorithm Configurations

When a delegation is secured with DNSSEC, then a compliant validating resolver
supporting at least one of the DS algorithms will either return validated data
or SERVFAIL. When the delegation has multiple DS algorithms, there is not
expectation that any particular one will be used for validation.

Downgrade-to-insecure resistance: It is currently only allowed to treat
responses as insecure when the resolver supports *none* of the algorithms
advertised via DS. Otherwise, and on-path attacker could downgrade to insecure
by simply stripping RRSIGs with supported algorithms from the response (leaving
only unsupported ones). The appropriate reaction to bogus responses like this is
SERVFAIL.

*No* algorithm "downgrade" resistance: If a resolver supports several algorithms
that are advertised in a delegation's DS RRset, and responses contain RRSIGs for
all of these algorithms, an on-patch attacker may generally force an algorithm
of their choice to be used for validation, by stripping (or invalidating) the
other RRSIG records. This is because according to {{Section 5.11 of !RFC6840}},
"Validators SHOULD accept any single valid path.  They SHOULD NOT insist that
all algorithms signaled in the DS RRset work". (However, some resolvers can be
configured to insist, e.g., Unbound's `harden-algo-downgrade` setting.)

The notion of "algorithm downgrade" seems to imply a hierarchy of algorithms.
This is not the intended meaning. Instead, the term is commonly used to express
that control over the selection of a validation path effectively is with the
attacker, not with the validator.


# Change History (to be removed before publication)

* draft-thomassen-dnsop-multialgo-problem-statement-00

> Initial public draft.
