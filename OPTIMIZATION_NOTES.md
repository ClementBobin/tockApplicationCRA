# Terminal Optimization and Caching Implementation

## Overview

This implementation adds command caching and bulk data fetching to significantly improve performance, especially on Windows where terminal process creation is slower.

## Changes Made

### 1. Command Caching System

A new caching system has been added to the Rust backend (`src-tauri/src/lib.rs`) that:

- **Caches command results for 60 seconds** to avoid redundant tock CLI calls
- **Automatically invalidates cache** when data is modified (start, stop, add activity)
- **Thread-safe implementation** using Mutex for concurrent access
- **Memory-efficient** with automatic TTL-based expiration

### 2. Bulk Month Fetching

A new command `get_activities_for_month` has been added that:

- **Fetches an entire month of activities in a single operation**
- **Reduces calendar load time** from 30-31 sequential commands to 1 bulk command
- **Uses the same caching system** to avoid repeated fetches
- **Formats data** with clear date separators for easy parsing

### 3. Updated Calendar Implementation

The HistoryTab component now:

- **Uses bulk month fetching** instead of day-by-day sequential calls
- **Dramatically reduces UI lag** during month navigation
- **Maintains the same UI/UX** with improved performance

## Performance Improvements

### Before:
- Calendar load: 30-31 sequential `tock report --date` commands
- Each command spawns a new process
- No caching - same data fetched repeatedly
- Significant UI lag on Windows

### After:
- Calendar load: 1 bulk command for entire month
- Results cached for 60 seconds
- Subsequent month views use cached data
- Minimal UI lag across all platforms

## Technical Details

### Caching Strategy

```rust
// Read operations use cache
execute_tock_command_cached(vec!["report", "--date", &date], true)

// Write operations invalidate cache
if result.success {
    get_cache().invalidate();
}
```

### Cache Key Format

- Single date: `report --date 2026-01-15`
- Month report: `month_report_2026_1`

### TTL (Time To Live)

- Default: 60 seconds
- Automatically removes stale entries
- Can be adjusted in `CommandCache::new(ttl_seconds)`

## API Changes

### New Backend Command

```rust
#[tauri::command]
fn get_activities_for_month(year: u32, month: u32) -> CommandResult
```

### New Frontend API

```typescript
getActivitiesForMonth: async (year: number, month: number): Promise<CommandResult>
```

### Output Format

```
=== 2026-01-01 ===
Time Tracking Report for 2026-01-01
=====================================
📁 project1: 2h 30m
  09:00 - 11:30 (2h 30m) | description

=== 2026-01-02 ===
Time Tracking Report for 2026-01-02
=====================================
📁 project2: 1h 15m
  14:00 - 15:15 (1h 15m) | description
```

## Compatibility

- ✅ Windows (primary target for optimization)
- ✅ macOS
- ✅ Linux
- ✅ Backward compatible with existing tock CLI

## Future Improvements

Potential enhancements:

1. **Configurable cache TTL** - Allow users to set cache duration
2. **Smart cache invalidation** - Only invalidate affected dates
3. **Persistent cache** - Save cache to disk for app restarts
4. **Background prefetch** - Preload adjacent months in background
5. **Compression** - Compress cached data for large datasets

## Testing

To verify the changes work correctly:

1. Navigate to the History/Calendar tab
2. Change months - first load will fetch data, subsequent navigations use cache
3. Add/modify an activity - cache is automatically invalidated
4. Return to calendar - fresh data is fetched and cached again

## Notes

- Cache is in-memory only (cleared on app restart)
- TTL ensures data freshness even without modifications
- Write operations always execute immediately (never cached)
- Read operations always check cache first before executing command
