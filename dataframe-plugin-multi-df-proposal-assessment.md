# Multi-DataFrame Plugin: Proposal Assessment & Implementation Plan

**Date:** 2026-01-04
**Issue:** [#3411 - New dataframe transform: join (left, right)](https://github.com/marimo-team/marimo/issues/3411)
**Proposal:** Dataframe plugin with access to all notebook dataframes, transform selection, and optional joins

---

## Executive Summary

This document provides a comprehensive analysis of implementing multi-dataframe functionality in marimo. After thorough exploration of the current architecture and evaluation of three implementation approaches, we recommend **creating a new `ui.dataframe_explorer` widget** as a purpose-built tool for multi-dataframe workflows.

**Key Findings:**
- Current `ui.dataframe` is designed for single-dataframe operations
- Infrastructure exists (RuntimeContext.globals) to access all notebook variables
- A separate explorer widget provides the best balance of simplicity and power
- Implementation timeline: 15 weeks to production-ready v1.0

---

## Table of Contents

1. [Issue Context](#issue-context)
2. [Current Architecture Analysis](#current-architecture-analysis)
3. [Proposed Solutions Comparison](#proposed-solutions-comparison)
4. [Recommended Approach](#recommended-approach)
5. [Technical Architecture](#technical-architecture)
6. [Implementation Plan](#implementation-plan)
7. [Risk Assessment](#risk-assessment)
8. [Success Metrics](#success-metrics)

---

## Issue Context

### Original Request
**Issue #3411** requests the ability to "perform basic joins by primary keys within marimo's data transform feature."

**Current Limitation:**
The dataframe plugin only accepts a single dataframe as input, making joins impossible.

### Enhanced Proposal
The user proposes extending this to:
- Dataframe plugin accesses all dataframes in the notebook
- Users can select and apply transforms to multiple dataframes
- Optionally join results or raw tables
- Could be a completely new plugin if more appropriate

---

## Current Architecture Analysis

### Core Components

**Location:** `/marimo/_plugins/ui/_impl/dataframes/`

**Key Files:**
- `dataframe.py` - Main widget (346 lines)
- `transforms/types.py` - 11 transform types
- `transforms/apply.py` - Transform execution engine
- `transforms/handlers.py` - Narwhals-based implementations

### Existing Transform Types (11)
1. Column Conversion
2. Rename Column
3. Sort Column
4. Filter Rows
5. Group By
6. Aggregate
7. Select Columns
8. Shuffle Rows
9. Sample Rows
10. Explode Columns
11. Unique/Distinct

### Architecture Strengths
- **Narwhals Integration:** Unified interface for Pandas, Polars, PyArrow, Ibis, DuckDB
- **Incremental Transforms:** Snapshot caching for performance
- **Type Safety:** Generic types separate frontend (S) from Python (T) values
- **RPC Communication:** Function registry enables clean frontend/backend separation
- **RuntimeContext:** Already provides access to all notebook variables via `ctx.globals`

### Key Discovery: Variable Access Patterns

The infrastructure to access multiple dataframes already exists:

```python
from marimo._runtime.context import get_context

ctx = get_context()
all_variables = ctx.globals  # dict[str, Any]
other_df = ctx.globals.get("dataframe_name")
```

**Implication:** We don't need to reinvent variable access—the runtime already supports it.

---

## Proposed Solutions Comparison

### Option A: Extend Existing ui.dataframe

**API Design:**
```python
# Backward compatible
df_widget = mo.ui.dataframe(df)

# New multi-dataframe API
df_widget = mo.ui.dataframe(
    [df1, df2, df3],
    names=["sales", "customers", "products"]
)

# Auto-discovery
df_widget = mo.ui.dataframe(auto_discover=True)
```

**Pros:**
- ✅ Backward compatible
- ✅ Unified interface - one API to learn
- ✅ Reuses existing infrastructure
- ✅ Simpler maintenance (one codebase)
- ✅ Natural evolution of existing widget

**Cons:**
- ❌ API complexity - overloaded constructor
- ❌ Type confusion - `df: DataFrameType | list[DataFrameType]`
- ❌ UI complexity - need dataframe selector in transform panel
- ❌ Feature bloat - widget doing too much
- ❌ Performance concerns - loading multiple large dataframes
- ❌ Cognitive load - mixing simple viewing with complex operations

**Assessment:** Violates single responsibility principle; increases complexity for all users even those with simple use cases.

---

### Option B: Create New ui.dataframe_explorer

**API Design:**
```python
# New purpose-built widget
explorer = mo.ui.dataframe_explorer({
    "sales": df_sales,
    "customers": df_customers,
    "products": df_products
})

# Auto-discovery
explorer = mo.ui.dataframe_explorer(auto_discover=True)

# Access result
result_df = explorer.value
```

**Pros:**
- ✅ Separation of concerns - simple viewing vs complex exploration
- ✅ Clean API - purpose-built for multi-dataframe workflows
- ✅ Type safety - no overloading
- ✅ Richer features - room for exploration-specific functionality
- ✅ Performance control - lazy-load dataframes
- ✅ Future-proof - can grow without affecting simple use cases
- ✅ Clear UX - users know when to use which widget
- ✅ Aligns with marimo philosophy - batteries-included but simple when simple

**Cons:**
- ❌ Code duplication - some shared logic with ui.dataframe
- ❌ Learning curve - two widgets to learn
- ❌ Maintenance burden - two widgets to maintain
- ❌ Discovery - users may not know explorer exists

**Assessment:** Best aligns with marimo's philosophy. Provides power without sacrificing simplicity.

---

### Option C: Hybrid - ui.dataframe.with_context()

**API Design:**
```python
# Simple single-dataframe (unchanged)
df_widget = mo.ui.dataframe(df_sales)

# Enable multi-dataframe mode via method
explorer = mo.ui.dataframe(df_sales).with_context(
    additional={"customers": df_customers}
)

# Alternative: factory method
explorer = mo.ui.dataframe.multi(
    primary=df_sales,
    context={"customers": df_customers}
)
```

**Pros:**
- ✅ Progressive enhancement - start simple, add complexity when needed
- ✅ Clear upgrade path - natural progression
- ✅ Shared codebase - inherits functionality
- ✅ Discoverable - IDE autocomplete reveals `.with_context()`
- ✅ Backward compatible

**Cons:**
- ❌ API complexity - method chaining may be unfamiliar
- ❌ Implementation complexity - careful inheritance needed
- ❌ Return type confusion - `.with_context()` returns different type
- ❌ Dual personality - dataframe_context is both a dataframe and not

**Assessment:** Interesting middle ground but adds API complexity without clear benefits over Option B.

---

## Recommended Approach

### 🏆 Option B: New ui.dataframe_explorer

**Rationale:**

#### 1. Alignment with Marimo Philosophy
- **Simple for simple cases:** `ui.dataframe` remains lightweight for viewing/transforming single dataframes
- **Powerful when needed:** `ui.dataframe_explorer` for complex multi-dataframe workflows
- **Batteries-included:** Adds powerful data exploration without complexity for basic use
- **Clear mental model:** Two tools, two use cases, no confusion
- **No hidden state:** Each widget manages its own state explicitly

#### 2. Technical Advantages
- **Clean architecture:** No overloaded constructors or type confusion
- **Performance:** Can optimize differently (lazy loading, progressive rendering)
- **Extensibility:** Room for exploration-specific features without affecting simple widget
- **Maintainability:** Clear boundaries make code easier to reason about

#### 3. User Experience
- **Discovery:** Clear documentation on when to use each widget
- **Learning:** Start with simple widget, graduate to explorer for advanced needs
- **Power:** Explorer can have richer UI without cluttering simple widget

#### 4. Future Features
The explorer can eventually support:
- Automatic relationship detection
- Data profiling across dataframes
- Visual query builder
- Schema visualization
- Data lineage tracking
- Template pipelines for common patterns

---

## Technical Architecture

### High-Level Component Structure

```
Backend:
/marimo/_plugins/ui/_impl/dataframe_explorer/
├── explorer.py                    # Main UIElement class
├── dataframe_manager.py          # Manages multiple dataframes
├── multi_transforms/
│   ├── join.py                   # JOIN transform
│   ├── concat.py                 # CONCAT transform
│   └── union.py                  # UNION transform
└── pipeline.py                   # Transform pipeline manager

Frontend:
/frontend/src/plugins/impl/dataframe-explorer/
├── ExplorerPlugin.tsx            # Main React component
├── components/
│   ├── DataFrameSelector.tsx    # Dataframe selection UI
│   ├── TransformBuilder.tsx     # Transform configuration
│   ├── PipelineViewer.tsx       # Visual pipeline representation
│   └── SchemaViewer.tsx         # Schema comparison
└── forms/
    ├── JoinFormRenderer.tsx      # Join configuration form
    ├── ConcatFormRenderer.tsx    # Concat configuration form
    └── UnionFormRenderer.tsx     # Union configuration form
```

### Core Backend API

```python
class dataframe_explorer(UIElement[dict[str, Any], DataFrameType]):
    """Multi-dataframe exploration and transformation widget.

    Examples:
        # Explicit dataframes
        explorer = mo.ui.dataframe_explorer({
            "sales": sales_df,
            "customers": customers_df
        })

        # Auto-discover from globals
        explorer = mo.ui.dataframe_explorer(auto_discover=True)

        # Access result
        result = explorer.value
    """

    def __init__(
        self,
        dataframes: dict[str, DataFrameType] | None = None,
        *,
        auto_discover: bool = False,
        include_patterns: list[str] | None = None,
        exclude_patterns: list[str] | None = None,
        page_size: int = 10,
        show_download: bool = True,
        lazy: bool | None = None,
        on_change: Optional[Callable[[DataFrameType], None]] = None
    ):
        # Implementation
```

### New Transform Types

#### 1. JOIN Transform
```python
@dataclass
class JoinTransform:
    type: Literal[TransformType.JOIN]
    left_source: str          # Dataframe identifier or "result"
    right_source: str
    left_on: list[str]        # Join columns from left
    right_on: list[str]       # Join columns from right
    how: Literal["inner", "left", "right", "outer", "cross", "semi", "anti"]
    suffix: tuple[str, str] = ("_x", "_y")
```

**Supported Join Types:**
- **Inner:** Return rows with matches in both tables
- **Left:** All rows from left, matched rows from right
- **Right:** All rows from right, matched rows from left
- **Outer:** All rows from both tables
- **Cross:** Cartesian product
- **Semi:** Rows from left that have matches in right (no columns from right)
- **Anti:** Rows from left that don't have matches in right

**Narwhals Support:** ✅ Full support via `.join()`

#### 2. CONCAT Transform
```python
@dataclass
class ConcatTransform:
    type: Literal[TransformType.CONCAT]
    sources: list[str]              # Dataframe identifiers
    axis: Literal[0, 1]             # 0=vertical, 1=horizontal
    join: Literal["inner", "outer"] = "outer"
    ignore_index: bool = True
```

**Use Cases:**
- **Vertical (axis=0):** Stack dataframes with similar columns (union-like)
- **Horizontal (axis=1):** Combine dataframes side-by-side

**Narwhals Support:** ✅ Via `nw.concat()`

#### 3. UNION Transform
```python
@dataclass
class UnionTransform:
    type: Literal[TransformType.UNION]
    sources: list[str]
    deduplicate: bool = True
```

**Difference from CONCAT:**
- Automatically deduplicates rows (UNION vs UNION ALL semantics)
- Requires matching schemas

**Narwhals Support:** ✅ Via `nw.concat()` + `unique()`

### Data Flow Architecture

```
┌─────────┐  ┌─────────┐  ┌─────────┐
│  DF 1   │  │  DF 2   │  │  DF 3   │
└────┬────┘  └────┬────┘  └────┬────┘
     │            │            │
     └────────────┴────────────┘
                  │
                  ▼
         ┌────────────────┐
         │ DataFrameMgr   │
         │ - sources: {}  │
         │ - metadata: {} │
         └────┬───────────┘
              │
              ▼
    ┌──────────────────────┐
    │ Transform Pipeline   │
    │                      │
    │ 1. Filter (DF1)      │
    │ 2. JOIN (DF1+DF2)    │
    │ 3. Group (result)    │
    └──────┬───────────────┘
           │
           ▼
      ┌─────────┐
      │ Result  │
      └─────────┘
```

### Transform Pipeline Execution

```
Initial State:
  sources = {"sales": LazyFrame, "customers": LazyFrame}
  result = None

Transform 1: Filter(df="sales", column="amount", op=">", value=100)
  → result = sources["sales"].filter(col("amount") > 100)

Transform 2: Join(left="result", right="customers", on="customer_id")
  → left = result (or sources["sales"] if result is None)
  → right = sources["customers"]
  → result = left.join(right, on="customer_id")

Transform 3: GroupBy(columns=["country"], agg="sum")
  → result = result.group_by("country").agg(sum("amount"))

Final: result.collect()
```

### UI/UX Design

#### Layout Structure
```
┌─────────────────────────────────────────────────────────────┐
│  DataFrames: [sales ▼] [customers] [products]    [+ Add]   │
├─────────────────────────────────────────────────────────────┤
│  Transform Pipeline:                                         │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ 1. Filter (sales): amount > 100                      │  │
│  │ 2. Join: sales ⟕ customers on customer_id           │  │
│  │ 3. Group By: country, Aggregate: sum(amount)         │  │
│  │ [+ Add Transform ▼]                                  │  │
│  └──────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────────┤
│  Preview: (10 / 1,234 rows)                                │
│  ┌──────────┬──────────┬──────────┬──────────┐            │
│  │ country  │ customer │ total    │ ...      │            │
│  ├──────────┼──────────┼──────────┼──────────┤            │
│  │ USA      │ ACME     │ 10000    │          │            │
│  │ UK       │ TechCo   │ 8500     │          │            │
│  └──────────┴──────────┴──────────┴──────────┘            │
├─────────────────────────────────────────────────────────────┤
│  [Copy Python Code] [Copy SQL] [Download ▼] [Apply]       │
└─────────────────────────────────────────────────────────────┘
```

#### Join Transform UI
```
Add Transform: [Join ▼]

┌─────────────────────────────────────────────────────────┐
│ Join Configuration                                      │
│                                                         │
│ Left DataFrame:  [sales ▼]                            │
│ Left Column(s):  [customer_id ▼]  [+ Add Column]      │
│                                                         │
│ Right DataFrame: [customers ▼]                        │
│ Right Column(s): [id ▼]           [+ Add Column]      │
│                                                         │
│ Join Type: ◉ Inner  ○ Left  ○ Right  ○ Outer         │
│                                                         │
│ Handle Duplicate Columns:                              │
│   Suffix: [_left] [_right]                            │
│                                                         │
│ [Cancel] [Add Transform]                               │
└─────────────────────────────────────────────────────────┘
```

---

## Implementation Plan

### Phase 1: Foundation (Weeks 1-2)
**Goal:** Core infrastructure for multi-dataframe support

**Backend:**
- [ ] Create `/marimo/_plugins/ui/_impl/dataframe_explorer/` directory
- [ ] Implement `DataFrameManager` class
- [ ] Implement auto-discovery from `RuntimeContext.globals`
- [ ] Add multi-dataframe transform types to `types.py`
- [ ] Create basic `dataframe_explorer` UIElement class
- [ ] Set up RPC functions

**Frontend:**
- [ ] Create `/frontend/src/plugins/impl/dataframe-explorer/` structure
- [ ] Implement basic `ExplorerPlugin.tsx`
- [ ] Create dataframe selector UI
- [ ] Transform pipeline UI skeleton

**Tests:**
- [ ] Unit tests for `DataFrameManager`
- [ ] Auto-discovery tests
- [ ] Basic integration tests

**Deliverable:** Basic explorer widget displaying multiple dataframes

---

### Phase 2: JOIN Transform (Weeks 3-4)
**Goal:** Full JOIN functionality

**Backend:**
- [ ] `JoinTransform` dataclass
- [ ] `JoinHandler` with Narwhals
- [ ] All join types (inner, left, right, outer, cross, semi, anti)
- [ ] Column conflict handling with suffixes
- [ ] Python/SQL code generation for joins

**Frontend:**
- [ ] `JoinFormRenderer.tsx`
- [ ] Column selector for joins
- [ ] Join type selector
- [ ] Suffix configuration UI
- [ ] Join preview/validation
- [ ] Suggested joins feature

**Tests:**
- [ ] Unit tests for all join types
- [ ] Column conflict tests
- [ ] Cross-library join tests (Pandas + Polars)
- [ ] Frontend form tests

**Deliverable:** Fully functional JOIN transform

---

### Phase 3: CONCAT & UNION (Weeks 5-6)
**Goal:** Vertical and horizontal concatenation

**Backend:**
- [ ] `ConcatTransform` dataclass
- [ ] `UnionTransform` dataclass
- [ ] Narwhals-based concat handler
- [ ] Axis parameter handling (0 vs 1)
- [ ] Column alignment logic
- [ ] Deduplication for UNION
- [ ] Code generation

**Frontend:**
- [ ] `ConcatFormRenderer.tsx`
- [ ] Multi-dataframe selector
- [ ] Axis selector (vertical/horizontal)
- [ ] Join type selector (inner/outer)
- [ ] Preview component

**Tests:**
- [ ] Vertical concat tests
- [ ] Horizontal concat tests
- [ ] Union with deduplication
- [ ] Column mismatch handling

**Deliverable:** CONCAT and UNION transforms functional

---

### Phase 4: Transform Pipeline (Week 7)
**Goal:** Multi-step pipeline with mixed single/multi transforms

**Backend:**
- [ ] `MultiTransformPipeline` class
- [ ] Pipeline validation logic
- [ ] Dependency resolution
- [ ] Incremental execution with caching
- [ ] Handle "result" as transform source
- [ ] Pipeline serialization

**Frontend:**
- [ ] Pipeline visualization
- [ ] Drag-to-reorder transforms
- [ ] Transform insertion/deletion
- [ ] Validation feedback
- [ ] Dataflow diagram

**Tests:**
- [ ] Complex pipeline tests
- [ ] Dependency validation
- [ ] Execution order tests
- [ ] Cache invalidation tests

**Deliverable:** Complete transform pipeline system

---

### Phase 5: UX Polish (Week 8)
**Goal:** Refinement and user experience improvements

**Tasks:**
- [ ] Relationship detection algorithm
- [ ] Schema viewer component
- [ ] Data profiling (row counts, types)
- [ ] Error handling and feedback
- [ ] Loading states and progress
- [ ] Performance optimization (lazy loading)
- [ ] Keyboard shortcuts
- [ ] Accessibility (ARIA labels, keyboard nav)
- [ ] Responsive design
- [ ] Dark mode support

**Deliverable:** Polished, production-ready UI

---

### Phase 6: Documentation & Examples (Week 9)
**Goal:** Comprehensive documentation

**Tasks:**
- [ ] API documentation (docstrings)
- [ ] User guide: "When to use explorer vs dataframe"
- [ ] Tutorial: "Multi-dataframe workflows"
- [ ] Cookbook examples:
  - Sales analysis with multiple tables
  - Data integration from multiple sources
  - Comparing datasets
- [ ] Video walkthrough
- [ ] FAQ section

**Deliverable:** Full documentation suite

---

### Phase 7: Testing & Hardening (Week 10)
**Goal:** Comprehensive testing and edge case handling

**Tasks:**
- [ ] Edge case testing (empty dfs, single row, etc.)
- [ ] Performance testing with large datasets
- [ ] Memory profiling
- [ ] Cross-browser testing
- [ ] Mobile responsiveness
- [ ] Accessibility audit
- [ ] Security review (SQL injection prevention)
- [ ] Load testing (10+ dataframes)
- [ ] Integration with marimo ecosystem

**Deliverable:** Battle-tested, robust implementation

---

### Phase 8: Beta Release (Week 11)
**Goal:** Soft launch for feedback

**Tasks:**
- [ ] Mark as experimental in docs
- [ ] Beta announcement (Discord, blog)
- [ ] User feedback collection
- [ ] Usage pattern monitoring
- [ ] Bug triage and fixes
- [ ] Performance metrics collection

**Deliverable:** Beta release with active feedback loop

---

### Phase 9: Iteration & Refinement (Weeks 12-14)
**Goal:** Address feedback and add requested features

**Potential Features:**
- [ ] Advanced SQL editor mode
- [ ] Custom transform scripting
- [ ] Visual relationship graph
- [ ] Data quality checks
- [ ] Export pipeline as reusable function
- [ ] Template pipelines (common patterns)

**Deliverable:** Refined, feature-complete v1.0

---

### Phase 10: v1.0 Release (Week 15)
**Goal:** Production release

**Tasks:**
- [ ] Remove experimental flag
- [ ] Final documentation review
- [ ] Release notes
- [ ] Blog post
- [ ] Video demo
- [ ] Social media announcement
- [ ] Update gallery with examples

**Deliverable:** Production-ready `ui.dataframe_explorer` v1.0

---

## Edge Cases and Challenges

### 1. Column Name Conflicts
**Challenge:** After join, columns from both tables may have same names

**Solution:**
- Default suffixes ("_x", "_y")
- User-configurable suffixes
- Automatic detection and warning in UI
- Preview shows final column names

### 2. Type Mismatches in Joins
**Challenge:** Joining on columns with incompatible types

**Solution:**
- Pre-join type validation
- Automatic type coercion suggestions
- Clear error messages with fix suggestions
- Option to add implicit cast transform

### 3. Memory Management
**Challenge:** Loading multiple large dataframes

**Solution:**
- Lazy evaluation by default (Narwhals LazyFrames)
- Progressive loading with limits
- Warning when total size exceeds threshold
- Streaming preview for large results

### 4. Transform Order Dependencies
**Challenge:** Join result referenced before join executes

**Solution:**
- Pipeline validates dependencies
- UI shows dataflow clearly
- Cannot select "result" until multi-df transform applied
- Clear error messages for invalid pipelines

### 5. Auto-Discovery Ambiguity
**Challenge:** What counts as a "dataframe"?

**Solution:**
```python
def is_dataframe(obj: Any) -> bool:
    return (
        can_narwhalify(obj) and
        not isinstance(obj, (list, dict, set)) and
        hasattr(obj, '__len__') and
        len(obj) > 0
    )
```

### 6. Schema Inference After Joins
**Challenge:** Column types may change after operations

**Solution:**
- Re-infer schema after each transform
- Cache schema per transform step
- Update UI column selectors dynamically

### 7. Code Generation Complexity
**Challenge:** Generating readable Python/SQL for complex pipelines

**Solution:**
- Template-based code generation
- Comments explaining each step
- Option to export as function

### 8. Relationship Detection Accuracy
**Challenge:** False positives in auto-detected relationships

**Solution:**
- Heuristics: exact name match, "_id" suffix, same type
- Show as "suggestions" not auto-applied
- Allow user to dismiss/accept

### 9. Performance with Many DataFrames
**Challenge:** UI becomes unwieldy with 10+ dataframes

**Solution:**
- Search/filter dataframe list
- Group by prefix/pattern
- Show only "active" dataframes in pipeline
- Collapse/expand sections

### 10. Backward Compatibility
**Challenge:** Ensuring `ui.dataframe` isn't affected

**Solution:**
- Completely separate implementation
- Shared utilities extracted to common module
- No changes to existing `ui.dataframe` class
- Integration tests for both widgets

---

## Testing Strategy

### Unit Tests
**Backend:**
- `test_dataframe_manager.py` - Auto-discovery, relationship detection
- `test_join_transform.py` - All join types, suffixes
- `test_concat_transform.py` - Vertical/horizontal concat
- `test_union_transform.py` - Union with deduplication
- `test_multi_pipeline.py` - Complex pipelines, dependencies

**Frontend:**
- `ExplorerPlugin.test.tsx` - Component rendering
- `JoinForm.test.tsx` - Form validation
- `PipelineViewer.test.tsx` - Pipeline visualization

### Integration Tests
```python
def test_full_workflow():
    """End-to-end: create explorer, apply transforms, get result"""
    explorer = mo.ui.dataframe_explorer({
        "df1": pd.DataFrame({"a": [1, 2], "id": [1, 2]}),
        "df2": pd.DataFrame({"b": [3, 4], "id": [1, 2]})
    })

    # Simulate join from frontend
    explorer._convert_value({
        "transforms": [{
            "type": "join",
            "left_source": "df1",
            "right_source": "df2",
            "left_on": ["id"],
            "right_on": ["id"],
            "how": "inner"
        }]
    })

    result = explorer.value
    assert "a" in result.columns
    assert "b" in result.columns
```

### Performance Tests
- Large dataframe performance (1M+ rows)
- Multiple dataframes memory usage (10+ dataframes)
- Lazy evaluation verification

### Cross-Library Tests
```python
@pytest.mark.parametrize("df_type", ["pandas", "polars", "pyarrow"])
def test_join_across_libraries(df_type):
    """Test joining dataframes of different types"""
```

### Visual Regression Tests
- Screenshot matching for UI components

---

## Risk Assessment

### Technical Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Performance degradation with large datasets | High | Lazy evaluation, pagination, warnings |
| Memory issues with multiple dataframes | High | Lazy loading, streaming, memory limits |
| Narwhals compatibility issues | Medium | Comprehensive cross-library tests |
| Frontend bundle size increase | Medium | Code splitting, lazy component loading |
| Transform pipeline complexity | Medium | Clear validation, helpful error messages |

### User Experience Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Discovery - users don't know explorer exists | Medium | Clear docs, examples, tutorial |
| Confusion about when to use which widget | Medium | Decision guide in documentation |
| Learning curve for multi-df operations | Low | Progressive disclosure, tooltips |
| Complex UI overwhelming users | Low | Clean design, hide advanced features |

### Project Risks

| Risk | Severity | Mitigation |
|------|----------|------------|
| Scope creep | Medium | Strict phase boundaries, MVP focus |
| Timeline delays | Medium | Buffer weeks, phased rollout |
| Maintenance burden | Low | Clean architecture, good tests |
| Feature divergence from ui.dataframe | Low | Extract shared utilities |

---

## Success Metrics

### Adoption Metrics
- **Target:** 20% of notebooks with multiple dataframes use explorer within 3 months
- **Measure:** Usage telemetry (opt-in)

### User Satisfaction
- **Target:** 4.5/5 average rating in user surveys
- **Measure:** Post-release survey

### Performance Benchmarks
- **Target:** <200ms for join transform on 10K rows
- **Target:** <100MB memory overhead for 5 dataframes
- **Measure:** Automated performance tests

### Code Quality
- **Target:** >90% test coverage
- **Target:** <10 bugs per 1000 lines of code
- **Measure:** Coverage reports, issue tracking

### Documentation
- **Target:** 100% API coverage in docs
- **Target:** ≥5 comprehensive examples
- **Measure:** Doc review, example gallery

---

## Future Enhancements (Post v1.0)

### Advanced Features
1. **Visual Query Builder** - Drag-and-drop pipeline construction
2. **Data Lineage Tracking** - Show data provenance through transforms
3. **Automatic Optimization** - Query optimization suggestions
4. **Collaborative Pipelines** - Share and reuse transform pipelines
5. **Template Library** - Common pipeline patterns (ETL, analysis)
6. **Integration with mo.sql** - Seamless SQL query integration
7. **Data Quality Checks** - Built-in validation and profiling
8. **Custom Transforms** - User-defined transform functions
9. **Pipeline Export** - Save pipeline as reusable Python function
10. **Real-time Updates** - Live data refresh and streaming

### Performance Enhancements
1. **Distributed Execution** - Dask/Ray backend for large-scale data
2. **Query Pushdown** - Optimize operations at database level
3. **Incremental Computation** - Only recompute affected parts
4. **Smart Caching** - Predictive cache warming

### AI Features
1. **AI-Suggested Joins** - ML-based relationship detection
2. **Natural Language Queries** - "Join sales and customers on ID"
3. **Anomaly Detection** - Highlight data quality issues
4. **Auto-profiling** - Automatic data analysis and insights

---

## Migration Strategy

### Rollout Plan

**Phase 1: Introduction (v0.1)**
- Launch `ui.dataframe_explorer` as experimental
- Documentation: "For advanced multi-dataframe workflows"
- `ui.dataframe` unchanged
- Examples in gallery

**Phase 2: Maturation (v0.2-0.3)**
- User feedback incorporation
- Feature additions
- Performance optimization
- API stabilization

**Phase 3: Promotion (v0.4+)**
- Graduate from experimental
- Full documentation
- Tutorial/cookbook
- Consider: `ui.dataframe.explore()` shortcut

### No Deprecation
- `ui.dataframe` and `ui.dataframe_explorer` coexist permanently
- Clear documentation on use cases:
  - **ui.dataframe:** Single dataframe viewing/transformation
  - **ui.dataframe_explorer:** Multi-dataframe analysis, joins, pipelines

---

## Critical Implementation Files

### Backend (Priority Order)

1. **`/marimo/_plugins/ui/_impl/dataframe_explorer/explorer.py`**
   - Core widget implementation
   - Main entry point for multi-dataframe functionality
   - ~400-500 lines estimated

2. **`/marimo/_plugins/ui/_impl/dataframe_explorer/dataframe_manager.py`**
   - Manages multiple dataframes
   - Auto-discovery logic
   - Schema tracking
   - ~250-300 lines estimated

3. **`/marimo/_plugins/ui/_impl/dataframe_explorer/multi_transforms/join.py`**
   - JOIN transform implementation
   - Most complex multi-df operation
   - ~200-250 lines estimated

4. **`/marimo/_plugins/ui/_impl/dataframe_explorer/pipeline.py`**
   - Transform pipeline manager
   - Dependency resolution
   - Incremental execution
   - ~300-350 lines estimated

5. **`/marimo/_plugins/ui/_impl/dataframes/transforms/types.py`** (extend)
   - Add multi-dataframe transform types
   - JOIN, CONCAT, UNION definitions
   - ~100-150 lines added

### Frontend (Priority Order)

1. **`/frontend/src/plugins/impl/dataframe-explorer/ExplorerPlugin.tsx`**
   - Main React component
   - Coordinates multi-dataframe UI
   - ~800-1000 lines estimated

2. **`/frontend/src/plugins/impl/dataframe-explorer/components/TransformBuilder.tsx`**
   - Transform configuration UI
   - Form rendering logic
   - ~400-500 lines estimated

3. **`/frontend/src/plugins/impl/dataframe-explorer/forms/JoinFormRenderer.tsx`**
   - Join-specific form
   - Column selection, join type
   - ~300-400 lines estimated

4. **`/frontend/src/plugins/impl/dataframe-explorer/components/PipelineViewer.tsx`**
   - Visual pipeline representation
   - Transform list, reordering
   - ~250-300 lines estimated

5. **`/frontend/src/plugins/impl/dataframe-explorer/components/DataFrameSelector.tsx`**
   - Dataframe selection dropdown
   - ~150-200 lines estimated

---

## Conclusion

The proposed `ui.dataframe_explorer` widget represents a natural evolution of marimo's data transformation capabilities. By creating a dedicated tool for multi-dataframe workflows, we maintain the simplicity of the existing `ui.dataframe` while providing powerful exploration features for advanced users.

**Key Takeaways:**
1. ✅ **Technically Feasible** - Infrastructure exists via RuntimeContext.globals
2. ✅ **Architecturally Sound** - Clean separation of concerns
3. ✅ **User-Friendly** - Progressive disclosure, clear use cases
4. ✅ **Future-Proof** - Room for advanced features without bloat
5. ✅ **Aligned with Philosophy** - Simple by default, powerful when needed

**Recommendation:** Proceed with Option B implementation following the 15-week phased roadmap.

---

## Appendix: Example Usage

### Basic Join
```python
import marimo as mo
import pandas as pd

# Sample data
sales = pd.DataFrame({
    "order_id": [1, 2, 3],
    "customer_id": [101, 102, 101],
    "amount": [250, 150, 300]
})

customers = pd.DataFrame({
    "id": [101, 102, 103],
    "name": ["Acme Corp", "TechCo", "DataInc"],
    "country": ["USA", "UK", "USA"]
})

# Create explorer
explorer = mo.ui.dataframe_explorer({
    "sales": sales,
    "customers": customers
})

# User applies join transform via UI
# Result available as:
result = explorer.value
```

### Auto-Discovery
```python
# Multiple dataframes in notebook
df_sales_2023 = pd.read_csv("sales_2023.csv")
df_sales_2024 = pd.read_csv("sales_2024.csv")
df_customers = pd.read_csv("customers.csv")
df_products = pd.read_csv("products.csv")

# Auto-discover all dataframes
explorer = mo.ui.dataframe_explorer(
    auto_discover=True,
    include_patterns=["df_*"],  # Only dataframes starting with df_
    exclude_patterns=["*_temp"]  # Exclude temporary dataframes
)

# Explorer shows all discovered dataframes
# User can build pipeline in UI
```

### Complex Pipeline
```python
# Create explorer with multiple sources
explorer = mo.ui.dataframe_explorer({
    "sales": sales_df,
    "customers": customers_df,
    "products": products_df
})

# User builds pipeline via UI:
# 1. Filter sales: amount > 100
# 2. Join sales with customers on customer_id
# 3. Join result with products on product_id
# 4. Group by country, product_category
# 5. Aggregate: sum(amount), count(order_id)

# Final result
summary = explorer.value
```

---

**Document Version:** 1.0
**Last Updated:** 2026-01-04
**Author:** Claude (Anthropic)
**Status:** Proposal - Awaiting Approval
