# BigQuery Model Config Labels - Test Results

## Docker Images Built ✅

1. **dbt-bigquery:patched-model-labels** ✅
   - Includes forked dbt-common with `get_current_node()` / `set_current_node()`
   - Includes forked dbt-core with node context setting
   - Includes patched dbt-bigquery adapter

2. **tt-dbt:patched-labels** ✅
   - Built from dbt-bigquery:patched-model-labels
   - Includes sqlfluff (installed with --no-deps to preserve our patches)

## Changes Verified in Images ✅

### dbt-common
```bash
$ docker run --rm --entrypoint sh tt-dbt:patched-labels -c "grep -A 3 'def get_current_node' /usr/local/lib/python3.11/site-packages/dbt_common/events/contextvars.py"

def get_current_node() -> Optional['ManifestNode']:
    """
    Get the full node object from execution context.
```
✅ Context variable functions are present

### dbt-core  
```bash
$ docker run --rm --entrypoint sh tt-dbt:patched-labels -c "grep -A 5 'Set the full node in context' /usr/local/lib/python3.11/site-packages/dbt/task/base.py"

                # Set the full node in context before execution
                # This allows adapters to access full node config (e.g., labels)
                from dbt_common.events.contextvars import set_current_node
                set_current_node(ctx.node)
```
✅ Node is being set in context before execution

### dbt-bigquery
```bash
$ docker run --rm --entrypoint sh tt-dbt:patched-labels -c "grep -A 8 'def execute' /usr/local/lib/python3.11/site-packages/dbt/adapters/bigquery/impl.py"

    def execute(self, sql, auto_begin=False, fetch=None):
        """Override execute to extract model config labels from execution context."""
        try:
            from dbt_common.events.contextvars import get_current_node
```
✅ Adapter extracts labels from context

## Test Run

**Command**: `dbt run -m test_cost_tracking_labels --debug`

**Result**: Model executed successfully

**Latest Job**: https://console.cloud.google.com/bigquery?project=tt-jarmo-dev&j=bq:US:23cb8547-42ce-4871-905c-4ad9540fc43e&page=queryresults

**Debug Output**: 
- ✅ Model compiled
- ✅ SQL executed
- ❌ Context shows `labels from context=None`

## Current Status

**Infrastructure**: ✅ All code changes are in place  
**Docker Images**: ✅ Built and contain all patches  
**Test Run**: ⚠️ Executes but labels not flowing through context

## Issue

The adapter's `execute()` method may not be called during table materialization. Table materializations likely call connection manager methods directly, bypassing the adapter's `execute()`.

## Next Steps to Investigate

1. **Check BigQuery Console**: Verify if labels appear on the job despite debug showing `None`
   - URL: https://console.cloud.google.com/bigquery?project=tt-jarmo-dev&j=bq:US:23cb8547-42ce-4871-905c-4ad9540fc43e

2. **Check Execution Flow**: The table materialization might use `connections.execute()` directly instead of `adapter.execute()`

3. **Possible Solutions**:
   - **Option A**: Add label extraction to `connections.execute()` instead of `adapter.execute()`
   - **Option B**: Ensure table materialization calls `adapter.execute()`  
   - **Option C**: Use pre-model-hook to call a function that extracts and sets labels

## Alternative: Current Workaround Works

The materialization override (injecting labels as SQL comments) DID work but caused cache errors. This confirms:
- ✅ Labels can be accessed in materializations
- ✅ Labels can be passed to connection manager
- ✅ Labels make it to BigQuery jobs

We just need to find the right hook point in the execution flow.

## Files Status

### Cleanup Done ✅
- ❌ Removed `macros/materializations/table.sql` (workaround)
- ❌ Removed `macros/set_bigquery_labels.sql` (not needed)
- ❌ Removed `on-run-start` hook from dbt_project.yml

### Implementation Files ✅
- ✅ `/dbt-common/dbt_common/events/contextvars.py` - Context variables
- ✅ `/dbt-core/core/dbt/task/base.py` - Sets node in context
- ✅ `/dbt-core/core/MANIFEST.in` - Includes jsonschemas
- ✅ `/dbt-adapters/dbt-bigquery/src/dbt/adapters/bigquery/impl.py` - Extracts from context
- ✅ `/dbt-adapters/dbt-bigquery/src/dbt/adapters/bigquery/connections.py` - Uses labels
- ✅ `/dbt-adapters/dbt-bigquery/docker/Dockerfile` - Includes our forks
- ✅ `/tt-dbt/Dockerfile` - Preserves patches when installing sqlfluff

## Versions

- dbt-core: 1.11.0-b3 (with our patches)
- dbt-bigquery: 1.11.0rc1 (with our patches)
- dbt-common: 1.27.0 (with our patches)
