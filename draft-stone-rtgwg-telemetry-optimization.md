---
title: "Telemetry-Driven Congestion Optimization"
abbrev: "Telemetry-Driven Congestion Optimization"
category: info

docname: draft-stone-rtgwg-telemetry-optimization-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Routing"
workgroup: "Routing Area Working Group"
keyword:
 - congestion
 - optimization

author:
 -
    fullname: Andrew Stone
    organization: Nokia
    email: andrew.stone@nokia.com

normative:
  RFC3209:
  RFC9256:

informative:
  RFC3630:
  RFC4655:
  RFC5305:
  RFC5036:
  RFC5440:
  RFC6241:
  RFC7011:
  RFC7471:
  RFC7950:
  RFC8051:
  RFC8231:
  RFC8253:
  RFC8281:
  RFC8426:
  RFC8570:
  RFC8571:
  RFC8664:
  RFC9350:
  RFC9522:
  RFC9552:
  RFC9603:
  RFC9830:
  RFC9857:
  I-D.gandhi-pce-pm:
  GNMI:
    title: "gRPC Network Management Interface (gNMI)"
    target: https://github.com/openconfig/reference/blob/master/rpc/gnmi/gnmi-specification.md
    author:
      org: OpenConfig
    date: false

...

--- abstract


This document describes Telemetry-Driven Congestion Optimization
(TDCO), a solution in which a centralized controller uses
near-real-time per-tunnel demand and per-interface traffic
measurements to prevent and alleviate link overload autonomously.
The controller changes the paths of the tunnels under its control
through existing mechanisms such as the Path Computation Element
Communication Protocol (PCEP), BGP Segment Routing (SR) Policy, or
NETCONF/YANG. TDCO avoids placing Traffic Engineering (TE)
tunnels where they would cause overload and, when overload occurs,
re-routes only as many TE tunnels as are needed to relieve the
overload onto paths with spare capacity that meet their full TE
constraints.

This document describes the solution and the safety properties
needed to operate it without operator approval of individual
actions. It defines no protocol extensions.


--- middle



# Introduction

Network links can become overloaded when traffic grows unexpectedly,
when link capacity is degraded, or when traffic is re-routed after a
failure.

An overloaded link degrades the traffic that it carries. Negative
effects may begin before a link is completely full, because short
traffic bursts can exceed the link capacity even when the average load
over a measurement interval is below it.

Alleviating an overload generally requires moving only some of the
traffic on the link, onto paths that have spare capacity and that
still meet the Traffic Engineering (TE) requirements of the traffic
that is moved. A router upstream of an
overloaded link can steer traffic quickly, but it does not in general
know the load on the links of the alternate path or the TE
constraints of the traffic that it steers. Network-wide
re-optimization and capacity planning take both into account, but
they operate on longer timescales and often move many TE tunnels at
once from a snapshot. A centralized controller that holds a network-wide view of the
topology, the TE tunnel paths, and the measured traffic is in a
position to close these gaps.

This document describes Telemetry-Driven Congestion Optimization
(TDCO), in which a centralized controller prevents and alleviates
link overload by steering the TE tunnels whose paths it controls and
that operator policy includes in optimization. The controller changes
those paths through existing mechanisms, for example the Path
Computation Element Communication Protocol (PCEP), BGP Segment
Routing (SR) Policy, or NETCONF/YANG.
The controller combines the TE topology and link
capacities, the measured demand of each TE tunnel, and the measured
traffic on each interface into a view of the load on every link, and
uses that view in two ways:

Proactive component:
: When the controller computes a path for a TE tunnel whose demand is known,
it prefers paths on which that demand does not push any link past
the trigger threshold. Traffic is therefore kept off links that are
about to become overloaded, not only removed from links that already
are.

Reactive component:
: When the measured load on one or more links exceeds the trigger
threshold, the controller re-routes only as many TE tunnels as are
needed to bring each of those links back to the target threshold, preferring
paths that do not fill any other link past the target threshold. Subsequent telemetry confirms the effect and drives any
further adjustment, forming a closed loop.

A TE tunnel that was moved off a link returns to it only if the link
then stays at or below the target threshold. This keeps headroom on
the link and prevents TE tunnels from being moved back and forth.

TDCO has three defining properties:

No human in the loop:
: The controller detects the overload, selects the TE tunnels to move,
computes alternate paths, and re-routes the selected TE tunnels, all
without operator approval of individual actions. Instead, safe
operation relies on automated safety controls, such as requiring an
overload to be sustained before acting, suspending optimization when
telemetry is stale, and damping re-routes, as described in
{{safety}}.

Measured demand:
: TE tunnel demand is learned from near-real-time streaming telemetry,
flow collection, or PCEP performance metrics, rather than from
signalled reservations or planning data.

Full TE-based computation:
: Paths are computed with each TE tunnel's complete TE constraint set
and with bandwidth awareness, so that offloaded traffic does not
overload other links or violate TE policies or Service Level
Agreements (SLAs). Adjustments are made by changing the paths of
TE tunnels at their headends, and are incremental.

The purpose of this document is to describe this solution, and the
safety properties needed to run it without operator approval of
individual actions, so that it can be deployed with a common
understanding of its behavior.

TDCO applies equally to RSVP-TE signalled LSPs and to SR Policies on
the SR-MPLS and SRv6 data planes, and accounts for both in a single
view of link load. This document does not define new protocol
mechanisms. The algorithms used to select and place TE tunnels are
left to implementations.

## Relationship to Existing Work {#existing-work}

TDCO builds on and is related to the following work:

* {{RFC9522}} gives an overview of traffic engineering,
  including online, reactive congestion management. TDCO is an
  instance of that approach, carried out by a controller.

* {{RFC8051}} describes use cases for stateful PCE, including
  optimization based on the traffic that LSPs carry. TDCO describes
  one such use case in detail, including the safety properties needed
  to run it autonomously, and applies it to any controller that can
  change TE tunnel paths, whether or not that controller is a PCE.

* {{RFC8426}} describes models for the coexistence of RSVP-TE and SR
  LSPs. TDCO follows the centralized management of available capacity
  model ({{Section 3.2 of RFC8426}}).

# Terminology {#terminology}

This document uses the terms Path Computation Element (PCE), Path
Computation Client (PCC), and Label Switched Path (LSP) as defined in
{{RFC4655}} and {{RFC5440}}, and the stateful PCE terms delegation and
PCE-initiated LSP as defined in {{RFC8231}} and {{RFC8281}}.

Where this document uses the term LSP in a general sense, the
concepts described apply equally to SR Policies on both the SR-MPLS
and SRv6 data planes.

In addition, this document uses the following terms:

Controller:
: A centralized entity that performs TDCO for the TE tunnels whose
paths it controls. A stateful PCE {{RFC8231}} is
one example of a controller.

TE Tunnel:
: An SR Policy {{RFC9256}} or an RSVP-TE signalled LSP {{RFC3209}}
whose path the controller is permitted to change, using one of the
mechanisms described in {{enactment}}. Examples include LSPs
delegated to the controller {{RFC8231}}, LSPs initiated by the
controller {{RFC8281}}, and SR Policy candidate paths that the
controller originates.

TE tunnel demand:
: The measured rate of traffic carried by a TE tunnel, as learned by
the controller from streaming telemetry, flow collection, or PCEP
performance metrics. TE tunnel demand is distinct from any bandwidth
that is signalled or reserved for the TE tunnel.

Dark traffic:
: Traffic carried on a link that is not carried in a TE tunnel, for
example native IPv4, native IPv6, or MPLS traffic that follows
shortest paths such as Label Distribution Protocol (LDP) {{RFC5036}},
native SR, or Flexible Algorithm {{RFC9350}} traffic.

Link load:
: The load on one direction of a link, as determined by the controller
from the measured utilization learned through streaming telemetry or
from the utilization advertised in the IGP or BGP-LS, adjusted for TE
tunnel path changes that are in flight and for
bandwidth reservations that the measurements do not reflect. Link
load includes dark traffic (see {{link-load}}). Link load is the
value against which overload detection and placement decisions are
checked.

Trigger threshold:
: The configured link load, expressed as an absolute value or as a
percentage of link capacity, above which a link is considered
overloaded.

Target threshold:
: The configured link load, less than or equal to the trigger threshold, to which
the reactive component of TDCO aims to bring an overloaded link. It
is also the highest load to which a link may be filled when moving TE
tunnels that still have a valid path.

Link overload:
: The condition in which the link load of a link exceeds its trigger
threshold.

Mandatory placement:
: A path computation for a TE tunnel that has no valid path and
therefore must be placed, for example a new TE tunnel, a TE tunnel
whose path has failed, or a TE tunnel whose path violates its
constraints. All other path changes move a TE tunnel that already
has a valid path; they are discretionary and are checked against the
target threshold (see {{path-computation}}).

Least congested path:
: Among the paths that satisfy the TE constraints of a TE tunnel, the
path that the controller considers to be the least congested. It is
used only when no path that satisfies those constraints keeps every
link at or below the applicable threshold. How congestion is
compared between paths, and how the optimization objective is used
among equally congested paths, is left to the implementation.

Optimization policy:
: The operator policy that determines which TE tunnels are included
in TDCO and which are excluded from it (see {{policy}}).


# The Controller {#controller}

In TDCO, the controller is a centralized entity that computes and
changes the paths of the TE tunnels under its control. The
controller:

* Maintains a network-wide view that combines the TE topology and link
  capacities, the state and paths of the TE tunnels, the bandwidth
  accounting for each link and the measured traffic.

* Decides on its own initiative when a TE tunnel is to be placed or
  moved, and onto which path, using the proactive and
  reactive components.

* Enacts those decisions by changing the paths of TE tunnels at their
  headends.

* Observes the result and makes further adjustments as needed.

Because the network-wide state and the decisions are held in one
place, the controller can coordinate the moves that it makes, account for TE
tunnels whose path changes are still in flight, and apply the
automated safety controls described in {{safety}}.

## Enacting Path Changes {#enactment}

TDCO does not depend on a particular mechanism for changing the path
of a TE tunnel. The controller needs a mechanism that lets it set the
path of a TE tunnel at its headend, learn the current state and path
of that TE tunnel, and learn when a path change is complete. Existing
mechanisms include:

PCEP:
: A stateful PCE {{RFC8231}} updates the paths of LSPs that are
delegated to it, and can create LSPs itself {{RFC8281}}. Segment
Routing paths are carried using the extensions for SR-MPLS
{{RFC8664}} and SRv6 {{RFC9603}}. PCEP state reports tell the
controller the current path and operational state of each LSP.

BGP SR Policy:
: The controller originates SR Policy candidate paths and advertises
them to headends in BGP {{RFC9830}}. This mechanism applies to SR
Policies only. Changing the path of a TE tunnel means advertising an
updated candidate path. BGP SR Policy does not itself report the
state of the SR Policy at the headend. The controller learns that
state by other means, for example from the BGP-LS advertisement of
SR Policies {{RFC9857}}.

NETCONF/YANG:
: The controller configures TE tunnel paths on the headend through
NETCONF {{RFC6241}}, using YANG {{RFC7950}} data models for TE
tunnels or SR Policies, and reads the operational state through the
same data models.

A single controller can use different mechanisms for different TE
tunnels or headends. The procedures in this document apply in the same
way whichever mechanism is used. The mechanisms differ in how quickly
a path change takes effect and in how the controller learns that it
has taken effect. These differences affect the in-flight accounting
described in {{in-flight}}.

When this document refers to a TE tunnel that is "controlled by" the
controller, it means a TE tunnel whose path the controller is
permitted to change through one of these mechanisms.

## Optimization Policy {#policy}

An optimization policy configured by the operator determines which
controlled TE tunnels are included in TDCO and which are excluded
from it. An excluded TE tunnel is not moved by the reactive component
and is not moved by TDCO back toward its preferred path. It still
receives a path when it needs a mandatory placement.

The demand of an excluded TE tunnel remains part of the link load, in
the same way as the demand of TE tunnels that are not controlled by
the controller, and is taken into account when other TE tunnels are
placed or moved.


# Inputs {#inputs}

## Topology and Capacity {#topology}

The controller learns the network topology, link capacities, and TE
attributes through BGP-LS {{RFC9552}} or by participating in the IGP.

## TE Tunnel Demand {#demand}

The controller learns the actual traffic demand of each TE tunnel from
near-real-time measurements. Possible sources include:

* Streaming telemetry, for example gNMI over gRPC {{GNMI}}.

* Flow collection, for example IPFIX {{RFC7011}}.

* PCEP performance metrics sent by the headends
  {{I-D.gandhi-pce-pm}}.

Demand is tracked per TE tunnel. Because the controller uses measured demand
rather than signalled reservations or planning data, its view reflects
the traffic that the network is actually carrying.

## Link Load {#link-load}

The controller learns the load on each link from near-real-time
measurements. Possible sources include:

* Streaming telemetry of interface counters, for example gNMI over
  gRPC {{GNMI}}.

* The Unidirectional Utilized Bandwidth advertised in the IGP TE
  metric extensions {{RFC7471}} {{RFC8570}}.

* The Unidirectional Utilized Bandwidth carried in BGP-LS {{RFC8571}}.

Each of these sources measures all traffic on the link, including dark traffic. Dark traffic
is therefore accounted for in link load, even though it is not steered.

The sources differ in the detail they provide. Streaming telemetry can
break the load down by traffic type, for example native IPv4, native
IPv6, and MPLS traffic, although the breakdowns available differ
between network device platforms. The IGP and BGP-LS advertise a
single utilization value for each direction, and how current that
value is depends on the advertisement interval and on any thresholds
that govern re-advertisement. The sources can be used alone or
together.

The controller combines the measured interface traffic and the measured TE
tunnel demand to determine the link load. The exact formula is
implementation-specific.

The link load is used for detecting overload and for placement
decisions.

## In-Flight Path Changes {#in-flight}

Telemetry lags behind path changes. After a TE tunnel is moved, for
example by make-before-break, it can take one or more measurement
intervals before the measurements show its traffic leaving the old
path and arriving on the new one.

This needs to be accounted for in both directions:

* On the links of the old path, the TE tunnel's demand is treated as
  already removed. Otherwise an overloaded link continues to appear
  overloaded, and the controller moves more TE tunnels than are needed.

* On the links of the new path, the TE tunnel's demand is treated as
  already added. Otherwise the capacity taken by the move still appears
  free, and the controller can give it to another TE tunnel.

The controller applies these adjustments until the move is complete
and the measurements reflect it. How completion is determined, and how
the adjustments are made, are implementation-specific.

Responsiveness is bounded by the telemetry or collection interval and
by any holding timers that apply before a threshold crossing is acted
upon.


# Proactive Component: Avoiding Busy Links {#proactive}

The controller needs to perform a mandatory placement for numerous reasons,
including, but not limited to, the creation of a new TE tunnel, a link
failure, or a constraint violation.

When the controller performs a mandatory placement for a TE tunnel whose
demand is known, it prefers any path on which the TE tunnel's demand
does not push the link load of any link past the trigger
threshold. This is a preference rather than a constraint: if no path
avoids the trigger threshold, the TE tunnel is placed on the least
congested path that satisfies its other TE constraints, rather than
being rejected, and any resulting overload is handled
by the reactive component. Traffic is therefore steered away
from links that would become overloaded, not only removed from links that
already are.

A TE tunnel that was moved off a link by the reactive component may
revert to its preferred path through normal re-optimization, only when
every link of that path stays at or below the target threshold after the
TE tunnel's demand is added.


# Reactive Component: Overload Alleviation {#reactive}

When the measured load on a link exceeds the trigger threshold, and
this is sustained as described in {{safety}}, the controller calculates
the excess of the link load above the target threshold. It
then selects TE tunnels, from those that the optimization policy
includes in TDCO (see {{policy}}), whose combined demand on the link
covers the excess, and re-routes only those TE tunnels.
If a selected TE tunnel cannot be placed on an
alternate path that satisfies its constraints and that does not fill
any link past the target threshold, it is placed on the least
congested path that satisfies its constraints, as described in
{{path-computation}}, and the controller can select other TE tunnels to
cover the remaining excess.

One or many links can be overloaded at the same time, for example when
a failure shifts traffic onto several links. The controller takes all
of the overloaded links into consideration together when selecting the
TE tunnels to move, rather than relieving each link in isolation.

The controller aims to move no more TE tunnels than are needed,
because each move adds signalling load and churn and carries a risk
of traffic disruption. Choosing the
smallest set of TE tunnels that covers the excess on every overloaded
link, given that each TE tunnel needs a feasible alternate path, is a
combinatorial problem, and
the fewest TE tunnels are not always the best choice: the largest TE
tunnels are the hardest to place and the most disruptive to move. The
choice of which TE tunnels are best suited to be moved is therefore
left to the implementation.

Each overloaded link is continuously re-evaluated with new telemetry as
traffic moves, to ensure that the target threshold is met. During
re-evaluation, the controller takes into account TE tunnels whose path
changes are still in flight, so that it does not move
more TE tunnels than are needed.

The overload of a link can be unresolvable, for example because it is
caused mainly by dark traffic, by TE tunnels that are not controlled
by the controller, or by TE tunnels that the optimization policy
excludes from TDCO, because no movable TE tunnel has a feasible
alternate path, or because the demand of a single TE tunnel exceeds
the headroom of every feasible alternate path. TE tunnels are moved
as a whole and are not split across paths.

# Path Computation {#path-computation}

Alternate paths are computed with each TE tunnel's full TE constraint
set, for example its metric bounds, affinities, diversity
requirements, and bidirectional co-routing constraints, and with its
optimization objective, for example lowest latency. Among the paths
that satisfy the constraints and keep every link at or below the
applicable threshold, the controller selects the path that best meets
the optimization objective. The least congested path is used only
when no such path exists.

Path computation is bandwidth-aware: it uses the measured
demand of the TE tunnel being placed or moved and the link
load of candidate links, checked against the trigger and
target thresholds. As a result, traffic
moved off one link is kept off links that it would overload whenever
another path allows it, and the TE tunnel's TE requirements continue
to be met.

The trigger threshold is used to detect an overloaded link and to
choose a path for a mandatory placement. A mandatory placement avoids
pushing any link past the trigger threshold when it can. When it
cannot, the TE tunnel is still placed, on the least congested path,
and any resulting overload is handled by the reactive component.

Every other path change, whether made by the reactive component, by a
TE tunnel reverting to its preferred path, or by re-optimization, uses
the target threshold. The TE tunnel is moved onto a path on which
every link stays at or below the target threshold after its demand is
added. If no such path exists, the TE tunnel stays on its current path
as long as every link of that path is at or below the trigger
threshold. Otherwise, it is moved onto the least congested path that
satisfies its other TE constraints, and any resulting overload is
handled by the reactive component. When the target is set below the
trigger, the gap between them leaves headroom, so small increases in
load do not cause a new overload and TE tunnels are not moved back and
forth.

The adjustment is local and incremental. Only the TE tunnels required
to relieve the overloaded link are moved. The reactive component does
not perform network-wide re-optimization. Periodic or operator-requested
re-optimization is a separate process, and the moves that it makes are
checked against the target threshold.

# Bandwidth Management {#bandwidth}

The controller manages bandwidth for both RSVP-TE signalled TE tunnels
and SR Policies. It maintains the bandwidth accounting for each link
itself, rather than relying on the network. SR Policies have no per-hop
reservation state in the network, so the controller is the only place
where their bandwidth is accounted for. Because the same accounting covers both kinds of TE tunnel,
RSVP-TE signalled TE tunnels and SR Policies can share links, with
the placement of each taking into account the bandwidth used by the
other.

TDCO does not perform bandwidth admission control: bandwidth determines
where a TE tunnel is placed, not whether it is placed. A TE tunnel is
never rejected or torn down for lack of bandwidth. When no path that satisfies the TE constraints of the TE
tunnel has enough headroom for its demand, the TE tunnel is placed on
the least congested of those paths, for example the path that
minimizes the highest link load along it, and any resulting overload
is handled by the reactive component. This applies
equally to RSVP-TE signalled TE tunnels and SR Policies.

This is the centralized management of available capacity model
described in {{Section 3.2 of RFC8426}}.

The controller keeps a single link load for each link, which combines
the measurement and signalled inputs described in {{link-load}} and
{{rsvp-te}}.

## RSVP-TE Bandwidth Signalling {#rsvp-te}

RSVP-TE signalled TE tunnels may or may not signal a bandwidth
reservation. It is generally assumed that RSVP-TE signalled TE tunnels
that are controlled by the controller do not signal a bandwidth reservation
for admission, and that the controller computes and manages their
bandwidth itself, using telemetry. A signalled reservation causes each node along the
path to perform its own admission against the signalled value. That
value can differ from the measured demand that the controller uses.
Unlike the bandwidth checks made by the controller, which only
influence path selection, admission at the nodes is a hard check: it can cause a path computed by the
controller to be rejected, or can cause other LSPs to be preempted.

When RSVP-TE signalled TE tunnels do signal a bandwidth reservation,
the controller can account for the reservation by considering the maximum of
the signalled bandwidth and the bandwidth measured by telemetry as the
TE tunnel demand.

The controller also accounts for bandwidth reserved by RSVP-TE LSPs that it
does not control. Such LSPs may still be reported to the controller, or
may be unknown to it. When an LSP that the controller does not control is
known to it, for example through PCEP state reports or NETCONF/YANG
operational state from the headend, the controller learns its
reservation and path, and can measure its demand using the same
telemetry sources as for controlled TE tunnels. When an LSP is not known
to the controller, the controller learns the bandwidth reserved on each link from
the Unreserved Bandwidth advertised in the IGP TE extensions for OSPF
({{Section 2.5.8 of RFC3630}}) and IS-IS ({{Section 3.6 of RFC5305}}),
or carried as a link attribute in BGP-LS {{RFC9552}}. In both cases the
reservation is taken into consideration for the link load.

# Operational Considerations {#operational}

TDCO acts on measured traffic without operator approval of individual
actions. Its safe operation therefore depends on the network it is
deployed in, on acting only on measurements that are current,
sustained, and reflect the effect of earlier actions, and on bounding
how much it changes the network. This section describes where TDCO is
applicable, what an operator needs to configure, the safety controls
that an implementation should provide, and how operators retain
control and visibility. The specific knobs and controls are
implementation-specific.

## Applicability {#applicability}

TDCO is applicable to networks in which:

* The TE tunnels that carry a significant share of the traffic on the
  links to be protected are controlled by the controller and included
  in TDCO by the optimization policy. TDCO can only relieve overload
  by moving traffic that it controls and is permitted by policy to
  move.

* The demand of each controlled TE tunnel can be measured in near real
  time, and the load of each link can be measured.

* Headends can change the path of a TE tunnel without disrupting its
  traffic, for example by make-before-break for RSVP-TE signalled TE
  tunnels or by a candidate path update for SR Policies.

* The topology offers alternate paths that satisfy the TE constraints
  of the TE tunnels to be moved.

* A single controller is responsible for the TE tunnels whose paths
  share a set of links. Independent controllers that steer TE tunnels
  across the same links without coordination can react to the same
  overload and move more traffic than is needed.

TDCO addresses sustained overload, measured over one or more
collection intervals. It does not react to individual traffic bursts
shorter than the collection interval. It also
cannot relieve an overload that is caused mainly by dark traffic, by
TE tunnels that are not controlled by the controller, or by TE tunnels
that the optimization policy excludes from TDCO.

## Configuration {#configuration}

An operator deploying TDCO needs to choose:

Collection interval:
: The telemetry or flow-collection interval bounds how quickly an
overload is detected. Shorter intervals improve responsiveness at
the cost of additional collection load on the network devices, the
collectors, and the controller.

Thresholds:
: The trigger and target thresholds, configured globally or per link.
The trigger threshold is typically set below link capacity, so that
steering begins before traffic bursts cause loss. The gap between
the trigger and target thresholds determines how far below the
overload point a link is brought, and therefore how much headroom
remains before the link triggers again.

Holding time:
: The time, or number of consecutive measurement intervals, for which
a threshold crossing must be sustained before it is acted upon.
A longer holding time reduces sensitivity to transient
load at the cost of slower response.

Optimization policy:
: Which TE tunnels are included in TDCO and which are excluded, as
described in {{policy}}. Excluding TE tunnels reduces the traffic
that TDCO can move, and therefore the overloads that it can
relieve.

## Safety Controls {#safety}

The number of TE tunnels moved for a single overload event is
determined by the reactive component. The controls below are
independent of that per-event selection and limit TDCO's overall
behavior, including when its inputs are wrong. An implementation
should consider:

Sustained threshold crossing:
: A link is treated as overloaded only when its link load remains
above the trigger threshold for the configured holding time. A single
sample or a short traffic burst therefore does not cause TE tunnels
to be moved.

Telemetry validity:
: Optimization is suspended for the affected links or TE tunnels when
telemetry is unavailable or stale, so that steering decisions are not
made on data that no longer reflects the network. Where a TE tunnel
has a signalled bandwidth reservation (see {{rsvp-te}}), the
controller can use the signalled value as its demand while telemetry
is unavailable. Suspension applies to discretionary path changes;
mandatory placements are still performed, using the last known or
signalled demand.

In-flight accounting:
: Telemetry lags behind path changes. Until subsequent measurements
reflect the TE tunnels that have already been moved, the controller
accounts for those moves so that it does not react
again to load that is already being removed.

Re-route damping:
: The frequency of path re-routing, either per TE tunnel or globally,
can be limited to reduce network-wide churn and potential
high-frequency oscillation.

## Operator Control and Visibility {#control}

Control:
: Operators need to be able to disable TDCO, globally or for
individual links, and to include TE tunnels in TDCO or exclude them
from it through the optimization policy described in {{policy}},
for example to exclude TE tunnels that are pinned to an explicit
path for service reasons.

Visibility:
: Because individual actions are not approved by an operator, the
controller needs to report the actions that it takes, including the
overloaded link, its link load before and after the action, and the
TE tunnels that were moved and their old and new paths. The
controller also needs to report overloads that it cannot resolve, so
that the operator can act on them, for example by adding capacity.

# Security Considerations

This document uses existing protocols and does not introduce new
protocol mechanisms. The security considerations of the mechanisms
used to change TE tunnel paths apply: PCEP {{RFC5440}},
stateful PCE {{RFC8231}}, and PCE-initiated LSPs {{RFC8281}}; BGP SR
Policy {{RFC9830}}; and NETCONF {{RFC6241}}. The security
considerations of BGP-LS {{RFC9552}} and of the telemetry and
flow-collection protocols in use also apply.

Because TDCO acts on measured data without operator approval of
individual actions, the integrity of its inputs is particularly
important:

* Falsified or manipulated telemetry, flow records, or PCEP
  performance metrics could cause the controller to re-route TE tunnels
  unnecessarily, to fail to alleviate a genuine overload, or to steer
  traffic toward links chosen by an attacker. Telemetry and
  flow-collection sessions need to be authenticated and integrity
  protected, for example by running gNMI over gRPC with TLS, or by
  using the TLS or DTLS transports for IPFIX described in {{RFC7011}}.
  PCEP sessions can be protected using TLS {{RFC8253}}. The input
  plausibility control in {{safety}} limits the effect of inputs that
  are falsified but well formed.

* Telemetry collectors, and any processing pipeline between them and
  the controller, are on the controller's decision path. Compromise of a collector
  or of the pipeline has the same effect as falsified telemetry, even
  when each individual telemetry session is protected. These components
  need the same level of protection as the controller itself.

* Compromise of the trigger and target threshold configuration could
  be used to induce steering or to suppress it. Protection of the
  configuration and management interfaces of the controller is therefore
  important.

* An attacker able to inject bursty traffic might attempt to provoke
  repeated re-routing. The sustained threshold crossing and re-route
  damping controls in {{safety}} limit the rate at which TDCO
  responds, and therefore limit the churn that such traffic can cause.

* A controller already controls the paths of the TE tunnels under its
  control, whichever mechanism it uses to change them. TDCO does not
  change that, but autonomous operation increases the importance of
  protecting the controller itself and the sessions it uses to change
  paths, whether PCEP, BGP, or NETCONF.

Per-tunnel demand and flow records reveal the traffic matrix of the
network and, potentially, the traffic volumes of individual customers.
This information needs to be protected in transit, as described above,
and at rest in the controller and in the collectors, with access restricted to
authorized operators and systems.


# IANA Considerations

This document has no IANA actions.


--- back

