# ADR 0011: Invocation-scoped credential callbacks

- Status: Accepted
- Date: 2026-11-09
- Supersedes: [ADR 0009](0009-local-credential-delivery.md)

## Decision

The `syntara.container/v1alpha1` ABI uses a bidirectional Execute stream.
The first message from [ep-worker](../../docs/glossary.md#ep-worker) or [syntara-plugin](../../docs/glossary.md#syntara-plugin) starts the invocation.
The [runtime-server](../../docs/glossary.md#runtime-server) sends a credential
request with its invocation ID, a fresh request ID, and a logical name: the
CredentialRecipe ID declared by the installed step's integration. The supplied
credential handler resolves that name within the invocation's authority and responds with the
same IDs and either a JSON object or a safe error category. Credential IDs,
references, paths, and provider error details do not cross this boundary.
Sensitive responses are separate from business events. The production channel
must authenticate both peers and provide confidentiality, using mTLS.

The generic `ContainerWorkloadRequest` no longer carries file-based credential
leases. Its existing opaque work-item credential handles remain platform-side;
the callback protocol does not send them to [runtime-server](../../docs/glossary.md#runtime-server) or prescribe their
mapping to requested names.

The [runtime-server](../../docs/glossary.md#runtime-server) accepts at most 16 concurrent reads per invocation, bounds
documents to 1 MiB, and limits each read to 30 seconds and the invocation
deadline. Scope closure, cancellation, deadline, disconnect, and shutdown
revoke pending reads. Late, duplicate, and mismatched responses are discarded.
Successful reads are recorded before the step can observe them so the existing
bounded output scanner can reject leaks. An access failure never includes
provider details. Neither [ep-worker](../../docs/glossary.md#ep-worker) nor [syntara-plugin](../../docs/glossary.md#syntara-plugin) may log sensitive messages. The callback grants
access only for the invocation; it is not a sandbox for arbitrary Python code.

In [syntara-plugin-contracts](../../docs/glossary.md#syntara-plugin-contracts), `ContainerClient` invokes the supplied resolver on the thread
consuming the Execute stream. Each resolver receives its declared name,
invocation ID, cancellation signal, and monotonic deadline through a
`CredentialRequestContext`; it should use the remaining time to bound network
and operation timeouts and observe cancellation while waiting. The `ContainerClient` implementation
does not own a resolver executor or queue. The [ep-worker](../../docs/glossary.md#ep-worker) or [syntara-plugin](../../docs/glossary.md#syntara-plugin) process
owns resolver concurrency and queue limits. Python cannot preempt arbitrary
resolver code that ignores cancellation or its deadline. The synchronous
`ContainerRuntimeService` implementation owns at most 16 reader/step worker pairs across the process. It rejects
additional invocations with a retryable `RATE_LIMITED` failure and retains a
capacity slot while either worker remains blocked. Shutdown signals
cancellation and waits briefly for these owned workers, but does not attempt
unsafe thread termination.

The async `AsyncContainerRuntimeService` used by [runtime-server](../../docs/glossary.md#runtime-server) owns a
per-service pool of eight daemon step workers by default. A running handler
retains its service capacity slot until it returns, even after cancellation or
a timeout. Shutdown signals cancellation and rejects new work, but daemon
workers do not prevent the Python process from exiting.

Closing the send stream from [ep-worker](../../docs/glossary.md#ep-worker) or [syntara-plugin](../../docs/glossary.md#syntara-plugin) revokes credential reads but leaves the
invocation running so it can return a terminal event. A malformed follow-up
message processed before the terminal event is committed cancels the
invocation and returns a cancellation failure; later messages may be ignored.
The [runtime-server](../../docs/glossary.md#runtime-server) shutdown allows active invocations to finish during the configured grace
period; remaining reads and invocations are revoked when that period ends.

The [syntara-plugin](../../docs/glossary.md#syntara-plugin) command serves supplied credentials from memory over a private socket in
source mode or an authenticated channel to its managed image. It writes no
delivery files and keeps rejecting credentials for independently managed
[runtime-server](../../docs/glossary.md#runtime-server) instances.

## Consequences

This is an in-place breaking protocol change under the existing `v1alpha1`
identifier. Earlier `v1alpha1` peers are incompatible; the Health ABI value
does not distinguish them. Old images must be rebuilt and the Execution Plane
must implement the matching stream and per-attempt name-to-credential mapping.
The SDK does not call the credential authority or perform that mapping itself.
Callers using `ContainerClient` provide a context-aware resolver and own its
concurrency policy. The [ep-worker](../../docs/glossary.md#ep-worker) and [syntara-plugin](../../docs/glossary.md#syntara-plugin) processes should apply their
invocation cancellation and deadline to downstream resolver work.
