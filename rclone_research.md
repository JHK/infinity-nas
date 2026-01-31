# rclone Research: Movie Library with Local Cache

## Use Case

Movie library stored on Backblaze B2 with local SSD/HDD cache.

### Assumptions/Specifications

- **Library size:** 5TB (larger than cache)
- **Cache size:** 500GB local SSD/HDD
- **Individual file size:** < 500GB (cache size > largest file)
- **Network speeds:**
  - Local ethernet: 1Gbit/s (~125MB/s)
  - Internet upload: 40Mbit/s (~5MB/s)
  - Internet download: Variable (B2 egress)

### Requirements

- Appear as infinitely-sized storage (library size >> cache size)
- Minimize re-downloads (B2 egress is expensive)
- Persistent cache across restarts
- Reliable writes (no data loss)
- Reasonable performance for streaming playback

### Key Parameters

| Parameter | Purpose |
|-----------|---------|
| `--vfs-cache-mode full` | Cache reads and writes; enables seeking in files |
| `--vfs-cache-max-age 87600h` | Prevent time-based cache eviction; rely on size limits |
| `--vfs-cache-max-size 500G` | Control disk usage; LRU eviction when exceeded |
| `--cache-dir /path/to/disk` | Cache location on local disk |
| `--vfs-write-back 5s` | Delay before upload starts after file close |
| `--vfs-write-wait 300s` | Wait duration for uploads on unmount |
| `--buffer-size 64M` | Buffer for streaming performance |
| `--vfs-read-ahead 256M` | Prefetch for smooth playback |

## Operational Scenarios

### 1. Normal Operation - Read

**Flow:**
1. File requested → check local cache
2. If cached and valid → serve from disk (no B2 call)
3. If not cached → download from B2, store in cache, serve to application
4. Cache persists on disk at `--cache-dir`

**Performance:** Cache hits are local disk speed. Cache misses require B2 download.

### 2. Normal Operation - Write

**Flow:**
1. Application writes file → data written to local cache immediately
2. Write operation returns success (async)
3. File closed → wait `--vfs-write-back` duration
4. Upload to B2 begins in background
5. Upload completes → file safely on remote

**Critical:** Local write succeeds before B2 upload. Application doesn't wait for upload.

### 3. Process Restart (Clean)

**When everything synced:**
- Cache directory persists on disk
- Files remain cached (not re-downloaded)
- Age timer runs even while unmounted
- If restart within max-age → cache still valid
- If restart after max-age → files evicted, re-download needed

**With pending uploads:**
- `--vfs-write-wait` causes unmount to wait for uploads
- If uploads complete within timeout → safe
- If timeout exceeded → unmount proceeds anyway, data may be lost
- Check logs for "timed out waiting" warnings

### 4. Process Crash/SIGKILL

**During read:** No data loss, just cache corruption potential

**During write to cache:**
- Partially written files in cache → corrupted
- File not yet on B2 → data loss

**During upload to B2:**
- Local cache has complete file
- Upload incomplete → file missing/partial on B2
- No automatic retry on restart

### 5. Cache Eviction

**Triggers:**
- Cache size exceeds `--vfs-cache-max-size` → LRU eviction
- File age exceeds `--vfs-cache-max-age` → time-based eviction
- Manual deletion of cache directory

**Behavior:**
- Oldest files removed first when size limit hit
- Files in active use never evicted
- Next access requires re-download from B2

### 6. Large File Upload (5GB movie)

**Sequence:**
1. Copy 5GB file → writes to cache (~5GB disk I/O time)
2. Copy command completes
3. File closed → wait 5s (write-back)
4. Upload to B2 begins (5GB upload time, depends on bandwidth)
5. During upload: file already in cache, can be read locally
6. Upload completes → file fully synced

**Disk space required:** Up to 2x file size temporarily (cache + any temp files)

### 7. Bulk Upload Exceeding Cache Size

**Scenario:** Initial library upload (5TB) via local ethernet (1Gbit/s) with 500GB cache.

**Observed behavior (validated via testing):**
1. Write speed (125MB/s) >> upload speed (5MB/s)
2. Writes succeed immediately to cache
3. Cache temporarily exceeds max-size limit during bulk operations
4. Files with pending uploads are protected from eviction
5. After uploads complete → LRU eviction brings cache back under limit
6. **No data loss in normal operation**

**Key findings:**
- `--vfs-cache-max-size` is a soft target, not a hard limit
- Cache can grow beyond max-size when files are queued for upload
- Eviction only occurs AFTER files successfully upload to B2
- Write operations don't block or fail when cache exceeds limit

**Data loss risks (exceptional cases only):**
- Disk completely fills (no space for cache growth) → writes may fail
- Unclean shutdown (crash/SIGKILL) with pending uploads → potential loss
- Network failures preventing upload completion

**Time calculation:**
5TB @ 5MB/s upload = ~12 days continuous upload time.

## Concerns and Limitations

### Data Integrity

**Risk areas:**
- Unclean shutdown with pending uploads (highest risk)
- Process killed during write operations
- Disk completely full (no space for cache growth)
- Network failures preventing upload completion

**Validated behaviors (via testing):**
- Normal bulk uploads: No data loss even when exceeding cache size
- Cache protects files with pending uploads from eviction
- Cache can temporarily grow beyond max-size to ensure upload completion

**Mitigations:**
- Use `--vfs-write-wait` with reasonable timeout
- Ensure adequate disk space (cache can exceed max-size temporarily)
- Implement proper shutdown procedures (SIGTERM, not SIGKILL)
- Verify uploads complete: `rclone rc vfs/stats` before shutdown
- Post-upload verification: `rclone check` to confirm data integrity

### Cache Consistency

**Issues:**
- External changes to B2 not reflected in cache immediately
- Use `--dir-cache-time` and `--poll-interval` if files change outside mount
- For read-only libraries, default settings are fine

### Performance

**Read performance:**
- Cache hit: Local disk speed (excellent)
- Cache miss: B2 download + local write (slow first access)
- Streaming: Use `--vfs-read-ahead` for prefetching

**Write performance:**
- Local write: Fast (disk speed)
- B2 upload: Async, doesn't block operations
- Bottleneck: Upload bandwidth and B2 API rate limits

### Cost Considerations

**B2 Egress:**
- Cache misses incur download costs
- Large cache with long max-age minimizes re-downloads
- Monitor cache hit ratio

**B2 API Calls:**
- Directory listings cached per `--dir-cache-time`
- Metadata checks for file validity
- Generally negligible cost

### Recovery Scenarios

**Incomplete upload detection:**
- No built-in mechanism
- Need external verification: compare local cache to B2 listing
- Consider scripting: `rclone check` or `rclone sync --checksum`

**Cache corruption:**
- Delete corrupted files from `--cache-dir`
- Will re-download on next access
- Or delete entire cache and rebuild

**Lost data:**
- If file in cache but never uploaded → permanent loss
- No transaction log or upload queue persistence
- Critical data needs separate backup strategy

## Monitoring and Verification

### Recommended Checks

**Pre-shutdown:**
```bash
# Check for pending uploads (requires --rc)
rclone rc vfs/stats
```

**Post-upload verification:**
```bash
# Verify cache contents match B2
rclone check /path/to/cache b2:movie-library
```

**Cache statistics:**
```bash
# Monitor cache size
du -sh /path/to/rclone-cache

# Check cache age
find /path/to/rclone-cache -type f -mtime +7
```

### Logging

Enable detailed logging for troubleshooting:
```bash
--log-file /var/log/rclone.log --log-level INFO
```

Key log patterns:
- `vfs cache: upload` - Upload started
- `vfs cache: written` - Upload completed
- `vfs cache: timed out waiting` - Upload timeout warning

## Production Recommendations

1. **Set max-age very high** (years) - rely on size limits for eviction
2. **Use dedicated disk** for cache with ample space
3. **Implement graceful shutdown** - always SIGTERM with write-wait; check vfs/stats before unmount
4. **Monitor cache hit ratio** - optimize cache size based on access patterns
5. **Periodic verification** - script to check cache vs B2 consistency (`rclone check`)
6. **Test failure scenarios** - simulate crashes, verify data integrity
7. **Document recovery procedures** - how to detect and fix incomplete uploads
8. **Consider write patterns** - bulk uploads work safely; ensure adequate disk space for temporary cache growth

## Known Issues and Workarounds

### Cache Rebuild on Remount (Some Reports)
- Usually caused by permission issues or wrong cache-dir path
- Verify cache-dir persists and is writable
- Check logs for "opening cache" messages

### Upload Timeouts
- Long uploads may exceed write-wait timeout
- Increase timeout or split large files
- Consider direct upload for huge files: `rclone copy`

### Memory Usage
- Large file counts can increase memory usage
- Monitor with `--rc` and `vfs/stats`
- Consider limiting concurrent transfers: `--transfers 4`