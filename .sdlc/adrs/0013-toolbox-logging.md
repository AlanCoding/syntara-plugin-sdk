# ADR 0013: Terminal structured Toolbox logs

- Status: Accepted
- Date: 2026-11-13
- Amends: [ADR 0012](0012-container-toolbox.md)

## Decision

The invocation-scoped Toolbox exposes `log(level, message, attributes=None)`,
with an optional keyword-only `event_name`. The SDK assigns the timestamp and
records valid events in call order. The language-neutral protobuf defines the
version 1 event and batch, including severity, JSON attribute and size bounds.
Invalid and excess records are dropped and counted; logging never changes the
step outcome. The collector and wire batch do not inspect or scrub records.
During direct local execution, [syntara-plugin](../../docs/glossary.md#syntara-plugin) applies its existing local diagnostic
scrubber when rendering records to stderr.

The [runtime-server](../../docs/glossary.md#runtime-server) sends at most one nonempty or drop-count-bearing batch
on Execute immediately before the terminal result or handled failure, without
waiting for acknowledgment. Cancellation flushes records already collected when
[runtime-server](../../docs/glossary.md#runtime-server) can send a terminal failure; disconnect, a crash, or a stuck process
may prevent delivery. The [syntara-plugin](../../docs/glossary.md#syntara-plugin) process renders the same events on stderr.
Credential reads remain exclusively on their separate callback surface.

## Consequences

This adds a typed message from [runtime-server](../../docs/glossary.md#runtime-server) under the pre-release `v1alpha1` ABI. Existing
peers that treat unknown messages as protocol violations must be upgraded; the
Health response cannot detect this difference. The [ep-worker](../../docs/glossary.md#ep-worker) must accept
the notification and own correlation and forwarding to its configured
collector. This decision adds neither live streaming nor an SDK exporter.
Plugin authors must keep secrets out of logs.
