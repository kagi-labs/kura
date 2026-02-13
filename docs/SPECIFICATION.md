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

## 5. Organizational Structure (Dynamic Collections)
- **On-Demand Namespacing:** Kura does not require pre-defined collections. Collections are created dynamically whenever an agent or user specifies a new path (e.g., `kura_save(collection="research/ai-safety", ...)`).
- **Infinite Hierarchy:** Supports nested collections for deep organization (e.g., `learning/rust/concurrency`).
- **Discovery & Listing:** 
    - `list_collections()`: Returns all active high-level "storage rooms."
    - `search_across_collections(query)`: Global search that respects dynamic boundaries.
- **Smart Routing:** If no collection is specified, data defaults to `general/`, but agents are encouraged to "label" their data at creation time.
