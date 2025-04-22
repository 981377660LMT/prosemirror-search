## 1.1.0 (2025-04-22)

### Bug fixes

Fix another issue in regular expression replacement content reuse.

### New features

`SearchResult` objects now have a `matchStart` property indicating where the node that it matches against starts.

## 1.0.0 (2024-07-14)

### New Features

The new `filter` query option makes the query skip matches for which a predicate returns false.

### Bug Fixes

Fix an issue where replacement that reused parts of the match inside an inline node with content preserved the wrong pieces of content.

## 0.1.0 (2024-06-21)

### Breaking Changes

First numbered release.
