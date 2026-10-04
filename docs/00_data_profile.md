# 00 Data Profile: GitHub Archive (Raw Ingestion)

## 1. File Metadata & Volume
- **Sample File:** `2026-09-28-13.json.gz` (13:00 UTC)
- **Compressed Size:** ~14.7 MB
- **Total Events (1 Hour):** 94,941
- **Uniqueness:** 94,941 unique `id`s (no duplicates within the single hourly file).
- **Entities Observed:**
  - Unique Actors: 39,041
  - Unique Repositories: 56,074

## 2. Event Type Distribution
| Event Type | Event Count | Percentage |
| :--- | :--- | :--- |
| **PushEvent** | 74,118 | 78.07% |
| **CreateEvent** | 14,695 | 15.48% |
| **DeleteEvent** | 3,402 | 3.58% |
| **PullRequestEvent** | 1,075 | 1.13% |
| **IssueCommentEvent** | 536 | 0.56% |
| **Other (10 types)** | 1,115 | 1.18% |

## 3. Schema & Payload Observations
- **Top Skew:** Over 93.5% of events are either `PushEvent` or `CreateEvent`. Analytical models focusing on repository development velocity should prioritize these.
- **Sparse Struct Inference:** Because the GitHub Archive stores polymorphic event payloads in a single stream, letting the engine infer a universal struct creates wide, sparse columns with excessive `NULL`s.
- **PullRequestEvent Fields:** Contains nested objects:
  - `action`: e.g., `opened`, `closed`, `merged`
  - `number`: PR identifier within repo
  - `pull_request`: Contains `id`, `url`, `head` (commit SHA, ref, repo), and `base` (target branch SHA, ref, repo).
- **PushEvent Changes:** Commit summaries and authors are absent or minimal in recent payloads, requiring metrics to rely on `size` / `distinct_size` rather than iterating `commits` arrays.

## 4. Modeling & ELT Implications
1. **Raw Storage Strategy:** Land raw files as immutable append-only records with columns `(id, type, actor JSON, repo JSON, payload JSON, created_at, _source_file, _loaded_at)`.
2. **Deduplication:** Although this single hour has 100% unique IDs, overlapping windows and backfills will create duplicates. Deduplication on `id` in staging is mandatory.
3. **Partitioning:** Batch landing partitioned by `dt=YYYY-MM-DD` and `hour=H`.
