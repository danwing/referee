---
title: "A Referee to Authenticate Home Servers"
abbrev: "Referee"
category: std

docname: draft-xyzzy-referee-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: WG Working Group
keyword:
 - home servers
 - tls
 - tofu

venue:
#  group: IOTOPS
#  type: Working Group
#  mail: iotops@ietf.org
#  arch: https://mailarchive.ietf.org/arch/browse/iotops/
  github: "danwing/referee"
  latest: "https://danwing.github.io/referee/draft-xyzzy-referee.html"

author:
 -
    ins: D. Wing
    name: Dan Wing
    organization: Cloud Software Group Holdings, Inc.
    abbrev: Cloud Software Group
    email: danwing@gmail.com


normative:

informative:


--- abstract

Obtaining and maintaining PKI certificates for devices in a home
network is difficult for both technical and human factors reasons. This
document describes an alternative approach to securely identify and
authenticate home servers using a referee system. The referee allows
bootstrapping a network of devices by trusting only one system in
the home network.

--- middle

# Introduction

Most existing TLS communications require the server obtaining a
certificate signed by a Certification Authority trusted by the client.
Within a home network this is fraught with complications of both
human factors and technical nature.

This document describes a Referee System, where a Referee is entrusted
(once) to help clients identify and authenticate (many) servers within
the home network.  The Referee System purposefully avoids using the
public key infrastructure (PKI).

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Requirements Evaluation

Using requirements from {{?I-D.rbw-home-servers}}, the proposal in this
document has the following summarized characteristics:

|    Solution                 | Reduce CA             | Eliminate CA     | Existing CA Support | Existing Client Support | Revoke Auth |
|----------------------------:|:---------------------:|:----------------:|:-------------------:|:-----------------------:|:-----------:|
| Referee                     | Yes                   |  Yes             |   N/A               |   No                    |   Yes       |
{: #table1 title="Summary of Referee Against Requirements"}


# Operation Overview

## Referee

The Referee function is implemented within any always-on device
within the home (e.g., router, smart home hub, NAS).  The Referee
contains a database of local hostnames and their Referee public
key fingerprints.

Clients authenticate to the Referee using the TLS 'referee'
extension.  Thus, clients wishing to participate in a 'referee'
system need to be updated to support the TLS 'referee' extension.

The Referee may need to revoke authorization for a server (e.g.,
replacement of a printer). This is accomodated by the client procedures
which will query the Referee if the client's cached public key does
not match the server's public key. This revocation happens immediately
after the Referee updates the public key associated with that server
name.


## Servers

A server supporting this specification is expected to be a printer
(using IPPS or HTTPS), NAS, IoT device, router (especially its
HTTPS-based management console or its ssh server), or similar.

Each in-home device supporting Referee has a fixed public key, which
persits for the lifetime of the device.  During installation of the
device to a Referee network, the device's hostname and public key
fingerprint are stored into the Referee Server.  Several options
exist for this step, detailed in {{bootstrapping}}.

A server supporting this specification also supports the new
TLS ClientHello extension "referee". Upon receiving such a ClientHello,
the server responds with its Referee public key {{!RFC7250}} rather
than a PKI certificate.

> Note: if a factory reset changes the device's Referee public key,
  the new key will need to be re-enrolled with the Referee and the old
  key removed.

## Clients

A client supporting this specification is first configured with the
DNS name of its Referee server.  It authenticates to the Referee server
using one of the bootstrapping mechanisms (see {{bootstrapping}}). This
step occurs only once for each home network the client joins, as each
home network is responsible for being a Referee for its own devices.

When connecting to a server with an .internal domain, a client supporting
this specification includes the TLS extension "referee" in its TLS
ClientHello.  This causes the server to respond with its Referee raw
public key {{!RFC7250}} rather than a PKI certificate (which the client
might or might not trust, based on how that PKI certificate was signed).

For the client, there are two situations that may occur:  it has
not previously cached the association of hostname to public key or it
has cached that information.

* If not previously cached, the client queries that network's Referee with the
DNS name of the server (e.g., printer.internal).  The Referee responds
with the public key fingerprint of that server.  The client checks if
the public key fingerprint (from the Referee) matches the public key
of the server (from the TLS handshake). If they match, communication
with the server continues.  The server MAY also cache the server name
and public key.  If they do not match, the client aborts this
coummunication session; further actions by the client are an
implementation detail.

* If previously cached, the client determines if the cached public key matches the public
key just obtained.  If they match, communication continues.  If they
do not match, the client aborts the communication; futher actions
by the client are an implementation detail.



# Bootstrapping Server Public Keys to the Referee {#bootstrapping}

## QR Scan

## manual entry

## TOFU

## Outstanding questions and issues

* Currently the text suggests clients should fallback to PKI if
  Referee validation fails.  But is such fallback harmful or is it
  worthwhile?

* If client has multiple Referees configured, how does client know
  which Referee to use for the network it has joined?  If SSID,
  wither Ethernet?  Maybe during TLS handshake the server could
  indicate the server's Referee??

* printer.internal or printer.local are handy names.  Are they
  suitable for this system, or do we need site-specific names
  containing a unique identifier like a UUID, e.g.,
  printer.2180be87-3e00-4c7f-a366-5b57fce4cbf7.internal?  If we need
  unique name, can we CNAME from printer.internal to the unique name?




## Operational Notes {#operational-notes}

The Referee has to always be available.  The client cache helps reduce
load on the Referee but new clients (e.g., new devices, guest users, restored
devices) and client cache invalidation will always cause some traffic to
the Referee.

When the Referee is unavailable, clients behavior devolves to what
we have today:  servers will need to obtain a real PKI certificate
signed by a Certification Authority already trusted by the clients, or
else clients will need to manually trust individual certificates.


# Security Considerations

See {{operational-notes}} describing client behavior when the Referee
is unavailable.


# IANA Considerations

Register TLS ClientHello extension "referee".


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
