---
title: "Intentionally Temporarily Degraded or Insecure"
abbrev: "Intentionally Temporarily Degraded or Insecure"
docname: draft-hardaker-dnsop-intentionally-temporary-insec-latest
category: bcp
ipr: trust200902

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
  -
    ins: W. Hardaker
    name: Wes Hardaker
    org: Google
    email: ietf@hardakers.net


normative:
  RFC1035:
  RFC2119:
  RFC4033:
  RFC4035:
  RFC4509:
  RFC6781:
  RFC8174:
  RFC4034:

informative:
  RFC7929:
  RFC7671:
  RFC7672:
  RFC6698:
  RFC4255:
  RFC9904:
  RFC9905:
  RFC5702:
  RFC6605:

--- abstract

Performing DNSKEY algorithm transitions with DNSSEC signing is
unfortunately challenging to get right in practice without decent
tooling support.  This document weighs the correct, completely secure
way of rolling keys against an alternate, significantly simplified,
method that takes a zone through an insecure state first.

--- middle

# Introduction

Performing DNSKEY {{RFC4035}} algorithm transitions with DNSSEC
{{RFC4033}} signing is unfortunately challenging to get right in
practice without decent tooling support.  This document weighs the
correct, completely secure way of rolling keys against an alternate,
significantly simplified, method that takes a zone through an insecure
state.

Section 4.1.4 of {{RFC6781}} describes the necessary steps required
when a new signing key is published for a zone that uses a different
signing algorithm than the currently published keys.  These are the
steps that MUST be followed when zone owners wish to have
uninterrupted DNSSEC protection for their zones.  The steps in this
document are designed to ensure that all DNSKEY records and all DS
{{RFC4509}} records (and the rest of a zone records) are properly
validatable by validating resolvers throughout the entire process.
Whenever possible, this procedure SHOULD be followed.

Unfortunately, there are a number of these steps that are challenging
to accomplish either because the timing is tricky to get right or
because current software doesn't support automating the process
easily.  Some examples:

1. The second step in Section 4.1.4 of {{RFC6781}} requires that a new
   key with the new algorithm (which we refer to as K_new) be created,
   but not yet published.  This step requires that both the old key
   (K_old) and K_new both sign and generate signatures for the zone,
   but without K_new actually being published in the zone even though
   its signatures.  Put another way, only K_old can exist in the zone
   even though signatures from both keys must be included.  After this
   odd mix has been published for a sufficient time length, based on
   the TTL, can K_new be safely introduced and published into the zone
   as well.

2. Sometimes one of the goals is to transfer zone management to new
   authoritative server software. But, if the newly desired algorithm
   isn't supported in the existing (to be replaced) DNSSEC signing
   software, then the transfer to the new software must be
   accomplished first.  However, if there isn't an overlap between the
   algorithms available in both software sets, it becomes practically
   impossible to even transfer the zone since neither software set can
   use both K_old and K_new.


Although some DNSSEC signing solutions may automate the algorithm
rollover steps (making operator involvement unnecessary), many other
tools do not yet support automated algorithm updates.  In these
environments, the most challenging step is requiring that certain
RRSIGs be published without the corresponding DNSKEYs that created
them.  This will likely require operators to use a text editor on the
contents of a signed zone to carefully select zone records to extract
before publication.  This introduces potentially significant operator
error(s).

This document proposes an alternate approach that MAY be used to
perform algorithm DNSKEY rollovers in these situations, which is
potentially more operationally robust but less secure.

## Requirements notation

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY",
   and "OPTIONAL" in this document are to be interpreted as described
   in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear
   in all capitals, as shown here.

# Transitioning temporarily through insecurity

An alternate approach to properly rolling DNSKEYs to a new algorithm,
is to intentionally make the zone become insecure while the DNSKEYs
and algorithms are swapped.  At a high level, this means removing all
DS records from the parent zone first, then remove the old key and
introduce the new key with its new algorithm during this period.  Zone
TTLs can be significantly shortened during this period to minimize the
period of insecurity.

Below are the enumerated steps required by this alternate transition
mechanism.  Note that there are still two critical waiting time
requirements (steps 2 and 6) that must be followed carefully.

1. Optional: lower the TTLs of both the zone's DS record, and the TTL
   of the DNSKEY RRset.  Note that in some operational deployments the
   parent zone may set the TTL of the DS record.

2. Remove all DS records from the parent zone.

3. Ensure the zone is considered unsigned by all validating resolvers
   by waiting 2 times the maximum TTL length for the DS record, and/or
   2 times the largest TTL found in the zone (whichever is larger).
   This is the most critical timing as all records associated with
   K_old must be cleared from validating resolver caches.  (The author
   of this document failed to wait the required time once.  It was not
   pretty.)

4. Replace the old DNSKEY(s) using the old algorithm with new
   DNSKEY(s) using the new algorithm(s) in the zone and publish the
   zone.

6. Wait 2 times the largest TTL found in the zone to ensure
   the new DNSKEYs will be found by validating resolvers.

7. Add the DS record(s) for the new DNSKEYs to the parent zone.

8. If the TTLs were modified in the optional step 1, change them back
   to their preferred values.

# Operational considerations

The process of replacing a DNSKEY with an older algorithm {{RFC9904}},
such as RSAMD5 {{RFC4034}} or RSASHA1 {{RFC9905}} with a more modern
one {{RFC9904}} such as RSASHA512 {{RFC5702}} or ECDSAP256SHA256
{{RFC6605}} can be a daunting task if the zone's current tooling
doesn't provide an easy-to-use solution.  For example, this may be the
case for zone owners using command line tools integrated into their
zone production environment.

This document describes an alternative approach to rolling DNSKEY
algorithms that may be significantly less prone to operational
mistakes.  However, it is paramount that operators understand of the
security considerations of using this approach.

The document recommends waiting 2 times TTL values in certain cases
for added assurance that the waiting period is long enough for caches
to expire.  In reality, waiting slightly longer than 1 TTL may be
sufficient but requires accepting added risks with propagation timing
and clock synchronization.

# Security considerations

DNSSEC provides data integrity protection for DNS data.  This document
specifically calls out a reason why a zone owner may desire to
deliberately turn off this protection while changing the zone's
DNSKEY's cryptographic algorithms.  Thus, this technique is
potentially harmful if an attacker knows when this will occur and can
use that time window to launch DNS modification attacks (for example,
cache poisoning attacks) against validating resolvers or other
validating DNS infrastructure.

Most importantly, this will deliberately break certain types of DNS
records that must be validatable for them to be effective.  This
includes for example, but not limited to, all DS records for the
zone's own children, DANE {{RFC6698}}{{RFC7671}}{{RFC7672}}, PGP key
fingerprints {{RFC7929}}, and SSHFP{{RFC4255}} fingerprints.  Zone
owners must carefully consider which records within their zone and
their zone's children depend on DNSSEC being available before using
the procedure outlined in this document.

Given all of this, it leaves the question of: "why would a zone owner
want to deliberately turn off security temporarily then?", to which
there is one principal answer: if the the complexity of executing an
algorithm role the correct way is difficult (or impossible), then the
chances of introducing an error that causes an operational outage may
be significantly higher than the chances of the zone being attacked
during the insecure transition period.  Simply put, an invalid zone
created by a botched algorithm roll is potentially worse than an
unsigned but still available zone.

--- back

# Acknowledgments

The author has discussed the pros and cons of this approach with
multiple people, including:

- Vladimír Čunát
- Peter van Dijk
- Viktor Dukhovni
- Warren Kumari.
- Scott Rose
- Tuomo Soini
- Paul Wouters

# Github Version of this document

While this document is under development, it can be viewed, tracked,
issued, pushed with PRs, ... here:

https://github.com/hardaker/draft-hardaker-dnsop-intentionally-temporarily-insecure
