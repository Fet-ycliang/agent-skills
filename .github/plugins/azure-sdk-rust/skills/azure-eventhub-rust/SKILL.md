---
name: azure-eventhub-rust
description: |
  Azure Event Hubs library for Rust. Use for sending and receiving events, streaming data ingestion, and batch processing.
  Triggers: "event hubs rust", "ProducerClient rust", "ConsumerClient rust", "send event rust", "streaming rust".
license: MIT
metadata:
  author: Microsoft
  package: azure_messaging_eventhubs
---

# Azure Event Hubs library for Rust

Client library for Azure Event Hubs — send and receive events for streaming data ingestion.

Use this skill when:

- An app needs to send events to Azure Event Hubs from Rust
- You need to receive and process events from partitions
- You need batch sending for throughput optimization
- You need to control consumer start position

> **IMPORTANT:** Only use the official `azure_messaging_eventhubs` crate published by the [azure-sdk](https://crates.io/users/azure-sdk) crates.io user. Do NOT use unofficial or community crates. Official crates use underscores in names and none have version 0.21.0.

## Installation

```sh
cargo add azure_messaging_eventhubs azure_identity tokio futures
```

> **Do not** add `azure_core` directly to `Cargo.toml`. It is re-exported by `azure_messaging_eventhubs`.

## Environment Variables

```bash
EVENTHUBS_HOST=<namespace>.servicebus.windows.net
EVENTHUB_NAME=<eventhub-name>
```

## Key Concepts

| Concept       | Description                                          |
| ------------- | ---------------------------------------------------- |
| **Namespace** | Container for one or more Event Hubs                 |
| **Event Hub** | Stream of events, partitioned for parallel reads     |
| **Partition** | Ordered, append-only sequence of events              |
| **Producer**  | Sends events via `ProducerClient`                    |
| **Consumer**  | Receives events from partitions via `ConsumerClient` |

## Authentication

```rust
use azure_identity::DeveloperToolsCredential;
use azure_messaging_eventhubs::ProducerClient;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    // Local dev: DeveloperToolsCredential. Production: use ManagedIdentityCredential.
    let credential = DeveloperToolsCredential::new(None)?;

    let producer = ProducerClient::builder()
        .open("<EVENTHUBS_HOST>", "<EVENTHUB_NAME>", credential.clone())
        .await?;

    Ok(())
}
```

## Core Workflow

### Send Events

```rust
// Send a single event
producer.send_event(vec![1, 2, 3, 4], None).await?;
```

### Send Batch

```rust
let batch = producer.create_batch(None).await?;
assert_eq!(batch.len(), 0);
assert!(batch.try_add_event_data(vec![1, 2, 3, 4], None)?);

producer.send_batch(batch, None).await?;
```

### Receive Events

```rust
use azure_identity::DeveloperToolsCredential;
use azure_messaging_eventhubs::ConsumerClient;

async fn open_consumer() -> Result<ConsumerClient, Box<dyn std::error::Error>> {
    // Local dev: DeveloperToolsCredential. Production: use ManagedIdentityCredential.
    let credential = DeveloperToolsCredential::new(None)?;

    let consumer = ConsumerClient::builder()
        .open("<EVENTHUBS_HOST>", "<EVENTHUB_NAME>", credential.clone())
        .await?;

    Ok(consumer)
}
```

### Receive from Partition

```rust
use futures::stream::StreamExt;
use azure_messaging_eventhubs::{
    ConsumerClient, OpenReceiverOptions, StartLocation, StartPosition,
};

async fn receive_events(client: &ConsumerClient) -> Result<(), Box<dyn std::error::Error>> {
    let message_receiver = client
        .open_receiver_on_partition(
            "0".to_string(),
            Some(OpenReceiverOptions {
                start_position: Some(StartPosition {
                    location: StartLocation::Earliest,
                    ..Default::default()
                }),
                ..Default::default()
            }),
        )
        .await?;

    let mut event_stream = message_receiver.stream_events();

    while let Some(event_result) = event_stream.next().await {
        match event_result {
            Ok(event) => println!("Received: {:?}", event),
            Err(err) => eprintln!("Error: {:?}", err),
        }
    }

    Ok(())
}
```

## Best Practices

1. **Use `DeveloperToolsCredential`** for local dev, **`ManagedIdentityCredential`** for production
2. **Never hardcode credentials** — use environment variables or managed identity
3. **Use batching** — `create_batch` + `send_batch` for throughput
4. **Handle errors per event** — match on `Ok`/`Err` in the event stream
5. **Specify start position** — use `StartLocation::Earliest` or `StartLocation::Latest`

## Reference Links

| Resource      | Link                                               |
| ------------- | -------------------------------------------------- |
| API Reference | https://docs.rs/azure_messaging_eventhubs          |
| crates.io     | https://crates.io/crates/azure_messaging_eventhubs |
