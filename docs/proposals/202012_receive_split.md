---
title: Thanos Routing Receive and Ingesting Receive
type: proposal
menu: proposals
status: accepted
---

### Related Tickets

* Hashring improvements: https://github.com/thanos-io/thanos/issues/3141
* Previous proposal implementations: https://github.com/thanos-io/thanos/pull/3845, https://github.com/thanos-io/thanos/pull/3580
* Distribution idea: https://github.com/thanos-io/thanos/pull/2675

### Summary

This document describes the motivation and design of running Receive in a stateless mode that does not have capabilities to store samples, it only routes remote write
to further receives based on hashring. This allows setting optional deployment model were only Routing Receives are using hashring files and does the routing and replication. That allows ingesting Receivers to not handle any routing or hashring, only receiving multi tenant writes.

### Motivation

[@squat](https://github.com/squat):

> Currently, any change to the hashring configuration file will trigger all Thanos Receive nodes to flush their multi-TSDBs, causing them to enter an unready state until the flush is complete. This unavailability during a flush allows for a clear state transition, however it can result in downtimes on the order of five minutes for every configuration change. Moreover, during configuration changes, the hashring goes through an even longer period of partial unreadiness, where some nodes begin and finish flushing before and after others. During this partial unreadiness, the hashring can expect high internal request failure rates, which cause clients to retry their requests, resulting in even higher load. Therefore, when the hashring configuration is changed due to automatic horizontal scaling of a set of Thanos Receivers, the system can expect higher than normal resource utilization, which can create a positive feedback loop that continuously scales the hashring.


### Goals

* Be able to run component that just does remote write forwarding part of Receive and can be part of hashring.
* Be able to run Receive without hashring functionalities and awareness.

### Proposal

We propose allowing to run Thanos Receive in a mode that only forwards/replicates remote write (distributor mode). You can enable that mode by simply
not specifying:

```yaml
  --receive.local-endpoint=RECEIVE.LOCAL-ENDPOINT
                                 Endpoint of local receive node. Used to
                                 identify the local node in the hashring
                                 configuration.
```

We can call this mode a "Routing Receive". Similarly, we can skip specify any hashring to Thanos Receive (`--receive.hashrings-file=<path>`), explicitly purposing it only for ingesting. We can call this mode "Ingesting Receive".

User can also mix all of those two modes for various federated hashrings etc. So instead of what we had before:

![Before](https://docs.google.com/drawings/d/e/2PACX-1vTfko27YB_3ab7ZL8ODNG5uCcrpqKxhmqaz3lW-yhGN3_oNxkTrqXmwwlcZjaWf3cGgAJIM4CMwwkEV/pub?w=960&h=720)

We have:

![After](https://docs.google.com/drawings/d/e/2PACX-1vTVrtCGjR4iMbrU7Kj6QAn1a1m4fr-kvoQVDAK4lzQ_wWfXfpLLEE9HB948-WHI5ZG6s1iGWt51R593/pub?w=960&h=720)

This allows us to (optionally) model deployment in a way that avoid expensive re-configuration of the stateful ingesting Receives after the hashring configuration file has changed.

In comparison to previous proposal (as mentioned in [alternatives](#previous-proposal-separate-receive-route-command) we have big adventages:

1. We can reduce number of components in Thanos system, we can reuse similar component flags and documentation. Users has to learn about one less command and in result Thanos design is much more approachable. Less components mean less maintainance, code and other implicit duties: Separate changelogs, issue confusions, boilerplates, etc.
2. Allow consistent pattern with Query. We don't have separate StorAPI component for proxying, we have that baked into Querier. This has been proven to be flexible and understandable, so I would like to propose similar pattern in Receive.
3. This is more future proof for potential advanced cases like chain of routers -> receives -> routers -> receives for federated writes.

### Open Questions

* Buffering the incoming requests.

### Plan

* Receive without `--receive.hashrings` does not forward or replicate requests, it routes straight to multi-tsdb.
* Receive without ` --receive.local-endpoint` will assume that no storage is needed, so will skip creating any resources for multi TSDB.
* Add changes to the documentation (it's simplistic now). Mention two modes.


(Inherited from previous proposal, no idea what it is about)
### Alternative Solutions

#### Previous Proposal: Separate receive-route command

1. Split the receive component into receive-route and receive (and ensure ease of resharding events).
1. Evaluate any effects on performance by simulating scenarios and collecting and analyzing metrics.
1. Use consistent hashing to avoid reshuffling time series after resharding events. The exact consistent hashing mechanism to be used needs some further research.
1. Migration: We document how the new architecture can be set up to have the same general deployment of the old architecture. (We run router and receiver on the same node).

*Pros:*

1. Splitting functionality can be used to make resharding events easier.
2. Right now the receiver handles ingestion, routing and writing, thus leading to too many responsibilities in a single process.
   This also makes the receiver more difficult to operate and understand for Thanos users.
   Splitting the receiver component into two different components could potentially have the following benefits:
   1. Resharding events become faster and cause no downtime in ingestion.
   2. Deployment becomes easier to understand for Thanos users.
   3. Each component consists of less code.

*Cons:*

There is no possible way to have a single-process receiver. The user must have a router + a receiver running.

#### Flag for current Receive --receive-route

Idea would be similar same as in [Proposal](#Proposal), but there will be explicit flag to turn off local storage capabilities.

I think we can have much more understandable logic if we simply not configure hashring for ingesting receives and not configure local hashring endpoint to notify that such receiver instance will never store anything.

