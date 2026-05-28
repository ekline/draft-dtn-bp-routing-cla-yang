---
title: "YANG Data Models for Bundle Protocol Routing and Convergence Layers"
abbrev: "BP Routing and CLA YANG"
docname: draft-ek-dtn-bp-routing-cla-yang-latest
category: std
ipr: trust200902
area: Internet
workgroup: Delay-Tolerant Networking
keyword:
  - DTN
  - Bundle Protocol
  - BPv7
  - YANG
  - RIB
  - FIB
  - Convergence Layer
  - TCPCLv4

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    fullname: "Erik Kline"
    organization: Aalyria Technologies
    email: "ek.ietf@gmail.com"

normative:
  RFC2119:
  RFC6241:
  RFC6991:
  RFC7950:
  RFC8040:
  RFC8174:
  RFC8340:
  RFC8342:
  RFC8343:
  RFC8349:
  RFC9171:
  RFC9172:
  RFC9174:
  RFC9758:
  I-D.ietf-dtn-eid-pattern:

informative:
  RFC4838:
  RFC5326:
  RFC9675:
  I-D.ietf-dtn-bibect:

--- abstract

This document defines YANG data models for two aspects of a Bundle
Protocol Agent (BPA): the routing information base (RIB) and forwarding
information base (FIB) used to direct bundles toward their destinations,
and the convergence layer adapters (CLAs) that exchange bundles with
peer BPAs.  A minimal companion module defines the BPA's node
identifiers, providing a common augmentation point for the routing and
CLA modules.

A complete YANG data model for a Bundle Protocol Agent (covering
fragmentation, reassembly, status report generation, endpoint
registration, storage management, CRC handling, and BPSec policy) is
explicitly out of scope for this document and is expected to be
addressed by a forthcoming companion document, structured as an
additive revision of the minimal BPA module defined here.

--- middle

# Introduction

The Bundle Protocol version 7 (BPv7) {{RFC9171}} provides delay- and
disruption-tolerant data transfer across a heterogeneous internetwork
of nodes connected by potentially intermittent links.  A Bundle
Protocol Agent (BPA) is the software entity at a node that originates,
forwards, and delivers bundles; it does so by consulting a routing
information base (RIB), selecting a route from a forwarding
information base (FIB) derived from that RIB, and handing matching
bundles to a convergence layer adapter (CLA) that implements bundle
transfer over an underlying transport.

To date, the IETF has produced numerous specifications describing how
BPAs interoperate, but no standardized YANG {{RFC7950}} data model
exists for managing BPAs through NETCONF {{RFC6241}}, RESTCONF
{{RFC8040}}, or other YANG-driven mechanisms.  Operators deploying
BPAs across diverse hardware and software platforms have therefore
relied on implementation-specific configuration interfaces, hindering
multi-vendor interoperability and operational uniformity.

This document begins addressing that gap by defining YANG models for
the routing and convergence-layer aspects of a BPA.  These are the
two subsystems most directly involved in determining how bundles flow
through a DTN, and they are also the two subsystems where
multi-vendor operational consistency is most urgently needed.

## Scope

This document defines five YANG modules:

* `ietf-dtn-types` — common types and identities for DTN YANG
  modules, including the Endpoint Identifier (EID) type and the EID
  scheme identity hierarchy.

* `ietf-dtn-bp` — a minimal BPA model containing only the BPA's node
  identifiers, providing a common augmentation point (`/bp:bp`) for
  other DTN YANG modules.

* `ietf-dtn-bp-routing` — the routing information base (RIB) and
  forwarding information base (FIB), structured along the lines of
  {{RFC8349}} but adapted to the EID-based naming and CLA-based
  next-hop resolution of the Bundle Protocol.

* `ietf-dtn-cla` — base configuration and operational state for
  convergence layer adapters, with provision for CLA-type-specific
  augmentation.

* `ietf-dtn-cla-tcpclv4` — an illustrative augmentation of the base
  CLA module with TCPCLv4-specific configuration {{RFC9174}}.
  Analogous augmentation modules can be defined for other CLA types.

## Non-Goals

The following are explicitly out of scope for this document:

* **Complete BPA modeling.**  Configuration of fragmentation,
  reassembly, status report generation, endpoint registration,
  storage management, CRC handling, and BPSec policy are deferred
  to a future revision of the `ietf-dtn-bp` module.  The skeleton
  defined here is intentionally minimal so that the future revision
  can be purely additive.

* **Contact Graph Routing (CGR) and other dynamic routing
  protocols.**  This document defines static routing only.  The
  `routing-protocol-type` identity defined herein is the
  extensibility hook for future routing protocol modules.  A
  companion CGR YANG module, including contact plan modeling, is
  envisioned as separate future work.

* **Policy-based routing on criteria other than destination EID.**
  Source-EID matching, extension-block matching (e.g., for QoS
  blocks), and other policy criteria are anticipated as future
  augmentations of the route `match` container defined here.  The
  schema is structured to make such extensions purely additive.

* **Endpoint registration and local delivery configuration.**  These
  are anticipated for a companion delivery-management module.

* **Application-level interfaces.**  Application-to-BPA APIs are
  outside the IETF YANG modeling scope.

## Terminology

{::boilerplate bcp14-tagged}

This document uses the terminology of {{RFC9171}} for Bundle Protocol
concepts (bundle, BPA, EID, convergence layer, administrative
endpoint, etc.) and the terminology of {{RFC7950}} for YANG
concepts (module, container, list, leaf, leafref, identity,
augmentation, etc.).

The following additional terms are used:

EID pattern:
: A string that matches a set of EIDs, as defined in
  {{I-D.ietf-dtn-eid-pattern}}.

Route:
: A configured or computed association between a set of bundles
  (described by match criteria) and a forwarding action (described
  by a next hop and optional forwarding policy).

Equal-cost group:
: The set of routes that are applicable to a given bundle and that
  share the same preference value.  Selection within an equal-cost
  group is governed by the routes' forwarding-policy mode.

Forwarding-policy mode:
: A property of a route describing how the BPA selects among
  multiple applicable routes in the same equal-cost group.  Modes
  defined here include `mode-none` (deterministic selection),
  `mode-ecmp` (equal-cost multipath), and `mode-ucmp` (weighted
  multipath).

## Tree Diagram Notation

Tree diagrams in this document use the notation defined in
{{RFC8340}}.

# Architectural Overview

## Module Structure

The five modules defined here are organized around a common
augmentation point.  The minimal `ietf-dtn-bp` module defines the
top-level `/bp:bp` container with only the BPA's node identifiers as
direct children.  The routing and CLA modules each augment `/bp:bp`
with their respective subtrees.  CLA-type-specific modules in turn
augment individual CLA list entries.

~~~~
+---------------------+
| ietf-dtn-types      |   (typedefs, identities)
+---------------------+
            ^  ^  ^
            |  |  |
   +--------+  |  +---------+
   |           |            |
+--+-------------+   +------+------+
| ietf-dtn-bp    |   | (others)    |
| /bp:bp         |   |             |
+----------------+   +-------------+
        ^
        | augment
        |
   +----+------------------+
   |                       |
+--+-------------+    +----+-----------+
| ietf-dtn-bp-   |    | ietf-dtn-cla   |
| routing        |    |                |
+----------------+    +----------------+
                              ^
                              | augment
                              |
                      +-------+--------+
                      | ietf-dtn-cla-  |
                      | tcpclv4 (and   |
                      | other type-    |
                      | specific       |
                      | modules)       |
                      +----------------+
~~~~

This structure allows the future BPA-completion document to revise
`ietf-dtn-bp` additively (introducing fragmentation, reassembly,
storage, etc., as new children of `/bp:bp` or as new sibling
containers) without disturbing the routing or CLA modules.

## Endpoint Identifiers

Endpoint Identifiers (EIDs) are represented in this model as strings,
following the URI form mandated by {{RFC9171}}.  The `ietf-dtn-types`
module defines a permissive `eid` typedef that accepts any URI-form
string within length bounds.  Scheme-specific validation (notably
the `ipn` scheme as updated by {{RFC9758}}) is left to
implementations.

EID patterns, used to specify the set of EIDs matched by a route,
are defined as a separate `eid-pattern` typedef.  The grammar and
semantics of EID patterns are defined in {{I-D.ietf-dtn-eid-pattern}}.

A BPA's node IDs, exposed via the `/bp:bp/node-id` leaf-list, also
identify the BPA's administrative endpoints per Section 6.1 of
{{RFC9171}}.  For the `dtn` scheme, the node ID and the
administrative endpoint EID are identical.  For the `ipn` scheme,
the administrative endpoint is the EID with service number 0 sharing
the same node component (per {{RFC9758}}).  Any EID whose
scheme-specific node component matches the node component of an
entry in the `node-id` list is considered local to the BPA and is
subject to local delivery rather than forwarding.

## Relationship to RFC 8349

The routing module follows the broad structure of {{RFC8349}}:

* A `control-plane-protocols/control-plane-protocol` list keyed by
  protocol type and instance name, providing the extensibility
  point for adding routing protocols (static routing here; CGR and
  others in future modules).

* A `ribs/rib` list for the RIBs maintained by configured
  protocols, with operational `routes` lists populated from each
  RIB.

* A `fib` container exposing the routes currently selected for
  forwarding.

The routing module diverges from {{RFC8349}} in three ways:

1. **Route keying.**  Static routes are keyed by an administrative
   `name`, not by destination prefix.  This admits future
   augmentations that match on criteria other than destination
   (source EID, extension blocks, etc.) without disturbing the
   schema.

2. **Match expressed as a container.**  Each static route contains
   a `match` container holding match criteria.  This revision
   defines a single criterion (`destination-eid-pattern`); future
   companion documents are expected to augment the `match`
   container additively.

3. **Explicit FIB.**  A `fib` container is provided as operational
   state separate from the RIB.  The BPA's route selection
   algorithm (see {{route-selection}}) determines FIB membership
   from RIB content; this can differ from per-RIB best-route
   selection alone, especially once policy-based matching extends
   the model.

## Route Selection Algorithm {#route-selection}

A BPA, given a bundle to forward, performs route selection as
follows.

1. The BPA constructs a *candidate list* by enumerating all routes
   across all configured RIBs in ascending order of `preference`.
   Ties in preference are broken by route `name` in lexicographic
   order, for determinism.

2. The BPA walks the candidate list.  For each route, it evaluates
   all leaves and child containers within the route's `match`
   container against the bundle.  A route is *applicable* if and
   only if every match criterion is satisfied.

3. The BPA locates the *equal-cost group*: the maximal set of
   applicable routes that share the same `preference` value,
   beginning at the first applicable route found in step 2.

4. If the equal-cost group contains exactly one route, the BPA
   forwards the bundle via that route's `next-hop`.

5. If the equal-cost group contains more than one route, the BPA
   selects among them according to the `forwarding-policy/mode` of
   the routes in the group:

   * `mode-none`: the BPA selects one route deterministically
     (e.g., the lexicographically first by `name`) and uses it
     for every bundle that matches.

   * `mode-ecmp`: the BPA distributes bundles across the routes in
     the group by hashing the tuple specified by `hash-input`
     (defaulting to `(source EID, destination EID)`).

   * `mode-ucmp`: the BPA distributes bundles across the routes in
     the group in proportion to their configured `weight` values,
     normalized within the group.

   Routes in the same equal-cost group SHOULD have identical
   `forwarding-policy` configuration.  If they differ, the BPA MUST
   treat the group as if `mode-none` were in effect and select the
   first applicable route by `name`.

6. The schema does not enforce non-overlapping match criteria.
   Operators are responsible for ordering routes by `preference`
   such that the intended forwarding policy is expressed by step 2.

Future companion documents defining new forwarding-policy modes
(notably modes that replicate a bundle across multiple routes in a
group) MUST specify the interaction of the new mode with primary-
block flags (custody transfer, deletion-on-forward, do-not-fragment)
and with BPSec security blocks.

## Forward Compatibility

Three points in the schema are explicitly reserved as augmentation
hooks for future work:

* The `match` container under each static route.  Future modules
  add policy criteria (source EID, extension block matching, etc.)
  by augmenting `match` additively.

* The `mode-parameters` choice under each route's
  `forwarding-policy` container.  New forwarding modes derived
  from `forwarding-policy-mode` add their parameters as new cases.

* The `active-contact-info` container under each CLA's `state`
  container.  Future contact-plan modules augment this container
  with operational state describing the currently-active contact
  bound to the CLA, if any.

# Module Tree Diagrams

## ietf-dtn-bp

~~~~
module: ietf-dtn-bp
  +--rw bp
     +--rw node-id*           dtn-types:eid
     +--rw primary-node-id?   -> ../node-id
~~~~

## ietf-dtn-bp-routing

~~~~
module: ietf-dtn-bp-routing

  augment /bp:bp:
    +--rw routing
       +--rw control-plane-protocols
       |  +--rw control-plane-protocol* [type name]
       |     +--rw type           identityref
       |     +--rw name           string
       |     +--rw description?   string
       |     +--rw static-routes
       |        +--rw route* [name]
       |           +--rw name                 string
       |           +--rw preference?          uint32
       |           +--rw match
       |           |  +--rw destination-eid-pattern
       |           |          dtn-types:eid-pattern
       |           +--rw next-hop
       |           |  +--rw next-hop-type    identityref
       |           |  +--rw (next-hop-resolution)?
       |           |     +--:(eid)
       |           |     |  +--rw next-hop-eid       dtn-types:eid
       |           |     +--:(cla)
       |           |     |  +--rw cla-binding
       |           |     |          -> /bp:bp/cla:convergence-layers
       |           |     |             /cla:cla/cla:name
       |           |     +--:(multicast)
       |           |        +--rw multicast-member*  dtn-types:eid
       |           +--rw forwarding-policy
       |              +--rw mode?              identityref
       |              +--rw (mode-parameters)?
       |                 +--:(ecmp)
       |                 |  +--rw hash-input?  identityref
       |                 +--:(ucmp)
       |                    +--rw weight       uint32
       +--rw ribs
       |  +--rw rib* [name]
       |     +--rw name           string
       |     +--rw description?   string
       |     +--ro routes
       |        +--ro route* [name]
       |           +--ro name                       string
       |           +--ro source-protocol?           leafref
       |           +--ro preference?                uint32
       |           +--ro destination-eid-pattern?
       |           |        dtn-types:eid-pattern
       |           +--ro installed-time?            yang:date-and-time
       +--ro fib
          +--ro entry* [name]
             +--ro name                       string
             +--ro rib?                       leafref
             +--ro destination-eid-pattern?   dtn-types:eid-pattern
             +--ro next-hop-eid?              dtn-types:eid
             +--ro cla-binding?               leafref
             +--ro selected-at?               yang:date-and-time
             +--ro statistics
                +--ro bundles-forwarded?     yang:counter64
                +--ro bytes-forwarded?       yang:counter64
                +--ro forwarding-failures?   yang:counter64
~~~~

## ietf-dtn-cla

~~~~
module: ietf-dtn-cla

  augment /bp:bp:
    +--rw convergence-layers
       +--rw cla* [name]
          +--rw name                  string
          +--rw type                  identityref
          +--rw enabled?              boolean
          +--rw direction?            identityref
          +--rw local-eid?            -> /bp:bp/node-id
          +--rw bound-interface*      if:interface-ref
          +--rw mtu?                  uint64
          +--rw peer* [peer-eid]
          |  +--rw peer-eid    dtn-types:eid
          |  +--rw address?    string
          |  +--ro source*     identityref
          +--ro state
             +--ro oper-status?         identityref
             +--ro last-state-change?   yang:date-and-time
             +--ro active-peers?        yang:gauge32
             +--ro bundles-tx?          yang:counter64
             +--ro bundles-rx?          yang:counter64
             +--ro bytes-tx?            yang:counter64
             +--ro bytes-rx?            yang:counter64
             +--ro active-contact-info
~~~~

## ietf-dtn-cla-tcpclv4

~~~~
module: ietf-dtn-cla-tcpclv4

  augment /bp:bp/cla:convergence-layers/cla:cla:
    +--rw tcpclv4
       +--rw listener* [local-address local-port]
       |  +--rw local-address    inet:ip-address
       |  +--rw local-port       inet:port-number
       +--rw segment-mru?         uint64
       +--rw transfer-mru?        uint64
       +--rw keepalive-interval?  uint16
       +--rw idle-timeout?        uint16
       +--rw tls
          +--rw require?            boolean
          +--rw min-version?        enumeration
          +--rw cipher-suite*       string
          +--rw node-certificate?   string
~~~~

# YANG Modules

## ietf-dtn-types

~~~~ yang
<CODE BEGINS> file "ietf-dtn-types@2026-05-27.yang"
module ietf-dtn-types {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:ietf-dtn-types";
  prefix dtn-types;

  organization
    "IETF Delay-Tolerant Networking (DTN) Working Group";

  contact
    "WG Web:   <https://datatracker.ietf.org/wg/dtn/>
     WG List:  <mailto:dtn@ietf.org>";

  description
    "This module defines common YANG types and identities used by
     other Delay-Tolerant Networking (DTN) YANG modules.

     Copyright (c) 2026 IETF Trust and the persons identified as
     authors of the code.  All rights reserved.

     Redistribution and use in source and binary forms, with or
     without modification, is permitted pursuant to, and subject
     to the license terms contained in, the Revised BSD License
     set forth in Section 4.c of the IETF Trust's Legal Provisions
     Relating to IETF Documents
     (https://trustee.ietf.org/license-info).

     This version of this YANG module is part of RFC XXXX
     (https://www.rfc-editor.org/info/rfcXXXX); see the RFC
     itself for full legal notices.";

  revision 2026-05-27 {
    description
      "Initial revision.";
    reference
      "RFC XXXX: YANG Data Models for Bundle Protocol Routing
                and Convergence Layers";
  }

  /* -------------------- Identities -------------------- */

  identity eid-scheme {
    description
      "Base identity for URI schemes used in Bundle Protocol
       Endpoint Identifiers (EIDs).  Derived identities correspond
       to specific schemes registered for use in BP EIDs.";
    reference
      "RFC 9171, Section 4.2.5.1";
  }

  identity scheme-dtn {
    base eid-scheme;
    description
      "The 'dtn' URI scheme for Bundle Protocol Endpoint
       Identifiers.";
    reference
      "RFC 9171, Section 4.2.5.1.1";
  }

  identity scheme-ipn {
    base eid-scheme;
    description
      "The 'ipn' URI scheme for Bundle Protocol Endpoint
       Identifiers, with textual representation as updated by
       RFC 9758.";
    reference
      "RFC 9171, Section 4.2.5.1.2;
       RFC 9758";
  }

  /* -------------------- Typedefs -------------------- */

  typedef eid {
    type string {
      length "1..1024";
      pattern '[a-zA-Z][a-zA-Z0-9+\-.]*:.*';
    }
    description
      "A Bundle Protocol Endpoint Identifier (EID), encoded as a
       URI.  An EID begins with a scheme name followed by ':'
       and a scheme-specific part.

       The detailed format of the scheme-specific part is defined
       by each scheme:

       - The 'dtn' scheme is defined in Section 4.2.5.1.1 of
         RFC 9171.

       - The 'ipn' scheme is defined in Section 4.2.5.1.2 of
         RFC 9171, with the textual representation as updated by
         RFC 9758.  RFC 9758 supersedes the original textual
         representation of ipn-scheme EIDs.

       The pattern enforced by this typedef is intentionally
       permissive to allow for future scheme extensions.
       Implementations of this YANG model SHOULD validate
       scheme-specific syntax in addition to the loose pattern
       admitted here.";
    reference
      "RFC 9171, Section 4.2.5.1;
       RFC 9758";
  }

  typedef eid-pattern {
    type string {
      length "1..1024";
    }
    description
      "A pattern that matches one or more Bundle Protocol
       Endpoint Identifiers.  The grammar and semantics of EID
       patterns are defined in [I-D.ietf-dtn-eid-pattern].

       An EID pattern is used to specify the set of destination
       (or, in future extensions, source) EIDs to which a route
       or policy applies.  A pattern that exactly equals an EID
       (with no wildcard or range constructs) matches only that
       single EID.

       Implementations SHOULD validate pattern syntax against the
       grammar defined in [I-D.ietf-dtn-eid-pattern] when
       accepting configuration.";
    reference
      "I-D.ietf-dtn-eid-pattern";
  }
}
<CODE ENDS>
~~~~

## ietf-dtn-bp

~~~~ yang
<CODE BEGINS> file "ietf-dtn-bp@2026-05-27.yang"
module ietf-dtn-bp {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:ietf-dtn-bp";
  prefix bp;

  import ietf-dtn-types {
    prefix dtn-types;
    reference "RFC XXXX";
  }

  organization
    "IETF Delay-Tolerant Networking (DTN) Working Group";

  contact
    "WG Web:   <https://datatracker.ietf.org/wg/dtn/>
     WG List:  <mailto:dtn@ietf.org>";

  description
    "This module defines a minimal YANG data model for a Bundle
     Protocol Agent (BPA), comprising only the BPA's set of node
     identifiers.  It provides a common augmentation point
     ('/bp:bp') for other DTN YANG modules, including those for
     routing and convergence-layer management.

     A complete YANG data model for a BPA (covering fragmentation,
     reassembly, status report generation, endpoint registration,
     storage management, CRC handling, and BPSec policy) is out
     of scope for the document defining the initial revision of
     this module and is expected to be addressed by a future
     revision that adds child nodes to '/bp:bp' additively.

     Copyright (c) 2026 IETF Trust and the persons identified as
     authors of the code.  All rights reserved.

     Redistribution and use in source and binary forms, with or
     without modification, is permitted pursuant to, and subject
     to the license terms contained in, the Revised BSD License
     set forth in Section 4.c of the IETF Trust's Legal Provisions
     Relating to IETF Documents
     (https://trustee.ietf.org/license-info).

     This version of this YANG module is part of RFC XXXX
     (https://www.rfc-editor.org/info/rfcXXXX); see the RFC
     itself for full legal notices.";

  revision 2026-05-27 {
    description
      "Initial revision.";
    reference
      "RFC XXXX";
  }

  container bp {
    description
      "Top-level container for Bundle Protocol Agent configuration
       and operational state.  Other DTN YANG modules augment
       this container to add routing, convergence-layer, and
       related subtrees.";

    leaf-list node-id {
      type dtn-types:eid;
      description
        "A node identifier of this Bundle Protocol Agent (BPA),
         expressed as an Endpoint Identifier (EID).

         Per Section 6.1 of RFC 9171, each node ID also
         identifies an administrative endpoint at which this BPA
         receives administrative records (status reports and
         other control bundles).  For the 'dtn' scheme the node
         ID and the administrative endpoint EID are identical;
         for the 'ipn' scheme the administrative endpoint is the
         EID with service number 0 sharing the same node
         component, per RFC 9758.

         Any EID whose scheme-specific node component matches the
         node component of an entry in this list is considered
         local to this BPA and is subject to local delivery
         processing rather than forwarding.

         A BPA typically has one node ID per EID scheme it
         supports.  Multiple node IDs within the same scheme are
         permitted but unusual; their use is implementation-
         defined.

         At least one node ID SHOULD be configured for any BPA
         that originates or accepts bundles.  This module does
         not impose a 'min-elements 1' constraint on the
         leaf-list so as to permit a configured-but-disabled BPA
         to exist transiently with no node IDs declared.";
      reference
        "RFC 9171, Section 4.2.5.1 and Section 6.1;
         RFC 9758";
    }

    leaf primary-node-id {
      type leafref {
        path "../node-id";
      }
      description
        "Designates one entry in the 'node-id' leaf-list as the
         primary node ID of this BPA.  The primary node ID is
         used as the source EID for bundles originated by this
         BPA when no source EID is specified by the originating
         application, and as the default 'local-eid' for CLA
         instances that do not explicitly configure one.

         If this leaf is unset, the BPA selects a primary node
         ID in an implementation-defined manner, typically by
         choosing the first entry in 'node-id' in declaration
         order.";
    }
  }
}
<CODE ENDS>
~~~~

## ietf-dtn-bp-routing

~~~~ yang
<CODE BEGINS> file "ietf-dtn-bp-routing@2026-05-27.yang"
module ietf-dtn-bp-routing {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:ietf-dtn-bp-routing";
  prefix bp-rt;

  import ietf-dtn-types {
    prefix dtn-types;
    reference "RFC XXXX";
  }
  import ietf-dtn-bp {
    prefix bp;
    reference "RFC XXXX";
  }
  import ietf-dtn-cla {
    prefix cla;
    reference "RFC XXXX";
  }
  import ietf-yang-types {
    prefix yang;
    reference "RFC 6991";
  }

  organization
    "IETF Delay-Tolerant Networking (DTN) Working Group";

  contact
    "WG Web:   <https://datatracker.ietf.org/wg/dtn/>
     WG List:  <mailto:dtn@ietf.org>";

  description
    "This module defines a YANG data model for the Bundle
     Protocol Routing Information Base (RIB) and Forwarding
     Information Base (FIB).  Its structure is inspired by the
     routing data model defined in RFC 8349, adapted to the
     Bundle Protocol's EID-based naming, EID-pattern-based
     matching, and convergence-layer-based next-hop resolution.

     This revision supports static routing only.  Additional
     routing protocols (notably Contact Graph Routing) are
     expected to be addressed by future companion documents that
     define new identities derived from 'routing-protocol-type'
     and augment the corresponding 'control-plane-protocol' list
     entries.

     Copyright (c) 2026 IETF Trust ...";

  revision 2026-05-27 {
    description "Initial revision.";
    reference "RFC XXXX";
  }

  /* -------------------- Identities -------------------- */

  identity routing-protocol-type {
    description
      "Base identity for Bundle Protocol routing protocol types.";
  }

  identity proto-static {
    base routing-protocol-type;
    description
      "Static routing.  Routes installed by configuration only.";
  }

  identity proto-direct {
    base routing-protocol-type;
    description
      "Direct routes derived from local CLA peer configuration.
       Implementations MAY automatically install direct routes
       for each peer EID configured on an active CLA.";
  }

  identity next-hop-type {
    description
      "Base identity for the kind of next-hop resolution used by
       a route.";
  }

  identity nh-type-eid {
    base next-hop-type;
    description
      "Next hop is identified by the EID of a peer BPA.  The
       local BPA resolves the peer EID to one or more convergence
       layer adapters at forwarding time.";
  }

  identity nh-type-cla {
    base next-hop-type;
    description
      "Next hop is identified by direct reference to a CLA
       instance, bypassing peer-EID resolution.  Used for
       point-to-point links and opportunistic contacts.";
  }

  identity nh-type-multicast {
    base next-hop-type;
    description
      "Next hop is a set of peer EIDs representing the recipients
       of a multicast bundle.  The local BPA replicates the
       bundle and forwards one copy to each listed member.";
  }

  identity forwarding-policy-mode {
    description
      "Base identity for forwarding-policy modes governing
       selection among routes in an equal-cost group.

       Future modules MAY derive additional modes from this
       identity, including modes that replicate a bundle across
       multiple routes within a group.  Such modules MUST
       specify the interaction of the new mode with primary-
       block flags (notably custody transfer, deletion-on-
       forward, and do-not-fragment) and with BPSec security
       blocks.";
  }

  identity mode-none {
    base forwarding-policy-mode;
    description
      "No multipath behavior.  When multiple applicable routes
       share the same preference, the BPA selects one
       deterministically and forwards all matching bundles via
       that route.";
  }

  identity mode-ecmp {
    base forwarding-policy-mode;
    description
      "Equal-Cost Multi-Path.  Bundles matching the equal-cost
       group are distributed across applicable routes by hash,
       according to the 'hash-input' parameter.";
  }

  identity mode-ucmp {
    base forwarding-policy-mode;
    description
      "Unequal-Cost Multi-Path.  Bundles matching the equal-cost
       group are distributed across applicable routes in
       proportion to their configured weights.";
  }

  identity hash-input {
    description
      "Base identity for hash-input selections used by ECMP-mode
       routes.";
  }

  identity hash-per-bundle {
    base hash-input;
    description
      "Each bundle is independently assigned to a route via a
       fresh random or pseudo-random selection.  No flow
       affinity is preserved.";
  }

  identity hash-per-flow-eid-pair {
    base hash-input;
    description
      "Bundles are assigned to routes by hashing the tuple
       (source EID, destination EID).  All bundles in the same
       (source, destination) flow take the same path within an
       equal-cost group.";
  }

  identity hash-per-flow-with-creation-ts {
    base hash-input;
    description
      "Bundles are assigned to routes by hashing the tuple
       (source EID, destination EID, creation timestamp).
       Provides flow stickiness with looser grouping than
       'hash-per-flow-eid-pair'.";
  }

  /* -------------------- Augmentation -------------------- */

  augment "/bp:bp" {
    description
      "Augments the Bundle Protocol Agent with routing
       configuration and operational state.";

    container routing {
      description
        "Top-level container for Bundle Protocol routing.";

      container control-plane-protocols {
        description
          "Configured instances of routing protocols (and static
           routing) that contribute routes to one or more RIBs.";

        list control-plane-protocol {
          key "type name";
          description
            "An instance of a routing protocol or static routing
             configuration.  Multiple instances of the same
             protocol type MAY coexist.";

          leaf type {
            type identityref {
              base routing-protocol-type;
            }
            description
              "The routing protocol type of this instance.";
          }

          leaf name {
            type string {
              length "1..64";
            }
            description
              "An administrative name for this protocol
               instance.";
          }

          leaf description {
            type string;
            description
              "Free-form description of this protocol instance.";
          }

          container static-routes {
            when "derived-from-or-self(../type, "
               + "'bp-rt:proto-static')";
            description
              "Static routes configured under this protocol
               instance.  Present only when 'type' is
               'proto-static'.";

            list route {
              key "name";
              description
                "A statically configured route.

                 Routes are keyed by an administrative 'name'
                 rather than by destination, to permit future
                 extensions that match on criteria other than
                 destination (notably source EID and extension
                 block matching, defined in companion
                 documents).";

              leaf name {
                type string {
                  length "1..64";
                }
                description
                  "Administrative name for this route.";
              }

              leaf preference {
                type uint32;
                default "100";
                description
                  "Route preference.  Smaller values are more
                   preferred.  Routes are evaluated against
                   incoming bundles in ascending preference
                   order; the first route whose 'match' criteria
                   are satisfied is selected for forwarding.

                   See the Route Selection Algorithm section of
                   RFC XXXX for the full selection algorithm.";
              }

              container match {
                description
                  "Criteria a bundle must satisfy to be
                   eligible for forwarding via this route.

                   This revision defines a single match
                   criterion: destination EID pattern.  Future
                   companion documents are expected to augment
                   this container to add source EID matching,
                   extension block matching, and other
                   policy-based matching criteria.  Such
                   augmentations are purely additive: existing
                   leaves and their semantics are preserved.";

                leaf destination-eid-pattern {
                  type dtn-types:eid-pattern;
                  mandatory true;
                  description
                    "Pattern matching the set of destination
                     EIDs to which this route applies.  Pattern
                     grammar is defined in
                     [I-D.ietf-dtn-eid-pattern].";
                }
              }

              container next-hop {
                description
                  "Forwarding next hop for bundles matching
                   this route.";

                leaf next-hop-type {
                  type identityref {
                    base next-hop-type;
                  }
                  mandatory true;
                  description
                    "Discriminator selecting the next-hop kind.";
                }

                choice next-hop-resolution {
                  description
                    "The next-hop value, whose form depends on
                     'next-hop-type'.";

                  case eid {
                    when "derived-from-or-self(../next-hop-type,"
                       + " 'bp-rt:nh-type-eid')";
                    leaf next-hop-eid {
                      type dtn-types:eid;
                      mandatory true;
                      description
                        "The EID of the peer BPA to which
                         matching bundles SHOULD be forwarded.";
                    }
                  }

                  case cla {
                    when "derived-from-or-self(../next-hop-type,"
                       + " 'bp-rt:nh-type-cla')";
                    leaf cla-binding {
                      type leafref {
                        path "/bp:bp/cla:convergence-layers"
                           + "/cla:cla/cla:name";
                      }
                      mandatory true;
                      description
                        "Direct reference to a CLA instance
                         used to forward bundles matching this
                         route.";
                    }
                  }

                  case multicast {
                    when "derived-from-or-self(../next-hop-type,"
                       + " 'bp-rt:nh-type-multicast')";
                    leaf-list multicast-member {
                      type dtn-types:eid;
                      min-elements 1;
                      description
                        "Set of peer EIDs constituting the
                         multicast fanout for this route.  The
                         BPA replicates the bundle and forwards
                         one copy to each listed member.";
                    }
                  }
                }
              }

              container forwarding-policy {
                description
                  "Forwarding-policy parameters governing
                   selection within an equal-cost group of
                   routes (those sharing the same preference and
                   matching the same bundle).

                   Routes that are the sole applicable route in
                   their preference group are unaffected by this
                   container.";

                must "not(derived-from-or-self("
                   + "../next-hop/next-hop-type,"
                   + " 'bp-rt:nh-type-multicast'))"
                   + " or derived-from-or-self(mode,"
                   + " 'bp-rt:mode-none')" {
                  error-message
                    "A multicast next-hop is incompatible with "
                  + "non-trivial forwarding-policy modes.";
                  description
                    "Multicast next-hops already implement
                     replication semantics; combining them with
                     ECMP or UCMP is not defined.";
                }

                leaf mode {
                  type identityref {
                    base forwarding-policy-mode;
                  }
                  default "bp-rt:mode-none";
                  description
                    "The forwarding-policy mode for this route.";
                }

                choice mode-parameters {
                  description
                    "Parameters specific to the configured
                     mode.  Each derived mode identity that
                     requires parameters defines a case here.";

                  case ecmp {
                    when "derived-from-or-self(../mode,"
                       + " 'bp-rt:mode-ecmp')";
                    leaf hash-input {
                      type identityref {
                        base hash-input;
                      }
                      default "bp-rt:hash-per-flow-eid-pair";
                      description
                        "Selects the tuple over which the BPA
                         hashes to assign bundles to routes
                         within the equal-cost group.";
                    }
                  }

                  case ucmp {
                    when "derived-from-or-self(../mode,"
                       + " 'bp-rt:mode-ucmp')";
                    leaf weight {
                      type uint32 {
                        range "1..max";
                      }
                      mandatory true;
                      description
                        "Relative weight of this route within
                         the equal-cost group.  Weights are
                         normalized across all members of the
                         group.";
                    }
                  }
                }
              }
            }
          }
        }
      }

      container ribs {
        description
          "Routing Information Bases (RIBs) maintained by this
           BPA.

           This revision defines RIBs only as operational state
           populated by control-plane protocols.  A future
           revision MAY introduce configuration controls for
           RIB selection and filtering.";

        list rib {
          key "name";
          description
            "A named RIB.  Most deployments will use a single
             default RIB; multiple RIBs MAY be defined for
             scenarios requiring routing-domain separation.";

          leaf name {
            type string {
              length "1..64";
            }
            description "Administrative name of this RIB.";
          }

          leaf description {
            type string;
            description "Free-form description of this RIB.";
          }

          container routes {
            config false;
            description
              "Routes currently installed in this RIB by
               configured control-plane protocols.";

            list route {
              key "name";
              description
                "An installed route.";

              leaf name {
                type string;
                description
                  "Name of this route as reported by the
                   installing protocol.";
              }

              leaf source-protocol {
                type leafref {
                  path
                    "../../../../control-plane-protocols"
                  + "/control-plane-protocol/name";
                }
                description
                  "Reference to the control-plane protocol
                   instance that installed this route.";
              }

              leaf preference {
                type uint32;
                description "Preference of this route.";
              }

              leaf destination-eid-pattern {
                type dtn-types:eid-pattern;
                description
                  "Destination EID pattern, replicated here
                   from the route's match container for ease
                   of inspection.";
              }

              leaf installed-time {
                type yang:date-and-time;
                description
                  "Time at which this route was installed.";
              }
            }
          }
        }
      }

      container fib {
        config false;
        description
          "Forwarding Information Base (FIB).  Operational
           state describing the routes currently selected by
           the BPA for forwarding, derived from the RIBs by
           the route selection algorithm specified in
           RFC XXXX.

           FIB entries are keyed by route name (matching the
           installing route) rather than by destination,
           because future policy extensions may produce
           multiple selected routes whose destinations
           overlap.";

        list entry {
          key "name";
          description "A selected forwarding entry.";

          leaf name {
            type string;
            description
              "Name of the selected route (matching the route
               name in the installing RIB).";
          }

          leaf rib {
            type leafref {
              path "../../../ribs/rib/name";
            }
            description
              "RIB from which this entry was selected.";
          }

          leaf destination-eid-pattern {
            type dtn-types:eid-pattern;
            description
              "Destination EID pattern of the selected route.";
          }

          leaf next-hop-eid {
            type dtn-types:eid;
            description
              "Resolved peer EID, when applicable.";
          }

          leaf cla-binding {
            type leafref {
              path "/bp:bp/cla:convergence-layers/cla:cla"
                 + "/cla:name";
            }
            description
              "Resolved CLA instance used to transmit bundles
               matching this entry, when applicable.";
          }

          leaf selected-at {
            type yang:date-and-time;
            description
              "Time at which this FIB entry was last
               computed.";
          }

          container statistics {
            description
              "Per-FIB-entry forwarding statistics.";

            leaf bundles-forwarded {
              type yang:counter64;
              description
                "Total bundles forwarded via this entry since
                 it was installed.";
            }
            leaf bytes-forwarded {
              type yang:counter64;
              description
                "Total bytes forwarded via this entry since
                 it was installed.";
            }
            leaf forwarding-failures {
              type yang:counter64;
              description
                "Total forwarding attempts via this entry
                 that failed.";
            }
          }
        }
      }
    }
  }
}
<CODE ENDS>
~~~~

## ietf-dtn-cla

~~~~ yang
<CODE BEGINS> file "ietf-dtn-cla@2026-05-27.yang"
module ietf-dtn-cla {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:ietf-dtn-cla";
  prefix cla;

  import ietf-dtn-types {
    prefix dtn-types;
    reference "RFC XXXX";
  }
  import ietf-dtn-bp {
    prefix bp;
    reference "RFC XXXX";
  }
  import ietf-yang-types {
    prefix yang;
    reference "RFC 6991";
  }
  import ietf-interfaces {
    prefix if;
    reference "RFC 8343";
  }

  organization
    "IETF Delay-Tolerant Networking (DTN) Working Group";

  contact
    "WG Web:   <https://datatracker.ietf.org/wg/dtn/>
     WG List:  <mailto:dtn@ietf.org>";

  description
    "Base YANG data model for Bundle Protocol Convergence Layer
     Adapters (CLAs).  CLA-specific modules augment the 'cla'
     list defined here with parameters appropriate to each CLA
     type (TCPCLv4, UDPCL, LTP, BIBE, BTP-U, etc.).

     Copyright (c) 2026 IETF Trust ...";

  revision 2026-05-27 {
    description "Initial revision.";
    reference "RFC XXXX";
  }

  /* -------------------- Identities -------------------- */

  identity cla-type {
    description
      "Base identity for convergence layer adapter types.";
  }

  identity cla-tcpclv4 {
    base cla-type;
    description "TCP Convergence Layer Protocol Version 4.";
    reference "RFC 9174";
  }

  identity cla-udpcl {
    base cla-type;
    description
      "UDP Convergence Layer.  A datagram-based CLA suitable for
       single-bundle exchanges over UDP.";
  }

  identity cla-ltp {
    base cla-type;
    description
      "Licklider Transmission Protocol convergence layer.";
    reference "RFC 5326";
  }

  identity cla-bibe {
    base cla-type;
    description
      "Bundle-in-Bundle Encapsulation convergence layer.";
    reference "I-D.ietf-dtn-bibect";
  }

  identity cla-direction {
    description
      "Base identity for the directional capability of a CLA
       instance.";
  }

  identity direction-bidirectional {
    base cla-direction;
    description
      "CLA supports both transmission and reception.";
  }

  identity direction-tx-only {
    base cla-direction;
    description "CLA supports transmission only.";
  }

  identity direction-rx-only {
    base cla-direction;
    description "CLA supports reception only.";
  }

  identity cla-oper-status {
    description
      "Base identity for CLA operational status values.";
  }

  identity oper-up {
    base cla-oper-status;
    description "CLA is up and able to transfer bundles.";
  }

  identity oper-down {
    base cla-oper-status;
    description
      "CLA is administratively or operationally down.";
  }

  identity oper-degraded {
    base cla-oper-status;
    description
      "CLA is partially operational (e.g., some peers reachable
       but not others).";
  }

  identity peer-source {
    description
      "Base identity for the provenance of a CLA peer entry.";
  }

  identity peer-source-static {
    base peer-source;
    description "Peer was configured statically.";
  }

  identity peer-source-discovery {
    base peer-source;
    description
      "Peer was learned via a neighbor-discovery mechanism.
       Specific discovery mechanisms are defined by future
       documents.";
  }

  identity peer-source-contact-plan {
    base peer-source;
    description
      "Peer was instantiated by a contact-plan-driven routing
       protocol.  Defined here as a placeholder; the contact
       plan data model is expected to be specified in a future
       companion document.";
  }

  /* -------------------- Augmentation -------------------- */

  augment "/bp:bp" {
    description
      "Augments the Bundle Protocol Agent with convergence-layer
       configuration and state.";

    container convergence-layers {
      description
        "Container for the BPA's convergence layer adapters.";

      list cla {
        key "name";
        description
          "A convergence layer adapter instance.

           Each entry represents one configured CLA.  CLA-type-
           specific configuration is provided by augmentations
           that are conditional on 'type'.";

        leaf name {
          type string {
            length "1..64";
          }
          description
            "Administrative name of this CLA instance.

             Implementations SHOULD NOT change a CLA's name
             during its lifetime; external references to a CLA
             (notably from the routing module and from future
             contact-plan modules) depend on name stability.";
        }

        leaf type {
          type identityref {
            base cla-type;
          }
          mandatory true;
          description "The CLA type of this instance.";
        }

        leaf enabled {
          type boolean;
          default "true";
          description
            "Administratively enables this CLA instance.";
        }

        leaf direction {
          type identityref {
            base cla-direction;
          }
          default "cla:direction-bidirectional";
          description
            "Directional capability of this CLA instance.";
        }

        leaf local-eid {
          type leafref {
            path "/bp:bp/bp:node-id";
          }
          description
            "The local node ID this CLA advertises to peers.

             If unset, the CLA defaults to the BPA's primary
             node ID (see '/bp:bp/bp:primary-node-id').  Setting
             this leaf is useful in multi-persona deployments
             where different CLAs advertise different node IDs
             of the same BPA (e.g., a 'dtn:' identity on one
             network and an 'ipn:' identity on another).";
        }

        leaf-list bound-interface {
          type if:interface-ref;
          description
            "Network interfaces to which this CLA is
             restricted.  If empty, the CLA may use any
             interface.";
        }

        leaf mtu {
          type uint64;
          units "bytes";
          description
            "Maximum transfer unit at this CLA: the largest
             single bundle (or bundle fragment) that may be
             transmitted via this CLA in one transfer.  Unset
             means no CLA-imposed limit beyond those imposed by
             the underlying transport.";
        }

        list peer {
          key "peer-eid";
          description
            "Peer endpoints known to this CLA.  Used by CLAs
             that maintain explicit peer relationships (e.g.,
             TCPCLv4); CLAs that do not maintain peer state
             SHOULD leave this list empty.";

          leaf peer-eid {
            type dtn-types:eid;
            description "The peer BPA's node ID.";
          }

          leaf address {
            type string;
            description
              "Transport-layer address of the peer, in a
               CLA-specific format.  CLA-specific augmentations
               SHOULD provide a strongly-typed alternative leaf
               (e.g., 'inet:host' plus 'inet:port-number' for
               TCPCLv4) and SHOULD NOT require this leaf when
               the typed alternative is present.";
          }

          leaf-list source {
            type identityref {
              base peer-source;
            }
            config false;
            description
              "Provenance of this peer entry: how it came to be
               in this list (static configuration, neighbor
               discovery, contact-plan instantiation, etc.).";
          }
        }

        container state {
          config false;
          description
            "Operational state of this CLA instance.";

          leaf oper-status {
            type identityref {
              base cla-oper-status;
            }
            description "Current operational status.";
          }

          leaf last-state-change {
            type yang:date-and-time;
            description
              "Time of the most recent change in
               'oper-status'.";
          }

          leaf active-peers {
            type yang:gauge32;
            description
              "Number of peers currently in an active state on
               this CLA.";
          }

          leaf bundles-tx {
            type yang:counter64;
            description "Total bundles transmitted.";
          }

          leaf bundles-rx {
            type yang:counter64;
            description "Total bundles received.";
          }

          leaf bytes-tx {
            type yang:counter64;
            description "Total bytes transmitted.";
          }

          leaf bytes-rx {
            type yang:counter64;
            description "Total bytes received.";
          }

          container active-contact-info {
            description
              "Container reserved for future augmentation by
               modules that bind CLAs to scheduled contacts.

               This revision defines no child nodes; a future
               contact-plan YANG module is expected to augment
               this container with operational state describing
               the currently-active contact bound to this CLA,
               if any.";
          }
        }
      }
    }
  }
}
<CODE ENDS>
~~~~

## ietf-dtn-cla-tcpclv4

~~~~ yang
<CODE BEGINS> file "ietf-dtn-cla-tcpclv4@2026-05-27.yang"
module ietf-dtn-cla-tcpclv4 {
  yang-version 1.1;
  namespace "urn:ietf:params:xml:ns:yang:ietf-dtn-cla-tcpclv4";
  prefix cla-tcpcl;

  import ietf-dtn-bp {
    prefix bp;
    reference "RFC XXXX";
  }
  import ietf-dtn-cla {
    prefix cla;
    reference "RFC XXXX";
  }
  import ietf-inet-types {
    prefix inet;
    reference "RFC 6991";
  }

  organization
    "IETF Delay-Tolerant Networking (DTN) Working Group";

  contact
    "WG Web:   <https://datatracker.ietf.org/wg/dtn/>
     WG List:  <mailto:dtn@ietf.org>";

  description
    "Augmentation of the base CLA YANG module with configuration
     parameters specific to TCPCLv4 (RFC 9174).  This module is
     provided as an illustrative example of the CLA-specific
     augmentation pattern; analogous modules can be defined for
     other CLA types.

     Copyright (c) 2026 IETF Trust ...";

  reference "RFC 9174";

  revision 2026-05-27 {
    description "Initial revision.";
    reference "RFC XXXX";
  }

  augment "/bp:bp/cla:convergence-layers/cla:cla" {
    when "derived-from-or-self(cla:type, 'cla:cla-tcpclv4')";
    description
      "Adds TCPCLv4-specific configuration to a CLA instance.";

    container tcpclv4 {
      description
        "TCPCLv4-specific configuration.";

      list listener {
        key "local-address local-port";
        description
          "Local TCP endpoints on which this TCPCLv4 instance
           accepts inbound connections.";

        leaf local-address {
          type inet:ip-address;
          description "Local IP address to bind.";
        }
        leaf local-port {
          type inet:port-number;
          default "4556";
          description
            "Local TCP port to bind.  Default 4556 per
             RFC 9174.";
          reference "RFC 9174, Section 5";
        }
      }

      leaf segment-mru {
        type uint64;
        units "bytes";
        default "1048576";
        description
          "Maximum size of an inbound transfer segment.";
        reference "RFC 9174, Section 4.3";
      }

      leaf transfer-mru {
        type uint64;
        units "bytes";
        description
          "Maximum size of an inbound transfer (bundle).
           Unset means no CLA-imposed limit.";
        reference "RFC 9174, Section 4.3";
      }

      leaf keepalive-interval {
        type uint16;
        units "seconds";
        default "60";
        description
          "Interval between keepalive messages.  Zero
           disables keepalives.";
        reference "RFC 9174, Section 4.6";
      }

      leaf idle-timeout {
        type uint16;
        units "seconds";
        default "600";
        description
          "Idle session timeout.  Zero disables the timeout.";
      }

      container tls {
        description
          "TLS configuration for this TCPCLv4 instance.";

        leaf require {
          type boolean;
          default "false";
          description
            "When true, TLS MUST be used; sessions that
             cannot establish TLS are rejected.";
          reference "RFC 9174, Section 4.4";
        }

        leaf min-version {
          type enumeration {
            enum tls12 {
              description "TLS 1.2 (RFC 5246).";
            }
            enum tls13 {
              description "TLS 1.3 (RFC 8446).";
            }
          }
          default "tls13";
          description "Minimum acceptable TLS version.";
        }

        leaf-list cipher-suite {
          type string;
          description
            "Permitted TLS cipher suites, in IANA registry
             string form (e.g., 'TLS_AES_256_GCM_SHA384').
             If empty, implementation defaults apply.";
        }

        leaf node-certificate {
          type string;
          description
            "Reference to or PEM-encoded form of the X.509
             certificate this node presents during TLS
             negotiation.  The encoding and lookup of this
             leaf are implementation-specific.";
        }
      }
    }
  }
}
<CODE ENDS>
~~~~

# Worked Examples

This section illustrates the data models with two small examples in
abbreviated YANG instance-data form.

## Static Route via Peer EID

A BPA with a single `ipn`-scheme node ID, one TCPCLv4 CLA, and one
static route directing all `ipn:42.*` traffic to peer `ipn:7.0` over
the TCPCLv4 CLA:

~~~~
/bp:bp/node-id = ["ipn:5.0"]
/bp:bp/primary-node-id = "ipn:5.0"

/bp:bp/cla:convergence-layers/cla:cla[name="tcpcl-public"]:
  type             = cla:cla-tcpclv4
  enabled          = true
  local-eid        = "ipn:5.0"

  /cla-tcpcl:tcpclv4/listener:
    [local-address="0.0.0.0", local-port=4556]

/bp:bp/bp-rt:routing/control-plane-protocols/control-plane-protocol
       [type="bp-rt:proto-static", name="default"]:
  static-routes/route[name="ipn-42-network"]:
    preference                            = 100
    match/destination-eid-pattern         = "ipn:42.*"
    next-hop/next-hop-type                = bp-rt:nh-type-eid
    next-hop/next-hop-eid                 = "ipn:7.0"
    forwarding-policy/mode                = bp-rt:mode-none
~~~~

## ECMP across Two CLAs

Two static routes at the same preference, each directing to a
different CLA, forming an ECMP group hashed per (source, destination)
flow:

~~~~
/bp:bp/bp-rt:routing/control-plane-protocols/control-plane-protocol
       [type="bp-rt:proto-static", name="default"]:
  static-routes/route[name="ipn-42-via-cla-a"]:
    preference                            = 100
    match/destination-eid-pattern         = "ipn:42.*"
    next-hop/next-hop-type                = bp-rt:nh-type-cla
    next-hop/cla-binding                  = "cla-a"
    forwarding-policy/mode                = bp-rt:mode-ecmp
    forwarding-policy/hash-input          = bp-rt:hash-per-flow-eid-pair

  static-routes/route[name="ipn-42-via-cla-b"]:
    preference                            = 100
    match/destination-eid-pattern         = "ipn:42.*"
    next-hop/next-hop-type                = bp-rt:nh-type-cla
    next-hop/cla-binding                  = "cla-b"
    forwarding-policy/mode                = bp-rt:mode-ecmp
    forwarding-policy/hash-input          = bp-rt:hash-per-flow-eid-pair
~~~~

# Security Considerations

The YANG modules specified in this document define schemas for data
that is designed to be accessed via network management protocols
such as NETCONF {{RFC6241}} or RESTCONF {{RFC8040}}.  The lowest
NETCONF layer is the secure transport layer, and the mandatory-to-
implement secure transport is Secure Shell (SSH).  The lowest
RESTCONF layer is HTTPS, and the mandatory-to-implement secure
transport is TLS.

The Network Configuration Access Control Model (NACM) provides the
means to restrict access for particular NETCONF or RESTCONF users
to a preconfigured subset of all available NETCONF or RESTCONF
protocol operations and content.

There are a number of data nodes defined in these YANG modules
that are writable, creatable, and deletable (i.e., `config true`,
which is the default).  These data nodes may be considered
sensitive or vulnerable in some network environments.  Write
operations to these data nodes can have a negative effect on
network operations.  Specifically:

* `/bp:bp/node-id`, `/bp:bp/primary-node-id`: Modifying a BPA's
  node identifiers changes the identity it presents to peers and
  may invalidate authentication state, contact plan entries, and
  static routes configured by peers.  Unauthorized modification
  can also enable identity spoofing.

* `/bp:bp/bp-rt:routing/.../static-routes/route`: Unauthorized
  creation, modification, or deletion of static routes can cause
  bundles to be misforwarded, replayed, dropped, or delivered to
  attacker-controlled endpoints.  This is the most operationally
  consequential subtree in the modules defined here and SHOULD be
  protected with the strictest NACM rules.

* `/bp:bp/cla:convergence-layers/cla:cla`: Modifying CLA
  configuration can disable an entire transport path, alter the
  set of peers a CLA accepts, or change TLS or other security
  parameters (in CLA-specific augmentations).  Of particular
  sensitivity in TCPCLv4 are the `tls/require`, `tls/min-version`,
  and `tls/node-certificate` leaves; tampering with these can
  downgrade or disable transport security.

Some of the readable data nodes in these YANG modules may be
considered sensitive or vulnerable in some network environments.
It is thus important to control read access to these data nodes:

* The full set of routes in the RIB and FIB reveals the BPA's
  view of network topology, including peer EIDs and contact
  arrangements.  In adversarial environments, this information
  can be used to plan traffic-analysis or denial-of-service
  attacks.

* CLA peer lists reveal the identities of correspondent BPAs.

* Operational counters (bundles-tx, bytes-tx, etc.) may permit
  inference of traffic patterns even when bundle contents are
  protected.

The models defined here do not themselves provide bundle-level
security; bundle confidentiality, integrity, and authentication
are provided by BPSec {{RFC9172}}.  A future YANG module for
BPSec policy configuration is anticipated.

# IANA Considerations

This document registers the following URIs in the "ns" subregistry
within the "IETF XML Registry" {{!RFC3688}}:

~~~~
URI: urn:ietf:params:xml:ns:yang:ietf-dtn-types
Registrant Contact: The IESG.
XML: N/A; the requested URI is an XML namespace.

URI: urn:ietf:params:xml:ns:yang:ietf-dtn-bp
Registrant Contact: The IESG.
XML: N/A; the requested URI is an XML namespace.

URI: urn:ietf:params:xml:ns:yang:ietf-dtn-bp-routing
Registrant Contact: The IESG.
XML: N/A; the requested URI is an XML namespace.

URI: urn:ietf:params:xml:ns:yang:ietf-dtn-cla
Registrant Contact: The IESG.
XML: N/A; the requested URI is an XML namespace.

URI: urn:ietf:params:xml:ns:yang:ietf-dtn-cla-tcpclv4
Registrant Contact: The IESG.
XML: N/A; the requested URI is an XML namespace.
~~~~

This document registers the following YANG modules in the "YANG
Module Names" registry {{!RFC6020}}:

~~~~
name:         ietf-dtn-types
namespace:    urn:ietf:params:xml:ns:yang:ietf-dtn-types
prefix:       dtn-types
reference:    RFC XXXX

name:         ietf-dtn-bp
namespace:    urn:ietf:params:xml:ns:yang:ietf-dtn-bp
prefix:       bp
reference:    RFC XXXX

name:         ietf-dtn-bp-routing
namespace:    urn:ietf:params:xml:ns:yang:ietf-dtn-bp-routing
prefix:       bp-rt
reference:    RFC XXXX

name:         ietf-dtn-cla
namespace:    urn:ietf:params:xml:ns:yang:ietf-dtn-cla
prefix:       cla
reference:    RFC XXXX

name:         ietf-dtn-cla-tcpclv4
namespace:    urn:ietf:params:xml:ns:yang:ietf-dtn-cla-tcpclv4
prefix:       cla-tcpcl
reference:    RFC XXXX
~~~~

--- back

# Acknowledgements
{:numbered="false"}

The authors thank the members of the IETF DTN working group for
discussions that shaped this work.

# Open Issues
{:numbered="false"}

The following issues are flagged for working group discussion:

1. **Author identification.**  Author full name and email address
   are placeholders in this -00 revision and require update before
   working group adoption.

2. **'min-elements 1' on '/bp:bp/node-id'.**  The leaf-list is
   currently unconstrained to permit transient configurations
   without node IDs.  Whether to require at least one node ID via
   YANG (with an associated `must` or `min-elements` constraint)
   versus leaving this to operator discipline is open.

3. **CLA peer 'address' format.**  This document treats the
   generic peer 'address' leaf as an opaque string, with CLA-
   specific augmentations expected to provide typed alternatives.
   An alternative is to forbid the generic leaf entirely and
   require typed addresses; this is more rigorous but precludes
   minimal CLA configurations.

4. **FIB selection visibility.**  The current FIB does not surface
   which RIB routes were considered but not selected.  Operators
   may wish for diagnostic state describing rejection reasons
   (e.g., "route X rejected because next-hop CLA is down").  This
   could be added under each FIB entry without disrupting the
   schema.

5. **Multiple RIBs.**  The schema supports multiple named RIBs but
   does not specify how routes from different RIBs interact during
   FIB construction.  Real deployments will need explicit
   semantics; whether to define these in this document or defer to
   a companion document is open.

