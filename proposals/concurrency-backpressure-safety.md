# Proposal: Concurrency and Backpressure Safety for TRACE Evidence Producers

**Status:** discussion draft; non-normative

This document proposes a platform-neutral safety model for TRACE evidence production under concurrent authorized activity. It does not change the TRACE wire format or conformance requirements.

## Motivation

TRACE evidence can be individually valid while the system producing, publishing, indexing, or verifying that evidence becomes unstable under concurrency.

The failure mode does not require a malicious participant. Multiple legitimate producers, collectors, verifiers, retries, or downstream consumers can create a positive feedback loop:

```text
concurrent evidence production
        -> write/update contention
        -> retry or duplicate delivery
        -> verification/indexing fan-out
        -> more queued work
        -> more retries
```

Useful evidence throughput may remain flat or fall while CPU, bandwidth, storage mutations, and verifier work continue to grow.

A related correctness risk exists when transport or persistence fails after the evidenced workload has already executed. Repeating the workload merely to regenerate or republish evidence is materially different from retrying evidence delivery, especially when the workload carries significant authority or external side effects.

## Proposed principle

> Evidence-delivery retries should not implicitly authorize repetition of the evidenced workload.

Where an action has already occurred, an implementation should be able to retain its evidence durably and retry publication independently.

Concurrent producers should likewise be able to identify duplicate, stale, conflicting, or superseded work without relying on unbounded retry loops.

## Candidate implementation guidance

The following is intentionally implementation-neutral. It could remain informative guidance, become OPTIONAL conformance behavior, or be split between specification guidance and conformance tests after review.

### 1. Stable event identity and idempotency

An evidence event should have an identity that survives transport retries. A collector should be able to recognize duplicate delivery without causing the underlying workload to execute again.

### 2. Execution/publication separation

Where practical, evidence should be durably retained before external publication. Publication retries should operate on retained evidence rather than implicitly repeating the action that produced it.

### 3. Predecessor or epoch binding for shared mutable state

When evidence publication mutates shared state, a producer should be able to bind its proposed transition to the state or epoch it observed. Stale writers should be rejected or reconciled instead of silently overwriting newer state.

This requirement is conceptual; implementations may use sequence numbers, epochs, version identifiers, object hashes, transactional compare-and-swap, or equivalent mechanisms.

### 4. Bounded retries and backoff

Retry behavior should have a finite budget and bounded cadence. Contention should not produce immediate fetch/recompute/retry loops. Implementations should be able to transition to a protected, deferred, or quarantined state when publication cannot safely progress.

### 5. Supersession and cancellation

Obsolete queued evidence work should be identifiable and cancellable while retaining enough lineage to explain what superseded it. A newer valid event should not require every obsolete verification or publication task to finish first.

### 6. Backpressure and circuit state

When evidence production exceeds safe publication or verification throughput, an implementation should stop adding expensive work rather than merely transferring pressure to another subsystem such as CPU, bandwidth, storage, or a downstream verifier.

Backpressure should therefore reduce work creation, not only delay writes.

### 7. Conflict preservation

Different evidence claiming the same stable event identity should not be silently overwritten. The implementation should preserve or surface the conflict for investigation.

## Optional metadata concepts for later discussion

This proposal does **not** recommend a wire-format change at this stage. If concrete test vectors demonstrate that additional metadata is necessary, candidate concepts could include equivalents of:

- stable event or operation identity;
- producer/stream epoch;
- predecessor or state binding;
- delivery attempt number;
- supersession relationship;
- protected/backpressure state.

The names above are illustrative. The desired safety properties may be achievable using existing TRACE constructs or implementation-local metadata.

## Suggested conformance and interoperability vectors

A future test suite could model the following conditions without prescribing a transport:

1. two authorized producers race from the same predecessor;
2. transport fails after workload execution but before evidence publication;
3. a producer restarts with an unpublished durable evidence record;
4. duplicate evidence delivery occurs;
5. publication repeatedly conflicts with newer shared state;
6. verifier/indexer throughput falls below producer throughput;
7. a newer event supersedes queued work for an older event;
8. conflicting bytes or claims appear under the same event identity.

Expected safety properties could include:

- no silent overwrite;
- no implicit workload re-execution;
- bounded retries;
- durable preservation of already-produced evidence;
- graceful backpressure;
- explicit conflict/supersession handling;
- deterministic recovery after restart.

## Scope boundary

This proposal is about evidence-pipeline correctness and availability under legitimate concurrency. It is not a denial-of-service specification and does not define repository-, database-, or cloud-provider-specific limits.

It also does not propose that pressure telemetry become an authority signal. Operational pressure may justify deferring work, but it should not by itself change the truth value, provenance, or authorization represented by TRACE evidence.

## Questions for project review

1. Should these properties remain informative implementation guidance, become OPTIONAL conformance behavior, or be divided between the two?
2. Does TRACE already provide sufficient event identity for retry/deduplication binding, or is explicit guidance needed?
3. Should backpressure/protected state ever be represented in TRACE evidence, or remain strictly implementation-local?
4. Which failure vectors belong in the core conformance suite versus implementation guidance?
5. Can all desired properties be expressed without changing the TRACE wire format?

## Change strategy

The preferred first iteration is additive and implementation-neutral. Any normative or wire-format proposal should be derived only after concrete test vectors demonstrate a portability or interoperability requirement and should follow the project's normal review process.
