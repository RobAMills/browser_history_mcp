# Changelog

All notable changes to the Browser History MCP Server will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.1] - 2025-12-31

### Fixed
- **Critical bug fix**: `search_browser_history` now correctly handles both dictionary and list return types from `get_browser_history`
  - Previously failed with error: `'str' object has no attribute 'get'` when searching
  - Root cause: Function expected simple list but received `BrowserHistoryResult` dict with metadata when using `all_browsers=True`
  - Fix: Added type checking to detect dict vs list responses and extract `history_entries` appropriately
  - Updated caching logic to store only history entries list, not full metadata dict
  - Location: `server/browser_utils.py` lines 637-665

### Tested
- Verified search functionality works correctly after fix:
  - Search for "github" returns hundreds of results ✅
  - Search for "spotify" returns relevant entries ✅
  - No more type errors during search operations ✅

### Documentation
- Added comprehensive "Use with Goose Desktop" section to README.md
  - Method 1: Via Settings UI with step-by-step instructions
  - Method 2: Via configuration file with JSON examples
  - Verification steps to confirm installation
  - Troubleshooting common issues
  - Updated Quick Start section to highlight Goose Desktop integration
- Added troubleshooting section to README.md documenting the search fix
- Includes issue description, root cause, fix details, and verification steps
- Created CHANGELOG.md to track all changes and improvements

## [1.0.0] - 2025-12-30

### Added
- Initial release of Browser History MCP Server
- Multi-browser support (Firefox, Chrome, Safari)
- Browser history retrieval and analysis tools
- Session analysis with intelligent time-based clustering
- Smart categorization of websites
- Domain analytics and visit pattern analysis
- Learning insights and productivity metrics
- Real-time database access with privacy-first design
- Comprehensive API with 8 core tools and 6 analysis prompts
