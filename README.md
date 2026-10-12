# Session 11 - Bài 3: Gradle Dependency Caching

- Workflow: [`.github/workflows/ci.yml`](.github/workflows/ci.yml), bước `actions/setup-java@v5` có `cache: 'gradle'`
- Báo cáo so sánh thời gian: [`report.md`](report.md)

| | Run #1 (cold) | Run #2 (warm) |
|---|---:|---:|
| `./gradlew test` | 49s | **18s (−63%)** |
| Job Run Unit Tests | 1m 5s | **37s (−43%)** |
| Cache | lưu mới | `Cache restored successfully` |
