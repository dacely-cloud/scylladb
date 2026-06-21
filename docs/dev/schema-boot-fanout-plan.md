# Schema Boot Fanout And Startup Backlog Plan

## Failure Signature

Observed on a single-shard developer node with many stale non-system keyspaces:

- process stuck in systemd `activating (start)`
- status stuck at `loading non-system sstables`
- native CQL never registered on port 9042
- Scylla metrics showed `system` reads with tens of thousands queued
- disk queues were idle: `disk_reads=0`, `sstables_read=0`
- SIGQUIT diagnostics showed schema reads waiting behind system read admission

The stale keyspaces were the trigger. The bug was not row cache pressure. Boot
could enqueue too much schema, compaction, and audit preprocessing work before
CQL was available.

## Root Cause

Startup blocks before CQL registration in:

- `replica/database.cc`: `parse_system_tables()`
- `replica/distributed_loader.cc`: `init_non_system_keyspaces()`
- regular compaction submission during table load
- group0 audit known-entity synchronization

The loader launched one schema load fiber per keyspace, then multiplied that by
table-level schema work. Separately, regular compaction submission could retain
thousands of task objects, and audit preprocessing eagerly copied all known
table names even when exact per-request rule evaluation would have been enough.

## Implemented Fixes

### Schema Boot Fanout

The boot schema loader now uses named throttle helpers in
`replica/schema_boot.hh`:

- `max_concurrent_keyspace_schema_partitions`
- `max_concurrent_keyspace_population`
- `max_concurrent_tables_per_keyspace`
- `max_concurrent_views_per_keyspace`

`replica/database.cc` and `replica/distributed_loader.cc` use those helpers
instead of unbounded `coroutine::parallel_for_each()` for keyspace schema reads,
keyspace population, table add/column-mapping checks, and view add operations.
All schema work is still performed; only retained/in-flight work is bounded.

### Column-Mapping Probe

The boot path now checks column-mapping existence with a `LIMIT 1` query while
keeping internal query caching enabled. It no longer reads a full mapping row
set when it only needs to know whether at least one mapping exists.

### Regular Compaction Submission Backlog

`compaction_manager::submit()` now has a hard retained-task cap for regular
compaction submissions:

- `regular_compaction_task_backlog_limit()`
- `regular_compaction_task_backlog_full()`

When the cap is hit, more regular submissions are postponed instead of
allocating more task objects. Task completion signals reevaluation, and
postponed work drains only while the cap has room.

Metrics:

- `scylla_compaction_manager_regular_compaction_task_backlog`
- `scylla_compaction_manager_regular_compaction_task_backlog_limit`
- `scylla_compaction_manager_postponed_compactions`

### Audit Known-Table Cache

Audit rule preprocessing now keeps the eager known-table cache only when audit
rules are non-empty and table count is at or below
`max_preprocessed_known_tables`.

Above the cap, group0 skips constructing the known-table hash set. Audit matching
remains exact through the existing per-request rule evaluation path; only the
startup optimization cache is skipped.

## Regression Tests

- `test_boot_schema_fanout_helpers_throttle_work_without_dropping_items`
- `regular_compaction_submission_backlog_is_bounded_test`
- `test_preprocessed_large_known_table_set_uses_bounded_lazy_path`

These verify that schema helpers throttle without dropping work, compaction
submission cannot retain more task objects than the cap, and audit matching
still works when the eager known-table cache is skipped.

## Validation Performed

Validated with Scylla's frozen toolchain flow:

```console
./tools/toolchain/dbuild ninja build/dev/scylla build/dev/test/boost/combined_tests build/dev/test/boost/compaction_group_test build/dev/test/boost/audit_rule_test
./tools/toolchain/dbuild ./build/dev/test/boost/combined_tests -t reader_concurrency_semaphore_test/test_boot_schema_fanout_helpers_throttle_work_without_dropping_items -- -c1 -m1G
./tools/toolchain/dbuild ./build/dev/test/boost/compaction_group_test -t regular_compaction_submission_backlog_is_bounded_test -- -c1 -m1G
./tools/toolchain/dbuild ./build/dev/test/boost/audit_rule_test --run_test=test_preprocessed_rules_match_fast_and_slow_paths,test_preprocessed_empty_rules_skip_cache,test_preprocessed_large_known_table_set_uses_bounded_lazy_path -- -c1 -m1G
```

All targeted regressions passed with `*** No errors detected`.

An isolated 2 GiB run against a filesystem copy of `/var/lib/scylla` reached
`init - serving`, with no read queue and no malloc failures. That run is not a
complete proof for the original 8k+ logical-table reproducer because the copied
workdir's system schema loaded zero non-system keyspaces, even though stale data
directories were present on disk.

## Validation Still Required

Reproduce with a real logical schema containing a large number of one-table
keyspaces and a full `scylla` server binary, then confirm:

- CQL starts successfully
- `scylla_database_queued_reads{class="system"}` remains bounded
- compaction task backlog stays at or below
  `scylla_compaction_manager_regular_compaction_task_backlog_limit`
- audit startup either loads a bounded known-table cache or explicitly skips
  eager audit known-table caching above the cap
