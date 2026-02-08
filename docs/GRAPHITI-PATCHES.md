# Graphiti Patches and Fixes

Documentation of patches applied to Graphiti for OpenClaw Context Graph integration.

## Overview

These patches fix issues encountered when using Graphiti's temporal knowledge graph with OpenClaw. They should be re-applied after Graphiti upgrades.

---

## Patch 1: Pydantic None Guard in edges.py

### Problem

During search, Neo4j can return incomplete records with `None` values for `uuid` and `fact` fields. This causes Pydantic validation errors:

```
uuid Input should be a valid string
fact Input should be a valid string
```

### Root Cause

The `EdgeResult` model in Graphiti expects string values, but Neo4j queries can return `None` when records are incomplete or during edge resolution when the LLM returns malformed data.

### Fix

Add null guards in `graphiti_core/edges.py` where EdgeResult objects are created:

```python
# Before
EdgeResult(
    uuid=record['uuid'],
    fact=record['fact'],
    ...
)

# After
EdgeResult(
    uuid=record['uuid'] or str(uuid4()),
    fact=record['fact'] or '',
    ...
)
```

### Location

`graphiti_core/edges.py` - all places where `EdgeResult` is instantiated from Neo4j records.

---

## Patch 2: SearchResults Iteration Fix in query.py

### Problem

Code was trying to iterate `SearchResults` object directly:

```python
for result in results:  # Wrong - SearchResults isn't directly iterable
    ...
```

### Fix

Iterate `.edges` and `.nodes` separately:

```python
# Before (broken)
for result in results:
    edges.append(result)

# After (correct)
for edge in results.edges:
    edges.append(edge)
for node in results.nodes:
    nodes.append(node)
```

### Affected Functions

All search functions in `query.py`:
- `search_context()`
- `search_current()`
- `search_at_time()`
- `search_timeline()`
- `get_meeting_prep()`
- `search_all()`

---

## Patch 3: API Parameter Rename

### Problem

Graphiti API changed parameter names:
- `num_results` → removed (uses default or `limit`)
- `search_filters` → `search_filter` (singular)

### Fix

Update all calls to `_search()` method:

```python
# Before
results = await self.client._search(
    query=query,
    num_results=10,
    search_filters=filters
)

# After
results = await self.client._search(
    query=query,
    search_filter=filters
)
```

---

## Patch 4: LLM Duplicate Facts Warning

### Problem

During edge resolution, the LLM sometimes returns out-of-range indices for duplicate facts:

```
LLM returned invalid duplicate_facts idx values
```

### Status

This is a **non-fatal warning** - ingestion still completes. No patch needed, but logged for awareness.

---

## Re-applying Patches After Upgrade

After upgrading Graphiti (`pip install --upgrade graphiti-core`):

1. Check if patches are still needed:
```bash
# Test search
./kg search-current "test query"
```

2. If Pydantic errors occur, re-apply the None guard patch to `edges.py`

3. If iteration errors occur, verify `query.py` uses `.edges` and `.nodes`

4. Document any new issues in this file

---

## Testing Patches

Verify patches are working:

```bash
# Should return results without Pydantic errors
./kg search-current "recent decisions"

# Should show temporal data (valid_at dates)
./kg timeline "project status"

# End-to-end ingestion test
./kg ingest --dry-run
```

Expected output includes:
- `valid_at` dates on every fact
- No Pydantic validation errors
- Expired facts filtered out by default

---

## Version Compatibility

| Graphiti Version | Patches Required |
|------------------|------------------|
| < 0.5.0 | All patches |
| 0.5.x | Verify each patch |
| Future | Re-test after upgrade |

---

## Related Files

- `projects/context-graph/query.py` - Search functions
- `graphiti_core/edges.py` - Edge model (external package)
- `reference/CONTEXT_GRAPH_TEMPORAL_SOP.md` - Temporal usage patterns
- `PATCHES.md` - Quick reference for re-application

---

## Automated Self-Testing Infrastructure

### Problem

Patches can silently break after Graphiti upgrades, API changes, or system updates. Manual discovery of broken pipelines wastes time and creates write-only graphs (data goes in but can't be retrieved).

### Solution: Daily Self-Test Cron

`self_test.py` runs 14 tests daily at 01:30 UTC:

| # | Test | What It Catches |
|---|------|-----------------|
| 1 | Neo4j connection | Database connectivity issues |
| 2 | Graphiti initialization | Client setup failures |
| 3 | Search with temporal filters | Broken search pipeline |
| 4 | Temporal fields present | Missing `valid_at` dates |
| 5 | Timeline search works | Time-range query failures |
| 6 | Entity types importable | Import/module issues |
| 7 | Extraction config + group_id classifier | Ingestion config errors |
| 8 | CLI `kg search-current` end-to-end | Full pipeline test |
| 9 | Episode count sanity check | Data integrity |
| 10 | All episodes have group_ids | Classification issues |
| 11 | Pydantic None guard patch still applied | Patch regression |
| 12 | API compatibility (no deprecated params) | API drift detection |
| 13 | SearchResults iteration | `.edges`/`.nodes` access |
| 14 | Write + read round-trip | End-to-end validation |

### Cron Configuration

```cron
# Daily Graphiti health check - 01:30 UTC
30 1 * * * /path/to/python /path/to/self_test.py >> /var/log/graphiti-selftest.log 2>&1
```

### Self-Healing Behavior

When a test fails:
1. Cron sub-agent attempts automated fix
2. Fix attempt is logged with full context
3. Alert sent with:
   - What broke
   - What fix was attempted
   - Whether fix succeeded
   - Manual intervention needed (if any)

### Log Review

Check logs regularly for patterns:

```bash
# Recent failures
grep -E "(FAIL|ERROR)" /var/log/graphiti-selftest.log | tail -20

# Automated fixes applied
grep "FIX APPLIED" /var/log/graphiti-selftest.log

# Full test history
tail -100 /var/log/graphiti-selftest.log
```

### Adding New Tests

When discovering a new issue:

1. Add test case to `self_test.py`
2. Document the patch in this file
3. Add automated fix logic if possible
4. Update the test count in cron comments

### Key Principle

**Don't wait for manual discovery.** Every known failure mode should have:
- A test that catches it
- A fix that attempts repair
- A log that records what happened
- An alert if human intervention needed

---

## Maintainer Notes

These patches address Graphiti's evolving API. As Graphiti stabilizes, some patches may become unnecessary. Always test after upgrades before assuming patches are still required.

Contact: Check Graphiti GitHub issues for upstream fixes before patching locally.
