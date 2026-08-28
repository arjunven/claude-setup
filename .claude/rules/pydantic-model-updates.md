---
paths:
  - "backend/**/*.py"
description: How to derive modified versions of Pydantic models
---

# Pydantic Model Updates

When you need a new version of a Pydantic model with some fields changed:

1. **`model_copy(deep=True)`** the parent model to get an independent copy
2. **Construct new child objects** at the leaf that changed — e.g. `op.V_out = FloatParam(value=2.5, owner="app", ...)`. Don't reconstruct the whole parent; unchanged siblings come from the deep copy for free.
3. **New objects get Pydantic validation at construction** — typos in `Literal` fields, wrong types, and constraint violations are caught immediately.

## Avoid

- **Mutating fields on existing nested models** — `param.value = 2.5` bypasses Pydantic validation (unless `validate_assignment=True` is set, which we don't use).
- **`model_dump()` → modify dict → `model_validate()` round-trip** — loses type safety during the dict phase, and string-keyed dict access doesn't catch typos.

## Example

```python
# Good: deep copy parent, construct new leaf
corrected_op = op.model_copy(deep=True)
corrected_op.V_out = FloatParam(
    value=new_vout,
    owner="app",
    name=v_out.name,
    unit=v_out.unit,
    message="Output voltage auto-adjusted",
)

# Avoid: mutating fields on the copy
corrected_op.V_out.value = new_vout    # no validation
corrected_op.V_out.owner = "app"       # typo here would be silent

# Avoid: dict round-trip
data = op.model_dump()
data["V_out"]["value"] = new_vout      # string key, no autocomplete
return OperatingPoint.model_validate(data)
```
