# dbt Fork Implementation Plan
## Goal: Pass Model Config (Labels) to Adapters During Execution

## Architecture Overview

```
dbt-core (execution)
     ↓
  sets full node in context
     ↓
dbt-common (context variables)
     ↓
dbt-bigquery adapter (reads from context)
     ↓
  extracts labels → BigQuery job
```

## Required Forks

### 1. dbt-common Fork
**Repository**: https://github.com/dbt-labs/dbt-common  
**Our fork**: TBD - needs to be cloned/forked

**Changes needed**: Add context variable for full node

### 2. dbt-core Fork  
**Repository**: https://github.com/dbt-labs/dbt-core  
**Our fork**: /Users/jarmo/git/tokenterminal/dbt-core ✅

**Changes needed**: Set node in context before execution

### 3. dbt-bigquery Fork
**Repository**: Part of dbt-adapters  
**Our fork**: /Users/jarmo/git/tokenterminal/dbt-adapters/dbt-bigquery ✅

**Changes needed**: Read node from context and extract labels

---

## Implementation Steps

### Step 1: Fork & Setup dbt-common

```bash
cd /Users/jarmo/git/tokenterminal
git clone https://github.com/dbt-labs/dbt-common.git
cd dbt-common
git checkout v1.27.0  # Match version from dbt-core setup.py line 75
```

### Step 2: Add Context Variable to dbt-common

**File**: `dbt-common/dbt_common/events/contextvars.py`

Add at the end of the file:

```python
from contextvars import ContextVar
from typing import Optional, TYPE_CHECKING

if TYPE_CHECKING:
    from dbt.contracts.graph.nodes import ManifestNode

# Context variable for passing the full node during execution
# This allows adapters to access model config (like labels) at execution time
_CURRENT_NODE: ContextVar[Optional['ManifestNode']] = ContextVar('current_node', default=None)


def set_current_node(node: Optional['ManifestNode']) -> None:
    """
    Set the full node object in context for adapter access.
    
    This should be called by dbt-core before executing a node,
    allowing adapters to access the full node configuration including
    model-specific settings like labels, tags, etc.
    
    Args:
        node: The full ManifestNode being executed, or None to clear
    """
    _CURRENT_NODE.set(node)


def get_current_node() -> Optional['ManifestNode']:
    """
    Get the full node object from execution context.
    
    Returns:
        The current node being executed, or None if not in execution context
    """
    return _CURRENT_NODE.get()
```

### Step 3: Update dbt-core Execution Path

**File**: `dbt-core/core/dbt/task/base.py`

In the `compile_and_execute` method (around line 282), add:

```python
def compile_and_execute(self, manifest: Manifest, ctx: ExecutionContext):
    result = None
    with (
        self.adapter.connection_named(self.node.unique_id, self.node)
        if get_flags().INTROSPECT
        else nullcontext()
    ):
        ctx.node.update_event_status(node_status=RunningStatus.Compiling)
        fire_event(
            NodeCompiling(
                node_info=ctx.node.node_info,
            )
        )
        with collect_timing_info("compile", ctx.timing.append):
            ctx.node = self.compile(manifest)

        # for ephemeral nodes, we only want to compile, not run
        if not ctx.node.is_ephemeral_model or self.run_ephemeral_models:
            ctx.node.update_event_status(node_status=RunningStatus.Executing)
            fire_event(
                NodeExecuting(
                    node_info=ctx.node.node_info,
                )
            )
            
            # NEW: Set the full node in context before execution
            from dbt_common.events.contextvars import set_current_node
            set_current_node(ctx.node)
            
            try:
                with collect_timing_info("execute", ctx.timing.append):
                    result = self.run(ctx.node, manifest)
                    ctx.node = result.node
            finally:
                # Clean up context after execution
                set_current_node(None)

    return result
```

Add import at top of file:
```python
from dbt_common.events.contextvars import set_current_node  # Add to existing imports
```

### Step 4: Update BigQuery Adapter

**File**: `dbt-adapters/dbt-bigquery/src/dbt/adapters/bigquery/impl.py`

Replace the current `execute` method with:

```python
def execute(self, sql, auto_begin=False, fetch=None):
    """Override execute to extract model config labels from execution context."""
    try:
        from dbt_common.events.contextvars import get_current_node
        from dbt.adapters.bigquery.connections import set_model_labels
        
        # Get the full node from execution context
        node = get_current_node()
        if node and hasattr(node, 'config'):
            # Extract labels from node config
            labels = getattr(node.config, 'labels', None)
            if isinstance(labels, dict) and labels:
                # Set labels in connection context for raw_execute to use
                set_model_labels(labels)
                logger.debug(f"Set model labels from node config: {labels}")
    except Exception as e:
        logger.debug(f"Could not extract model labels from node: {e}")
    
    return super().execute(sql, auto_begin=auto_begin, fetch=fetch)
```

**File**: `dbt-adapters/dbt-bigquery/src/dbt/adapters/bigquery/connections.py`

The context variable and extraction code is already in place! ✅

### Step 5: Update Dependencies

**File**: `dbt-core/core/setup.py`

Update line 75 to point to our forked dbt-common:

```python
# Before:
"dbt-common>=1.27.0,<2.0",

# After (for local development):
# Use editable install: pip install -e /path/to/our/dbt-common

# Or (for production):
"dbt-common @ git+https://github.com/token-terminal/dbt-common@v1.27.0-tt-labels",
```

---

## Testing

### 1. Unit Test (dbt-common)

**File**: `dbt-common/tests/unit/test_contextvars.py`

```python
from dbt_common.events.contextvars import set_current_node, get_current_node

def test_current_node_context():
    """Test that node can be set and retrieved from context."""
    # Initially None
    assert get_current_node() is None
    
    # Create a mock node
    mock_node = type('MockNode', (), {'unique_id': 'model.test.my_model'})()
    
    # Set in context
    set_current_node(mock_node)
    
    # Retrieve from context
    retrieved = get_current_node()
    assert retrieved is mock_node
    assert retrieved.unique_id == 'model.test.my_model'
    
    # Clear context
    set_current_node(None)
    assert get_current_node() is None
```

### 2. Integration Test (dbt-bigquery)

Test with actual model:

```sql
-- models/test_labels.sql
{{ config(
    materialized='table',
    labels={
        'team': 'data',
        'cost_center': 'analytics'
    }
) }}

select 1 as id
```

Run and verify:
```bash
dbt run --select test_labels --debug
# Check logs for "Set model labels from node config"
# Check BigQuery Console for job labels
```

---

## Build & Install

### Build Order

1. **dbt-common** (dependency of dbt-core)
```bash
cd /Users/jarmo/git/tokenterminal/dbt-common
pip install -e .
```

2. **dbt-core** (uses dbt-common)
```bash
cd /Users/jarmo/git/tokenterminal/dbt-core/core
pip install -e .
```

3. **dbt-bigquery** (uses dbt-core)
```bash
cd /Users/jarmo/git/tokenterminal/dbt-adapters/dbt-bigquery
pip install -e .
```

### Docker Build

Update `dbt-bigquery/docker/Dockerfile`:

```dockerfile
FROM python:3.11.2-slim-bullseye

# Install our forked dbt-common first
COPY ../../../dbt-common /tmp/dbt-common
RUN pip install /tmp/dbt-common

# Install our forked dbt-core
COPY ../../../dbt-core/core /tmp/dbt-core
RUN pip install /tmp/dbt-core

# Install our patched dbt-bigquery
COPY . /tmp/dbt-bigquery
RUN pip install /tmp/dbt-bigquery

ENTRYPOINT ["dbt"]
```

---

## Version Management

### Tag Strategy

- **dbt-common**: `v1.27.0-tt-labels`
- **dbt-core**: `v1.11.0b3-tt-labels`  
- **dbt-bigquery**: `v1.11.0rc1-tt-labels`

### Git Workflow

```bash
# In each repo
git checkout -b feat/model-config-labels
git commit -m "feat: pass model config to adapters via execution context"
git tag v1.X.Y-tt-labels
git push origin feat/model-config-labels
git push origin v1.X.Y-tt-labels
```

---

## Benefits of This Approach

1. **✅ Clean Architecture** - Uses context variables (already in dbt)
2. **✅ Thread-Safe** - ContextVar is thread-local
3. **✅ Minimal Changes** - Small, focused modifications
4. **✅ Generic** - Works for any adapter, any config field
5. **✅ Future-Proof** - Can be upstreamed to dbt-labs
6. **✅ No Hacks** - Proper execution flow

---

## Alternative: Workaround (Current)

If we can't fork dbt-common right now, our current materialization override works:

```sql
-- macros/materializations/table.sql injects labels as SQL comment
/* dbt:labels:{"team":"data"} */

-- connections.py extracts them before execution
```

**Pros**: Works without dbt-common fork  
**Cons**: Only works for table materialization, hacky

---

## Next Actions

1. ⬜ Clone and fork dbt-common
2. ⬜ Implement context variable in dbt-common
3. ⬜ Update dbt-core execution path
4. ⬜ Simplify BigQuery adapter impl.py
5. ⬜ Test end-to-end
6. ⬜ Build Docker images
7. ⬜ Update tt-dbt to use forked images
