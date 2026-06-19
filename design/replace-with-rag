Ready for review
Select text to add comments on the plan
Plan: Replace SQLite with Qdrant + Embeddings (RAG)
Context
The current server uses SQLite with FTS5 for all storage — both structured metadata (package versions, sources, symbols, dependencies) and document retrieval (BM25 full-text search on document_chunks). The goal is to replace this with a vector-based retrieval store to enable semantic similarity search rather than keyword matching, improving query_docs result quality.

Honest scope caveat: A true full replacement is possible but requires Qdrant to carry all roles SQLite plays today — metadata filtering for version selection, audit logs, structured lookups, and vector search. Qdrant supports rich payload filtering, so this is feasible without keeping any SQLite. The indexer-as-writer / server-as-reader split maps cleanly to Qdrant's HTTP API used by both processes.

Recommended Stack
Concern	Choice	Reason
Vector store	Qdrant (local Docker or binary)	Best .NET SDK, payload-level filtering (replaces SQL WHERE clauses), supports hybrid dense+sparse search, runs fully offline
Embedding model	text-embedding-3-small (OpenAI API)	No GPU required, 1536-dim, cheap (~$0.02/1M tokens), good quality for code/doc search
Fallback / offline	ONNX local model (e.g., BAAI/bge-small-en-v1.5 via ML.NET + ONNX Runtime)	Zero-cost, air-gapped environments
Add IEmbeddingService abstraction so the model can be swapped via config.

Qdrant Collection Design
Replace all SQLite tables with these Qdrant collections:

sources (payload-only, no vectors)
Point ID: hash(name)
Payload: name, environment, service_index, kind (nuget/docs), last_indexed_at
libraries (vector for semantic search + payload for exact lookup)
Point ID: hash(source_name + package_id)
Payload: source_name, package_id, normalized_package_id, kind, display_name, source_environment
Vector: embedding of package_id + " " + display_name
Replaces: libraries table + libraries_fts
library_versions (payload-only)
Point ID: hash(source_name + package_id + version)
Payload: library_point_id, source_name, package_id, version, title, description, summary, tags[], authors[], is_listed, is_prerelease, is_deprecated, published_at, indexed_at, content_hash, target_frameworks[], project_url, icon_url
Replaces: library_versions + target_frameworks tables
document_chunks (primary RAG collection)
Point ID: hash(source_name + package_id + version + path + ordinal)
Payload: library_version_point_id, source_name, package_id, version, artifact_path, kind (readme/xml_documentation/text), member_name, ordinal, content, content_hash
Vector: embedding of content (or member_name + "\n" + content for XML docs)
Replaces: document_chunks + document_chunks_fts
symbols (vector for semantic lookup + payload for pattern match)
Point ID: hash(source_name + package_id + version + fqn + target_framework)
Payload: library_version_point_id, source_name, package_id, version, namespace, fully_qualified_name, kind, signature, containing_type, assembly_path, target_framework, xml_documentation_member, xml_documentation_content
Vector: embedding of fully_qualified_name + " " + signature + " " + xml_documentation_content
Replaces: symbols table
Note on LIKE queries: Qdrant has MatchText payload filter that does substring match on text fields — use MatchText on fully_qualified_name to replace LIKE '%query%'
dependencies (payload-only)
Point ID: hash(source_name + package_id + version + dep_package_id + target_framework)
Payload: library_version_point_id, package_id, version, dep_package_id, version_range, target_framework
Replaces: dependencies table (if needed; currently only stored, not queried by MCP tools)
index_runs (payload-only)
Point ID: random UUID per run
Payload: source_name, status, started_at, completed_at, duration_ms, indexed_count, changed_count, unchanged_count, error_count, errors[]
Replaces: index_runs + index_run_errors tables
Code Changes
New files to add
File	Purpose
src/DevContextMcp.Infrastructure/Embeddings/IEmbeddingService.cs	Interface: Task<float[]> EmbedAsync(string text)
src/DevContextMcp.Infrastructure/Embeddings/OpenAIEmbeddingService.cs	OpenAI text-embedding-3-small
src/DevContextMcp.Infrastructure/Embeddings/OnnxEmbeddingService.cs	Local ONNX fallback
src/DevContextMcp.Infrastructure/Indexer/Persistence/QdrantIndexStore.cs	Implements IIndexStore — upserts collections on publish
src/DevContextMcp.Infrastructure/Server/QdrantNuGetReadStore.cs	Implements INuGetReadStore — queries via Qdrant .NET SDK
src/DevContextMcp.Infrastructure/QdrantCollectionNames.cs	Constants for collection names
Files to modify
File	Change
src/DevContextMcp.Infrastructure/DevContextMcp.Infrastructure.csproj	Add Qdrant.Client and Azure.AI.OpenAI (or OpenAI) NuGet packages
src/DevContextMcp.Server/Program.cs	Switch DI registration from SqliteNuGetReadStore → QdrantNuGetReadStore
src/DevContextMcp.Indexer/Program.cs	Switch from SqliteIndexStore → QdrantIndexStore
src/DevContextMcp.Server/Configuration/DevContextMcpOptions.cs	Add QdrantUrl, EmbeddingProvider, EmbeddingModel fields
src/DevContextMcp.Indexer/Configuration/IndexingOptions.cs	Same embedding config fields
Files that do NOT change (clean interface boundary)
All *Handler.cs in DevContextMcp.Server.Core.Services — these call INuGetReadStore, no SQLite awareness
All *Tool.cs in DevContextMcp.Server.Tools
All MCP resources
All model/record classes
IIndexStore, INuGetReadStore interfaces
Query Mapping (SQLite → Qdrant)
Current SQL operation	Qdrant equivalent
libraries_fts BM25 search	Vector similarity search on libraries collection
Exact normalized_package_id = ?	Payload filter MatchValue("normalized_package_id", ...)
document_chunks_fts MATCH	Vector similarity search on document_chunks, filtered by package_id + version
Symbol LIKE '%query%'	MatchText("fully_qualified_name", query) payload filter + optional vector re-rank
Version selection filters (is_listed, is_prerelease, is_deprecated)	Payload filters combined with Must/Should
Latest version (ORDER BY published_at DESC LIMIT 1)	OrderBy("published_at") + Limit(1) scroll
Content hash deduplication	Point ID is hash(...) — upsert is naturally idempotent
Schema migration	Qdrant collection RecreateCollection with schema version in collection metadata
Idempotency Strategy
Each point's ID is a deterministic hash of its natural key (e.g., SHA256(source+package_id+version+path+ordinal)), so re-indexing the same content is a no-op upsert. Content hash in payload enables skip-if-unchanged logic just like the current content_hash column.

Configuration
Add to appsettings.json:

{
  "Qdrant": {
    "Url": "http://localhost:6334",
    "ApiKey": ""
  },
  "Embeddings": {
    "Provider": "openai",
    "Model": "text-embedding-3-small",
    "ApiKey": "",
    "BatchSize": 100
  }
}
What Gets Better vs. What Gets Worse
Before (FTS5)	After (RAG)
query_docs for conceptual questions	Keyword matching only	Semantic similarity — finds relevant chunks even with different vocabulary
Library search by name	BM25 on package_id/title	Semantic + exact — handles synonyms, misspellings
Symbol search	LIKE pattern	MatchText + vector re-rank
Version listing	Instant SQL query	Payload-only scroll — still fast
Indexing speed	Fast (no embedding)	Slower — embedding API calls per chunk (batch to mitigate)
Offline use	Fully offline	Needs Qdrant running; OpenAI needs internet (swap to ONNX for offline)
Operational complexity	Single .db file	Qdrant process + embedding provider
Verification
Unit tests: Mock IEmbeddingService and QdrantClient; test QdrantNuGetReadStore query construction
Integration test: Stand up a local Qdrant instance (Docker), index a small NuGet package, run SearchDocumentsAsync and assert results come back
End-to-end: Run the indexer against a test source, then start the server, connect via MCP client, call query_docs and resolve_library, compare result quality to SQLite baseline
Idempotency: Index the same source twice, assert point counts don't grow
Version selection: Verify ListVersionsAsync returns correct ordering and is_prerelease / is_deprecated flags are respected via payload filters