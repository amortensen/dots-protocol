---
title: DDoS Open Threat Signaling Protocol
docname: draft-mortensen-dots-protocol-00
date: 2015-10-19

area: Security
wg: DOTS
kw: Internet-Draft
cat: standards

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

informative:
  RFC1518:
  RFC1519:
  RFC2373:
  RFC4271:

-- abstract

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
defense have proven effective [CITE], but the heterogenous methods employed to
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
[I-D.draft-ietf-dots-requirements].

Architectural Overview
======================

Protocol
========

Signal Channel
--------------

* Role/Purpose

* L4 Protocol

  * UDP

  * TCP

  * Should use QUIC? Quick readup on it plz

* Security

  * CurveCP? DTLS?

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


Data Channel
------------

* Role/Purpose

* Security

  * HTTPS

  * Authentication

    * Bidirectional cert auth

  * Authorization

    * Password to limit access? e.g. admin v. read-only?

* Endpoints

  * Content-Types

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

