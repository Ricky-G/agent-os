# AAIF Proposal: Standard Agent Signals

**Proposal Type:** Standard Specification  
**Submitter:** Imran Siddique  
**Affiliation:** Microsoft  
**Date:** 2026-01-27

---

## 1. Executive Summary

We propose that AAIF adopt a standard set of signals for controlling autonomous AI agents, based on POSIX process signals. This would enable interoperability between agent frameworks (LangChain, CrewAI, AutoGen, etc.) and governance tools.

**Key Benefits:**
- One governance layer for all frameworks
- Portable agent policies
- Standard debugging/auditing tools
- Reduced vendor lock-in

## 2. Problem Statement

### Current State

Each agent framework implements control differently:

```
LangChain:  agent.stop()
CrewAI:     crew.stop()
AutoGen:    agent.reset()
Custom:     ???
```

### Consequences

1. **No portable governance**: A policy engine built for LangChain doesn't work with CrewAI
2. **No standard tooling**: Each framework needs its own debugger, profiler, auditor
3. **Enterprise barrier**: Organizations can't adopt agents without standardized control

### Industry Parallel

This is the state containers were in before OCI (Open Container Initiative). Docker, rkt, and others were incompatible. OCI solved this with a standard runtime spec.

**We need an OCI for agents.**

## 3. Proposed Solution

### 3.1 Standard Signal Set

| Signal | Value | Behavior | Catchable |
|--------|-------|----------|-----------|
| `SIGKILL` | 9 | Immediate termination | No |
| `SIGSTOP` | 19 | Pause execution | No |
| `SIGCONT` | 18 | Resume from pause | No |
| `SIGINT` | 2 | Graceful shutdown | Yes |
| `SIGTERM` | 15 | Request termination | Yes |
| `SIGUSR1` | 10 | User-defined | Yes |
| `SIGUSR2` | 12 | User-defined | Yes |

### 3.2 Why These Signals

**SIGKILL (non-catchable termination)**
- Critical for safety: A misbehaving agent MUST be stoppable
- Cannot be caught: Even a compromised agent can be terminated

**SIGSTOP/SIGCONT (pause/resume)**
- Enables inspection without termination
- Useful for debugging, auditing, rate limiting

**SIGINT/SIGTERM (graceful shutdown)**
- Allows cleanup, state saving
- Framework can implement custom handling

### 3.3 Interface Specification

```python
class AgentSignal(IntEnum):
    SIGINT = 2
    SIGKILL = 9
    SIGUSR1 = 10
    SIGUSR2 = 12
    SIGTERM = 15
    SIGCONT = 18
    SIGSTOP = 19

class SignalHandler(Protocol):
    def send_signal(self, agent_id: str, signal: AgentSignal) -> bool: ...
    def register_handler(self, signal: AgentSignal, handler: Callable) -> bool: ...
    def get_state(self, agent_id: str) -> Literal["RUNNING", "STOPPED", "TERMINATED"]: ...
```

## 4. Adoption Path

### Phase 1: Specification (Q1 2026)
- Finalize RFC-003 (Agent Signals)
- Publish reference implementation
- Create conformance test suite

### Phase 2: Framework Integration (Q2 2026)
- Work with LangChain, CrewAI, AutoGen maintainers
- Provide adapter libraries for easy adoption
- Document migration guides

### Phase 3: Tooling (Q3 2026)
- Standard debugger (`agent-gdb`)
- Standard profiler (`agent-perf`)
- Standard auditor (`agent-audit`)

## 5. Compatibility

### Backward Compatibility

Frameworks can adopt incrementally:

```python
# Wrapper for existing framework
class LangChainSignalAdapter:
    def __init__(self, agent):
        self.agent = agent
    
    def send_signal(self, agent_id, signal):
        if signal == AgentSignal.SIGKILL:
            self.agent.stop()  # Map to existing API
            return True
        # ... more mappings
```

### Forward Compatibility

Signal values are fixed (matching POSIX). New signals can be added without breaking existing implementations.

## 6. Reference Implementation

Available at: https://github.com/imran-siddique/agent-os

Key files:
- `packages/control-plane/src/agent_control_plane/signals.py`
- `docs/rfcs/RFC-003-Agent-Signals.md`
- `tests/test_kernel_critical.py` (conformance tests)

## 7. Prior Art

| Standard | Domain | Relevance |
|----------|--------|-----------|
| POSIX signals | Operating systems | Direct inspiration |
| OCI Runtime Spec | Containers | Similar interoperability problem |
| OpenTelemetry | Observability | Standard across vendors |
| MCP (Model Context Protocol) | LLM tools | Agent-adjacent standard |

## 8. Stakeholder Support

**Seeking endorsement from:**
- [ ] LangChain (langchain-ai)
- [ ] CrewAI (joaomdmoura)
- [ ] AutoGen (microsoft)
- [ ] DSPy (stanfordnlp)
- [ ] Semantic Kernel (microsoft)

**Initial discussions:**
- [TBD - will link to GitHub discussions]

## 9. Risks and Mitigations

| Risk | Mitigation |
|------|------------|
| Frameworks don't adopt | Provide adapter libraries with zero code changes |
| Specification too rigid | Allow user-defined signals (SIGUSR1/2) |
| Performance overhead | Signals are metadata, not data path |
| Security concerns | Non-catchable signals ensure governance |

## 10. Success Metrics

| Metric | Target | Timeline |
|--------|--------|----------|
| Frameworks implementing spec | 3+ | 6 months |
| Governance tools using spec | 5+ | 12 months |
| Production deployments | 100+ | 18 months |

## 11. Resources Required

- **From AAIF**: Standards process guidance, communication channels
- **From submitter**: Reference implementation, conformance tests, documentation
- **From community**: Feedback, framework-specific adapters

## 12. Conclusion

Standard agent signals would unlock:
1. Portable governance across frameworks
2. Standard tooling (debuggers, profilers, auditors)
3. Enterprise confidence in agent adoption

The specification is minimal (7 signals), backward-compatible, and based on 50+ years of proven OS design.

**We request AAIF consider this for the standards track.**

---

## Appendix A: Full Specification

See: [RFC-003: Standard Agent Signals](../rfcs/RFC-003-Agent-Signals.md)

## Appendix B: Conformance Tests

```python
# Minimum conformance tests
def test_sigkill_terminates(handler):
    handler.send_signal("agent-1", AgentSignal.SIGKILL)
    assert handler.get_state("agent-1") == "TERMINATED"

def test_sigkill_not_catchable(handler):
    caught = False
    def try_catch(sig): nonlocal caught; caught = True
    handler.register_handler(AgentSignal.SIGKILL, try_catch)
    handler.send_signal("agent-1", AgentSignal.SIGKILL)
    assert handler.get_state("agent-1") == "TERMINATED"
    # Handler should NOT have been called

def test_sigstop_pauses(handler):
    handler.send_signal("agent-1", AgentSignal.SIGSTOP)
    assert handler.get_state("agent-1") == "STOPPED"

def test_sigcont_resumes(handler):
    handler.send_signal("agent-1", AgentSignal.SIGSTOP)
    handler.send_signal("agent-1", AgentSignal.SIGCONT)
    assert handler.get_state("agent-1") == "RUNNING"
```

## Appendix C: Contact

**Submitter:** Imran Siddique  
**GitHub:** https://github.com/imran-siddique  
**Project:** https://github.com/imran-siddique/agent-os
