# AMB Architecture Notes

> Design notes for the Agent Message Bus.

## Overview

AMB provides pub/sub messaging between agents with pluggable backends.

## Supported Backends

| Backend | Description | Use Case |
|---------|-------------|----------|
| InMemory | No external dependencies | Development, testing |
| Redis | Via redis-py | Production (moderate scale) |
| RabbitMQ | Via pika | Guaranteed delivery |
| Kafka | Via confluent-kafka | High throughput |

## Basic Usage

```python
from amb_core import MessageBus, InMemoryBroker

# Development setup
broker = InMemoryBroker()
bus = MessageBus(adapter=broker)

# Publish
await bus.publish("agent.events", {"type": "action", "data": "..."})

# Subscribe
async def handler(message):
    print(f"Received: {message}")

await bus.subscribe("agent.events", handler)
```

## Features

### Backpressure

When consumers fall behind, publishers are slowed to prevent unbounded queue growth.

```python
broker = InMemoryBroker(
    max_queue_size=10000,
    backpressure_threshold=0.8  # Slow publishers at 80% capacity
)
```

### Durability (Optional)

For critical workloads, enable write-ahead logging:

```python
from amb_core.persistence import FileMessageStore

store = FileMessageStore("/var/amb/wal")
bus = MessageBus(adapter=broker, persistence=store)
```

### Priority Lanes

Route urgent messages separately:

```python
await bus.publish("agent.events.high", message, priority="high")
await bus.publish("agent.events.low", message, priority="low")
```

## Limitations

- Performance numbers are theoretical based on backend capabilities, not measured in this codebase
- The adapters for Redis/RabbitMQ/Kafka are designed but may need production hardening
- No benchmark suite currently exists

## Code Location

```
packages/amb/
├── src/amb_core/
│   ├── bus.py          # Main MessageBus class
│   ├── adapters/       # Backend adapters
│   └── persistence/    # WAL implementation
└── tests/
```

## Running Tests

```bash
cd packages/amb
pip install -e ".[dev]"
pytest
```
