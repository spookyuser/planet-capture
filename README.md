# planet-capture

An on-device search engine for your browser history. Indexes the pages you visit in Chrome, Arc, Brave, and Safari — then lets you search them with keywords, semantic similarity, or LLM-reranked hybrid search. Ideal for agentic workflows that need to answer "where did I read that?"

planet-capture combines BM25 full-text search, vector semantic search, and LLM re-ranking — all running locally via node-llama-cpp with GGUF models.

## Quick Start

```sh
bun install -g https://github.com/spookyuser/planet-capture

# See which browsers were detected
planet-capture browsers

# Index your browser history + fetch visited pages
planet-capture index

# Generate embeddings for semantic search
planet-capture embed

# Search
planet-capture search "rust async runtime"          # Fast BM25 keyword search
planet-capture vsearch "how do I debounce input"    # Semantic search
planet-capture query "typescript generics tutorial" # Hybrid + reranking (best quality)

# Look up a specific page
planet-capture get "https://example.com/post"

# Look up a page by docid (shown in search results)
planet-capture get "#abc123"

# Restrict results to one browser
planet-capture search "api docs" --browser=chrome

# Export results for an agent
planet-capture query "rust lifetimes" --files --min-score 0.3
```

### Indexing

```sh
# Full indexing (discover URLs from browsers, fetch any new pages)
planet-capture index

# Common flags
planet-capture index --since=2024-01-01    # Only include history after this date
planet-capture index --rate-limit=5        # Fetches per second (default 2)
planet-capture index --max-pages=500       # Cap fetches per run
planet-capture index --discover-only       # Only read browser history, don't fetch
planet-capture index --dry-run             # Preview without writing

# Split indexing into two steps
planet-capture discover                    # Only read browser history
planet-capture fetch                       # Only fetch pending pages
```

Pages are fetched, run through Readability for main-content extraction, hashed,
and stored in SQLite. Repeat visits to the same URL dedupe on content hash.

### URL Filters

Exclude noisy URLs (social feeds, search result pages, localhost) from the index:

```sh
planet-capture filters list
planet-capture filters add "twitter.com/*/status"
planet-capture filters add "localhost:*"
planet-capture filters remove "twitter.com/*/status"
```

Filter patterns are glob-style and applied during discovery.

### Using with AI Agents

`--json` and `--files` output formats are designed for agentic workflows:

```sh
# Structured results for an LLM
planet-capture search "authentication" --json -n 10

# List all relevant URLs above a threshold
planet-capture query "error handling" --files --min-score 0.4

# Retrieve full page content
planet-capture get "https://example.com/post" --full
```

### MCP Server

planet-capture ships an MCP (Model Context Protocol) server so agents can
search your browser history without shelling out.

**Tools exposed:**
- `search` — Hybrid BM25 + vector + LLM reranking over indexed pages
- `get` — Retrieve a page by URL or docid
- `recent` — List recently visited pages
- `status` — Show index counts and browsers
- `multi_get` — Batch retrieve pages by docid

**Claude Desktop configuration** (`~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "planet-capture": {
      "command": "planet-capture",
      "args": ["mcp"]
    }
  }
}
```

**Claude Code** — configure MCP in `~/.claude/settings.json`:

```json
{
  "mcpServers": {
    "planet-capture": {
      "command": "planet-capture",
      "args": ["mcp"]
    }
  }
}
```

#### HTTP Transport

By default the MCP server uses stdio (launched as a subprocess by each client).
For a shared, long-lived server that avoids repeated model loading, use HTTP:

```sh
planet-capture mcp --http                    # localhost:8181
planet-capture mcp --http --port=8080        # custom port
```

LLM models stay loaded in VRAM across requests. Embedding/reranking contexts
are disposed after 5 min idle and transparently recreated on the next request.

Point any MCP client at `http://localhost:8181/mcp` to connect.

### SDK / Library Usage

Use planet-capture as a library in your own Node.js or Bun applications.

Install from a GitHub checkout or from the published package once available.

```typescript
import { createStore } from 'planet-capture'

const store = await createStore({ dbPath: './index.db' })

// Hybrid search (BM25 + vector + reranking)
const results = await store.search({ query: "rust async runtime" })
console.log(results.map(r => `${r.title} (${Math.round(r.score * 100)}%) ${r.file}`))

// Retrieve a page
const page = await store.get("https://example.com/post")
if (!("error" in page)) {
  console.log(page.title, page.url, page.browsers, page.visitCount)
}

// Run indexing from code
await store.index({ maxPages: 100 })

// Generate embeddings
await store.embed({ onProgress: ({ chunksEmbedded, totalChunks }) => {
  console.log(`${chunksEmbedded}/${totalChunks}`)
}})

// Inspect index state
const status = await store.getStatus()
console.log(status.totalPages, status.fetchedPages, status.browsers)

await store.close()
```

Key types exported for SDK consumers:

```typescript
import type {
  PlanetCaptureStore,  // The store interface
  SearchOptions,
  HybridQueryResult,
  PageResult,
  PageNotFound,
  SearchResult,
  ExpandedQuery,
  BrowserInfo,
  IndexStatus,
  IndexHealthInfo,
  IndexOptions,
  EmbedResult,
} from 'planet-capture'
```

Utility exports:

```typescript
import {
  extractSnippet,              // Extract a relevant snippet from text
  addLineNumbers,              // Add line numbers to text
  DEFAULT_MULTI_GET_MAX_BYTES, // Default max body size for multi-get (10KB)
  Maintenance,                 // Database maintenance operations
  getDefaultDbPath,            // Default index path (~/.planet-capture/index.db)
} from 'planet-capture'
```

The SDK requires explicit `dbPath` — no defaults are assumed. This makes it
safe to embed in any application without side effects.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     planet-capture Hybrid Search Pipeline                   │
└─────────────────────────────────────────────────────────────────────────────┘

                              ┌─────────────────┐
                              │   User Query    │
                              └────────┬────────┘
                                       │
                        ┌──────────────┴──────────────┐
                        ▼                             ▼
               ┌────────────────┐            ┌────────────────┐
               │ Query Expansion│            │  Original Query│
               │     (LLM)      │            │   (×2 weight)  │
               └───────┬────────┘            └───────┬────────┘
                       │                             │
                       └──────────────┬──────────────┘
                                      │
              ┌───────────────────────┼───────────────────────┐
              ▼                       ▼                       ▼
     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
     │ Original Query  │     │ Expanded Query 1│     │ Expanded Query 2│
     └────────┬────────┘     └────────┬────────┘     └────────┬────────┘
              │                       │                       │
      ┌───────┴───────┐       ┌───────┴───────┐       ┌───────┴───────┐
      ▼               ▼       ▼               ▼       ▼               ▼
  ┌───────┐       ┌───────┐ ┌───────┐     ┌───────┐ ┌───────┐     ┌───────┐
  │ BM25  │       │Vector │ │ BM25  │     │Vector │ │ BM25  │     │Vector │
  │(FTS5) │       │Search │ │(FTS5) │     │Search │ │(FTS5) │     │Search │
  └───┬───┘       └───┬───┘ └───┬───┘     └───┬───┘ └───┬───┘     └───┬───┘
      │               │         │             │         │             │
      └───────┬───────┘         └──────┬──────┘         └──────┬──────┘
              │                        │                       │
              └────────────────────────┼───────────────────────┘
                                       │
                                       ▼
                          ┌───────────────────────┐
                          │   RRF Fusion + Bonus  │
                          │  Original query: ×2   │
                          │  Top-rank bonus: +0.05│
                          │     Top 30 Kept       │
                          └───────────┬───────────┘
                                      │
                                      ▼
                          ┌───────────────────────┐
                          │    LLM Re-ranking     │
                          │  (qwen3-reranker)     │
                          └───────────┬───────────┘
                                      │
                                      ▼
                          ┌───────────────────────┐
                          │  Position-Aware Blend │
                          │  Top 1-3:  75% RRF    │
                          │  Top 4-10: 60% RRF    │
                          │  Top 11+:  40% RRF    │
                          └───────────────────────┘
```

## Score Normalization & Fusion

### Search Backends

| Backend | Raw Score | Conversion | Range |
|---------|-----------|------------|-------|
| **FTS (BM25)** | SQLite FTS5 BM25 | `Math.abs(score)` | 0 to ~25+ |
| **Vector** | Cosine distance | `1 / (1 + distance)` | 0.0 to 1.0 |
| **Reranker** | LLM 0-10 rating | `score / 10` | 0.0 to 1.0 |

### Fusion Strategy

The `query` command uses **Reciprocal Rank Fusion (RRF)** with position-aware blending:

1. **Query Expansion**: Original query (×2 weight) + LLM variations
2. **Parallel Retrieval**: Each query searches both FTS and vector indexes
3. **RRF Fusion**: Combine all result lists using `score = Σ(1/(k+rank+1))` where k=60
4. **Top-Rank Bonus**: Documents ranking #1 in any list get +0.05, #2-3 get +0.02
5. **Top-K Selection**: Take top 30 candidates for reranking
6. **Re-ranking**: LLM scores each chunk (yes/no with logprobs confidence)
7. **Position-Aware Blending**:
   - RRF rank 1-3: 75% retrieval, 25% reranker (preserves exact matches)
   - RRF rank 4-10: 60% retrieval, 40% reranker
   - RRF rank 11+: 40% retrieval, 60% reranker (trust reranker more)

### Score Interpretation

| Score | Meaning |
|-------|---------|
| 0.8 - 1.0 | Highly relevant |
| 0.5 - 0.8 | Moderately relevant |
| 0.2 - 0.5 | Somewhat relevant |
| 0.0 - 0.2 | Low relevance |

## Requirements

### Supported Browsers

| Browser | History | Bookmarks | Platform |
|---------|---------|-----------|----------|
| Chrome  | ✅       | ✅         | macOS    |
| Arc     | ✅       | ✅         | macOS    |
| Brave   | ✅       | ✅         | macOS    |
| Safari  | ✅       | ✅         | macOS    |

planet-capture reads browser history from local SQLite databases and bookmarks
from the JSON / plist files each browser ships with. No cloud sync, no
credentials required — just read access to `~/Library/Application Support/`.

### System Requirements

- **Node.js** >= 22
- **Bun** >= 1.0.0
- **macOS**: Homebrew SQLite (for extension support)
  ```sh
  brew install sqlite
  ```

### GGUF Models (via node-llama-cpp)

planet-capture uses three local GGUF models (auto-downloaded on first use):

| Model | Purpose | Size |
|-------|---------|------|
| `embeddinggemma-300M-Q8_0` | Vector embeddings (default) | ~300MB |
| `qwen3-reranker-0.6b-q8_0` | Re-ranking | ~640MB |
| `Qwen3-1.7B` | Query expansion | ~1.1GB |

Models are downloaded from HuggingFace and cached in `~/.cache/node-llama-cpp/`.

### Custom Embedding Model

Override the default embedding model via the `PLANET_CAPTURE_EMBED_MODEL`
environment variable. Useful for multilingual content where
`embeddinggemma-300M` has limited coverage.

```sh
export PLANET_CAPTURE_EMBED_MODEL="hf:Qwen/Qwen3-Embedding-0.6B-GGUF/Qwen3-Embedding-0.6B-Q8_0.gguf"
planet-capture embed -f    # Re-embed with the new model
```

> **Note:** When switching embedding models, re-index with `planet-capture embed -f`
> since vectors are not cross-compatible between models.

## Installation

### Global CLI install

```sh
bun install -g https://github.com/spookyuser/planet-capture
```

This repo runs `prepare` on install so Bun builds the CLI automatically.

### Development

```sh
git clone https://github.com/spookyuser/planet-capture
cd planet-capture
bun install
bun run build
bun link
```

### Run from source

```sh
# Run from source during development
bun src/cli/planet-capture.ts <command>
```
## Search Commands

```
┌──────────────────────────────────────────────────────────────────┐
│                        Search Modes                              │
├──────────┬───────────────────────────────────────────────────────┤
│ search   │ BM25 full-text search only                            │
│ vsearch  │ Vector semantic search only                           │
│ query    │ Hybrid: FTS + Vector + Query Expansion + Re-ranking   │
└──────────┴───────────────────────────────────────────────────────┘
```

### Options

```sh
# Search options
-n <num>           # Number of results (default: 10)
--browser=NAME     # Restrict to a specific browser
--min-score <num>  # Minimum score threshold (default: 0)
--full             # Show full page content
--line-numbers     # Add line numbers to output
--intent=TEXT      # Disambiguation intent (query only)

# Output formats
--files            # Output: docid,score,url
--json             # JSON with snippets
--csv              # CSV output
--md               # Markdown output
--xml              # XML output

# Get options
planet-capture get <url|#docid>  # Retrieve a page
-l <num>                         # Max lines to return
--from <num>                     # Start from line number
```

## Data Storage

Index stored at: `~/.planet-capture/index.db`

### Schema

```sql
browsers        -- Detected browsers with history/bookmarks paths
pages           -- Canonical URL → content hash + fetch status
page_sources    -- Which browsers contributed each URL (visit counts, timestamps)
content         -- Content-addressable: hash → extracted page text
pages_fts       -- FTS5 full-text index over page titles + bodies
content_vectors -- Embedding chunks (hash, seq, pos, 900 tokens each)
vectors_vec     -- sqlite-vec vector index
llm_cache       -- Cached LLM responses (query expansion, rerank scores)
url_filters     -- Glob patterns to exclude from discovery
indexer_state   -- Most recent indexing run metadata
```

## How It Works

### Indexing Flow

```
Browser History ──► Filter URLs ──► Fetch HTML ──► Readability ──► Hash Content
      │                                                    │              │
      ▼                                                    │              ▼
 page_sources                                              │         Generate docid
 (visits, browser)                                         │         (6-char hash)
      │                                                    │              │
      └───────────────────────────────────────────────────►└──► pages + content
                                                                          │
                                                                          ▼
                                                                     FTS5 Index
```

### Embedding Flow

Pages are chunked into ~900-token pieces with 15% overlap using smart boundary
detection, then embedded with node-llama-cpp:

```
Page ──► Smart Chunk (~900 tokens) ──► Format each chunk ──► node-llama-cpp ──► Store Vectors
          │                            "title | text"        embedBatch()
          │
          └─► Chunks stored with (hash, seq, pos) per chunk
```

### Smart Chunking

Instead of cutting at hard token boundaries, planet-capture uses a scoring
algorithm to find natural markdown/prose break points — headings, code fences,
blank lines, list items. The squared distance decay means a heading 200 tokens
back still beats a simple line break at the target.

## License

MIT
