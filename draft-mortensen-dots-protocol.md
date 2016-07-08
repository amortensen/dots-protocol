---
title: DDoS Open Threat Signaling Protocol
docname: draft-mortensen-dots-protocol-00
date: 2015-10-19

area: Security
wg: DOTS
kw: Internet-Draft
cat: std

coding: us-ascii
pi:
  toc: yes
  sortrefs: no
  symrefs: yes

author:
      -
        ins: A. Mortensen
        name: Andrew Mortensen
        org: Arbor Networks, Inc.
        street: 2727 S. State St
        city: Ann Arbor, MI
        code: 48104
        country: United States
        email: amortensen@arbor.net
      -
        ins: N. Teague
        name: Nik Teague
        org: Verisign, Inc.
        email: nteague@verisign.com

normative:
  RFC0768:
  RFC0791:
  RFC0793:
  RFC2119:
  RFC2460:
  RFC5246:
  RFC5952:
  RFC6555:
  RFC7159:
  RFC7230:
  RFC7231:
  RFC7540:
  I-D.ietf-dots-architecture:
  I-D.ietf-dots-requirements:
  REST:
    target: http://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf
    title: Architectural Styles and the Design of Network-based Software Architectures
    author:
      ins: R. Fielding
      name: Roy Thomas Fielding
      org: University of California, Irvine
    date: 2000
    seriesinfo:
      "Ph.D.": "Dissertation, University of California, Irvine"
    format:
      PDF: http://www.ics.uci.edu/~fielding/pubs/dissertation/fielding_dissertation.pdf

informative:
  RFC1518:
  RFC1519:
  RFC2373:
  RFC4271:
  RFC5575:
  I-D.tsvwg-quic-protocol:
  WISR:
    target: https://www.arbornetworks.com/images/documents/WISR2016_EN_Web.pdf
    title: Worldwide Infrastructure Security Report
    author:
      org: Arbor Networks, Inc.
    date: 2016
    format:
      PDF: https://www.arbornetworks.com/images/documents/WISR2016_EN_Web.pdf

--- abstract

This document describes Distributed-Denial-of-Service (DDoS) Open Threat
Signaling (DOTS), a signaling protocol for requesting and managing mitigation of
DDoS attacks.

DOTS mitigation requests over the signal channel permit domains to signal the
need for help fending off DDoS attacks, setting the scope and duration of the
requested mitigation.  Elements called DOTS servers field the signals for help,
and enable defensive countermeasures to defend against the attack reported by
the clients, reporting the status of the delegated defense to the requesting
clients. DOTS clients additionally may use the data channel to manage filters
and black- and white-lists to restrict or allow traffic to the clients' domains
arbitrarily.

The DOTS signal channel operates over UDP and if necessary TCP. The DOTS data
channel operates over HTTPS.


--- middle

Introduction
============

Distributed-Denial-of-Service attack scale and frequency continues to increase
year over year, and the trend shows no signs of abating [WISR]. In response to
the DDoS attack trends, service providers and vendors have developed various
approaches to sharing or delegating responsibility for defense, among them ad
hoc service relationships, filtering through peering relationships [CARISFLOW],
and proprietary solutions ([ARBOR], [VERISIGN]). Such hybrid approaches to DDoS
defense have proven effective [CITE], but the heterogeneous methods employed to
coordinate DDoS defenses across domain boundaries have necessarily limited their
scope and effectiveness, as the mechanisms in one domain have no traction in
another.

The DDoS Open Threat Signaling (DOTS) protocol provides a common mechanism to
achieve the coordinated attack response previously restricted to custom or
proprietary solutions. To meet the needs of network operators facing down modern
DDoS attacks, DOTS itself is a hybrid protocol, consisting of a signal channel
and a data channel. DOTS uses the signal channel, a lightweight and robust
communication layer, to signal the need for mitigation regardless of network
conditions, and uses the data channel, an HTTPS-based communication layer with
RESTful [REST] semantics, as vehicle for provisioning, configuration, and
filter management.

DOTS is not intended as a replacement for such protocols as BGP Flow
Specification [RFC5575] or as a general purpose mitigation application
programming interface (API), but rather as an advisory protocol enabling attack
response coordination between willing domains. Any DOTS-enabled device or
service is capable of triggering a request for help and shaping the scope and
nature of that help, with the details of the actual mitigation left to the
discretion of the operators of the attack mitigators. DOTS thereby permits all
participating parties to manage their own attack defenses in the manner most
appropriate for their own domains.


Terminology
-----------

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be
interpreted as described in {{RFC2119}}.

Terms used to define entity relationships, transmitted data, and methods of
communication are drawn from the terminology defined in
[I-D.ietf-dots-requirements].


Architecture
============

The architecture in which the DOTS protocol operates is assumed to be derived
from the architectural components and concepts described in
[I-D.ietf-dots-architecture].


DOTS Agents
-----------

All protocol communication is between a DOTS client and a DOTS server. The
logical agent termed a DOTS gateway is in practice a DOTS server placed
back-to-back with a DOTS client. As discussed in [I-D.ietf-dots-architecture],
any interface enabling the back-to-back DOTS server and client to act as a DOTS
gateway is implementation-specific. This protocol is therefore concerned only
with managing one or more bilateral relationships between DOTS clients and the
DOTS servers, a signaling mode known as Direct Signaling in the DOTS
architecture. This is shown in {{fig-proto-dir-sig}} below:

~~~~~
    +-----------+  signal channel  +-----------+
    |           |<---------------->|           |
    |DOTS client|                  |DOTS server|
    |           |<================>|           |
    +-----------+   data channel   +-----------+
~~~~~
{: #fig-proto-dir-sig title="DOTS protocol direct signaling"}

The DOTS architecture anticipates many-to-one and one-to-many deployments, in
which multiple DOTS clients maintain distinct signaling sessions with a single
DOTS server or a single DOTS client maintains distinct signaling sessions with
multiple DOTS servers, as shown below in {{fig-proto-mn-dir-sig}}:

~~~~~
    +----+      +----+      +----+
    | c1 |      | Sa |------| c2 |
    +----+      +----+      +----+
          \                /
           \              /
            \   +----+   /
             +--| Sb |--+
                +----+

    DOTS        DOTS        DOTS
    client 1    servers     client 2
~~~~~
{: #fig-proto-mn-dir-sig title="DOTS protocol direct signaling"}

DOTS server Sb has signaling sessions with DOTS clients c1 and c2. DOTS client
c2 has signaling sessions with DOTS servers Sa and Sb. Except where explicitly
defined in this protocol, all mechanisms to maintain multiple signaling sessions
are left to the implementation. Exceptions arising from detected mitigation
request conflicts are described in \{\{mit-conflict-detection\}\} below.


Protocol Overview
=================

The DOTS protocol consists of two channels, a signal channel and a data channel.
The signal channel is the minimal secure communication layer a DOTS client uses
to request mitigation for resources under the administrative control of the DOTS
client; the administrative control may be delegated. The data channel acts as a
management plane for DOTS, permitting DOTS client operators a limited ability to
adjust configuration and filtering for their mitigation requests.

Signal Channel
==============

The signal channel characteristics in [I-D.ietf-dots-requirements] describe a
secure, low-overhead protocol capable of operating in DDoS attack conditions. To
function in attack conditions, the signal channel must expect and absorb a
certain amount of signal lossiness and latency, with the limits established by
operators of deployed DOTS agents.

The purpose of the signaling channel is to convey DDoS mitigation request and
status information between participating agents (client and server or gateway).
Conditions during a DDoS attack are invariably hostile for connection oriented
protocols traversing affected paths.  Mechanisms such as Happy Eyeballs
[RFC6555] may be used to select a transport suitable for a given time and
prevailing network conditions.  For the purpose of this draft, however, a
default signaling transport based upon UDP [RFC5405] will be used.  UDP's
connectionless quality lends itself to being able to sustain loose
communications during an event which may heavily congest certain network paths
towards the DOTS signal originating network.  Key tenets of DOTS protocol design
is low communication overhead and efficient message packing to increase the
chances of successful transmission and receipt.  Desirable side-effects of
efficient packing are the removal of the possibility of fragmentation in
addition to a message size that is friendly towards encapsulation (e.g via GRE
[RFC2784] or MPLS [RFC3031]).  Large UDP packets may also be treated adversely
by middleboxes with restrictive policies or may fall foul of aggressive
filtering.

To assist in meeting the requirements for efficiency the signaling channel will
utilize Protocol Buffers [PROTOBUF] - "A language neutral, extensible mechanism
for serializing structured data".  Numbered fields in Protobufs allow for new
fields to be introduced at any time permitting extensibility within DOTS without
requiring versioning or refactoring.  The data channel may be used to provide a
mechanism by which schema updates or expansions may be communicated during
provisioning/session (re)establishment.

Data serialization alone does not provide for the requirements of peer mutual
authentication [SEC-001], message confidentiality [SEC-002], message replay
protection [SEC-003] or message integrity.  CurveCP [CCP] meets these
requirements and provides active and passive forward secrecy.  Key distribution
may be achieved via the data channel, via an online mechanism such as DANE
[RFC6698] or by out of band means.


Minimum Viable Information
--------------------------

DOTS is intended to be extensible and to evolve to meet the future needs in
communicating as yet unknown threats, however, it must be able to convey the
minimum amount of information required for an upstream mitigation platform to
successfully counter a DDoS attack.  A client may have limited visibility into
the full breadth of an attack and as such may not be well placed to provide
useful telemetry, in addition DDoS sources may or may not be spoofed and number
in the millions.  This provides challenges for the quality and usefulness of
telemetry and mitigation/countermeasure stipulations and as such this type of
information if conveyed can only be considered advisory.  In these instances the
minimum viable information required for the majority of mitigations to be
activated is that which pertains to the resource being targeted by the attack
(host, prefix, protocol, port, URI etc. [OP-006]).  [I-D.ietf-dots-requirements]
also identifies a mitigation lifetime period [OP-005] and mitigation efficacy
metric [OP-007].  The former may be considered for inclusion in the minimum
viable information set, however, the latter may only be relevant in updates.  An
explicit mitigation request/terminate flag is also required - a mitigation MUST
be explicitly requested by an operator.  Finally, each message should include a
message id or sequence number field as well as a field for the last received
message id or sequence number.  These may then be compared by the endpoints to
assist in tracking state and/or identifying loss.

Messages
--------

DOTS signaling using Protobufs may reduce the number of discrete messages to
just a single message superset per direction with function defined by the chosen
fields contained within the message.  We would, therefore, define a single
schema each for the client and server sides containing all relevant fields.
Tags 1 through 15 may benefit from only requiring a single byte to encode (vs
two for tags 16 through 2047) and these should be used for frequently occurring
message elements.


### Client Schema

The entire client schema is detailed in {{fig-client-schema}}.  It is not
expected that client messages will require all fields to be used simultaneously
but instead a subset to convey a given signal type.  The only fields which may
be common to all signals are seqno and lastsvrseqno which may be used to detect
loss or drop outs.

~~~~~
    message Client {
      // Client generated sequence number
      int64 seqno = 1;

      // Sequence number of last received server message
      int64 lastsvrseqno = 2;

      // Opaque client generated event identifier
      string eventid = 3;

      // Mitigation scope (target)
      string scope = 4;

      // Mitigation request lifetime in seconds
      int64 lifetime = 5;

      // Mitigation eficacy
      int32 eficacy = 6;

      // Request active mitigation list from server
      enum Active {
        FALSE = 0;
        TRUE = 1;
      }
      Active active = 13 [default = FALSE];

      // Heartbeat request (operator initiated)
      enum Ping {
        FALSE = 0;
        TRUE = 1;
      }
      Ping ping = 14 [default = FALSE];

      // Mitigation request
      enum Mitigate {
        FALSE = 0;
        TRUE = 1;
      }
      Mitigate mitigateReq = 15 [default = FALSE];

      extensions 100 to 199;
    }
~~~~~
{: #fig-client-schema title="Client Schema"}


### Server Schema

The entire server schema is detailed in {{fig-server-schema}}.  It is not
expected that server messages will require all fields to be used simultaneously
but instead a subset to convey a given signal type.  The only fields which may
be common to all signals are seqno and lastcliseqno which may be used to detect
loss or drop outs.

~~~~~
    message Server {
      // Server generated sequence number
      int64 seqno = 1;

      // Sequence number of last received Client message
      int64 lastcliseqno = 2;

      // May be bundled if sub-MTU
      message Mitigations {
        // Opaque Client generated event identifier
        string eventid = 1;

        // Mitigation state
        enum Enabled {
          FALSE = 0;
          TRUE = 1;
        }
        Enabled enabled = 2 [default = FALSE];

        // Remaining mitigation time to live
        int64 ttl = 3;

        // Dropped byte count
        int64 bytec = 4;

        // Dropped bps
        int64 bps = 5;

        // Dropped packet count
        int64 packc = 6;

        // Dropped pps
        int64 pps = 7;

        // Blacklist enabled
        enum Blacklist {
          False = 0;
          True = 1;
        }
        Blacklist blacklist = 8 [default = FALSE];

        // Whitelist enabled
        enum Whitelist {
          False = 0;
          True = 1;
        }
        Whitelist whitelist = 9 [default = FALSE];

        // Filters enabled
        enum Filters {
          False = 0;
          True = 1;
        }
        Filters filters = 10; [default = FALSE]

        extensions 100 to 199;
      }
      repeated Mitigations mitigations = 3;

      enum Ping {           // Heartbeat request (operator initiated)
        FALSE = 0;
        TRUE = 1;
      }
      Ping ping = 4 [default = FALSE];

      extensions 100 to 199;
    }
~~~~~
{: #fig-server-schema title="Server Schema"}


Interactions
------------

Practical interactions

Mitigation request:

~~~~~
    Client                       Server
      |                             |
      |---------Event (M=1)-------->|  // Mitigation request
      |                             |
      |<---------SvrResponse--------|  // Server acceptance
      |                             |
      |---------EventUpdate-------->|  // Efficacy update
      |                             |
      |<---------SvrResponse--------|  // Server feedback
      |                             |
      |---------Event (M=0)-------->|  // Mitigation termination
      |                             |
~~~~~

Active mitigation request:

~~~~~
    Client                       Server
      |                             |
      |-----------Status----------->|  // Status request
      |                             |
      |<-------StatusResponse-------|  // Status response
      |                             |
~~~~~

Heartbeat:

~~~~~
    Client                       Server
      |                             |
      |----------HeartBeat--------->|  // Client heartbeat
      |                             |
      |<---------HeartBeat----------|  // Server heartbeat
      |                             |
~~~~~


* Messages

  * All messages are a heartbeat

  * Client message

    * Seqno

    * Last seqno from server

    * Mit request

      * Mit requested - Boolean

      * Mit ID - opaque client-generated ID (absent if mit not requested)

      * Mit lifetime - unsigned long int (absent if mit not requested)

    * Mit scope - Object

      * identifier; OR

      * resource Prefix/IP/FQDN/URL, port, etc.

    * Attack telemetry?

  * Server message

    * Seqno

    * Last seqno from client

    * Mit statuses (Bundled if smaller than MTU and multiple mits from client)

      * Mit status

        * Mit enabled - Boolean

        * Mit ID - opaque client-generated ID from mit request above

        * Mitigation TTL

        * Dropped byte count

        * Dropped bps

        * Dropped packet count

        * Dropped pps

        * Blacklist enabled - Boolean

        * Whitelist enabled - Boolean

        * Filters enabled - Boolean


Data Channel {#data-channel}
============

Role {#data-channel-role}
----

The DOTS data channel serves as a management plane for the DOTS signal channel.
Using the conventions established in [REST], the data channel provides an
interface for configuration, black- and white-list management, traffic filter
management, and extensibility required for future operator needs (GEN-001
[I-D.ietf-dots-requirements]).


Limitations {#data-channel-limitations}
-----------

Unlike the DOTS signal channel, the data channel potentially offers DOTS client
operators limited direct control over the behavior of mitigations requested by
the DOTS client. However, the DOTS data channel is not a general purpose
application programming interface for mitigators with which a DOTS server is
communicating. Certain countermeasure profiles for DDoS attacks are widely
understood and deployed, but many remain specific to mitigation vendor
implementations, making abstraction all but impossible. The DOTS data channel in
this protocol is therefore focused on a limited subset of widely available and
well understood mitigation actions, namely black- and white-listing, and
rate-limiting.

While managing filters and rate-limit policy over the DOTS data channel
resembles the dissemination of flow specifications with a match and action on
match in [RFC5575], the similarity is restricted to [RFC5575]'s traffic-rate
action only in order to prevent a DOTS client from exerting influence over
traffic not destined for the DOTS client's domain.


Transport {#data-channel-transport}
---------

The DOTS data channel relies on the semantics described in [REST], meaning any
reliable application protocol enabling those semantics could be used. This
document anticipates HTTP/1.1 over TLS [RFC7230] will be most widely deployed at
the time of writing. Implementations of the DOTS protocol therefore MUST support
data channels using HTTP/1.1 over TLS. However, this document also leaves open
the possibility that the data channel MAY be implemented through such
application transports as HTTP/2 [RFC7540] or the Quick UDP Internet Connection
[I-D.tsvwg-quic-protocol] protocol, as well as other current and future
protocols supporting [REST] semantics and the security requirements described in
[I-D.ietf-dots-requirements]. Support for alternative secure REST transports for
the data channel are deployment- and implementation-specific.

DOTS data channel implementations MUST support the IPv4 [RFC0791] and IPv6
[RFC2460] protocols, and MUST support the "Happy Eyeballs" algorithm for dual
stack deployments discussed in [RFC6555].

Implementations of the DOTS data channel MUST use TLS version 1.2 or higher.
DOTS agents MUST NOT create a data channel with a peer agent requesting a lower
TLS version, and SHOULD drop the connection immediately on detecting the peer
DOTS agent does not support a required TLS version.

{{security-considerations}} offers a more detailed discussion of data channel
transport security, including cipher suites.


Authentication {#data-channel-authentication}
--------------

When establishing the data channel, the DOTS client and DOTS server MUST
mutually authenticate each other, per SEC-001 in [I-D.ietf-dots-requirements].
A common method for mutual authentication for HTTP/1.1 over TLS is an exchange
of X.509 certificates between client and server during the TLS handshake
[RFC5246]; similar mechanisms exist in HTTP/2 and in [I-D.tsvwg-quic-protocol].

Regardless of the underlying transport used, this document does not prescribe
the method of mutual authentication. The method of mutual authentication
used for the data channel is left to the discretion of the DOTS server operator.
Additional discussion of mutual authentication is below in
{{security-considerations}}.


Authorization {#data-channel-authorization}
-------------

TBD deployment-specific, see also security considerations.


Resources {#data-channel-resources}
---------

The DOTS server exposes data channel resources to the DOTS client as uniform
resource identifiers. The DOTS client sends requests related to the data channel
resources using the verbs defined in [RFC7231]: GET, POST, PUT, PATCH and
DELETE. The DOTS server responds to the DOTS client requests with a status code
and, if the request succeeded, available data returned by the request. The
status codes used in DOTS server responses are also defined in [RFC7231].


### Resource Root

The root resource or endpoint in the DOTS data channel is /dots/v1/data. The
root resource MUST be prefixed to all resources exposed through the data
channel.


### {+dataroot}/sessions

The /sessions endpoint is a read-only resource from which the DOTS client may
request the status of signaling sessions.


#### GET {+dataroot}/sessions

The DOTS client requests the list of signaling sessions by issuing a GET for the
/sessions resource:

~~~~~
    GET /dots/v1/data/sessions HTTP/1.1
    Host: dots-server.example.com
    Accept: application/json
~~~~~
{: #eg-client-status title="DOTS Client Requesting Session Status"}

If the DOTS client is authorized, the DOTS server responds to the GET with a
list of signaling session identifiers, as in the following example:

~~~~~
    HTTP/1.1 200 OK
    Cache-Control: no-cache
    Content-Type: application/json

    {
        "sessions": [
            {
                "id": <string>,
                "client": <ip_address>,
                "server": <ip_address>,
                "duration": <iso8601_duration>,
            },
            {
                ...
            }
        ]
    }
~~~~~

The top-level JSON key-value pairs in the response are as follows:

sessions:
: A list of dictionary objects describing active signaling sessions.  If empty,
no signaling sessions are active.

Each dictionary within the sessions list contains the following JSON key-value
pairs:

id:
: An opaque alphanumeric string identifying the signaling session.

client:
: The dotted-quad IPv4 or formatted IPv6 address [RFC5952] of the DOTS client in
  the signaling session.

server:
: The dotted-quad IPv4 or formatted IPv6 address [RFC5952] of the DOTS server in
  the signaling session.

duration:
: The [ISO8601] duration of the signaling session.

YANG model TBD


### {+dataroot}/filters

The /filters endpoint on a DOTS server is a read-write resource through which
a DOTS client may request that the DOTS server add, retrieve, modify and delete
traffic filters to an active mitigation requested through the signal channel.

A filter is a match and an action on match. As discussed above in
{{data-channel-limitations}}, actions are restricted to black- and white-listing
and rate-limiting. Matches in a filter dictionary may be any of the match types
discussed below. All matches MUST include a destination address or identifier;
DOTS server implementations MUST NOT accept filters missing a destination
address or prefix.

A filter can be represented as a map or dictionary with the following
attributes:

af:
: address family of the flow to filter, must be one of "ipv4" or "ipv6". This
  attribute is required in all filters.

dstpfx:
: destination prefix of the flow to filter, where the prefix format is
  daddr value must be

TBD: 


#### PUT {+dataroot}/filters/{+mitigation-id}

A POST request over the data channel to the /filters endpoint on a DOTS server
permits a DOTS client to manage traffic-rate policy for a mitigation:

~~~~~
    POST /dots/v1/data/filters/42 HTTP/1.1
    Host: dots-server.example.com
    Accept: application/json
    Content-Type: application/json
    Content-Length: NNNN

    {
        "filters": [
            {
                
            }
        ]
    }
~~~~~

#### GET {+dataroot}/filters/{+mitigation-id}

A GET request to the /filters endpoint on a DOTS server returns filters for a
mitigation requested by the DOTS client. The mitigation-id value MUST be the
DOTS client-generated mitigation ID used in a mitigation request previously sent
to the DOTS server over the signal channel, with the exception of the global
filter list as described below. A request listing the filters active during
a mitigation is shown below in {{eg-client-get-per-mit-filter}}:

~~~~~
    GET /dots/v1/data/filters/42 HTTP/1.1
    Host: dots-server.example.com
    Accept: application/json
~~~~~
{: #eg-client-get-per-mit-filter title="Filter GET"}

The DOTS server returns a list of active filters applied as part of the
mitigation on the DOTS client's behalf as in
{{eg-client-get-per-mit-filter-response}}:

~~~~~
    HTTP/1.1 200 OK
    Cache-Control: no-cache
    Content-Type: application/json

    {
        "id": 42,
        "filters": [
            {
            }
        ]
    }
~~~~~
{: #eg-client-get-per-mit-filter-response title="Filter GET Response"}

If the filter list is empty, no filters are applied as part of the mitigation.

If mitigation-id is omitted or if the mitigation-id value is 0, the
DOTS server MUST return the list off global filters the DOTS client has
previously added. Global filters submitted by a client are applied to all
mitigations requested by a DOTS client, and are applied before any 


### {+dataroot}/config

The /config data channel endpoint on a DOTS server is a read-write resource
through which a DOTS client may configure global signaling session behavior.

#### GET {+dataroot}/config

A GET request to the /config endpoint returns the current DOTS configuration for
the DOTS client:

~~~~~
GET /dots/v1/data/config HTTP/1.1
Host: dots-server.example.com
Accept: application/json
~~~~~
{: #eg-client-cfg-get title="DOTS Client Requesting Configuration"}

~~~~~
HTTP/1.1 200 OK
Cache-Control:
Content-Type: application/json

{
    "config": {
        "protected-resources": {
            <alnum_id>: [
            ]
        }
    }
}

~~~~~

  * Signaling session config endpoint

    * Resource identifier management

    * Max mit lifetime

    * Heartbeat interval

    * Permitted lossiness (missed heartbeat count, miss duration)

    * Action on signal loss (auto-mit, none, etc.)


### Serialization {#data-channel-resources-serialization}

Resource data is exchanged between DOTS client in a serialized format.
Implementations MUST support JSON [RFC7159] serialization of resource data. DOTS
clients MUST advertise support for JSON-encoded data from the DOTS server
through the HTTP Accept header [RFC7231] \(or an equivalent if not using HTTP),
using the MIME type defined in [RFC7159], application/json:

~~~~~
        GET /dots/v1/data/sessions HTTP/1.1
        Host: dots-server.example.com
        Accept: application/json
~~~~~
{: #eg-client-serial1 title="DOTS Client Advertising Required Serialization"}

Implementations MAY offer additional serialization formats as well. DOTS clients
MAY advertise support for additional serialization formats in requests to the
DOTS server through the HTTP Accept header [RFC7231] \(or an equivalent if not
using HTTP), as shown in the example HTTP/1.1 request below:

~~~~~
        GET /dots/v1/data/sessions HTTP/1.1
        Host: dots-server.example.com
        Accept: application/json; q=0.5, application/cbor
~~~~~
{: #eg-client-serial2 title="DOTS Client Supporting Additional Serializations"}

If a DOTS server does not support the media types in the DOTS client's Accept
header (or its equivalent), the DOTS server MUST respond with an status code
indicating an error in the client request. In HTTP deployments, the DOTS server
MUST return the 415 Unsupported Media Type error code defined in [RFC7231]. A
DOTS client request lacking indicated support for application/json content
suggests an invalid or malicious client implementation. After sending the 415
error response, DOTS servers SHOULD terminate the data channel connection with
the invalid client.


### Caching {#data-channel-resources-caching}

DOTS server responses sent over the DOTS data channel MUST NOT be cached by the
DOTS client.
DOTS server implementations MUST include the Cache-Control with a value of
"no-cache", as desc



* Endpoints

  * Black-/white-list endpoints

    * Similar to filter endpoint below

  * Status endpoint

    * GET

  * Filter endpoint

    * GET/POST/PUT/DELETE

      * GET -> *200 OK* + filter body

        * "GET /v1/filters/" returns all filters

      * POST -> *201 Created* + Location header for specific filter

      * PUT -> *204 No Content* on successful update

      * DELETE -> *204 No Content* on successful delete

  * Signaling session config endpoint

    * Resource identifier management

    * Max mit lifetime

    * Heartbeat interval

    * Permitted lossiness (missed heartbeat count, miss duration)

    * Action on signal loss (auto-mit, none, etc.)

  * Provisioning endpoint



Security Considerations
=======================

Data Channel Security
---------------------

The DOTS data channel acts as a management plane for DOTS signaling sessions.
As discussed in the security considerations of [I-D.ietf-dots-architecture], an
attacker with control over data channel may be able to blacklist or rate-limit
any flows under the administrative control of the DOTS client. Extra care must
therefore be taken when authenticating and authorizing the data channel.

DOTS server operators SHOULD enforce access control policies restricting which
addresses are able to contact

###
