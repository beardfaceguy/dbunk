# dbunk product spec

A protocol and schema for inter-agent fact-checking.

## Problem

AI research agents produce confident findings, and confident mistakes look the same. A reader cannot tell which statements were observed, which were inferred, and which were never checked. Reviews that read only the text check whether it hangs together, not whether it is true.

A manual review of one real 221-claim agent investigation packet, with evidence links, found the arithmetic and most source claims correct. It also found an understated credential consumer, a likely mis-attribution of usage between applications, an uninventoried configuration field and a wrong citation. Each would have shaped decisions. Finding them required reopening the sources independently, and many claims could not be checked because their evidence never reached the reviewer. The investigation's 30-day usage export showed 20 distinct model identifiers under one API-key label, and the investigation was evaluating a move of some workloads to a self-hosted model. The packet itself is private and is not published.

## v1 scope

1. A framework-neutral protocol document (`PROTOCOL.md`) for inter-agent fact-checking, layered on rubber_ducky's review protocol v1.3.
2. JSON Schemas (`schema/`) for claim packets, evidence-bundle manifests, reviewer responses with claim references and verification records, and researcher rebuttals.
3. Worked examples (`examples/`), including a sanitized multi-class packet and a full negotiation that replays through rubber_ducky's lifecycle.

## Deferred

- An MCP server implementing the protocol.
- dbunk's built-in reviewer agent (model loop, tool and path gates, sandboxed computation).
- A shared negotiation library extracted from rubber_ducky.
- Storage and transport of evidence bundles.

## Open product choices

- The license for dbunk and for rubber_ducky, which dbunk references.
- The claim-ID format, within the reserved-prefix rule.

## Success measure

A real investigation packet of at least 200 claims, rewritten in dbunk format and reviewed under the protocol, surfaces at least the significant findings an expert manual review found.

## Decisions

Proposed ADRs are in `docs/adr/`. Each is reviewed by an independent agent through rubber_ducky's ADR review mode before the decider accepts or rejects it.
