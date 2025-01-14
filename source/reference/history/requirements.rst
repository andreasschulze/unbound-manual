Requirements for Recursive Caching Resolver
===========================================

1. Introduction
---------------

This is the requirements document for a DNS name server and aims to
document the goals and non-goals of the project.  The DNS (the Domain
Name System) is a global, replicated database that uses a hierarchical
structure for queries.

Data in the DNS is stored in Resource Record sets (RR sets), and has a
time to live (TTL).  During this time the data can be cached.  It is
thus useful to cache data to speed up future lookups.  A server that
looks up data in the DNS for clients and caches previous answers to
speed up processing is called a caching, recursive nameserver.  

This project aims to develop such a nameserver in modular components, so
that also DNSSEC (secure DNS) validation and stub-resolvers (that do not
run as a server, but a linked into an application) are easily possible.

The main components are the Validator that validates the security
fingerprints on data sets, the Iterator that sends queries to the
hierarchical DNS servers that own the data and the Cache that stores
data from previous queries.  The networking and query management code
then interface with the modules to perform the necessary processing.

In Section 2 the origins of the Unbound project are documented. Section
3 lists the goals, while Section 4 lists the explicit non-goals of the
project. Section 5 discusses choices made during development.

2. History
----------

The unbound resolver project started by Bill Manning, David Blacka, and
Matt Larson (from the University of California and from Verisign), that
created a Java based prototype resolver called Unbound.  The basic
design decisions of clean modules was executed.

The Java prototype worked very well, with contributions from Geoff
Sisson and Roy Arends from Nominet.  Around 2006 the idea came to create
a full-fledged C implementation ready for deployed use.  NLnet Labs
volunteered to write this implementation.

3. Goals
--------

- A validating recursive DNS resolver
- Code diversity in the DNS resolver monoculture
- Drop-in replacement for BIND apart from config
- DNSSEC support
- Fully RFC compliant
- High performance

  - Even with validation

- Used as

  - Stub resolver
  - Full caching name server
  - Resolver library

- Elegant design of validator, resolver, cache modules

  - Provide the ability to pick and choose modules

-  Robust
-  In C, open source: The BSD license
-  Highly portable, targets include modern Unix systems, such as \*BSD, Solaris, linux, and maybe also the windows platform
-  Smallest as possible component that does the job
-  Stub-zones can be configured (local data or AS112 zones)

4. Non-Goals
------------

- An authoritative name server
- Too many Features

5. Choices
----------

- rfc2181 discourages duplicates RRs in RRsets. unbound does not create
  duplicates, but when presented with duplicates on the wire from the
  authoritative servers, does not perform duplicate removal.
  It does do some rrsig duplicate removal, in the msgparser, for dnssec qtype
  rrsig and any, because of special rrsig processing in the msgparser.
- The harden-glue feature, when yes all out of zone glue is deleted, when
  no out of zone glue is used for further resolving, is more complicated 
  than that, see below.

  Main points:

    - rfc2182 trust handling is used
    - data is let through only in very specific cases
    - spoofability remains possible

  Not all glue is let through (despite the name of the option). Only glue 
  which is present in a delegation, of type A and AAAA, where the name is
  present in the NS record in the authority section is let through.
  The glue that is let through is stored in the cache (marked as 'from the
  additional section'). And will then be used for sending queries to. It
  will not be present in the reply to the client (if RD is off).
  A direct query for that name will attempt to get a msg into the message
  cache. Since A and AAAA queries are not synthesized by the unbound cache,
  this query will be (eventually) sent to the authoritative server and its
  answer will be put in the cache, marked as 'from the answer section' and
  thus remove the 'from the additional section' data, and this record is 
  returned to the client.

  The message has a TTL smaller or equal to the TTL of the answer RR.
  If the cache memory is low; the answer RR may be dropped, and a glue
  RR may be inserted, within the message TTL time, and thus return the
  spoofed glue to a client. When the message expires, it is refetched and
  the cached RR is updated with the correct content.
  The server can be spoofed by getting it to visit a especially prepared 
  domain. This domain then inserts an address for another authoritative 
  server into the cache, when visiting that other domain, this address may
  then be used to send queries to. And fake answers may be returned.
  If the other domain is signed by DNSSEC, the fakes will be detected.

  In summary, the harden glue feature presents a security risk if
  disabled. Disabling the feature leads to possible better performance
  as more glue is present for the recursive service to use. The feature
  is implemented so as to minimise the security risk, while trying to 
  keep this performance gain.
- The method by which dnssec-lameness is detected is not secure. DNSSEC lame
  is when a server has the zone in question, but lacks dnssec data, such as
  signatures. The method to detect dnssec lameness looks at nonvalidated
  data from the parent of a zone. This can be used, by spoofing the parent,
  to create a false sense of dnssec-lameness in the child, or a false sense
  or dnssec-non-lameness in the child. The first results in the server marked
  lame, and not used for 900 seconds, and the second will result in a
  validator failure (SERVFAIL again), when the query is validated later on.

  Concluding, a spoof of the parent delegation can be used for many cases
  of denial of service. I.e. a completely different NS set could be returned,
  or the information withheld. All of these alterations can be caught by
  the validator if the parent is signed, and result in 900 seconds bogus.
  The dnssec-lameness detection is used to detect operator failures, 
  before the validator will properly verify the messages.

  Also for zones for which no chain of trust exists, but a DS is given by the
  parent, dnssec-lameness detection enables. This delivers dnssec to our
  clients when possible (for client validators).

  The following issue needs to be resolved:

    A server that serves both a parent and child zone, where
    parent is signed, but child is not. The server must not be marked
    lame for the parent zone, because the child answer is not signed.

  Instead of a false positive, we want false negatives; failure to 
  detect dnssec-lameness is less of a problem than marking honest 
  servers lame. dnssec-lameness is a config error and deserves the trouble.
  So, only messages that identify the zone are used to mark the zone
  lame. The zone is identified by SOA or NS RRsets in the answer/auth.
  That includes almost all negative responses and also A, AAAA qtypes.
  That would be most responses from servers.
  For referrals, delegations that add a single label can be checked to be
  from their zone, this covers most delegation-centric zones.

  So possibly, for complicated setups, with multiple (parent-child) zones
  on a server, dnssec-lameness detection does not work - no dnssec-lameness
  is detected. Instead the zone that is dnssec-lame becomes bogus.
- authority features.

  This is a recursive server, and authority features are out of scope.
  However, some authority features are expected in a recursor. Things like
  localhost, reverse lookup for 127.0.0.1, or blocking AS112 traffic.
  Also redirection of domain names with fixed data is needed by service
  providers. Limited support is added specifically to address this.

  Adding full authority support, requires much more code, and more complex
  maintenance.

  The limited support allows adding some static data (for localhost and so),
  and to respond with a fixed rcode (NXDOMAIN) for domains (such as AS112).

  You can put authority data on a separate server, and set the server in
  unbound.conf as stub for those zones, this allows clients to access data
  from the server without making unbound authoritative for the zones.
- The access control denies queries before any other processing.
  This denies queries that are not authoritative, or version.bind, or any.
  And thus prevents cache-snooping (denied hosts cannot make non-recursive
  queries and get answers from the cache).
- If a client makes a query without RD bit, in the case of a returned
  message from cache which is:

  .. code-block:: bash

    answer section: empty
    auth section: NS record present, no SOA record, no DS record,
                  maybe NSEC or NSEC3 records present.
    additional: A records or other relevant records.

  A SOA record would indicate that this was a NODATA answer.
  A DS records would indicate a referral.
  Absence of NS record would indicate a NODATA answer as well.

  Then the receiver does not know whether this was a referral
  with attempt at no-DS proof) or a nodata answer with attempt
  at no-data proof. It could be determined by attempting to prove
  either condition; and looking if only one is valid, but both
  proofs could be valid, or neither could be valid, which creates
  doubt. This case is validated by unbound as a 'referral' which
  ascertains that RRSIGs are OK (and not omitted), but does not
  check NSEC/NSEC3.
- Case preservation.

  Unbound preserves the casing received from authority servers as best 
  as possible. It compresses without case, so case can get lost there.
  The casing from the query name is used in preference to the casing
  of the authority server. This is the same as BIND. RFC4343 allows either
  behaviour.
- Denial of service protection

  If many queries are made, and they are made to names for which the
  authority servers do not respond, then the requestlist for unbound
  fills up fast.  This results in denial of service for new queries.
  To combat this the first 50% of the requestlist can run to completion.
  The last 50% of the requestlist get (200 msec) at least and are replaced
  by newer queries when older (LIFO).
  When a new query comes in, and a place in the first 50% is available, this
  is preferred.  Otherwise, it can replace older queries out of the last 50%.
  Thus, even long queries get a 50% chance to be resolved.  And many 'short'
  one or two round-trip resolves can be done in the last 50% of the list.
  The timeout can be configured.
- EDNS fallback.

  Is done according to the EDNS RFC (and update draft-00).
  Unbound assumes EDNS 0 support for the first query.  Then it can detect
  support (if the servers replies) or non-support (on a NOTIMPL or FORMERR).
  Some middleboxes drop EDNS 0 queries, mainly when forwarding, not when
  routing packets.  To detect this, when timeouts keep happening, as the
  timeout approached 5-10 seconds, and EDNS status has not been detected yet,
  a single probe query is sent.  This probe has a sub-second timeout, and
  if the server responds (quickly) without EDNS, this is cached for 15 min.
  This works very well when detecting an address that you use much - like
  a forwarder address - which is where the middleboxes need to be detected.
  Otherwise, it results in a 5 second wait time before EDNS timeout is
  detected, which is slow but it works at least.
  It minimizes the chances of a dropped query making a (DNSSEC) EDNS server
  falsely EDNS-nonsupporting, and thus DNSSEC-bogus, works well with
  middleboxes, and can detect the occasional authority that drops EDNS.
  For some boxes it is necessary to probe for every failing query, a
  reassurance that the DNS server does EDNS does not mean that path can
  take large DNS answers.
- 0x20 backoff.

  The draft describes to back off to the next server, and go through all
  servers several times.  Unbound goes on get the full list of nameserver
  addresses, and then makes 3 * number of addresses queries.
  They are sent to a random server, but no one address more than 4 times.
  It succeeds if one has 0x20 intact, or else all are equal.
  Otherwise, servfail is returned to the client.
- NXDOMAIN and SOA serial numbers.

  Unbound keeps TTL values for message formats, and thus rcodes, such
  as NXDOMAIN.  Also it keeps the latest rrsets in the rrset cache.
  So it will faithfully negative cache for the exact TTL as originally
  specified for an NXDOMAIN message, but send a newer SOA record if
  this has been found in the mean time.  In point, this could lead to a
  negative cached NXDOMAIN reply with a SOA RR where the serial number
  indicates a zone version where this domain is not any longer NXDOMAIN.
  These situations become consistent once the original TTL expires.
  If the domain is DNSSEC signed, by the way, then NSEC records are
  updated more carefully.  If one of the NSEC records in an NXDOMAIN is
  updated from another query, the NXDOMAIN is dropped from the cache,
  and queried for again, so that its proof can be checked again.
- SOA records in negative cached answers for DS queries.

  The current unbound code uses a negative cache for queries for type DS.
  This speeds up building chains of trust, and uses NSEC and NSEC3
  (optout) information to speed up lookups.  When used internally,
  the bare NSEC(3) information is sufficient, probably picked up from
  a referral.  When answering to clients, a SOA record is needed for
  the correct message format, a SOA record is picked from the cache
  (and may not actually match the serial number of the SOA for which the
  NSEC and NSEC3 records were obtained) if available otherwise network
  queries are performed to get the data.
- Parent and child with different nameserver information.

  A misconfiguration that sometimes happens is where the parent and child
  have different NS, glue information.  The child is authoritative, and
  unbound will not trust information from the parent nameservers as the
  final answer.  To help lookups, unbound will however use the parent-side
  version of the glue as a last resort lookup.  This resolves lookups for
  those misconfigured domains where the servers reported by the parent
  are the only ones working, and servers reported by the child do not.

