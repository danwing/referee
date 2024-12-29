---
title: "A Referee to Authenticate Servers in Local Domains"
abbrev: "Local Domain Referee"
category: std

docname: draft-wing-settle-referee-latest
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
  group: SETTLE
  type: ""
  mail: settle@ietf.org
  arch: https://mailarchive.ietf.org/arch/browse/settle/
  github: "danwing/referee"
  latest: "https://danwing.github.io/referee/draft-xyzzy-referee.html"

author:
 -
    ins: D. Wing
    name: Dan Wing
    organization: Citrix
    abbrev: Citrix
    email: danwing@gmail.com
    country: United States of America


normative:

informative:


--- abstract

Obtaining and maintaining PKI certificates for devices in a home
network is difficult for both technical and human factors reasons. This
document describes an alternative approach to securely identify and
authenticate home servers using a Referee system. The referee allows
bootstrapping a network of devices by trusting only one system in
a local domain -- the Referee.

--- middle

# Introduction

Most existing TLS communications require the server obtaining a
certificate signed by a Certification Authority trusted by the client.
Within a home network this is fraught with complications of both
human factors and technical natures.

This document describes a Referee system to authorize the legitimate
servers on a local domain.  A server, called a Referee, is entrusted
to help clients identify and authenticate servers within
the local domain.  The Referee system purposefully avoids Public Key
Infrastructure using X.509 {{?PKIX=RFC5280}}, instead using an
"allowlist" of public keys.

When clients connect to a server on the local domain and encounter
a self-signed certificate that might otherwise cause an authentication
failure (typically, a warning to the user), the client can query the
local domain's pre-authorized Referee system to learn if that server
has been enrolled with the Referee.  If so, the connection can continue --
much as a certificate signed by a trusted Certification Authority would
continue.


# Requirements Evaluation

Using requirements from {{?I-D.rbw-home-servers}}, the proposal in this
document has the following summarized characteristics:

|    Solution                 | Reduce CA             | Eliminate CA     | Existing CA Support | Existing Client Support | Revoke Auth |
|----------------------------:|:---------------------:|:----------------:|:-------------------:|:-----------------------:|:-----------:|
| Referee                     | Yes                   |  Yes             |   N/A               |   No                    |   Yes       |
{: #table1 title="Summary of Referee Against Requirements"}


# Operation

## Referee

The Referee function is implemented within any always-on device
within the home (e.g., router, smart home hub, NAS).  The Referee
contains a database of local hostnames and their Referee public
key fingerprints.

The Referee on a local network is named "referee.internal".

Clients authenticate to the Referee and use HTTP GET to fetch the
named public key fingerprint from the Referee server.  For example to
get public key fingerprint of a server named printer.internal,

~~~~
  GET /.well-known/referee/sha256/printer.internal HTTP/1.1
~~~~

The public key fingerprint is SHA-256 of the server's public key
returned as an octet-stream.

## Servers

A server supporting this specification is expected to be a printer
(using IPPS or HTTPS), file server (e.g., NAS or laptop), IoT device,
router (especially its HTTPS-based management console or its ssh
server), or similar.

Each in-home device supporting Referee has a fixed public key, which
persists for the lifetime of the device.  During installation of the
device to a Referee network, the device's hostname and public key
fingerprint are stored into the Referee Server.  Several options
exist for this step, detailed in {{bootstrapping}}.

If a server's Referee public key changes (e.g., factory reset,
public key algorithm, key length) the new key needs
to be enrolled with the Referee and the old key removed. Clients
will notice the mismatch and will query the Referee.  This
functionality might be automated; see {{key-lifetime}}.

## Clients

A client supporting this specification is first configured with the
DNS name of its Referee server.  It authenticates to the Referee server
using one of the bootstrapping mechanisms (see {{bootstrapping}}). This
step occurs only once for each home network the client joins, as each
home network is responsible for being a Referee for its own devices.

On a connection to a server on the local domain (see {{local}}) there
are two situations that may occur: the client has not previously cached the
association of hostname to public key or it has cached that
information.

* If not previously cached, the client queries that network's Referee
with the DNS name of the server (e.g., printer.internal).  The Referee
responds with the public key fingerprint of that server.  The client
checks if the public key fingerprint (from the Referee) matches the
public key of the server (from the TLS handshake). If they match,
communication with the server continues and the server name and its
public key MAY be cached by the client.  If they do not match, the
client aborts this communication session; further actions by the
client are an implementation detail.

* If previously cached, the client determines if the cached public key
matches the public key obtained from the TLS handshake with the
server.  If they match, communication continues.  If they do not
match, the queries the Referee to learn if a new key fingerprint is
available for that server.  If a different fingerprint is returned by
the Referee, the client verifies it matches the public key from the
TLS handshake.  If they match, the client replaces the information in
its cache.

Internally a client might form a unique identity for a local domain
server as hostname (e.g., printer.local) combined with the identity
of the Referee, such as the Referee's public key fingerprint.  In this
way, when the client is on a different network (which will have a
different Referee), a server name collision (e.g., local.printer) will
result in a unique internal identity for that server -- keeping all
the server-specific data separate (e.g., web forms, passwords, local
storage, etc.).

## Revoking Authorization

When the administrator revokes authorization for a server (e.g., replacement
of a printer), the administrator removes the old public key from the Referee
and installs the new key in the Referee.

When this replacement occurs, the clients that have not already cached
the server's public key will simply query the Referee, which has the server's
new public key.  The clients that have cached the server's previous public
key will notice the mismatch, pause their communication with the server, and
validate with the Referee that the new key is legitimate, and continue their
communication with the server.

Thus, revoking authentication has immediate effect because the clients
immediately validate a mismatch with the Referee.

# Bootstrapping the Referee {#bootstrapping}

## Clients to Referee

The clients have to be configured to trust their Referee. This is
a one time activity, for each home network the client joins.


Until service discovery is defined for a Referee system, the client
has to be configured to trust the Referee server's public key
fingerprint.  This can be done manually or using TOFU, and is
implementation specific.

> for discussion: To reduce initial bootstrap for client, perhaps use SVCB for
  client to bootstrap its first Referee?  This effectively achieves
  un-authenticated encryption to devices on the local network which is
  better than unencrypted traffic to those same devices.  For an
  attacker to abuse this, it requires an attacker advertise itself
  as the Referee and to maintain its status as Referee.

> for discussion: see {{key-lifetime}} regarding Referee key lifetime.

## Servers to Referee

Server names and their associated public key fingerprints have to
be populated into the Referee.

### Short Code or Scan Code

Short code printed on the Referee-capable server which can be scanned
by a smartphone application by the home administrator which is
authorized to push new associations to the Referee.  Alternatively,
the same information could be manually typed in by the home
administrator to the Referee's management GUI or CLI.

### TOFU

A client device which leans the Referee does not have an existing
entry for a (new) name is authorized to 'push' the (new) name and
its public key fingerprint to the Referee.



# Identifying Servers as Local {#local}

This section defines the domain names and IP addresses considered
"local".

## Local Domain Names

The following domain name suffixes are considered "local":

* ".local" (from {{?mDNS=RFC6762}})
* ".home-arpa" (from {{?Homenet=RFC8375}})
* ".internal" (from {{?I-D.davies-internal-tld}})
* both ".localhost" and "localhost" (Section 6.3 of {{?RFC6761}})

## Local IP Addresses

Additionally, if any host resolves to a local IP address and
connection is made to that address, those are also considered
"local":

* 10/8, 172.16/12, and 192.168/16 (from {{?RFC1918}})
* 169.254/16 and fe80::/10 (from {{?RFC3927}} and {{?RFC4291}})
* fc00::/7 (from {{?RFC4193}})
* 127/8 and ::1/128 (from {{Section 3.2.1.3 of ?RFC1122}} and {{?RFC4291}})


# Operational Notes {#operational-notes}

The Referee has to always be available.  The client cache helps reduce
load on the Referee but new clients (e.g., new devices, guest users, restored
devices) and client cache invalidation will always cause some traffic to
the Referee.

When the Referee is unavailable, clients behavior devolves to what
we have today:  servers will need to obtain a real PKI certificate
signed by a Certification Authority already trusted by the clients, or
else clients will need to manually trust individual certificates.


# Security Considerations

TODO: expand security considerations.

See {{operational-notes}} describing client behavior when the Referee
is unavailable.




# IANA Considerations

Register new .well_known URI for "referee".

--- back



# Issues for Further Discussion

## PKI Fallback

Currently the text suggests clients should fallback to PKI if Referee
validation fails.  This means certificate warnings for self-signed
certificates.  Is such fallback harmful or is it worthwhile?

## Multiple Networks: Multiple Referees

If client has multiple Referees configured (due to visiting multiple
networks), it is anticipated the client would do service discovery to
find the network's referee and validate if the (discovered) referee is
an already-known referee -- akin to {{?DNR=RFC9463}}.

If the client only knows of one referee, this problem never occurs.

## Unique Names

Printer.internal or printer.local are handy names and can be used
with a Referee system.

Unfortunately existing browsers have state that is tied to names --
web forms, cookies, and passwords.  Thus, those existing systems need names that contain a
unique identifier like a UUID, e.g.,
printer.2180be87-3e00-4c7f-a366-5b57fce4cbf7.internal.  Or perhaps
embedding part/all of the public key into the name itself, for
example:

~~~~~
  printer.2180be87-3e00-4c7f-a366-5b57fce4cbf7.internal
  nas.103a40ee-c76f-46da-84a1-054b8f18ae33.internal
  router.fb5f73ed-275a-431e-aecf-436f0c54d69d.internal
~~~~~

The Referee system is ambilvalnt about the name -- the Referee's name
and each server's name need only be unique on the current local
domain. Name collisions that occur between local domains are handled
by the client querying the other network's trusted Referee to check
legitimacy.

~~~~~
  printer.ee80be87-3e00-4c7f-a366-5b57fce4c999.internal
  nas.ee80be87-3e00-4c7f-a366-5b57fce4c999.internal
  router.ee80be87-3e00-4c7f-a366-5b57fce4c999.internal
~~~~~

The Referee system allows keeping the unique name the same for the
lifetime of the device while allowing changing its public key.

## Key Lifetime (Rotating Public Key) {#key-lifetime}

For security hygiene, the public keys in a server and in the Referee
are occasionally changed. This section discusses how such changes are
handled by a Referee system.

### Server

If a server's public key changes the new key has to be installed
into the network's Referee.  To automate such changes, the server could connect to the
Referee and prove possession of its (old) private key (using TLS
client authentication or using application-layer mechanism such as
JSON Web Signature) and publish its new public key using an HTTP PUT.

> Note: such a PUT mechanism also means an attacker in possession of
the server's private key can change the legitimate server's public key
fingerprint in the Referee to now point at an attacker-controlled
system, denying access to the legitimate server. It is still better than
unencrypted connections, which is the case today.


### Referee

If the Referee's public key changes all the clients have to
re-authenticate the Referee's new public key.  This is uncool.

To allow changing the Referee's public key without client
re-authentication, the client and Referee could do session resumption
for its subsequent connections to the Referee (Section 2.2 of
{{?RFC8446}}).  When doing session resumption with the Referee, the
client should also retrieve and cache the Referee's current public key
fingerprint so if, in the future, the Referee cannot perform session
resumption the client can still authenticate the Referee.

With the above technique, the client will only have to (manually)
re-authenticate the Referee when the Referee cannot perform session
resumption.

## Incrementatal Adoption

The Referee system requires support of both the client (to ask the
Referee for mediation) and installation of a Referee -- which could be
in the home router, NAS, or other always-on device.  This section
explores how to bootstrap Referee system even when the server does
not (yet) support Referee.


### Server Does Not Support Referee

In the case a server does not support Referee, it will not
register itself with the network's Referee.

The Referee could be told by a network administrator to connect to
such a server, extract public key, and install that public key and
name into the Referee. When a Referee-capable client connects to that
server and receives a certificate signed by a Certification Authority
the client does not trust, and that server is on a local domain, the
client can query the Referee for the server's public key.  The Referee
responds with the server's public key which the client compares to the
server's public key in the TLS handshake. If they match, the client
has finished authentication with the server.  If they don't match, the
client considers the TLS handshake to have failed and displays an
error.


# Acknowledgments
{:numbered="false"}

Thanks to Sridharan Rajagopalan for reviews and feedback.

