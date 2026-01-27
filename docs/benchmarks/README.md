# Design Notes

> Design documents for Agent OS components.

## Contents

| Document | Description |
|----------|-------------|
| [`cmvk-benchmarks.md`](cmvk-benchmarks.md) | CMVK design rationale |
| [`amb-benchmarks.md`](amb-benchmarks.md) | AMB architecture notes |

## Status

This is a research project. The code is functional but:

- **No formal benchmarks exist** - performance claims need validation
- **Not production-tested** - needs hardening for real deployments
- **APIs may change** - this is experimental software

## Running the Code

```bash
# Install
git clone https://github.com/imran-siddique/agent-os.git
cd agent-os
pip install -e ".[dev]"

# Run tests
pytest

# Try a demo
python examples/carbon-auditor/demo.py
```

## Future Work

- Create benchmark suite for claim verification (CMVK)
- Measure actual message bus throughput (AMB)
- Profile policy engine latency
- Test integrations with LangChain/CrewAI
