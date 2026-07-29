# Technical Specification: Issue #249 (feat: get_engagements_by_tag query)

## 1. Overview
This specification details the design for adding engagement tagging capability and a paginated query interface `get_engagements_by_tag` to the HireSettle smart contract.

## 2. Requirements & Constraints
1. **Creation Configuration**:
   - `EngagementConfig` extended with `pub tags: Option<Vec<String>>`.
   - `Engagement` and `EngagementSummary` updated to include `pub tags: Option<Vec<String>>`.
2. **Validation Rules**:
   - Maximum 10 tags per engagement (`TooManyTags`).
   - Maximum 32 characters per tag (`TagTooLong`).
   - Tag cannot be empty string (`TagEmpty`).
   - Duplicate tags in a single creation call are deduplicated when writing index records.
3. **Storage Indexing**:
   - Storage Key: `DataKey::TagEngagements(String)` (persistent storage).
   - TTL extended using standard persistent TTL (100,000 / 6,300,000 ledgers).
4. **Query Interface**:
   - `get_engagements_by_tag(env: Env, tag: String, page: u32, page_size: u32) -> Vec<String>`
   - `get_tag_engagement_count(env: Env, tag: String) -> u32`
   - Zero `page_size` or out-of-bounds `page` returns an empty `Vec<String>`.

## 3. Data Structures
```rust
pub enum DataKey {
    // ...
    TagEngagements(String),
}

pub struct EngagementConfig {
    // ...
    pub tags: Option<Vec<String>>,
}
```
