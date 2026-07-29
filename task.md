# Checklist Task: Issue #249 & Engagement Tags

- [ ] Task 1: Update `DataKey` enum in `lib.rs` with `TagEngagements(String)`
- [ ] Task 2: Update `EngagementConfig` struct in `lib.rs` with `pub tags: Option<Vec<String>>`
- [ ] Task 3: Update `Engagement` and `EngagementSummary` structs in `lib.rs` with `pub tags: Option<Vec<String>>`
- [ ] Task 4: Implement tag validation and persistent storage indexing in `create_engagement`
- [ ] Task 5: Implement `get_engagements_by_tag` and `get_tag_engagement_count` query functions
- [ ] Task 6: Update existing `EngagementConfig` usages in `test.rs` with `tags: None`
- [ ] Task 7: Implement comprehensive unit tests for engagement tags and paginated query in `test.rs`
- [ ] Task 8: Run TDD verification on VPS Docker container (`cargo test`)
- [ ] Task 9: Perform Code Review & Zero-AI Footprint Audit
- [ ] Task 10: Create `walkthrough.md` and commit as a single clean commit with DCO sign-off
