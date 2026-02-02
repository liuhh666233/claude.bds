---
name: Factor Migration
description: Migrate quantitative factors from Python to C++ (blitzer) with complete implementation and testing
---

# Factor Migration Skill

Migrate quantitative factor calculations from Python to C++ for the blitzer project.

## Overview

This skill guides the migration of factor computation logic from Python (pandas/numpy based) to C++ (streaming/incremental processing). The blitzer framework uses a specific architecture with state management, parallel execution, and Parquet output.

## Interactive Workflow

### Step 0: Gather User Input

**Ask user for the following information:**

1. **Factor Store Path**: Where the Python factor implementations are located
   - Default: `/home/lxb/github/factor-store`
   - Common locations: `factor-store`, `intraday-alpha-research`, `research-toolkit`

2. **Factor Name(s)**: Which factor(s) to migrate
   - Can be a single factor or comma-separated list
   - Example: `compute_err`, `compute_main_force_returns`, `compute_volatility`

Use `AskUserQuestion` tool:

```
Questions:
1. Factor Store Path - Where is the Python factor code?
   Options:
   - /home/lxb/github/factor-store (default)
   - /home/lxb/github/intraday-alpha-research
   - [Other - custom path]

2. Factor Name - Which factor(s) to migrate?
   [Free text input - e.g., compute_err, compute_volatility]
```

### Step 1: Locate and Read Python Factor

After getting user input, search for the factor:

```bash
# Find the factor file
grep -r "def ${FACTOR_NAME}" ${FACTOR_STORE_PATH} --include="*.py"
```

Read the factor implementation and analyze:
- Input data fields (Close, Volume, etc.)
- Window size and parameters
- Computation logic
- Output field names

### Step 2: Resolve Unknown Dependencies

When encountering functions/modules with unclear definitions (e.g., `npd.ffillna`, `npd.clean`, `bn.move_sum`), ask the user for source paths:

```
I found references to the following libraries that I need to understand:
- numpandas (npd.ffillna, npd.clean, npd.shift)
- bottleneck (bn.move_sum, bn.move_std)
- toolkit.stats (custom functions)

Please provide the source code paths for these libraries:

Options:
1. numpandas - Path to numpandas repo
   - Default: /home/lxb/github/numpandas
   - [Other - custom path]

2. research-toolkit - Path to toolkit/stats
   - Default: /home/lxb/github/research-toolkit
   - [Other - custom path]

3. Skip - I'll infer the behavior from usage context
```

**Common library locations:**
| Library | Default Path | Key Functions |
|---------|--------------|---------------|
| `numpandas` | `/home/lxb/github/numpandas` | `ffillna`, `clean`, `shift`, `align_backward` |
| `research-toolkit` | `/home/lxb/github/research-toolkit` | `standardize`, `rolling_correlation` |
| `bottleneck` | System package | `move_sum`, `move_std`, `move_median`, `nanargmax` |

After getting paths, read the relevant function implementations to understand:
- NaN handling behavior
- Edge case handling (empty arrays, all-NaN slices)
- Parameter defaults (min_count, ddof, etc.)

### Step 3: Check Prerequisites

**Inform user of required preparations:**

```markdown
## Prerequisites Checklist

Before proceeding, please ensure:

1. **Blitzer Codebase Access**
   - Path: `/home/lxb/github/blitzer`
   - You have write permissions

2. **Build Environment**
   - `nix develop` works in the blitzer directory
   - `ninja` is available in build directory

3. **Input Data Schema**
   The factor uses these fields from `PriceVolume1MinRow`:
   - [ ] `close` - Closing price
   - [ ] `volume` - Trading volume
   - [ ] `volume_rmb` - Volume in RMB (if using vwap)
   - [ ] Other: {list any additional fields}

4. **Reference Implementation**
   Python code location: `${FACTOR_STORE_PATH}/${FACTOR_FILE}`

Ready to proceed? [Y/n]
```

## Prerequisites

- Access to the blitzer codebase at `/home/lxb/github/blitzer`
- Python factor implementation with clear computation logic
- Understanding of the factor's input data and output schema

## Files to Modify

| File | Operation | Purpose |
|------|-----------|---------|
| `blitzer/features/feature_impls/feature_xxx.cc` | **Create** | Factor computation implementation |
| `blitzer/features/feature_impls/feature_common.h` | Modify | Add function declaration |
| `blitzer/features/slice_minbar_features.h` | Modify | Add FeatureResult fields + Parquet schema |
| `blitzer/features/slice_minbar_features.cc` | Modify | Register factor in BuildComputeFuncs |
| `blitzer/features/CMakeLists.txt` | Modify | Add source file |
| `config/slice_minbar_features_config.json` | Modify | Enable factor in runtime config |
| `blitzer/features/slice_minbar_features_test.cc` | Modify | Add unit tests |

## Workflow

### Step 1: Understand Python Implementation

Before writing any code, thoroughly analyze the Python factor:

1. **Identify inputs**: What data fields are used? (close, volume, etc.)
2. **Window size**: Rolling window parameters (e.g., 241 bars)
3. **Computation logic**: Step-by-step algorithm
4. **Edge cases**: NaN handling, min_count, boundary conditions
5. **Output fields**: What values are produced?

Create a mapping table like:

```markdown
| Python | C++ |
|--------|-----|
| `npd.ffillna(prices)` | Forward fill with `FindLastValidPrice()` |
| `bn.move_sum(x, window=241, min_count=1)` | `std::deque` + running sum |
| `np.sign(returns)` | `std::copysign(1.0, returns)` |
```

### Step 2: Create Implementation File

Create `blitzer/features/feature_impls/feature_xxx.cc`:

```cpp
#include <algorithm>
#include <cmath>
#include <deque>
#include <limits>

#include "blitzer/features/feature_impls/feature_common.h"
#include "blitzer/features/slice_minbar_features.h"
#include "marl/defer.h"
#include "marl/waitgroup.h"

namespace wonder::blitzer::features {

// ========== State Structure ==========

struct FeatureXxxState {
  static constexpr int kWindow = 241;

  // Price/volume history
  std::deque<double> prices;
  std::deque<double> volumes;

  // Derived data history
  std::deque<double> returns;

  // Running statistics
  double last_valid_price = std::numeric_limits<double>::quiet_NaN();

  // Output accumulators (for move_sum equivalent)
  std::deque<double> output_contrib;
  double output_sum = 0.0;
};

// ========== Helper Functions ==========

static double SafeReturn(double current, double previous) {
  if (!std::isfinite(previous) || previous <= 1e-9) {
    return std::numeric_limits<double>::quiet_NaN();
  }
  double ret = (current - previous) / previous;
  // Clean extreme values
  if (!std::isfinite(ret) || std::abs(ret) > 10.0) {
    return std::numeric_limits<double>::quiet_NaN();
  }
  return ret;
}

// ========== Single Stock Computation ==========

static void ComputeFeatureXxxSingle(const SliceMinbar& m,
                                    FeatureXxxState& st,
                                    FeatureResult& out,
                                    double adj_factor) {
  const double NaN = std::numeric_limits<double>::quiet_NaN();

  // 1. Get current price with adjustment
  double current_price = m.close * adj_factor;
  if (!std::isfinite(current_price) || current_price <= 0) {
    current_price = st.last_valid_price;  // Forward fill
  }

  // 2. Calculate return
  double current_return = SafeReturn(current_price, st.last_valid_price);

  // 3. Update last valid price
  if (std::isfinite(current_price) && current_price > 0) {
    st.last_valid_price = current_price;
  }

  // 4. Manage sliding window
  // ... (factor-specific logic)

  // 5. Compute output
  out.xxx_field = /* computed value */;
}

// ========== Batch Processing ==========

void ComputeFeatureXxx(const std::vector<SliceMinbar>& minbars,
                       std::vector<FeatureResult>& results,
                       const std::vector<double>& adj_factors,
                       const std::vector<SharedFeatureState>& shared_states,
                       marl::Scheduler& scheduler) {
  static std::vector<FeatureXxxState> states;
  InitializeAndValidateStates(states, minbars, "FeatureXxx");

  marl::WaitGroup wg;
  for (size_t i = 0; i < minbars.size(); ++i) {
    wg.add(1);
    marl::schedule([&, i] {
      defer(wg.done());

      const auto& minbar = minbars[i];
      if (!shared_states[i].is_valid) {
        return;
      }

      auto& result = results[i];
      auto& state = states[i];
      double adj_factor = adj_factors[i];

      ComputeFeatureXxxSingle(minbar, state, result, adj_factor);
    });
  }
  wg.wait();
}

}  // namespace wonder::blitzer::features
```

### Step 3: Update Header Files

**In `feature_common.h`**, add function declaration:

```cpp
void ComputeFeatureXxx(const std::vector<SliceMinbar>& minbars,
                       std::vector<FeatureResult>& results,
                       const std::vector<double>& adj_factors,
                       const std::vector<SharedFeatureState>& shared_states,
                       marl::Scheduler& scheduler);
```

**In `slice_minbar_features.h`**, add to `FeatureResult`:

```cpp
struct FeatureResult {
  // ... existing fields ...

  // New factor fields
  double xxx_field = std::numeric_limits<double>::quiet_NaN();
};
```

**In Parquet codec section**, add mapping:

```cpp
codec.Add("XxxField", &blitzer::features::FeatureResult::xxx_field);
```

### Step 4: Register Factor

**In `slice_minbar_features.cc`**, find `BuildComputeFuncs` and add:

```cpp
if (enabled.count("xxx_field")) {
  funcs.push_back(&ComputeFeatureXxx);
}
```

### Step 5: Update CMakeLists.txt

Add to `slice_minbar_features` target:

```cmake
target_sources(
  slice_minbar_features
  ...
  PRIVATE ...
          feature_impls/feature_xxx.cc
  ...)
```

### Step 6: Update Runtime Config

**In `config/slice_minbar_features_config.json`**, add new factor fields to enable them:

```json
{
  "enabled_features": [
    "prices_err",
    "upward_std_bias",
    "vsa_ratio",
    "vsa_low_to_max",
    "evv",
    "xxx_field"
  ]
}
```

**Note**: The factor name in the config must match the field name in `FeatureResult` and the key used in `BuildComputeFuncs`.

### Step 7: Add Unit Tests

**In `slice_minbar_features_test.cc`**:

1. Add to `all_features` in SetUp:
```cpp
std::unordered_set<std::string> all_features = {
  // ... existing ...
  "xxx_field"
};
```

2. Add test cases:
```cpp
TEST_F(SliceMinbarFeaturesTest, XxxInsufficientDataTest) {
  // Test with < window size data points
}

TEST_F(SliceMinbarFeaturesTest, XxxFullWindowTest) {
  // Test with full window
}

TEST_F(SliceMinbarFeaturesTest, XxxEdgeCasesTest) {
  // Test NaN, zero volume, extreme values
}
```

### Step 8: Build and Test

```bash
cd /home/lxb/github/blitzer
nix develop
cd build
ninja slice_minbar_features
ninja slice_minbar_features_test
./test/slice_minbar_features_test --gtest_filter="*Xxx*"
```

## Common Patterns

### Rolling Sum (move_sum equivalent)

```cpp
// Maintain deque and running sum
if (st.contrib_queue.size() >= kWindow) {
  st.running_sum -= st.contrib_queue.front();
  st.contrib_queue.pop_front();
}
st.contrib_queue.push_back(contribution);
st.running_sum += contribution;

// Output (with min_count handling)
out.field = (st.contrib_queue.size() >= min_count) ? st.running_sum : NaN;
```

### Forward Fill

```cpp
double current_price = m.close * adj_factor;
if (!std::isfinite(current_price) || current_price <= 0) {
  current_price = st.last_valid_price;  // Use previous valid value
}
// Update for next iteration
if (std::isfinite(current_price) && current_price > 0) {
  st.last_valid_price = current_price;
}
```

### Direction Detection with Forward Fill

```cpp
int direction = 0;
if (current_return > 0) direction = 1;
else if (current_return < 0) direction = -1;
else direction = st.last_direction;  // Forward fill when return == 0
st.last_direction = direction;
```

### Running Max/Min

```cpp
if (score > st.running_max) {
  st.running_max = score;
  is_new_max = true;
}
```

## Important Notes

### Price Handling
- Always apply `adj_factor` for stock split adjustment
- Forward fill invalid prices (NaN, <= 0)
- Use `volume_rmb / volume` when vwap price is needed

### Return Calculation
- Formula: `(price - lag_price) / lag_price`
- Clean extreme values: filter `|ret| > 10.0`
- Handle division by zero

### Test Data Setup
- Ensure `CreateTestBar` sets `volume_rmb = close * volume`
- Add new factor names to `all_features` in SetUp

### Debugging Tips
- Use `spdlog::debug()` for intermediate values
- Compare with Python output for specific timestamps
- Check window boundary conditions carefully

## Verification Checklist

- [ ] State structure with correct window size
- [ ] Price adjustment and forward fill
- [ ] Return calculation with cleaning
- [ ] Sliding window management
- [ ] Function declaration in header (`feature_common.h`)
- [ ] FeatureResult fields added (`slice_minbar_features.h`)
- [ ] Parquet codec mapping (`slice_minbar_features.h`)
- [ ] Factor registered in BuildComputeFuncs (`slice_minbar_features.cc`)
- [ ] CMakeLists.txt updated
- [ ] Runtime config updated (`config/slice_minbar_features_config.json`)
- [ ] Unit tests added (`slice_minbar_features_test.cc`)
- [ ] Compilation successful
- [ ] Tests pass
- [ ] Output matches Python implementation
