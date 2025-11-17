# MCP Filesystem Server - Tool Operations Guide

## Overview

This document provides detailed operational notes for the 8 MCP filesystem server tools based on testing and usage.

## Tool Inventory

### 1. init
**Purpose:** Initialize and verify allowed directories with optional operations

**Parameters:**
- `directory` (str, optional): Directory path to list contents
- `file_path` (str, optional): File path to read content

**Operation:**
- Validates configuration and accessibility of allowed directories
- Returns status message ("OK" if all directories accessible)
- Shows configuration source (command-line args vs config file)
- Can optionally list a directory and/or read a file in one call
- **Use Case:** Always call this first to verify your configuration before using other tools

**Example Usage:**
```python
# Verify configuration only
init(directory=None, file_path=None)

# Verify config and list a directory
init(directory="D:/projects", file_path=None)

# Verify config, list directory, and read a file
init(directory="D:/projects", file_path="D:/projects/readme.md")
```

**Return Format:**
```json
{
  "message": "OK",
  "isError": false,
  "details": {
    "allowed_dirs": ["D:/LocalProjects"],
    "allowed_extensions": [".py", ".js", ".ts", ".json", ".md", ".txt"],
    "accessible_dirs": ["D:/LocalProjects"],
    "inaccessible_dirs": [],
    "configuration_source": "command_line_args"
  }
}
```

---

### 2. list_directory
**Purpose:** List immediate contents of a directory (NON-RECURSIVE)

**Parameters:**
- `directory` (str): Absolute or relative path to directory
- `report_progress` (bool, default: true): Include progress information
- `batch_size` (int, default: 100): Items per progress batch

**Operation:**
- Lists only direct contents without entering subdirectories
- Fast operation for large directories
- Supports both `/` and `\` path separators

**Example Usage:**
```python
# List with progress reporting
list_directory(directory="D:/projects", report_progress=True, batch_size=100)

# Simple list without progress
list_directory(directory="D:/projects", report_progress=False)

# Windows path format also works
list_directory(directory="D:\\projects\\myapp")
```

**Tested Results:**
- Directory: `D:/LocalProjects/filesystem_server`
- Items found: 13 (immediate contents only)
- Processing time: Instantaneous (0 seconds)
- Speed: 86,411 items/second

**Contents Found:**
```
.git, .gitattributes, .gitignore, .venv, app.py, config.json,
images, LICENSE.txt, mcp_configuration_examples.json, pyproject.toml,
README.md, setup.py, uv.lock
```

**Key Feature:** Does NOT recurse into `.git`, `.venv`, or `images` subdirectories

---

### 3. list_directory_index
**Purpose:** List paginated directory contents (NON-RECURSIVE)

**Parameters:**
- `directory` (str): Absolute or relative path to directory
- `index` (int, default: 0): Page index (0-based)
- `index_size` (int, default: 10): Number of items per page

**Operation:**
- Lists only direct contents without entering subdirectories
- Returns a specific page of results for pagination
- Perfect for browsing large directories in manageable chunks
- Supports both `/` and `\` path separators

**Return Format:**
```json
{
  "error": false,
  "contents": ["item1", "item2", "..."],
  "total_items": 37,
  "index": 0,
  "index_size": 10,
  "start_position": 0,
  "end_position": 10,
  "items_in_page": 10,
  "total_pages": 4,
  "has_more": true,
  "has_previous": false,
  "processing_time": 0.001,
  "directory_path": "D:/LocalProjects/filesystem_server/.venv/Scripts",
  "normalized_path": "d:/LocalProjects/filesystem_server/.venv/Scripts"
}
```

**Use Cases:**
- Browsing large directories without overwhelming output
- Implementing pagination in UI/CLI applications
- Memory-efficient directory navigation
- Progressive directory exploration

**Example Usage:**
```python
# Get first 10 items (page 0)
list_directory_index("D:/projects", index=0, index_size=10)

# Get next 10 items (page 1)
list_directory_index("D:/projects", index=1, index_size=10)

# Get items 20-29 (page 2)
list_directory_index("D:/projects", index=2, index_size=10)
```

**Key Feature:** Pagination support with has_more/has_previous flags for easy navigation

---

### 4. read_file
**Purpose:** Read text file content

**Parameters:**
- `file_path` (str): Absolute or relative path to file

**Operation:**
- Reads file as UTF-8 text
- Returns entire file content as string
- Validates file type against allowed_extensions
- Not suitable for binary files

**Example Usage:**
```python
# Read a text file
read_file(file_path="D:/projects/readme.md")

# Read a Python file
read_file(file_path="D:/projects/app.py")

# Windows path format
read_file(file_path="D:\\projects\\config.json")
```

**Tested Results:**
- File: `D:/LocalProjects/filesystem_server/README.md`
- Size: 17,613 bytes
- Status: ✅ Successfully read
- Returns: Plain text content

**Tested Results:**
- File: `D:/LocalProjects/filesystem_server/LICENSE.txt`
- Size: 11,357 bytes
- Status: ✅ Successfully read (Apache License 2.0)
- Returns: Plain text content

---

### 5. read_file_binary
**Purpose:** Read file content as base64-encoded binary

**Parameters:**
- `file_path` (str): Absolute or relative path to file

**Operation:**
- Reads file as raw binary
- Encodes content as base64 string
- Suitable for images, PDFs, and other binary files
- Validates file type against allowed_extensions

**Example Usage:**
```python
# Read an image file as base64
read_file_binary(file_path="D:/projects/logo.png")

# Read any binary file
read_file_binary(file_path="D:/projects/document.pdf")

# Windows path format
read_file_binary(file_path="D:\\projects\\image.jpg")
```

**Tested Results:**
- File: `D:/LocalProjects/filesystem_server/LICENSE.txt`
- Size: 11,357 bytes original
- Base64 output: 11,560 characters
- Status: ✅ Successfully encoded

**Return Format:**
```json
{
  "content_base64": "ICAgICAgICAgICAgICAgICAg...",
  "encoding": "base64",
  "error": false
}
```

**Use Cases:**
- Reading images (PNG, JPG, GIF)
- PDFs and documents
- Audio/video files
- Any binary data that needs to be transmitted as text

---

### 6. list_resources
**Purpose:** Recursively list all files and directories in MCP resource format

**Parameters:**
- `directory` (str, optional): Starting directory (defaults to all allowed_dirs)
- `report_progress` (bool, default: true): Include progress information
- `batch_size` (int, default: 50): Resources per progress batch

**⚠️ WARNING:** This tool performs recursive scanning and can return very large datasets. When used on directories with many subdirectories and files (like those containing `.venv`, `.git`, or `node_modules`), it may return thousands of items. Consider using `list_directory` for non-recursive listing if you only need immediate contents.

**Example Usage:**
```python
# List all resources in all allowed directories
list_resources(directory=None, report_progress=True, batch_size=100)

# List resources in specific directory
list_resources(directory="D:/projects/myapp", report_progress=True)

# Get simple list without progress
list_resources(directory="D:/projects", report_progress=False)
```

**Operation:**
- Walks entire directory tree recursively
- Returns structured resource objects with metadata
- Includes file size and modification dates
- Progress tracking for large directory structures

**Tested Results:**
- Directory: `D:/LocalProjects/filesystem_server`
- Total items found: 4,045 (recursive scan)
- Items returned: 3,300 in first batch
- Processing time: 1.135 seconds
- Speed: 3,565 items/second
- Progress updates: 64 batches (at 50-item intervals)

**Resource Object Format:**
```json
{
  "id": "D:/LocalProjects/filesystem_server/app.py",
  "type": "file",
  "name": "app.py",
  "path": "D:/LocalProjects/filesystem_server/app.py",
  "metadata": {
    "size": 35942,
    "modified": "2025-11-11T12:34:56Z"
  },
  "actions": ["read", "read_binary"]
}
```

**Key Feature:** RECURSIVE - scans all subdirectories including `.venv`, `.git`, etc.

---

### 7. get_resource
**Purpose:** Get metadata for a specific file or directory

**Parameters:**
- `path` (str): Absolute or relative path to resource

**Operation:**
- Returns metadata without reading file content
- Shows available actions for the resource
- Provides file size and modification date for files
- Validates path against allowed_dirs

**Example Usage:**
```python
# Get metadata for a file
get_resource(path="D:/projects/app.py")

# Get metadata for a directory
get_resource(path="D:/projects/myapp")

# Windows path format
get_resource(path="D:\\projects\\readme.md")
```

**Return Format (File):**
```json
{
  "id": "D:/LocalProjects/filesystem_server/app.py",
  "type": "file",
  "name": "app.py",
  "path": "D:/LocalProjects/filesystem_server/app.py",
  "metadata": {
    "size": 35942,
    "modified": "2025-11-11T12:34:56Z"
  },
  "actions": ["read", "read_binary"]
}
```

**Return Format (Directory):**
```json
{
  "id": "D:/LocalProjects/filesystem_server",
  "type": "directory",
  "name": "filesystem_server",
  "path": "D:/LocalProjects/filesystem_server",
  "metadata": {},
  "actions": ["list"]
}
```

**Use Case:** Quick metadata lookup without reading file content

---

### 8. get_tool_operations_guide
**Purpose:** Retrieve comprehensive operational documentation

**Parameters:**
- None

**Operation:**
- Returns this complete guide as structured data
- Provides tool parameters, usage patterns, and best practices
- Includes performance benchmarks and warnings
- Essential reference for understanding all available tools

**Example Usage:**
```python
# Get the complete operations guide
get_tool_operations_guide()

# The guide includes detailed documentation for all 8 tools
# with usage examples, performance benchmarks, and best practices
```

**Return Format:**
```json
{
  "success": true,
  "guide_content": "# MCP Filesystem Server - Tool Operations Guide\n...",
  "format": "markdown",
  "file_path": "D:/LocalProjects/filesystem_server/TOOL_OPERATIONS.md",
  "size_bytes": 8595,
  "lines": 297,
  "note": "This is comprehensive documentation for all MCP filesystem tools..."
}
```

**Use Cases:**
- Quick reference for tool capabilities
- Understanding tool parameters and return formats
- Learning best practices and performance characteristics
- Troubleshooting and error handling guidance

**Key Feature:** Self-documenting API - always up-to-date tool reference

---

## Tool Comparison: list_directory vs list_directory_index vs list_resources_index vs list_resources

| Feature | list_directory | list_directory_index | list_resources |
|---------|---------------|---------------------|----------------|
| **Recursion** | No (immediate contents only) | No (immediate contents only) | Yes (entire tree) |
| **Pagination** | No (returns all items) | Yes (returns page of items) | No (returns all items) |
| **Speed** | Very fast (86K items/sec) | Very fast (instant) | Moderate (3.5K items/sec) |
| **Return Format** | Dict with full contents | Dict with page + metadata | MCP resource objects |
| **Metadata** | Progress info only | Pagination info (page, total, has_more) | File size, modified date |
| **Use Case** | Quick directory overview | Large directory pagination | Full workspace scan |
| **Example Output** | 37 items at once | 10 items per page | 4,045 items total |

**Recommendation:**
- Use `list_directory` for quick navigation of small/medium directories
- Use `list_directory_index` for paginating through large directories
- Use `list_resources` for comprehensive workspace analysis with metadata

---

## Configuration

**Current Test Configuration:**
- **Allowed Directories:** `D:/LocalProjects`
- **Allowed Extensions:** `.py`, `.js`, `.ts`, `.json`, `.md`, `.txt`
- **Configuration Source:** `command_line_args` (from mcp.json)

**Path Format Support:**
- Windows format: `D:\LocalProjects\filesystem_server` ✅
- Unix format: `D:/LocalProjects/filesystem_server` ✅
- Mixed format: `D:\LocalProjects/filesystem_server` ✅
- All formats automatically normalized

---

## Testing Summary

| Tool | Status | Notes |
|------|--------|-------|
| init | ✅ Working | Validates config, lists directories |
| list_directory | ✅ Working | Non-recursive, instant results |
| list_directory_index | ✅ Working | Pagination tested with Scripts folder (37 items) |
| read_file | ✅ Working | Successfully read README.md and LICENSE.txt |
| read_file_binary | ✅ Working | Successfully encoded LICENSE.txt as base64 |
| list_resources | ✅ Working | Scanned 4,045 items recursively |
| get_resource | ✅ Working | Metadata lookup for files and directories |
| get_tool_operations_guide | ✅ Working | Returns complete documentation |

---

## Best Practices

1. **Start with init:** Always validate configuration before operations
2. **Choose the right tool:**
   - Need quick directory view? → `list_directory`
   - Need paginated directory browsing? → `list_directory_index`
   - Need full workspace scan? → `list_resources`
   - Need text content? → `read_file`
   - Need binary/image? → `read_file_binary`
   - Need just metadata? → `get_resource`
   - Need tool documentation? → `get_tool_operations_guide`
3. **Path handling:** Server normalizes paths automatically - use any format
4. **Performance:** Non-recursive operations are significantly faster
5. **Pagination:** For large directories (100+ items), use `list_directory_index` instead of `list_directory`
6. **Security:** All operations validate against allowed_dirs and allowed_extensions

---

## Error Handling

All tools return structured error responses instead of raising exceptions:

```json
{
  "error": true,
  "error_message": "Access to this file is not allowed: D:/forbidden/file.txt"
}
```

Common error scenarios:
- Path not in allowed_dirs
- File extension not in allowed_extensions
- File/directory does not exist
- Permission denied

---

*Last Updated: November 17, 2025*
