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

Representative journal snippets from the failing node:

```text
Jun 21 02:04:33 node scylla[157948]: INFO  2026-06-21 02:04:33,062 [shard 0:main] init - loading non-system sstables
Jun 21 02:28:48 node scylla[157948]: 1        1        16K        system_schema.tables/data-query/active/need_cpu
Jun 21 02:28:48 node scylla[157948]: reads_queued_because_need_cpu_permits: 27267
Jun 21 04:58:04 node scylla[380858]: WARN  2026-06-21 04:58:04,700 [shard 0:main] seastar - Too long queue accumulated for main (8777 tasks)
Jun 21 04:58:20 node scylla[380858]: WARN  2026-06-21 04:58:20,734 [shard 0:main] seastar - (rate limiting dropped 21609 similar messages) Too long queue accumulated for main (26501 tasks)
Jun 21 04:58:30 node scylla[380858]: WARN  2026-06-21 04:58:30,735 [shard 0:comp] seastar - (rate limiting dropped 14523 similar messages) Too long queue accumulated for compaction (18728 tasks)
```

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

`compaction_manager::submit()` now has a startup-only retained-task cap for
regular compaction submissions:

- `regular_compaction_task_backlog_limit()`
- `regular_compaction_task_backlog_full()`

When the startup cap is hit, more regular submissions are postponed instead of
allocating more task objects. Task completion signals reevaluation, and
postponed work drains only while the cap has room.

The limiter is disabled before Scylla announces serving, via
`disable_startup_regular_compaction_backlog_limit()`. Once disabled, normal
runtime compaction scheduling is not capped by this startup guard. Any postponed
startup work is reevaluated when the guard is disabled.

Metrics:

- `scylla_compaction_manager_regular_compaction_task_backlog`
- `scylla_compaction_manager_regular_compaction_task_backlog_limit`
- `scylla_compaction_manager_postponed_compactions`

The backlog limit metric reports `0` after startup to show that the startup
guard is no longer active.

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
submission cannot retain more task objects than the cap during startup,
compaction submission can exceed the cap once the startup guard is disabled,
and audit matching still works when the eager known-table cache is skipped.

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

Final logical-schema validation used a repo-mounted workdir so data persisted
across `dbuild` container invocations:

```console
time cqlsh 127.0.0.1 19142 --connect-timeout=30 --request-timeout=120 -f /tmp/scylla-highcard-schema.cql
```

That created 4,200 real one-table keyspaces in 5m31s:

```text
 count
-------
  4200

 count
-------
  4200

 keyspace_name
---------------
       hc_4199

 table_name
------------
          t
```

The final restart from the same mounted workdir completed successfully:

```text
INFO  2026-06-21 06:40:32,536 [shard 0:main] database - Loading schema table keyspaces for 4205 keyspaces with concurrency 1
INFO  2026-06-21 06:40:32,817 [shard 0:main] database - Loading schema table tables for 4204 keyspaces with concurrency 1
INFO  2026-06-21 06:40:40,868 [shard 0:main] database - Populating 1 priority non-system keyspaces with concurrency 2
INFO  2026-06-21 06:40:40,870 [shard 0:main] database - Populating 4204 non-system keyspaces with concurrency 2
INFO  2026-06-21 06:40:43,288 [shard 0:main] group0_raft_sm - Skipping eager audit known-table cache for 4286 tables; cap is 4096; matching remains exact via per-request rule evaluation
INFO  2026-06-21 06:40:43,472 [shard 0:main] init - serving
INFO  2026-06-21 06:40:43,472 [shard 0:main] init - Scylla version 2026.3.0~dev-0.20260621.048d9b50079c initialization completed.
```

Post-boot CQL confirmed the logical schema was present:

```text
 count
-------
  4200

 count
-------
  4200

 keyspace_name
---------------
       hc_4199

 table_name
------------
          t
```

Post-boot metrics:

```text
scylla_compaction_manager_postponed_compactions{shard="0"} 0.000000
scylla_compaction_manager_regular_compaction_task_backlog{shard="0"} 0.000000
scylla_compaction_manager_regular_compaction_task_backlog_limit{shard="0"} 0.000000
scylla_database_queued_reads{class="system",shard="0"} 0.000000
scylla_database_reads_memory_consumption{class="system",shard="0"} 0.000000
scylla_memory_malloc_failed{shard="0"} 0
```

The restart log had no `ERROR`, `FATAL`, `bad_alloc`, `std::bad_alloc`,
`Aborting`, or `malloc_failed` lines. Non-fatal dev-environment warnings were
still present, including sysctl and I/O scheduler warnings, single-node gossip
warnings, and two oversized allocation warnings of 135,168 and 270,336 bytes.

## Validation Status

The high-cardinality logical-schema restart case is reproduced and closed for a
single-shard 4 GiB dev node with 4,200 real one-table keyspaces. Broader
multi-shard and production-hardware validation is still useful, but the original
startup fanout failure mode is covered by tests and by the full local restart
run above.
