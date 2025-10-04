# indexed-find

Fast keyword-based file search tool for file recovery with boolean and sequential queries.

## Unique Features

**What sets indexed-find apart:**

- **Multi-threaded indexing**: Parallel file reading, processing, and writing for 20-50x speedup
  - Intelligent memory management with configurable readahead limits
  - Single-threaded mode for safety, multi-threaded for speed
- **Sequential/ordered search**: Find keywords in a specific order across lines with `<` operator
  - `printf < format < buffer` matches even when keywords are separated by other content
- **Dual tokenization**: Indexes both `[a-z0-9]+` and `[a-z0-9_]+` patterns for comprehensive coverage
- **Context preservation**: Stores surrounding lines (160 char limit) for quick review without reopening files
- **Offset tracking**: Exact byte position for each keyword occurrence
- **Relative path storage**: Clean, portable filename storage relative to index directory
- **Memory-efficient**: Processes files one at a time (single-threaded) or with bounded queues (multi-threaded)
- **Append mode**: Add new files without re-indexing existing ones
- **Size filtering**: Skip files by max size (default 200KB, configurable)

## Performance

**Single-threaded vs Multi-threaded:**

```bash
# Single-threaded (safe, slower) - ~1-2 files/sec
indexed-find -n recovery index -d /lost+found -r
# 108,000 files = ~15-30 hours

# Multi-threaded with 4 threads - ~40-80 files/sec  
indexed-find -n recovery index -d /lost+found -r --threads 4
# 108,000 files = ~22-45 minutes

# Adjust memory limit (default 10% of RAM)
indexed-find -n recovery index -d /lost+found -r --threads 4 --max-readahead-sysmem 20
```

**How multi-threading works:**
- `--threads 1` (default): Single-threaded, no parallelism
- `--threads N` (N>1): 1 reader thread + (N-1) worker threads + 1 writer thread
  - **Reader**: Queues files from disk (blocks when memory limit hit)
  - **Workers**: Extract keywords with regex patterns in parallel
  - **Writer**: Batch INSERT to SQLite (1000 keywords or time-based flush)
  
**Memory management:**
- `--max-readahead-sysmem 10`: Uses 10% of system RAM for file queue
- Reader checks file size before loading, blocks if over limit
- Always allows at least 1 file to be read (even if over limit)
- Memory is released as workers finish processing

**When to use multi-threading:**
- Large datasets (10,000+ files): Use `--threads 4` or higher
- Fast storage (SSD): Higher thread counts work better
- Slow storage (HDD): Fewer threads (2-4) may be optimal
- Memory constrained: Reduce `--max-readahead-sysmem` percentage

```bash
chmod +x indexed-find
# Optionally move to your PATH
```

## Quick Start

```bash
# Index current directory
indexed-find -n mydata index -d .

# Index with recursion and custom size limit
indexed-find -n recovery index -d /lost+found -r --maxsize 300000

# Boolean search
indexed-find -n mydata search '(malloc && free) || calloc'

# Sequential search (ordered keywords)
indexed-find -n mydata search 'fopen < buffer < fread'

# Complex boolean with negation
indexed-find -n mydata search 'database && !deprecated'
```

## Usage

### Indexing

```bash
indexed-find -n <dataset> index -d <directory> [options]

Options:
  -d, --dir DIR           Directory to index (required)
  -r, --recursive         Recursively index subdirectories
  -a, --append            Append to existing index
  --overwrite --force     Overwrite existing index (both flags required)
  --maxsize BYTES         Max file size in bytes (default: 200000)
  --maxindexed N          Stop after indexing N files (for testing)
  --threads N             Number of threads (default: 1 = single-threaded)
                          N > 1: Uses 1 reader + (N-1) workers + 1 writer
  --max-readahead-sysmem PCT  
                          Percentage of system memory for readahead (default: 10%)
  --commit-time SECONDS   Min time between SQLite commits (default: 0 = after each file)
  -v, -v -v, -v -v -v    Verbosity levels
```

### Searching

```bash
indexed-find -n <dataset> search '<query>'

Boolean operators:
  &&    AND
  ||    OR
  !     NOT
  ()    Grouping

Sequential/ordered search (keywords must appear in order):
  # Multiple arguments
  indexed-find -n ds search signal desktop updater
  
  # Space-separated (quote the query)
  indexed-find -n ds search 'signal desktop updater'
  
  # Using < operator (also works)
  indexed-find -n ds search 'signal < desktop < updater'
```

### Listing

```bash
# List all files in index
indexed-find -n <dataset> -l
indexed-find -n <dataset> --list-files

# List all keywords in index
indexed-find -n <dataset> -k
indexed-find -n <dataset> --list-keywords
```

## Example Output

### Indexing (default verbosity)

```bash
$ indexed-find -n recovery index -d /lost+found -r
Total:15234, Indexed:8456, Skipped:6778
Indexing complete. Database: ~/.cache/indexor/ds/recovery/index.db
```

### Indexing (verbose: -v -v)

```bash
$ indexed-find -n recovery index -d . -v -v
Total:1, Indexed:1, Skipped:0 Indexing: config.c (4521 bytes)
Total:2, Indexed:2, Skipped:0 Indexing: parser.h (2341 bytes)
Total:3, Indexed:2, Skipped:1 Skipping: bigfile.tar (size: 5242880 > 200000)
Total:4, Indexed:3, Skipped:1 Indexing: utils.py (8934 bytes)
Total:5, Indexed:4, Skipped:1 Indexing: main.c (12456 bytes)
...
Total:148, Indexed:91, Skipped:57 Indexing: final.rs (6234 bytes)
Total:149, Indexed:91, Skipped:58 Skipping: cache.db (already indexed)
Total:149, Indexed:91, Skipped:58
Indexing complete. Database: ~/.cache/indexor/ds/recovery/index.db
```

### Search Results (boolean)

```bash
$ indexed-find -n recovery search '(chicken && cross) || road'

mystery.py:1247 (chicken)
  def solve_ancient_riddle():
→     # TODO: Why did the chicken cross the road?
      return "To get to the other side"

philosophy.c:891 (cross)
  /* The chicken's motivation remains unclear */
→ if (chicken.crosses(road) && !road.has_traffic()) {
      existential_crisis();

comedy.txt:42 (road)
  Q: Why did the rubber chicken cross the road?
→ A: She was stretching her legs. Get it? Rubber? Stretching?
  *crickets chirping*
```

### Search Results (sequential)

```bash
$ indexed-find -n recovery search 'malloc < memset < free'

memory_leak.c:156 (malloc)
  // Classic three-act tragedy
→ void *ptr = malloc(sizeof(despair) * 1000);
  memset(ptr, 0, sizeof(regret));

memory_leak.c:157 (memset)
  void *ptr = malloc(sizeof(despair) * 1000);
→ memset(ptr, 0, sizeof(regret));
  // free(ptr); // TODO: Do this someday maybe

memory_leak.c:158 (free)
  memset(ptr, 0, sizeof(regret));
→ // free(ptr); // TODO: Do this someday maybe
  printf("What could go wrong?\\n");
```

## Index Storage

Indexes are stored in: `~/.cache/indexor/ds/<dataset-name>/index.db`

Each keyword entry includes:
- Keyword (lowercased, max 40 chars)
- Filename (relative to indexed directory)
- Byte offset
- Prior line (up to 160 chars)
- Match line (up to 160 chars)
- Next line (up to 160 chars)

## Other Indexing Tools (Reference)

**Note:** This is an LLM-generated comparison and may not be fully accurate or comprehensive. Verify details before making decisions.

### General-purpose search engines
- **Recoll**: Desktop search with GUI, supports many formats, boolean queries, proximity search
- **Apache Lucene/Solr**: Enterprise search platform, highly scalable, complex queries
- **Elasticsearch**: Distributed search, JSON documents, RESTful API
- **Xapian**: Probabilistic search library, used by Notmuch and others

### Code-specific indexers
- **ctags/universal-ctags**: Function/class indexing for code navigation
- **GNU Global (gtags)**: Source code tagging system, integrates with editors
- **cscope**: Interactive source code browser for C
- **ripgrep (rg)**: Fast grep alternative with smart filtering
- **The Silver Searcher (ag)**: Code searching tool, gitignore-aware

### Document indexers
- **Sphinx**: Full-text search for documentation
- **Whoosh**: Pure Python search library
- **DocFetcher**: Desktop search for documents

### Database approaches
- **SQLite FTS5**: Full-text search extension for SQLite
- **PostgreSQL Full-Text Search**: Built-in text search capabilities
- **Manticore Search**: Open source search engine forked from Sphinx

Most of these tools don't support:
- Sequential/ordered keyword matching across content
- Dual tokenization patterns for code
- Context preservation with byte offsets
- File-size-based filtering during indexing

## License

Do whatever you want with it. It's recovery tools for lost files - help yourself!

## Author

Generated with Claude (Anthropic). Feel free to modify and improve!
