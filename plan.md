# Execution Plan: Issue #249

## Phase 1: Data Model & Storage Extension
- Add `TagEngagements(String)` to `DataKey`.
- Update `EngagementConfig`, `Engagement`, and `EngagementSummary` structs with `tags: Option<Vec<String>>`.

## Phase 2: Creation & Indexing Logic
- Add validation logic for tags in `create_engagement`.
- Implement per-tag persistent indexing with deduplication.

## Phase 3: Query Interface
- Implement `get_engagements_by_tag` with saturating arithmetic pagination.
- Implement `get_tag_engagement_count`.

## Phase 4: Test Suite & TDD Verification
- Write tests for tags, pagination, limits, and edge cases in `test.rs`.
- Verify full test suite execution in VPS Docker environment.
