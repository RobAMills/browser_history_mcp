# Browser History MCP Server - Architecture Documentation

## Overview

The Browser History MCP Server is a local Model Context Protocol (MCP) server that provides AI assistants with secure, privacy-preserving access to browser history data for analysis and insights. Built with Python 3.12+ using the FastMCP framework, it processes browser data entirely locally without any external data transmission.

## System Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    AI Assistant (Claude)                     │
│                  (MCP Client via stdio)                      │
└────────────────────────┬────────────────────────────────────┘
                         │ MCP Protocol
                         │ (JSON-RPC over stdio)
┌────────────────────────▼────────────────────────────────────┐
│                  FastMCP Server (main.py)                    │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  MCP Tools Layer                                     │   │
│  │  - check_browser_status()                            │   │
│  │  - get_browser_history()                             │   │
│  │  - analyze_browser_history()                         │   │
│  │  - search_browser_history()                          │   │
│  │  - suggest_categories()                              │   │
│  └──────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  MCP Prompts Layer                                   │   │
│  │  - productivity_analysis                             │   │
│  │  - learning_analysis                                 │   │
│  │  - research_topic_extraction                         │   │
│  └──────────────────────────────────────────────────────┘   │
└────────────────────────┬────────────────────────────────────┘
                         │
        ┌────────────────┴────────────────┐
        │                                 │
┌───────▼──────────┐            ┌────────▼─────────┐
│  browser_utils   │            │ analysis_utils   │
│  (Data Access)   │            │  (Processing)    │
└───────┬──────────┘            └────────┬─────────┘
        │                                │
        │                                │
┌───────▼──────────────────────────────────────────┐
│         Browser SQLite Databases                 │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐       │
│  │ Firefox  │  │  Chrome  │  │  Safari  │       │
│  │ places.  │  │ History  │  │ History. │       │
│  │ sqlite   │  │          │  │   db     │       │
│  └──────────┘  └──────────┘  └──────────┘       │
└──────────────────────────────────────────────────┘
```

## Core Components

### 1. Entry Point (`server/main.py`)

**Purpose**: MCP server initialization and tool/prompt registration

**Key Responsibilities**:
- Initialize FastMCP server instance
- Register MCP tools (functions callable by AI)
- Register MCP prompts (pre-defined analysis templates)
- Manage global history cache (`CACHED_HISTORY`)
- Route tool calls to appropriate utility functions

**Key Design Patterns**:
- **Caching**: Global `CachedHistory` object prevents redundant database queries
- **Workflow Orchestration**: Tools are numbered (Step 1, 2, 3) to guide usage
- **Error Handling**: Graceful degradation when browsers are locked or unavailable

### 2. Browser Data Access Layer (`server/browser_utils.py`)

**Purpose**: Cross-platform browser history extraction

**Key Functions**:
- `get_firefox_history()`: Queries Firefox `places.sqlite` database
- `get_chrome_history()`: Queries Chrome `History` database  
- `get_safari_history()`: Queries Safari `History.db` database
- `tool_detect_available_browsers()`: Auto-detects installed browsers
- `tool_get_browser_history()`: Unified interface for multi-browser retrieval

**Database Access Strategy**:
- **Read-only mode**: Opens SQLite databases with `mode=ro` URI parameter
- **Lock detection**: Checks for database locks (browser running)
- **Graceful fallback**: Returns partial results if some browsers fail
- **Cross-platform paths**: Auto-detects OS-specific database locations

**Browser-Specific Details**:
- **Firefox**: Timestamps in microseconds since Unix epoch, `moz_places` table
- **Chrome**: Timestamps in WebKit format (microseconds since 1601-01-01), `urls` table
- **Safari**: Timestamps in Cocoa format (seconds since 2001-01-01), `history_items` table

### 3. Analysis & Processing Layer (`server/analysis_utils.py`)

**Purpose**: Transform raw history into actionable insights

**Key Functions**:
- `categorize_browsing_history()`: Categorizes URLs using pattern matching
- `analyze_domain_frequency()`: Aggregates visit statistics by domain
- `tool_analyze_browsing_sessions()`: Clusters visits into sessions (2-hour gap threshold)
- `calculate_productivity_metrics()`: Computes productivity vs distraction ratios
- `identify_learning_paths()`: Detects learning patterns and technologies
- `tool_get_browsing_insights()`: Main comprehensive analysis orchestrator

**Analysis Pipeline**:
```
Raw History → Categorization → Session Clustering → Enrichment → Insights
     ↓              ↓                  ↓                ↓            ↓
  Entries      Categories         Sessions        Metadata     Metrics
```

**Session Analysis**:
- **Clustering**: Groups visits with <2 hour gaps into sessions
- **Enrichment**: Adds productivity ratio, focus score, time patterns
- **Classification**: Categorizes as work, learning, entertainment, mixed
- **Pattern Detection**: Identifies deep work, context switching, rabbit holes

### 4. Type System (`server/local_types.py`)

**Purpose**: Strict type definitions for data consistency

**Key Types**:
- `HistoryEntry`: Immutable dataclass for raw browser entries
- `HistoryEntryDict`: TypedDict for serializable history data
- `EnrichedSession`: Session with metadata (duration, productivity, focus)
- `BrowserInsightsOutput`: Complete analysis result structure
- `CachedHistory`: History cache with metadata and expiration

**Design Benefits**:
- Type safety across async boundaries
- Clear API contracts between layers
- IDE autocomplete and validation
- Serialization compatibility with MCP protocol

### 5. Categorization Engine (`server/BROWSING_CATEGORIES.py`)

**Purpose**: Domain and pattern-based URL classification

**Category Structure**:
```python
{
    'category_name': {
        'domains': ['example.com', ...],      # Exact domain matches
        'patterns': [r'regex_pattern', ...],  # URL regex patterns
        'subcategories': {                    # Optional subcategorization
            'subcat_name': ['domain', ...]
        }
    }
}
```

**Built-in Categories**:
- `development`: GitHub, Stack Overflow, documentation sites
- `social_media`: Twitter, Reddit, LinkedIn, Discord
- `entertainment`: YouTube, Netflix, Spotify, gaming
- `productivity`: Email, calendars, project management
- `learning`: Courses, tutorials, educational content
- `shopping`: E-commerce platforms
- `news`: News outlets and aggregators
- `other`: Uncategorized URLs

**Customization**: Users can edit this file to add personal sites/patterns

### 6. Prompt Templates (`server/prompts.py`)

**Purpose**: Pre-defined analysis workflows for AI assistants

**Available Prompts**:
- `productivity_analysis`: Time distribution, focus metrics, recommendations
- `learning_analysis`: Learning patterns, knowledge gaps, progression tracking
- `research_topic_extraction`: Topic clustering, research depth analysis
- `generate_insights_report`: Comprehensive behavioral analysis
- `compare_time_periods`: Trend analysis across time ranges
- `export_visualization`: Data export and visualization suggestions

## Data Flow

### Typical Analysis Workflow

1. **Browser Detection**:
   ```
   AI calls check_browser_status()
   → Detects Firefox, Chrome, Safari
   → Checks for database locks
   → Returns availability status
   ```

2. **History Retrieval**:
   ```
   AI calls get_browser_history(days=7)
   → Queries all available browser databases
   → Converts timestamps to ISO format
   → Caches results in CACHED_HISTORY
   → Returns unified history list
   ```

3. **Analysis**:
   ```
   AI calls analyze_browser_history(analysis_type="comprehensive")
   → Retrieves cached history (or fetches if needed)
   → Categorizes URLs by domain/pattern
   → Clusters into sessions (2hr gaps)
   → Enriches sessions with metadata
   → Calculates productivity metrics
   → Identifies learning paths
   → Returns comprehensive insights
   ```

4. **Search/Filter**:
   ```
   AI calls search_browser_history(query="python")
   → Searches cached history
   → Filters by URL and title
   → Returns matching entries
   ```

## Performance Optimizations

### Caching Strategy
- **Global Cache**: `CACHED_HISTORY` stores last query results
- **Cache Key**: `(time_period_days, browser_type)`
- **Invalidation**: Automatic when parameters change
- **Benefit**: Avoids re-querying databases for same time period

### Fast Mode
- **Purpose**: Limit analysis scope for faster results
- **Limits**:
  - Max 1000 entries for session analysis
  - Top 20 domains only
  - Reduced learning path depth
- **Use Case**: Quick summaries vs deep analysis

### Benchmarking
- Built-in timing for each analysis phase
- Logs performance metrics to console
- Helps identify bottlenecks

## Security & Privacy

### Privacy-First Design
- **Local Processing**: All analysis happens on user's machine
- **No Network Calls**: Zero external data transmission
- **Read-Only Access**: Databases opened in read-only mode
- **No Persistence**: No history stored beyond session cache

### Browser Lock Handling
- Detects when browsers are running (database locked)
- Provides clear user instructions to close browsers
- Graceful degradation: returns partial results from available browsers

## Extension Points

### Adding New Browsers
1. Implement `get_<browser>_history(days)` in `browser_utils.py`
2. Add database path detection for OS platforms
3. Handle browser-specific timestamp formats
4. Register in `tool_detect_available_browsers()`

### Adding Categories
1. Edit `BROWSING_CATEGORIES.py`
2. Add domain patterns and subcategories
3. No code changes required - automatic integration

### Custom Analysis
1. Create new function in `analysis_utils.py`
2. Register as MCP tool in `main.py` with `@mcp.tool()`
3. Define input/output types in `local_types.py`

## Dependencies

- **FastMCP**: MCP server framework (stdio transport)
- **SQLite3**: Built-in Python library for database access
- **Standard Library**: datetime, typing, collections, urllib, re

## Configuration

### MCP Client Setup (Claude Desktop)
```json
{
  "mcpServers": {
    "browser-history": {
      "command": "python",
      "args": ["/path/to/server/main.py"]
    }
  }
}
```

### Environment
- Python 3.12+ required
- No API keys or credentials needed
- Works offline completely

## Error Handling

- **Database Locked**: Returns partial results + user instructions
- **Browser Not Found**: Skips browser, continues with others
- **Empty History**: Returns empty analysis with metadata
- **Invalid Time Period**: Validates input parameters
- **Corrupted Database**: Catches SQLite errors, logs warnings

