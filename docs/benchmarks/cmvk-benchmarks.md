# CMVK Design Notes

> Design rationale for Cross-Model Verification Kernel.

## Concept

CMVK uses multiple LLM models to verify claims. The hypothesis is that different models have different failure modes, so requiring consensus should reduce errors.

## Architecture

```
Input Claim
    │
    ├──► Model A (e.g., GPT-4)
    ├──► Model B (e.g., Claude)
    └──► Model C (e.g., Gemini)
            │
            ▼
    Consensus Engine
            │
            ▼
    Decision (PASS/FAIL/FLAG)
```

## Key Design Decisions

### 1. Mathematical Decision, Not LLM Decision

The final pass/fail decision uses mathematical comparison (e.g., vector distance), not another LLM call. This makes decisions deterministic and auditable.

```python
# Decision is math, not inference
drift = np.linalg.norm(claim_vector - observed_vector)
if drift > threshold:
    return "FLAGGED"
```

### 2. Parallel Model Calls

Models are called in parallel to reduce latency. If one model times out, the others can still provide a result (with reduced confidence).

### 3. Configurable Thresholds

Different use cases have different tolerance for false positives vs false negatives. Thresholds are configurable.

## Limitations

**Not benchmarked:** The accuracy claims in previous documentation were hypothetical. Real benchmarks would require:
- A labeled dataset of true/false claims
- Blind evaluation by human experts
- Statistical significance testing

**Cost:** Multi-model verification is 2-4x more expensive than single-model.

**Latency:** Parallel calls still add ~2-3 seconds vs single model.

**Not a silver bullet:** If all models share the same training data bias, they'll all make the same mistake.

## Running the Code

```bash
cd packages/cmvk
pip install -e .
python -m cmvk.verify "The capital of France is Paris"
```

## Future Work

- Add support for open-source models (reduce cost, increase diversity)
- Create benchmark dataset for claim verification
- Explore weighted consensus based on model confidence
