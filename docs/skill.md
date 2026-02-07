---
name: explore
description: Explore websites and extract structured data (TABLES, PARAGRAPHS) using NavCLI browser automation
keywords:
  - browser
  - scraping
  - exploration
  - tables
  - paragraphs
  - extraction
  - data
examples:
  - Extract data tables from financial websites
  - Scrape large text paragraphs from articles
  - Explore SPA websites and extract structured content
  - Handle pagination for multi-page listings
requires:
  - pip install navcli
  - nav-server running
---

# Data Extraction with NavCLI

> **IMPORTANT**: For data extraction, focus on `tables` and `paragraphs` commands.
>
> [Project Homepage](https://make.datavoid.fun/navcli/) | [GitHub](https://github.com/wumu2013/navcli)

## Why Tables & Paragraphs?

| Command | Use Case |
|---------|----------|
| `nav tables` | Extract structured tabular data (spreadsheets, financial data, listings) |
| `nav paragraphs` | Extract large text blocks (articles, reports, documentation) |
| `nav g <url>` | Navigate + auto-fetch paragraphs on first call |

## Quick Start

```bash
pip install navcli
```

```bash
# Start server
nav-server

# Load session for authenticated access
nav load_session <name>

# DATA EXTRACTION - The most important commands
nav g <url>              # Navigate (auto-returns paragraphs)
nav tables               # Extract ALL tables with structured data
nav paragraphs           # Extract large text paragraphs (200+ chars)
nav paragraphs <len>     # Extract paragraphs with custom min length
```

## Data Extraction Commands

```bash
nav tables                # Extract all <table> elements with data
nav paragraphs            # Get large text paragraphs (200+ chars, default)
nav paragraphs 500       # Get paragraphs with 500+ chars
nav elements              # Get interactive elements (buttons, inputs, links)
nav links                 # Get all hyperlinks
nav forms                 # Get all forms
```

## Agent Workflow: Navigate → Observe → Interact → Feedback → Continue

This is the core pattern for intelligent browsing. Agent can iteratively explore and interact.

```bash
# 1 Navigate - Go to target page
nav g https://example.com/data-page

# 2 Observe - Check what data is available
nav tables                      # PRIORITY: Extract tabular data
nav paragraphs                  # PRIORITY: Extract text content
nav elements                    # Find interactive elements

# 3 Interact - Navigate/paginate if needed
nav find "Next"                 # Find pagination
nav c "a.next-page"             # Click next page
nav wait_idle                   # Wait for load

# 4 Feedback - Verify data extraction
nav tables                      # Extract updated table
nav paragraphs                  # Extract new content

# 5 Continue - Proceed to next data source
nav g https://example.com/page2
nav tables
```

## CLI Commands

### Navigation

```bash
nav g <url>       # goto
nav b             # back
nav f             # forward
nav r             # reload
```

### Interaction

```bash
nav c <sel> [--force]       # click element
nav t <sel> <txt>            # type text into input
nav dblclick <sel>           # double-click element
nav rightclick <sel>        # right-click element
nav clear <sel>             # clear input field
nav upload <sel> <path>     # upload file to input
```

### Query

```bash
nav elements              # Get interactive elements
nav text                  # Get page text content
nav html                  # Get page HTML content
nav screenshot            # Take page screenshot
nav state                 # Get full page state
nav url                   # Get current URL
nav title                 # Get page title
nav tables                # Get all tables with data
nav paragraphs            # Get paragraphs (min 200 chars)
nav paragraphs <len>      # Get paragraphs (custom min length)
nav links                 # Get all hyperlinks
nav forms                 # Get all forms
nav evaluate <js>        # Execute JavaScript expression
```

### Explore

```bash
nav find <text>           # Find first element containing text
nav findall <text>       # Find all elements containing text
nav inspect <sel>        # Inspect element details
nav wait [--selector <s>]  # Wait for selector
nav wait --seconds <n>   # Wait for seconds
nav wait_idle [timeout]  # Wait for network idle
nav scroll top|bottom|up|down  # Scroll page
```

### Session

```bash
nav cookies               # Show current cookies
nav clear_cookies         # Clear all cookies
nav save_session [name]   # Save session (default: ~/.navcli/sessions/session.json)
nav load_session [name]   # Load session (default: ~/.navcli/sessions/session.json)
```

### Control

```bash
nav quit                  # Close browser
nav shutdown             # Shutdown server
nav exit                 # Exit CLI
```

## API Endpoints

### Navigation

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/cmd/goto?url=<url>` | Navigate to URL |
| POST | `/cmd/back` | Go back in history |
| POST | `/cmd/forward` | Go forward in history |
| POST | `/cmd/reload` | Reload current page |

### Interaction

| Method | Endpoint | Description |
|--------|----------|-------------|
| POST | `/cmd/click?selector=<s>[&force=true]` | Click element |
| POST | `/cmd/type?selector=<s>&text=<t>` | Type text into input |
| POST | `/cmd/dblclick?selector=<s>` | Double-click element |
| POST | `/cmd/rightclick?selector=<s>` | Right-click element |
| POST | `/cmd/clear?selector=<s>` | Clear input field |
| POST | `/cmd/upload?selector=<s>&file_path=<p>` | Upload file |

### Query

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/query/elements` | Get interactive elements |
| GET | `/query/text` | Get page text content |
| GET | `/query/html` | Get page HTML content |
| GET | `/query/screenshot` | Get screenshot (base64) |
| GET | `/query/state` | Get current state |
| GET | `/query/url` | Get current URL |
| GET | `/query/title` | Get page title |
| GET | `/query/evaluate?expr=<js>` | Execute JavaScript |
| GET | `/query/links` | List all links |
| GET | `/query/forms` | List all forms |
| GET | `/query/tables` | Get all tables with data |
| GET | `/query/paragraphs?min_length=200` | Get large text paragraphs |

### Session

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/cmd/cookies` | Get all cookies |
| POST | `/cmd/cookies/set` | Set cookie (JSON body) |
| DELETE | `/cmd/cookies/clear` | Clear all cookies |
| POST | `/cmd/session/save?name=<name>` | Save session |
| POST | `/cmd/session/load?name=<name>` | Load session |
| POST | `/cmd/quit` | Close browser |
| POST | `/cmd/shutdown` | Shutdown server |

### Explore

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/explore/find?text=<t>` | Find first element by text |
| GET | `/explore/findall?text=<t>` | Find all elements by text |
| GET | `/explore/inspect?selector=<s>` | Inspect element details |
| POST | `/explore/wait?selector=<s>` | Wait for selector |
| POST | `/explore/wait?seconds=<n>` | Wait for seconds |
| POST | `/explore/wait?idle=true` | Wait for network idle |
| POST | `/explore/scroll?direction=<top\|bottom\|up\|down>` | Scroll page |

## Complete Examples

### Multi-Step Data Extraction Workflow

```bash
# Start on listing page
nav g https://example.com/items

# Observe: Find filter controls
nav elements
nav find "Filter"
nav find "Apply"

# Interact: Set filters
nav c "select[name='category']"
nav t "input[name='search']" "widget"
nav c "button.apply-filters"

# Feedback: Check results
nav wait_idle
nav tables                      # Extract filtered data

# Continue: Navigate to details
nav findall "View Details"
nav c "a.view-item-1"           # Click first item

# Observe: Check details page
nav state
nav text
```

### Pagination Pattern

```bash
nav findall "Next"       # Find pagination links
nav c "a.next-page"     # Click next page link
nav wait_idle
nav tables              # Get updated table
```

### Interactive Exploration Pattern

```bash
# Loop: Navigate -> Observe -> Interact -> Feedback
while [ needs more pages ]; do
    nav tables                      # Observe current page data
    nav findall "Next"              # Find next page link
    nav c "a.pagination-next"      # Interact: click next
    nav wait_idle                   # Wait for load
done
```

## Troubleshooting

| Issue | Fix |
|-------|-----|
| Empty tables | `nav wait_idle` then retry |
| Not logged in | `nav load_session <name>` |
| Element not found | Re-run `nav elements` to refresh |
| Timeout | `nav wait_idle 10` for slow pages |
