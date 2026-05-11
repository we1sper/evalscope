# Per-Subset Limit Configuration — Modification Report

## Overview

Extended `TaskConfig.limit` to accept per-subset limit values via a `Dict[str, Union[int, float]]` in addition to the existing scalar `int`/`float`. Subsets not listed in the dict are unlimited.

## Design Decision

Extended the existing `limit` field rather than adding a new `subset_limits` field. This keeps the API surface small and is fully backward compatible — all existing configs continue to work unchanged.

| `limit` value | Behavior |
|---|---|
| `None` | No limit (unchanged) |
| `int` (e.g., `5`) | Same cap for all subsets (unchanged) |
| `float` (e.g., `0.5`) | Same fraction for all subsets (unchanged) |
| `{"a": 10, "b": 0.5}` | Per-subset; subsets not in dict are unlimited |

## Files Changed

### 1. `evalscope/config.py`
- **Line 104**: Type annotation widened from `Optional[Union[int, float]]` to `Optional[Union[int, float, Dict[str, Union[int, float]]]]`
- **Lines 186-204**: `_validate_limit` validator extended with dict branch:
  - Rejects non-string keys
  - Parses each value through `parse_int_or_float`
  - Rejects negative values
  - Filters out zero values
  - Coerces empty dict to `None`

### 2. `evalscope/api/benchmark/benchmark.py`
- **Line 132**: `DataAdapter.limit` property return type updated to include dict variant

### 3. `evalscope/api/benchmark/adapters/default_data_adapter.py`
- **Lines 232-243**: Added `_get_subset_limit(self, subset: str) -> Optional[Union[int, float]]` helper that resolves the effective limit for a given subset:
  - `self.limit` is `None` → returns `None`
  - `self.limit` is a `dict` → returns `self.limit.get(subset)` (None if not found)
  - Otherwise → returns `self.limit` as-is (scalar)
- **Line 254**: `load_subset()` changed from `limit=self.limit` to `limit=self._get_subset_limit(subset)` — this is the **non-reformat code path**, where each subset is loaded independently via a DataLoader
- **Lines 216-219**: `load_subsets()` reformat path unchanged — the full `self.limit` (including dict) is passed to `DatasetDict.from_dataset()`, which resolves per-key

### 4. `evalscope/api/dataset/dataset.py`
- **Lines 305-349**: `DatasetDict.from_dataset()` limit param widened to accept dict. Limit resolution logic changed:
  - If `limit` is a dict → resolve `subset_limit = limit.get(key)` per key
  - If `limit` is scalar → use as-is
  - Apply resolved `subset_limit` per subset (handles `int`/`float`/`None`)

### 5. `evalscope/arguments.py`
- **Lines 44-59**: Added `_parse_limit_arg(value: str)` type function — tries `json.loads` first (returns dict or float), falls back to `float(value)`, raises `ArgumentTypeError` with a descriptive message on failure. This allows the CLI to accept both `--limit 10` and `--limit '{"a": 10, "b": 0.5}'`
- **Line 69**: Changed `type=float` → `type=_parse_limit_arg`; help text updated to describe the JSON dict format

### 6. `evalscope/utils/resource_utils.py`
- **Lines 205-220**: `compute_eval_total_count()` updated to resolve per-subset limit from dict when estimating total sample counts

### 7. Subset-looping custom adapters
- `evalscope/benchmarks/needle_haystack/needle_haystack_adapter.py` line 234
- `evalscope/benchmarks/general_arena/general_arena_adapter.py` line 112

Both replaced `limit=self.limit` with `limit=self._get_subset_limit(subset_name)` to support per-subset limits when iterating over subsets.

## Data Flow

```
TaskConfig.limit: int | float | dict[str, int|float] | None
    │
    ▼
DataAdapter.limit (property, returns TaskConfig.limit directly)
    │
    ▼
DefaultDataAdapter._get_subset_limit(subset)
    ├─ limit is None → None
    ├─ limit is dict → limit.get(subset)  (None if subset not in dict)
    └─ limit is scalar → limit
    │
    ├── non-reformat path ──► DataLoader(limit=<resolved int/float/None>)
    │
    └── reformat path ──► DatasetDict.from_dataset(limit=<full dict or scalar>)
                              │
                              └─ resolves limit per key internally
```

## Backward Compatibility

All existing scalar `limit` values (`int`, `float`, `None`) pass through the new code paths identically. The dict handling code is only triggered when `isinstance(limit, dict)` is `True`.

## Verification

- 18 unit-level assertions covering: scalar int, scalar float, None, zero→None coercion, dict values, empty dict→None, zero-in-dict filtering, negative rejection, string→number parsing, `_get_subset_limit` resolution, `DatasetDict.from_dataset` per-key truncation, YAML round-trip, and `compute_eval_total_count`
- CLI arg parsing: `--limit 10` → `10.0`, `--limit '{"a": 10}'` → `{"a": 10}`, `--limit not_a_number` → `ArgumentTypeError`
- All existing tests pass (test_models.py uses `limit=5` scalar, unchanged behavior)

## Usage Example

```bash
evalscope eval \
  --model Qwen/Qwen2.5-0.5B-Instruct \
  --datasets cmmlu \
  --dataset-args '{"cmmlu": {"subset_list": ["ancient_chinese", "astronomy"]}}' \
  --limit '{"ancient_chinese": 10, "astronomy": 20}'
```