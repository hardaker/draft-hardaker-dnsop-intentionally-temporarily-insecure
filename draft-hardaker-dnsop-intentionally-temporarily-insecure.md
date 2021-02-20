---
title: "Intentionally Temporarily Insecure"
abbrev: "Intentionally Temporarily Insecure"
docname: draft-hardaker-dnsop-intentionally-temporarily-insecure-00
category: bcp
ipr: trust200902

stand_alone: yes
pi: [toc, sortrefs, symrefs, docmapping]

author:
  -
    ins: W. Hardaker
    name: Wes Hardaker
    org: USC/ISI
    email: ietf@hardakers.net


normative:
  RFC2119:
  RFC4033:
  RFC4035:
  RFC4509:
  RFC6781:
  RFC8174:

informative:
  RFC7929:
  RFC7671:
  RFC7672:
  RFC6698:
  RFC4255:

--- abstract

Performing algorithm transitions in DNSSEC signing is unfortunately
challenging to get right in practice.  This document weighs the
correct, completely secure way of doing so against an alternate path
that takes a zone through an insecure state but using a significantly
simplified process. 

--- middle

# Introduction

Performing algorithm transitions in DNSSEC {{RFC4033}} signing is
unfortunately challenging to get right in practice.  This document
weighs the correct, completely secure way of doing so against an
alternate path that takes a zone through an insecure state but using a
significantly simplified process.

Section 4.1.4 of {{RFC6781}} describes the necessary steps required
when a new signing key is published for a zone that is of a different
algorithm than the currently published keys.  These are the steps that
MUST be followed when zone owners wish to have uninterrupted DNSSEC
protection for their zones.  The steps in this document are designed
to ensure that all DNSKEYs {{RFC4035}} and all DS {{RFC4509}} records
are properly validatable by validating resolvers throughout the entire
process.

Unfortunately, there are a number of these steps that are difficult to
accomplish either because they're tricky to get right timing-wise or
because current software doesn't support them easily.  For example,
the second step in Section 4.1.4 of {{RFC6781}} requires that a new
key be created, but not published, with the new algorithm (which we
refer to as K_new).  This step requires that K_new sign the zone, but
only its signatures, but not K_new itself, be put into a published
zone that is signed with the original key (K_old).  Only after this
has been published for a sufficient time length, based on the TTL, can
the K_new be safely introduced and published into the zone.

Similarly, after the new DS (DS_new) is published, K_old can be
removed but while its (still valid) signatures continue to be
published for another TTL.

Although many DNSSEC signing solutions may automate these steps,
making operator involvement unnecessary, many other tools do not
support automated algorithm updates to support these steps, and their
associated timing.  Specifically, the requirement that certain RRSIGs
be published without the corresponding DNSKEYs that created them will
likely require operators to use a text editor on the contents of a
signed zone, now introducing potential significant operator error.

This document proposes an alternate, potentially more operationally
robust but less secure approach to performing algorithm DNSKEY
rollovers.

## Requirements notation

   The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
   "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY",
   and "OPTIONAL" in this document are to be interpreted as described
   in BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear
   in all capitals, as shown here.

# Transitioning temporarily through insecurity

An alternate approach, especially when the toolsets being used do not
provide easy algorithm rollover approaches, is to intentionally make
the zone become insecure while the algorithm is swapped.  This means
removing all DS records from the parent zone during the removal of the
old key and the introduction of a new key with a new algorithm.  Zone
TTLs may be significantly shortened during this period to minimize the
period of insecurity.

Note that there are still some timing requirements that must be
followed carefully, although there are less than are required by the
proper procedure  defined in Section 4.1.4 of {{RFC6781}}.  These are
the functional steps required by this alternate transition mechanism:

1. Optional: lower the TTLs of the DS record if possible and the
   zone's SOA negative cache time

2. Remove all DS records from the parent zone

3. Wait 2 times the maximum TTL length for the DS record to expire.
   This is the most critical timing.  The author of this document
   failed to wait the required time once.  It was not pretty.

4. Remove the old keys from the zone

5. Add new key(s) with the new algorithm(s) to the zone and publish
   the zone
   
6. Wait 2 times the zone SOA's published negative cache time

7. Add the new DS record(s) to the parent zone

8. If the TTLs were modified in optional step 0, change them back to
   their preferred values.

# Operational considerations

The process of replacing a DNSKEY with an older algorithm, such as
RSAMD5 or RSASHA1 with a more modern one such as RSASHA512 or
ECDSAP256SHA256 can be a daunting task if the zone's current tooling
doesn't provide an easy-to-use solution.  This is the case for zone
owners that potentially use command line tools that are integrated
into their zone production environment.

This document describes an alternative approach to rolling DNSKEY
algorithms that may be significantly less prone to operational
mistakes.  However, understanding of the security considerations of
using this approach is paramount.

# Security considerations

DNSSEC provides an data integrity protection for DNS data.  This
document specifically calls out a reason why a zone owner may desire
to deliberately turn off DNSSEC while changing the zone's DNSKEY's
cryptographic algorithms.  Thus, this is deliberately turning off
security which is potentially harmful if an attacker knows when this
will occur and can use that time window to launch DNS modification
attacks (for example, cache poisoning attacks) against validating
resolvers or other validating DNS infrastructure.

Most importantly, this will deliberately break certain types of DNS
records that must be validatable for them to be effective.  This
includes for example, but not limited to, all DS records for child
zones, DANE {{RFC6698}}{{RFC7671}}{{RFC7672}}, PGP keys {{RFC7929}},
and SSHFP{{RFC4255}}.  Zone owners must carefully consider which
records within their zone depend on DNSSEC being available before
using the procedure outlined in this document.

Given all of this, it leaves the question of: "why would a zone owner
want to deliberately turn off security temporarily then?", to which
there is one principal answer.  Simply put, if the the complexity of
doing it the correct way is difficult with existing tooling then the
chances of performing the more complex procedure and introducing an
error, likely making the entire zone unavailable during that time
period, may be significantly higher than the chances of the zone being
attacked during the transition period of the simpler approach where
zone availability is less likely to be impacted.  Simply put, an
invalid zone created by a botched algorithm roll is potentially worse
than an unsigned but still available zone.

--- back

# Acknowledgments

The author has discussed the pros and cons of this approach with
multiple people, including Viktor Dukhovni and Warren Kumari.

# Github Version of this document

While this document is under development, it can be viewed, tracked,
issued, pushed with PRs, ... here:

https://github.com/hardaker/draft-hardaker-dnsop-intentionally-temporarily-insecure
