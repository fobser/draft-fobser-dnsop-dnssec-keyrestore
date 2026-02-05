---
title: "DNSSEC Key Restore"
category: info

docname: draft-fobser-dnsop-dnssec-keyrestore-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Operations and Management"
workgroup: "Domain Name System Operations"
keyword:
 - DNSSEC
 - disaster recovery
 - backup and restore
#venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
#  github: USER/REPO
#  latest: https://example.com/LATEST

author:
 -
    fullname: Florian Obser
    organization: RIPE NCC
    email: fobser@ripe.net
 -
    fullname: Martin Pels
    organization: RIPE NCC
    email: mpels@ripe.net

normative:
  RFC9364:
informative:
  RFC6781:
  RFC7583:
  RFC7958:
  RFC8078:
  RFC9499:
  RFC9824:
  Shamir:
    title: How to Share a Secret
    author:
     -
        fullname: Adi Shamir
    seriesinfo:
      "ACM Press": "Communications of the ACM, Vol. 22, No. 11, pp. 612-613"
      DOI: 10.1145/359168.359176
    date: 1979-11

--- abstract

This document describes the issues surrounding the handling of DNSSEC
private keys in a DNSSEC signer. It presents operational guidance in
case a DNSSEC private key becoming inoperable.

--- middle

# Introduction

DNSSEC {{RFC9364}} uses public key cryptography to provide integrity
protection of DNS data. From an operational point of view, it is
critically important to keep the private key secret under all
circumstances.

The private key is typically kept secret by using Hardware Security
Modules (HSMs). HSMs are designed to perform cryptographic operations
such as creating keys and signing messages without disclosing the
private key. Alternatively the DNSSEC signer is an appliance or
commodity server hardware and operational policy stipulates that the
private key must not leave the signer.

Operationally this is a risk because only a single key exists. The
key could become inoperable at any point due to hardware failure,
natural disaster, operator error, or malicious action.

It is difficult to create backups of the private key. After all, the
system is designed to prevent backups. A compromise is usually reached
by using a secret sharing scheme, e.g. {{Shamir}}. The private key is
split into N pieces inside of the HSM, which are then distributed to
key share holders. In case the private key becomes inoperable, M out
of the N key share holders need to come together to restore the secret
key.

A key sharing scheme does not mitigate all risk. When
more than N-M key shares become unavailable a restore cannot be
performed, because not enough key shares are available. This is
particularly challenging in small to medium sized teams.

Unlike the private key, a DNSSEC signed zone can be considered public
data with its integrity protected by signatures. Signed zones can be
added to the normal, established backup procedures.

The rest of the document describes procedures on how to restore DNSSEC
signing functionality with only a backup of the signed zone available.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses DNS terminology from {{RFC9499}}.
DNSSEC key states and timeline related abbreviations are defined in
{{RFC7583}}.

The following additional definitions are used within this document.

Inoperable (private key):
: The private part of a DNSKEY appearing in the chain of trust of
  the zone that can no longer be used for signing. Causes include
  hardware failure, natural disaster, operator error, or malicious
  action. A compromised key is not an inoperable private key since it
  can still be used for signing.

Operable (private key):
: The opposite of an inoperable private key. A key that can be used
  for signing.

# Scope

The procedures described in this document pertain to DNSSEC architectures
with pre-signed records. Online signing, such as described in {{RFC9824}},
is out of scope since it requires that each server carrying the zone holds
a copy of the signing key(s). Thus, the operational challenges are different
than described in the introduction.

The root zone is out of scope since the distribution of a new trust
anchor takes considerably longer than the RRSIG lifetime {{RFC7958}}.

# DNSSEC Key Restore

In case of a catastrophe where the DNSSEC private key becomes
inoperable and no functioning backups of the private key are
available, it is desirable to recover from this situation with DNS
resolution continuing to work for the effected zone(s) while
performing DNSSEC key restore operations.

This is possible because the moment the DNSSEC private key becomes
inoperable, the zone is still correctly signed and served by the
authoritative name servers. Signatures typically have a lifetime of
many days. That means that the operator has a lot of time to recover
from this situation without the zone becoming bogus and no longer
validating. Hasty and inappropriate action on the other hand could
lead to outages.

While the DNSSEC private key cannot be restored because no functioning
backups exist, the function of the zone can be restored.

The restore process uses slightly modified key rollover procedures
from {{RFC7583}}.

During the restore process, the signing software operates on a
pre-signed zone. That is, the zone already contains a DNSKEY RRset and
RRSIG RRsets. The signing software might try to remove these records
because the accompanying private key is no longer present. The operator
MUST prevent this, otherwise the zone will become bogus.

The signing software MUST NOT remove DNSKEYs until instructed to do so
and SHOULD NOT remove old RRSIGs. If a signer implementation does
not support keeping the old RRSIG records in place these records,
excluding the RRSIG for the old DNSKEY RRset, MUST be manually added back
to the zone before publication.

The exact process depends on which key(s) are inoperable and if the
zone is signed with a split KSK / ZSK key pair or a Combined Signing
Key (CSK).

Performing an Algorithm Rollover as described in {{RFC6781}} using the
procedures defined in this document is NOT RECOMMENDED. If an
algorithm rollover is not already in progress, signing using the
currently used algorithm should be restored first using the procedures
defined in this document. Once this has been completed a regular
algorithm rollover can be performed.

## Key Rollover Considerations

If a regular key rollover is in progress, the procedures described in
this document can be followed. They effectively cancel the ongoing key
rollover and perform a new one.

If an algorithm rollover is in progress the procedures described in
this document can be followed, with the exception that a that a new key
MUST be added to the zone per algortihm for which there is an inoperable
key.

## SOA considerations

When restoring an inoperable ZSK or CSK, the SOA record of the zone SHOULD NOT
be changed when introducing a new key in the DNSKEY RRset, because the SOA 
cannot be re-signed with the inoperable key. In case the SOA is changed, signed
responses for existing records will remain valid, but denial of existence 
proofs for non-existent record types will become bogus.

To ensure the zone is still propagated, any secondary name servers relying on
IXFR/AXFR need to be manually forced to load the new version of the zone.

## CDS/CDNSKEY considerations

For restoring an inoperable KSK or CSK, a new DS record needs to be added
to the parent zone. For child zones where this update process is ordinarily
handled using CDS/DNSKEY records (see {{RFC8078}}) the DS update needs to
be performed manually if the ZSK or CSK is inoperable. This is because
CDS/DNSKEY records added to the child zone cannot be signed with the inoperable
key, and thus cannot be cryptographically validated. Additionally, introducing
CDS/CDNSKEY records in the zone would change the type bitmap of the NSEC or 
NSEC3 record in the zone apex, which also cannot be re-signed with the 
inoperable key.

## KSK / ZSK split, KSK operable, ZSK inoperable

Since the old ZSK is inoperable, it cannot be used to create new
RRSIGs. Therefore the zone cannot be changed and only the Pre-Publication
method can be used. See {{RFC7583}} section 2.1.

Section 3.2.1 of {{RFC7583}} documents the timeline for this
method.

The following diagram shows the timeline of the restoration.
Time increases along the horizontal scale from left to right and the vertical
lines indicate events in the process. Significant times and time intervals
are marked.

                  |1|      |2|   |3|      |4|
                   |        |     |        |
    Key N         - - ----------->|<-Iret->|
                   |        |     |        |
    Key N+1        |<-Ipub->|<--->|<----- - -
                   |        |     |        |
    Key N                                 Trem
    Key N+1        Tpub    Trdy  Tact

                      ---- Time ---->

Event 1: The new ZSK is added to the DNSKEY RRset at its publication time
(Tpub).

The inoperable ZSK and all RRSIGs it created MUST remain in the zone.

The SOA record of the zone SHOULD NOT be changed at this point in time,
because it cannot be re-signed with the inoperable key. Any secondary
name servers relying on IXFR/AXFR need to be manually forced to load
the new version of the zone.

The new ZSK must be published long enough to guarantee that any cached
DNSKEY RRset contains the new ZSK. This interval is the publication
interval (Ipub), given by

    Ipub = Dprp + TTLkey

Dprp is the propagation delay, the time it takes for changes to
propagate to all authoritative nameserver instances. TTLkey is the TTL
of the DNSKEY RRset.

Event 2: The new ZSK can be used when it becomes ready at Trdy.

    Trdy = Tpub + Ipub.

At this point the zone can be changed again.

Event 3: At some later time, the zone is signed with the new ZSK. At this
point RRSIGs from the inoperable ZSK can be removed. The inoperable ZSK
MUST be retained in the DNSKEY RRset.

Event 4: The inoperable ZSK can be removed after the retire interval (Iret).

    Iret = Dsgn + Dprp + TTLsig

Dsgn is the delay needed to ensure that all existing RRsets are signed
with the new ZSK, Dprp is the propagation delay and TTLsig is the
maximum TTL of all RRSIG records.

Theoretically the Double-Signature method could be used as well. In this case
records in the zone can only be changed after the retire interval, which is at
least as long as the publication interval of the Pre-Publication
method. The Double-Signature retire interval is given by:

    Iret = Dsgn + Dprp + max(TTLkey, TTLsig)

## KSK / ZSK split, KSK inoperable

Since the old KSK is inoperable, the DNSKEY RRset cannot be
changed. Therefore, only the Double-DS method can be used. See
{{RFC7583}} section 2.2.

If the ZSK is inoperable as well, it MUST NOT be restored yet.

Section 3.3.2 of {{RFC7583}} documents the timeline for this
method.

The following diagram shows the timeline of the restoration.
The diagram follows the convention described in Section 4.1.

                |1|      |2|       |3|  |4|      |5|
                 |        |         |    |        |
    Key N       - ---------------------->|<-Iret->|
                 |        |         |    |        |
    Key N+1      |<-Dreg->|<-IpubP->|<-->|<------- -
                 |        |         |    |        |
    Key N                                        Trem
    Key N+1     Tsbm     Tpub      Trdy Tact

                     ---- Time ---->

Event 1: A new DS record is added to the DS RRset in the parent
zone, this is the submission time, Tsbm.

Event 2: After the registration delay, Dreg, the DS record is
published in the parent zone. This is the publication time (Tpub).

    Tpub = Tsbm + Dreg.

The DS record must be published long enough to guarantee that any
cached DS RRset contains the new DS record. This is the parent
publication interval (IpubP).

    IpubP = DprpP + TTLds

DprpP is the propagation delay of the parent zone, i.e. the time it
takes for changes to propagate to all authoritative servers of the
parent zone. TTLds is the TTL of the DS RRset at the parent.

Event 3: The new KSK can be used when it becomes ready at Trdy.

    Trdy = Tpub + IpubP

Event 4: At this point, Tact, the new KSK is added to the DNSKEY
RRset and used to generate the DNSKEY RRSIG. The old, inoperable
KSK can be removed. The ZSK MUST remain in the DNSKEY RRset.

If the ZSK is inoperable, the SOA record of the zone SHOULD NOT be
changed at this point in time, because it cannot be re-signed with
the inoperable key. Any secondary name servers relying on IXFR/AXFR
need to be manually forced to load the new version of the zone.
The ZSK signing function can be restored using the procedure in
the previous section.

To ensure that no caches have DNSKEY RRset with the old KSK, the old
DS record MUST remain in the parent zone for the duration of the
retire interval (Iret), given by:

    Iret = DprpC + TTLkey

DprpC is the child propagation delay, the time it takes for changes to
propagate to all authoritative nameserver instances of the child
zone. TTLkey is the TTL of the DNSKEY RRset.

Event 5: The old DS record can be removed from the parent zone at Trem.

    Trem = Tact + Iret

## CSK inoperable

Since the old CSK is inoperable, the DNSKEY RRset cannot be
changed. Therefore, only the Double-DS method can be used. See
{{RFC7583}} section 2.2.

Section 3.3.2 of {{RFC7583}} documents the timeline for this method.

Since the CSK is also used to sign the zone, the timing of the
Double-DS method needs to be adjusted.

The inoperable CSK and all RRSIGs it created MUST remain in the zone.

The following diagram shows the timeline of the restoration.
The diagram follows the convention described in Section 4.1.

                |1|      |2|       |3|  |4|      |5|
                 |        |         |    |        |
    Key N       - ---------------------->|<-Iret->|
                 |        |         |    |        |
    Key N+1      |<-Dreg->|<-IpubP->|<-->|<------- -
                 |        |         |    |        |
    Key N                                        Trem
    Key N+1     Tsbm     Tpub      Trdy Tact

                     ---- Time ---->

Event 1: A new DS record is added to the DS RRset in the parent zone,
this is the submission time, Tsbm.

Event 2: After the registration delay, Dreg, the DS record is published
in the parent zone. This is the publication time (Tpub).

    Tpub = Tsbm + Dreg.

The DS record must be published long enough to guarantee that any
cached DS RRset contains the new DS record. This is the parent
publication interval (IpubP) given by

    IpubP = DprpP + TTLds

DprpP is the propagation delay of the parent zone, i.e. the time it
takes for changes to propagate to all authoritative servers of the
parent zone. TTLds is the TTL of the DS RRset at the parent.

Event 3: The new CSK can be used when it becomes ready at Trdy.

    Trdy = Tpub + IpubP

Event 4: At this point the new CSK is added to the DNSKEY RRset and
used to generate the DNSKEY RRSIG.

The old, inoperable CSK MUST remain in the DNSKEY RRset. The RRSIGs
generated by the inoperable CSK MUST remain in the zone.

The SOA record of the zone SHOULD NOT be changed at this point in time,
because it cannot be re-signed with the inoperable key. Any secondary
name servers relying on IXFR/AXFR need to be manually forced to load
the new version of the zone.

To ensure that no caches have DNSKEY RRset with the old CSK, the old
DS record MUST remain in the parent zone for the duration of the
retire interval (Iret), given by:

    Iret = Dsgn + DprpC + max(TTLkey, TTLsig)

Dsgn is the delay needed to ensure that all existing RRsets are signed
with the new CSK. DprpC is the child propagation delay, the time it
takes for changes to propagate to all authoritative nameserver
instances of the child zone. TTLkey is the TTL of the DNSKEY RRset and
TTLsig is the maximum TTL of all RRSIG records.

Event 5: The old DS record can be removed from the parent zone at Trem.

    Trem = Tact + Iret

At the same time the old, inoperable CSK and all its signatures can be
removed as well.

# Security Considerations

All security considerations of {{RFC9364}} apply to this document.

# IANA Considerations

This document has no IANA actions.

--- back

# Acknowledgments
{:numbered="false"}

The document draws heavily from the work in {{RFC7583}} and we thank
the authors for their work:

* Stephen Morris
* Johan Ihren
* John Dickinson
* W. (Matthijs) Mekking

Additionally, we thank the following people for contributing ideas and feedback:

* Libor Peltan
* Peter Thomassen
* Anand Buddhdev
* Wes Hardaker
