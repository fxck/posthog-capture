# posthog-capture (Zerops fork)

A minimal fork of [PostHog/posthog](https://github.com/PostHog/posthog)'s `rust/` subtree, carrying a small **SASL Kafka patch** so [capture-rs](https://github.com/PostHog/posthog/tree/master/rust/capture) can run against managed-Kafka offerings that require SASL_PLAINTEXT or SASL_SSL — including [Zerops](https://zerops.io), MSK, Confluent Cloud, Aiven, and similar.

This fork exists to unblock [`fxck/recipe-posthog`](https://github.com/fxck/recipe-posthog)'s `capture` service.

## What's patched

Five files. Four new optional `KafkaConfig` fields propagated to the rdkafka `ClientConfig` in both the shared `common/kafka` crate and capture's own `sinks::kafka` sink (capture has its own parallel `KafkaConfig` struct and producer setup, so it needs the same patch).

| File | Patch |
|---|---|
| [`rust/common/kafka/src/config.rs`](rust/common/kafka/src/config.rs) | Add `kafka_security_protocol`, `kafka_sasl_mechanism`, `kafka_sasl_username`, `kafka_sasl_password` (all `Option<String>`) to `KafkaConfig`. |
| [`rust/common/kafka/src/kafka_producer.rs`](rust/common/kafka/src/kafka_producer.rs) | When set, write them to the rdkafka `ClientConfig`. Overrides `kafka_tls`'s `security.protocol` if both are configured. |
| [`rust/common/kafka/src/kafka_consumer.rs`](rust/common/kafka/src/kafka_consumer.rs) | Same as above for the consumer side. |
| [`rust/capture/src/config.rs`](rust/capture/src/config.rs) | Same four fields added to capture's own `KafkaConfig` struct. |
| [`rust/capture/src/sinks/kafka.rs`](rust/capture/src/sinks/kafka.rs) | Propagate to the rdkafka `ClientConfig` in capture's `KafkaSink::new`. |

Defaults preserve existing behavior: leaving the new fields unset gives plain TCP if `kafka_tls=false`, TLS if `kafka_tls=true` — both pre-patch paths.

To opt in, set the env vars:
```
KAFKA_SECURITY_PROTOCOL=SASL_PLAINTEXT   # or SASL_SSL
KAFKA_SASL_MECHANISM=PLAIN               # or SCRAM-SHA-256, etc.
KAFKA_SASL_USERNAME=<user>
KAFKA_SASL_PASSWORD=<password>
```

## Why this exists (the original blocker)

PostHog Cloud terminates SASL upstream of capture-rs (private VPC, plaintext between sidecars and the broker), so SASL fields never had to exist in the binary. Anyone trying to run capture-rs against an off-the-shelf managed Kafka — where the broker enforces SASL on every listener — finds the binary builds fine but disconnects immediately after the TCP handshake:

```
librdkafka: FAIL kafka:9092/bootstrap:
  Disconnected: connection closed by peer: receive 0 after POLLIN
librdkafka: Global error: AllBrokersDown
```

That's the broker rejecting an unauthenticated handshake. Pre-patch, there was no env var or struct field that would make rdkafka send SASL credentials — `KafkaConfig` only exposed `kafka_tls: bool`, which only flips the protocol to `ssl`.

## Maintenance

- **Pinned upstream SHA**: [`.upstream-sha`](.upstream-sha) records the PostHog commit this fork was rebased from. Rebases happen as needed when the patched files churn upstream.

## Building

The capture binary builds with cargo from `rust/`:

```bash
cd rust
cargo build --release -p capture
```

System deps: `build-essential`, `cmake`, `pkg-config`, `libssl-dev`, `libsasl2-dev` (Ubuntu 22.04 / 24.04 names). Build artifact at `target/release/capture` (~270 MB).

## Usage

The recipe at [fxck/recipe-posthog](https://github.com/fxck/recipe-posthog) uses this fork as `buildFromGit` for its `capture` service. The fork carries its own [`zerops.yml`](./zerops.yml) with the `capture` setup block — Zerops clones this repo and finds the build/run config here. See recipe-posthog for the full Zerops deploy story.

For other Kafka deployments, the four env vars above are the entire surface area — everything else is upstream PostHog behavior.

## How this fork compares to PostHog Cloud's capture

PostHog Cloud runs the **same upstream binary**, unpatched — because Cloud's capture pods sit inside a private VPC with Kafka brokers reachable over plaintext (SASL is terminated upstream). The fork closes the gap for any deployment where the broker itself speaks SASL on the wire, which is every off-the-shelf managed Kafka. Feature parity with Cloud's capture is otherwise 1:1: same `CAPTURE_MODE` values (`events` / `recordings` / `flags` / `ai`), same endpoints, same rdkafka client, same throughput characteristics.

For a full comparison of the broader self-host deploy (this fork + the rest of PostHog) against the official Helm chart and PostHog Cloud, see [`recipe-posthog`'s Honest comparison section](https://github.com/fxck/recipe-posthog#honest-comparison).

## License

MIT — inherits PostHog/posthog's LICENSE. Patches authored as derivative work under the same license.
