---
name: image-search-mcp
version: 1.0
description: "Search web images via DuckDuckGo. No key needed."
tags: ["images", "search", "duckduckgo", "download", "upload"]
---

## Image Search (DuckDuckGo - No Key Needed)

**Script:** `~/image_mcp/image_search.py`

### Usage

```bash
python3 ~/image_mcp/image_search.py "query text" max_results
```

### What it does:
1. Searches DuckDuckGo Images (no API key)
2. Returns URLs + titles + sources
3. Auto-downloads first result
4. Auto-uploads to uguu.se
5. Saves results JSON

### For teaching:
1. Search topic
2. Pick best image
3. Use `![](url)` in explanation

### Known Queries (Std 12 Economics Ch 1):
- `"time series graph economics"`
- `"simple bar diagram economics"`
- `"grouped bar diagram economics"`
- `"stacked bar diagram economics"`
- `"pie chart economics"`
