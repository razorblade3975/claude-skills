---
name: implementing-codegen-variables
description: Use when creating or modifying codegen variable classes in cg/python/vars/, or when building JSON param files for the algo engine simulator. Symptoms - writing a Variable subclass, using Cell/on/capture DSL, preparing a var_config dict or JSON for gen_data_algo_engine_sim.
---

# Implementing Codegen Variables

## Overview

Codegen variables are Python classes that define stateful computation graphs compiled to C++ plugins. They live in `cg/python/vars/` and are loaded by the factory system from JSON configs. Every variable exposes `self.value` (the output node) and `self.is_ready` (boolean readiness).

## When to Use

- Creating a new Variable subclass in `cg/python/vars/`
- Writing JSON param files for `gen_data_algo_engine_sim` or notebook sim workflows
- Debugging a variable that compiles but outputs zeros or wrong values

## Port/Product Format (Critical)

**The #1 source of silent failures.** Wrong format compiles fine but produces all zeros.

| Context | Format | Example |
|---------|--------|---------|
| JSON param for sim | Full internal_name string | `"BINANCE:PERP:AVAX/USDT"` |
| Jinja2 template | `{{ internal_name }}` | Resolved at render time |
| book_values.py `port` param | String or list | `"BINANCE:PERP:AVAX/USDT"` |
| pricing_models.py `product` param | String or list | `"BINANCE:PERP:AVAX/USDT"` |

Both string and list formats work — `lib.TradeUnit.load()` maps list elements positionally to `(product, inst_type, book_specific_id)`. The silent failure comes from **wrong list shape** (e.g., 3-element `["BINANCE", "PERP", "AVAXUSDT"]` instead of string `"BINANCE:PERP:AVAX/USDT"`). Use the full internal_name string for sim JSON params — it's always correct.

**Silent failure mode:** A malformed port compiles, runs, and produces only `timed_1000ms` events with all values = 0. The sim does not error — the product simply never matches any market data.

**Diagnostic:** If your output has only `timed_1000ms` events and no `quote`/`trade` events, the port format is wrong.

## Variable Class Template

```python
import codegen.core as cg
from codegen.core import fn, cast, Cell
from codegen import factory
from codegen import lib

class MyVariable(lib.Variable):
    def __init__(self, port, ...):
        product = lib.TradeUnit.load(port)

        # Readiness (two standard patterns)
        # Pattern A: slow-path clock (most variables)
        slow_path = lib.timed_clock(1000)
        self.is_ready = cg.capture(lib.is_ready(product), slow_path)
        # Pattern B: always ready (synthetic/sampler variables)
        # self.is_ready = True

        # Reset on readiness transitions (standard for stateful vars)
        reset_cond = lib.changed(self.is_ready)

        # Value computation using Cell + .on()
        value = Cell(0.0, type="double")
        value.on(reset_cond, 0.0)             # Reset when readiness changes
        value.on(event_condition, new_value_expression)

        self.value = value
```

## DSL Quick Reference

### State (Cell)
```python
c = Cell(0.0, type="double")     # Mutable state
c.on(condition, new_value)        # Update when condition true
c.prev()                          # Pre-update value (same cycle)
```

**Cell.on() semantics:** When multiple `.on()` calls trigger on the same event, they fire in **registration order** and each handler mutates the cell. Later handlers see the result of earlier ones. Use `c.prev()` to get the true start-of-event value. This is why handler registration order matters critically (e.g., capture direction BEFORE updating `last_px`).

### Events
| Function | Fires on |
|----------|----------|
| `lib.mkt_update(port)` | Any quote or trade |
| `lib.quote_update(port)` | Quote only |
| `lib.tick_update(port)` | Trade only |
| `lib.after_on_notify()` | Once per timestamp (after all events) |
| `cg.updated(node)` | When node value changes |

### Market Data
| Function | Returns |
|----------|---------|
| `lib.bidpx(port)`, `lib.askpx(port)` | Best bid/ask price |
| `lib.bidsz(port)`, `lib.asksz(port)` | Best bid/ask size |
| `lib.trade_price(port)`, `lib.trade_size(port)` | Last trade price/size |
| `lib.is_ready(port)` | Bool: has received first quote |

### Math / Logic
| Function | Notes |
|----------|-------|
| `fn("fabs")(x)` | Float absolute value |
| `lib.get_sign(x)` | Returns -1, 0, or +1 (int64_t) |
| `cast("double")(x)` | Type cast |
| `cg.If(cond, then, else)` | Ternary |
| `a & b`, `a \| b` | Logical AND/OR (not bitwise!) |

### Samplers
```python
from vars import samplers
event, samplecount, decay = samplers.make_multi_sampler(sampler_spec)
# Use: value.on(event, value * decay)  # decay on sample
```

### Pricing Models
```python
# From JSON spec (full loadable form):
pm = factory.load(["vars.pricing_models.MktSwitchMid", {"product": port}])

# Inside a Variable class (with default fallback):
from vars.pricing_models import MktSwitchMid, load_pm
if ref_pm is None:
    ref_pm = MktSwitchMid(port)
else:
    ref_pm = load_pm(port, ref_pm)  # handles alias resolution
px = ref_pm.price
ready = ref_pm.is_ready
```

**Note:** `factory.load()` only works for full `["module.Class", {params}]` specs. For alias strings, use `load_pm()` from `pricing_models.py`.

## Verification Workflow

**Always verify after writing a variable.** Create a JSON param file and run:

```bash
cd ~/src/qr
.venv/bin/python -m qr.sim.gen_data_algo_engine_sim \
  --instrument BINANCE:PERP:AVAX/USDT \
  --date 20251231 \
  --param /tmp/my_var_test.json
```

Then inspect the output:
```python
import polars as pl
df = pl.read_parquet("<output_dir>/result.parquet")
print(df['event_type'].value_counts())  # Should have quote/trade, NOT just timed_1000ms
print(df['my_var_value'].describe())    # Should have non-zero values
```

**Checklist:**
- [ ] Output has `quote` and `trade` events (not just `timed_1000ms`)
- [ ] `_value` columns have non-zero, non-null values
- [ ] `_is_ready` columns are 1 after warmup
- [ ] Row count is ~2M for a full day of BINANCE perp data

## JSON Param File Format

```json
{
    "var_name": ["vars.module.ClassName", {"param": "value"}]
}
```

Example with pricing model nesting:
```json
{
    "my_signal": ["vars.samplers.PXCSampleDir", {
        "port": "BINANCE:PERP:AVAX/USDT",
        "thresh_bps": 10.0,
        "pm": ["vars.pricing_models.MktSwitchMid", {
            "product": "BINANCE:PERP:AVAX/USDT"
        }]
    }]
}
```

## Common Mistakes

| Mistake | Symptom | Fix |
|---------|---------|-----|
| Wrong port format (wrong list shape) | All zeros, only `timed_1000ms` events | Use `"EXCHANGE:TYPE:BASE/QUOTE"` string |
| Cell.on() order wrong | Direction/diff captured after update | Register dependent `.on()` BEFORE the updating `.on()` |
| Missing `lib.is_ready()` guard | NaN/garbage during warmup | Guard events: `event & lib.is_ready(port)` |
| Using `&` for bitwise | Gets logical AND instead | Use `fn("&")(a, b)` for bitwise |
| Assuming function semantics | Silent wrong values | Read the source in `lib.py` before using unfamiliar functions |
| Missing reset on readiness | Stale values after reconnect | Add `value.on(lib.changed(is_ready), 0.0)` — see `trade_count.py:22` |
| Using `factory.load()` for PM alias | AttributeError on `.price` | Use `load_pm()` from `pricing_models.py` for alias resolution |
| PricingModel in JSON without wrapper | Column named `_price` not `_value` | Wrap with `PricingModelVariable` if you want `_value` column |
| No verification run | Broken variable ships | Always run gen_data_algo_engine_sim and inspect output |

## Existing Variable Files (Pattern Reference)

| File | Good example for |
|------|-----------------|
| `book_values.py` | Simple Cell + event pattern, is_ready slow-path |
| `pricing_models.py` | Pricing model loading, PricingModelVariable wrapper |
| `samplers.py` | Sampler classes, SamplerVariable (reset to 0 pattern), PXCSampleDir |
| `price_trend.py` | EMA/trend with sampler, normalization modes |
| `trade_count.py` | Trade event accumulation with decay |
| `trade_based.py` | Trade-based features, normalization |
| `book_indicators.py` | Complex multi-level book features, caching |
