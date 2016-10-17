---
title: Transport-Independent Path Layer State Management
abbrev: PLUS Statefulness
docname: draft-trammell-plus-statefulness-00
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: M. Kuehlewind
    name: Mirja Kuehlewind
    org: ETH Zurich
    email: mirja.kuehlewind@tik.ee.ethz.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: B. Trammell
    name: Brian Trammell
    org: ETH Zurich
    email: ietf@trammell.ch
    street: Gloriastrasse 35
    city: 8092 Zurich
    country: Switzerland
  -
    ins: J. Hildebrand 
    name: Joe Hildebrand 
    email: hildjj@cursive.net

informative:
  RFC0793:
  RFC2474:
  RFC7675:
  I-D.hamilton-quic-transport-protocol:
  draft-trammell-plus-abstract-mech:
    title: Abstract Mechanisms for a Cooperative Path Layer under Endpoint Control
    abbrev: Path Layer Mechanisms
    docname: draft-trammell-plus-abstract-mech-00
    date: 2016-09-28
    category: info
    ipr: trust200902
    area: Transport
    workgroup: PLUS BoF
    keyword: Internet-Draft
    author:
      -
        ins: B. Trammell
        name: Brian Trammell
        organization: ETH Zurich
        email: ietf@trammell.ch
        street: Gloriastrasse 35
        city: 8092 Zurich
        country: Switzerland
  IMC-GATEWAYS:
    title: An experimental study of home gateway characteristics (Proc. ACM IMC 2010)
    author:
      -
        ins: S. Hatonen
      -
        ins: A. Nyrhinen
      -
        ins: L. Eggert
      -
        ins: S. Strowes
      -
        ins: P. Sarolahti
      - 
        ins: M. Kojo

normative:
  RFC5103:
  RFC7011:
  RFC7398:

--- abstract

This document describes a simple state machine for stateful network devices on
a path between two endpoints to associate state with traffic traversing them
on a per-flow basis, as well as abstract signaling mechanisms for driving the
state machine. This state machine is intended to replace the de-facto use of
the TCP state machine or incomplete forms thereof by stateful network devices
in a transport-independent way, while still allowing for fast state timeout
of non-established or undesirable flows.

--- middle

# Introduction

The boundary between the network and transport layers was originally defined
to be that between information used (and potentially modified) hop-by-hop, 
and that used end-to-end. End-to-end information in the transport layer
is associated with state at the endpoints, but processing of network-layer
information was assumed to be stateless.

The widespread deployment of stateful middleboxes in the Internet, such as
network address and port translators (NAPT), firewalls that model the TCP
state machine to distinguish packets belonging from desirable flows from
backscatter and random attack traffic, and devices which keep per-flow state
for reporting and monitoring purposes (e.g. IPFIX {{RFC7011}} Metering
Processes), has broken this assumption, and made it more difficult to deploy
non-TCP transport protocols in the Internet.

The deployment of new transport protocols encapsulated in UDP with encrypted
transport headers (such as QUIC {{I-D.hamilton-quic-transport-protocol}}) 
will present a challenge to the operation of these devices, and their ubquity
likewise threatens to impair the deployability of these protocols. There are
two main causes for this problem: first, stateful devices often use an
internal model of the TCP state machine to determine when TCP flows start and
end, allowing them to manage state for these flows; for UDP flows, they must
rely on timeouts. These timeouts are generally short relative to those for TCP
{{IMC-GATEWAYS}}, requiring UDP- encapsulated transports either to generate
unproductive keepalive traffic for long-lived sessions, or to tolerate
connectivity problems and the necessity of reconnection due to loss of on-path
state.

This document presents a solution to this problem by defining a state machine
that is transport independent to be implemented at per-flow state-keeping
middleboxes as a replacement for incomplete TCP state modeling. Middleboxes
implementing this state machine using signals from a common UDP encapsulation
layer can have equivalent necessary state information to that provided by TCP,
reducing the friction between middleboxes and these new transport protocols.

# Terminology

In this document, the term "flow" is defined to be compatible with the
definition given in {{RFC7011}}: A flow is defined as a set of packets passing
a device on the network during a certain time interval. All packets belonging
to a particular Flow have a set of common properties. Each property is defined
as the result of applying a function to the values of:

1. one or more network layer header fields (e.g., destination IP
   address) or transport layer header fields (e.g., destination port
   number) that the device has access to;
2. one or more characteristics of the packet itself (e.g., number
   of MPLS labels, etc.);
3. one or more of the fields derived from packet treatment at the device
   (e.g., next-hop IP address, the output interface, etc.).

A packet is defined as belonging to a flow if it completely satisfies all the
defined properties of the flow. In general, the set of properties on deployed
devices includes the source and destination IP address, the source and
destination transport layer port number, the transport protocol number. The
differentiated services field {{RFC2474}} may also be included in the set of
properties defining a flow.

A bidirectional flow or biflow is defined as compatible with {{RFC5103}}, by
joining the "forward direction" flow with the "reverse direction" flow,
derived by reversing the direction of directional fields (ports and IP
addresses). Biflows are only relevant at devices positioned so as to see all
the packets in both directions of the biflow, generally on the endpoint side
of the service demarcation point for either endpoint as defined in the
reference path given in {{RFC7398}}.

# State Machine

The transport-independent state machine for on-path devices is shown in 
{{fig-states}}. It relies on four states, three configurable timeouts, and a set of
signals defined in {{abstract-signaling-mechanisms}}. The states are defined
as follows:

- zero: there is no state for a given flow at the device
- uniflow: a packet has been seen in one direction; state will be kept at the
  device until it is explicitly cancelled or until timeout t1 elapses without
  a packet.
- biflow: a packet has been seen in one direction and an indication that that 
  the receiving endpoint wishes to continue the association has been seen in 
  the opposite direction; state will be kept at the device until it is 
  explicitly canceled or until timeout t2 elapses without a packet.
- closing: an established biflow is shutting down due to an explicit close
  indication; state will be kept at the device until timeout t3 elapses.

The three timeouts are defined as follows:

- t1 is the unidirectional idle timeout. It can be considered equivalent to
  the idle timeout for transport protocols where the device has no information
  about session start and end (e.g. most UDP protocols).
- t2 is the bidirectional idle timeout. It can be considered equivalent to the 
  timeout for transport protocols where the device has information about session 
  start and end (e.g. TCP).
- t3 is the closing timeout: how long the device will account additional packets 
  to a flow after observing a stop signal, and is generally applied to ensuring
  a reordered stop signal doesn't create a new flow.

Selection of timeouts is a configuration and implementation detail, but
generally t3 <= t1 < t2.


~~~~~~~~~~~~~
   _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
  '                                         ' 
  '    +----+   packet    +-----+           '  
  '   /      \---------->/       \----+     '  UNIFLOW
  '  (  zero  )         ( uniflow )  packet '  ONLY 
  '   \      /<----------\       /<---+     '  STATES
  '    +----+   t1,stop   +-----+           '
  '_ _ _ ^_^ _ _ _ _ _ _ _ _ | _ _ _ _ _ _ _'
      t3 |  \                | association
         |   \__________     v 
       +-----+     t2   \  +----+
      /       \          \/      \----+    
     ( closing )<--------( biflow ) any-packet
      \       /   stop    \      /<---+    
       +-----+             +----+
    
~~~~~~~~~~~~~
{: #fig-states title="Transport-Independent State Machine for Stateful On-Path Devices"}

## Uniflow States

When a packet is received for a flow that the device has no state for, and it
is configured to forward the packet instead of dropping it, it moves that flow
from the zero state into the uniflow state and starts a timer t1. It resets
this timer for any additional packet it forwards in the same direction as long
as the flow remains in the uniflow state. When timer t1 expires on a flow in
the uniflow state, the device drops state for the flow and performs any
processing associated with doing so: tearing down NAT bindings, closing
associated firewall pinholes, exporting flow information, and so on. The
device may also drop state on a stop signal, if observed.

Some devices will only see one side of a communication, e.g. if they are
placed in a portion of a network with asymmetric routing. These devices use
only the zero and uniflow states (as marked in {{fig-states}}.) In addition,
true uniflows -- protocols which are solely unidirectional (e.g. some
applications over UDP) -- will also use only the uniflow-only states. In
either case, current devices generally don't associate much state with
observed uniflows flows, and timeout is generally sufficient to expire this
state.

## Biflow States

A uniflow transitions to the biflow state when the device observes an
association signal. An association signal consists of a packet sent in the
opposite direction from the uniflow packet(s), with certain properties as
defined in {{association-signaling}}. After transitioning to the biflow
state, the device starts a timer t2. It resets this timer for any packet it
forwards in either direction. The biflow state represents a fully established
bidirectional communication. When timer t2 expires, the device assumes that
the flow has shut down without signaling as such, and drops state for the
flow, performing any associated processing. When a stop signal is observed in
either direction, the flow transitions to the closing state.

When a flow enters the closing state, it starts a timer t3. While the stop
signal should be the last packet on a flow, the t3 timer ensures that
reordered packets after the stop signal will be accounted to the flow. When
this timer expires, the device drops state for the flow, performing any
associated processing.

## Additional States

Devices may augment the transitions in this state diagram depending on their
function. For example, a firewall that decides based on some information
beyond the signals used by this state machine to shut down a flow may
transition it directly to a blacklist state on shutdown. This document is
concerned only with states and transitions common to transport- and function-
independent state maintenance.

# Abstract Signaling Mechanisms

The state machine in {{state-machine}} requires three signals: a packet (the
first packet observed in a flow in the zero state), an association signal (allowing
a device to verify that an endpoint wishes a bidirectional communication to be
established or to continue), and a stop signal (noting that an endpoint wishes
to stop a bidirectional communication). Additional related signals may also be
useful, depending on the function a device provides. There are a few different
ways to implement these signals; here, we explore the properties of some
potential implementations.

We assume the following general requirements for these signals; parallel to
those given in {{draft-trammell-plus-abstract-mech}}:

- At least the endpoints can verify the integrity of the signals exposed, and
  shut down a transport association when that verification fails, in order to
  reduce the incentive for on-path devices to attempt to spoof these signals.
- Endpoints and devices on path can probabilistically verify that a originator
  of a signal is on-path.

## Association Signaling

An association signal indicates that endpoint that received the first packet seen
by the device is interested in continuing conversation with the sending
endpoint. In TCP, the association signal corresponding to a first packet with the
SYN flag set is a SYN-ACK packet sent in reply with an appropriate
acknowledgment number. However, a transport-independent signal cannot rely on
TCP-specific header fields.

Note that association signaling for this state machine is an in-band analogue to consent
signaling in ICE {{RFC7675}}.

Transport-independent, path-verifiable association signaling can be implemented
using a association token generated by one endpoint, present on each packet
sent in the flow by that endpoint, and a response token, derived from the
association token using a well-known, defined function, present on each packet
sent in the flow by the opposite endpoint. We can assume a transport
association has an initiator and a responder; under this assumption, the
association token is chosen by the initiator, and the response token generated
by the responder.

Any packet sent by the responder with a valid response token, and without a
stop signal (see {{stop-signaling}}), can then be taken to be a association signal
to continue a bidirectional communication. Note that, since it relies on a
widely-known function, this mechanism does allow on-path devices to forge
association signaling in a way that downstream on-path devices cannot detect.
However, in the presence of end-to-end signal integrity verification, this
forgery will be detected by the endpoint, which MUST terminate the association
on a forged association signal; the flow at the duped on-path device will
transition from biflow to closing within a single packet. This reduces any
attack against the association signaling mechanism to the disruption of a
connection, which on-path devices can do in any case by simply refusing to
forward packets.

Association tokens MUST be chosen by initiators to be hard to guess.
Cryptographic random number generators suffice here. Note there is a tradeoff
between per-packet overhead and state overhead at on-path devices, and
assurance that an association token is hard to guess. This tradeoff must be
evaluated at protocol design time.

There are a few considerations in choosing a function (or functions) to
generate the response token from the association token, and to verify a
response token given an association token.

- Is the function symmetric (that is, do on-path devices do the same
  processing to verify a response token as the responder does to generate it),
  or asymmetric (where different processing is used on-path than at the
  responder)? 
- Is the function one-way or two-way? For one-way functions, it is difficult
  to generate an association token from a valid response token; for two-way
  functions, this difficulty does not exist. Using a two-way function may make
  it simpler to design on-path devices with reduced per-flow state 
  requirements.
- If the function is asymmetric, is there only a single valid response token
  for each association token, or multiple valid response tokens?
- If the function is asymmetric, the verification process should be
  computationally as complex or less complex than the generation process.

Two possible functions for response token generation are simple one's
complement of the association token (a two-way, symmetric function), or taking
N bits from an SHA-2 hash of the association token (a one-way, symmetric
function). Both provide some assurance of knowledge of the association token
and implementation of the protocol. In any case, a concrete implementation of
association signaling must choose a single function, or mechanism for
unambiguously choosing one, at both endpoints as well as along the path.

Note that if a concrete protocol driving this state machine requires the
association and response tokens to be present on every packet, the token
present identifies the direction of traffic.

## Stop Signaling

A stop signal is a single bit directly carried or otherwise encoded in the protocol
header to indicate that a flow is ending, whether normally or abnormally, and
that state associated with the flow should be torn down. Upon decoding a stop
signal, a device on path should move the flow from uniflow state to null, or
from biflow state to closing.

Transports should send a stop signal only on the last packet sent in a
bidirectional flow. The closing timeout t3 is intended to ensure that any
packets reordered in delivery are accounted to the flow before state for it is
dropped.

We assume the encoding of a stop signal into a packet header, as with all
other signals, is integrity protected end-to-end. Stop signals, as association
signals, be forged by one on-path device to dupe other devices into moving
flows into the closing state. However, state will be re-established by the
continuing flow (and resulting association signals) after the closing timeout, and
an endpoint receiving a spoofed stop signal could enter a fast 
re-establishment phase of the upper layer transport protocol to minimize
disruption, further reducing the incentive to attackers to spoof stop signals.

<!--
## Header-Only Forwarding

[EDITOR'S NOTE: probably cut this, since it is extremely expensive to implement on path; potentially move it to the plus protocol document as an optional feature, since it makes some assumptions about PLUS we don't here: A network device in "no state" might only forward the unencrypted/header parts of a packet to transition to "biflow" state without forwarding unknow data. A PLUS receiver MUST reply with a (PLUS-only) packet (within a given time frame). On receiption of this PLUS-only packet, the network device tranits in "biflow" state and SHOULD resend the whole packet including the encrypted data while dropping a PLUS-only packet. If the network device was not able to store the encrypted data, it forwards the PLUS-only packet to the original sending endpoint which then can be used as a fast trigger that the original encrypted data was not received by the intented receiving endpoint.]
-->

## Timeout Exposure

Since one of the goals of these mechanisms is to reduce the amount of non-
productive keepalive traffic required for UDP-encapsulated transport
protocols, they MAY be deployed together with a path-to-receiver signal with
feedback as defined in {{draft-trammell-plus-abstract-mech}} asking for timeouts
t1, t2, and t3 for a given flow.

## Signal mapping for TCP

A mapping of TCP flags to transitions in to the state machine in {{state-
machine}} shows how devices currently using a model of the TCP state machine
can be converted to use this state machine. 

Simply speaking, the ACK flag {{RFC0793}} in the absence of the FIN or RST
flags is synonymous with the association signal, while the RST or final FIN flags
are stop signals. For a typical TCP flow:

1. The initial SYN places the flow into uniflow state,
2. The SYN-ACK sent in reply acts as a association signal and places the flow into biflow state,
3. Any RST moves the flow into closing state, or 
4. The final FIN-ACK (not the first half-close FIN) moves the flow into closing state.

# IANA Considerations

This document has no actions for IANA.

# Security Considerations

This document defines a state machine for transport-independent state
management on middleboxes, using in-band signaling, to replace the commonly-
implemented current practice of incomplete TCP state modeling on these
devices. It defines new signals for state management. While these signals can
be spoofed by any device on path that observes traffic in both directions, we
presume the presence of end-to-end integrity protection of these signals
provided by the upper-layer transport driving them. This allows such spoofing
to be detected and countered by endpoints, reducing the threat from on-path
devices to connection disruption, which such devices are trivially placed to
perform in any case.

# Acknowledgments

Thanks to Christian Huitema for discussions leading to this document.

This work is partially supported by the European Commission under Horizon 2020
grant agreement no. 688421 Measurement and Architecture for a Middleboxed
Internet (MAMI), and by the Swiss State Secretariat for Education, Research,
and Innovation under contract no. 15.0268. This support does not imply
endorsement.