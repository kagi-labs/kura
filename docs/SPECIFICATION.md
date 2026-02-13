# Project Kura (蔵) Technical Specification

## 1. Storage Interface (S3-Lite)
- Implementation of a subset of the Amazon S3 API.
- Support for `PutObject`, `GetObject`, `ListObjectsV2`, and `DeleteObject`.
- Authentication via pre-shared local keys.

## 2. MCP Tool Integration
- `kura_save(key, content, metadata)`: Persists agent output to the storehouse.
- `kura_load(key)`: Retrieves content for agent context.
- `kura_search(query)`: Searches metadata and content tags.

## 3. Sync Adapters
- **Local:** POSIX filesystem (optimized for SSD).
- **Cloud:** Pluggable sync to S3, Google Drive, or Dropbox.
- **Protocol:** Event-driven sync triggers on file close.

## 4. Conflict Resolution
- Last-write-wins by default.
- Versioning support for critical memory files.
