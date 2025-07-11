# TickTick MCP Troubleshooting Guide

This guide documents critical issues discovered through extensive testing and provides solutions for common problems.

## 🚨 Critical Issue: Silent Task Creation Failures

### Problem
Tasks created via API return success (200) but don't appear in TickTick apps due to hidden character restrictions.

### Root Cause
TickTick API has undocumented character filtering that silently drops tasks containing specific characters.

## 🔍 Character Compatibility Testing Results

We tested 10 different character types systematically. Here are the definitive results:

### ❌ Characters That Break Sync (AVOID)

| Character Type | Example | Status | Notes |
|---------------|---------|--------|-------|
| **Newlines** | `"Line 1\nLine 2"` | ❌ Breaks | Both literal `\n` and actual line breaks |
| **Backslashes** | `"C:\\Users\\folder"` | ❌ Breaks | Any backslash including escape sequences |

### ✅ Characters That Work (SAFE TO USE)

| Character Type | Example | Status | Notes |
|---------------|---------|--------|-------|
| **Emojis** | `"💻🚀📊✅❌🔧🎯📁⚡🤖"` | ✅ Works | All Unicode emojis tested successfully |
| **Arrows** | `"→ ← ↑ ↓ ↗ ↖ ↘ ↙ ⇒ ⇐"` | ✅ Works | All directional arrows work |
| **Bullets** | `"• ○ ■ ▪ ▫ ◦ ‣ ★ ☆"` | ✅ Works | All bullet point characters |
| **Unicode** | `"ñ é ü ç § ± © ® ™ ° ¿ ¡"` | ✅ Works | Extended Unicode characters |
| **JSON** | `"{ } [ ] [1,2,3] {\"key\": \"value\"}"` | ✅ Works | JSON syntax characters |
| **Markdown** | `"# Header *bold* **strong** - list"` | ✅ Works | Markdown formatting |
| **Quotes** | `"He said 'hello world' to everyone"` | ✅ Works | Regular quotes in content |
| **Mixed Safe** | `"💻 Code → Test 🚀 Deploy • Done"` | ✅ Works | Combinations of safe characters |

## 🔧 Working Commands

### Test Your Setup
```bash
# 1. Test authentication
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_get_projects","arguments":{}}}' | node src/index.js

# 2. Create a simple test task
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_create_task","arguments":{"title":"Test Task","content":"This should appear in your TickTick app"}}}' | node src/index.js

# 3. Test with safe special characters  
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_create_task","arguments":{"title":"💻 Special Chars Test","content":"Emojis 🚀, arrows →, bullets •, unicode é all work perfectly!"}}}' | node src/index.js
```

### Create Tasks Safely
```bash
# ✅ Good - Using safe characters
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_create_task","arguments":{"title":"Project Planning","content":"Phase 1: Research → Phase 2: Development → Phase 3: Testing 🚀"}}}' | node src/index.js

# ❌ Bad - Using problematic characters (will not sync)
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_create_task","arguments":{"title":"Bad Example","content":"Line 1\nLine 2\nPath: C:\\Users\\folder"}}}' | node src/index.js
```

## 🎉 BREAKTHROUGH: Task Reading Fixed!

### ✅ Task Reading Now Works!
**Solution**: Use correct endpoint pattern with both project_id and task_id

```bash
# ✅ WORKING - Read specific task
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_get_task_details","arguments":{"project_id":"YOUR_PROJECT_ID","task_id":"YOUR_TASK_ID"}}}' | node src/index.js
```

**Key Requirements**:
- Must provide both `project_id` and `task_id`
- Use endpoint pattern: `/project/{project_id}/task/{task_id}`
- Returns full task details with 200 status

### 🚨 Still Failing Operations
**Problem**: Update/Delete operations still return 500 errors
```bash
# ❌ Still failing:
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_update_task","arguments":{"task_id":"123","title":"Updated"}}}' | node src/index.js
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_delete_task","arguments":{"task_id":"123"}}}' | node src/index.js
```

### API Operations Status
| Operation | Status | Notes |
|-----------|--------|--------|
| Create Task | ✅ Works | With character limitations |
| Create Project | ✅ Works | Fully functional |
| **Read Task Details** | ✅ **FIXED!** | Requires project_id + task_id |
| Update Task | ❌ Fails | 500 errors |
| Delete Task | ❌ Fails | 500 errors |
| Get Projects | ✅ Works | Fully functional |

## 🗂️ Cache Implementation Issues & Solutions

### ✅ NEW: Task Cache System Working!

**Solution**: Implemented local cache system to solve the "give me all tasks" UX problem.

### 🔧 Cache Testing Results

| Feature | Status | Notes |
|---------|--------|--------|
| **Task Registration** | ✅ Works | Manual task ID registration to cache |
| **Cache Persistence** | ✅ Works | File saved to `~/.ticktick-mcp-cache.json` |
| **Auto-Registration** | ✅ Works | New tasks automatically cached on creation |
| **Cache Retrieval** | ✅ Works | List all cached tasks with formatting |
| **CSV Import** | ⚠️ Partial | Requires raw CSV data, not file path |

### 🚨 Common Cache Issues

#### 1. Directory Navigation Problems
**Problem**: Running MCP commands from wrong directory causes "Cannot find module" errors

```bash
# ❌ This fails when run from wrong directory:
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_get_cached_tasks","arguments":{}}}' | node src/index.js

# ✅ Use full path instead:
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_get_cached_tasks","arguments":{}}}' | node /Users/liadgez/Documents/mcp-services/ticktick-mcp/src/index.js
```

#### 2. CSV Import Parameter Confusion
**Problem**: CSV import expects raw CSV data string, not file path

```bash
# ❌ This fails - trying to pass file path:
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_import_from_csv","arguments":{"csv_file_path":"path/to/file.csv"}}}' | node src/index.js

# ✅ This works - passing raw CSV data:
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_import_from_csv","arguments":{"csv_data":"task_id,project_id,title\ntask001,project123,Test Task"}}}' | node src/index.js
```

### 🔧 Working Cache Commands

#### Test Cache Functionality
```bash
# 1. Register a task manually
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_register_task_id","arguments":{"task_id":"test123","project_id":"project456","title":"Test Task"}}}' | node /full/path/to/ticktick-mcp/src/index.js

# 2. View all cached tasks
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_get_cached_tasks","arguments":{}}}' | node /full/path/to/ticktick-mcp/src/index.js

# 3. Create task (auto-caches)
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_create_task","arguments":{"title":"Auto Cache Test","project_id":"YOUR_PROJECT_ID"}}}' | node /full/path/to/ticktick-mcp/src/index.js
```

#### Import CSV Data
```bash
# Create CSV data with proper format
CSV_DATA="task_id,project_id,title
task001,project123,Design homepage
task002,project123,Write documentation"

# Import to cache
echo "{\"jsonrpc\":\"2.0\",\"id\":1,\"method\":\"tools/call\",\"params\":{\"name\":\"ticktick_import_from_csv\",\"arguments\":{\"csv_data\":\"$CSV_DATA\"}}}" | node /full/path/to/ticktick-mcp/src/index.js
```

### 💡 Cache Usage Tips

1. **Cache File Location**: `~/.ticktick-mcp-cache.json`
2. **TTL**: Tasks expire after 24 hours (marked as "stale")
3. **Auto-Registration**: All new tasks via `ticktick_create_task()` are auto-cached
4. **Project Filtering**: Use `project_id` parameter in `ticktick_get_cached_tasks()` to filter by project
5. **CSV Format**: Must include `task_id,project_id,title` columns minimum

### 🎯 **CRITICAL: Inbox Project ID Discovery**

**Inbox Project ID**: `inbox125308274`

```bash
# ✅ Reading inbox tasks - WORKING METHOD:
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_get_task_details","arguments":{"project_id":"inbox125308274","task_id":"YOUR_TASK_ID"}}}' | node /full/path/to/ticktick-mcp/src/index.js

# ✅ Create task in inbox (gets inbox project ID automatically):
echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_create_task","arguments":{"title":"Test Task","content":"Test content"}}}' | node /full/path/to/ticktick-mcp/src/index.js
```

**Key Discovery**: Inbox has special project ID `inbox125308274` - not visible in projects list but required for task reading.

## 📋 Debugging Checklist

### Task Not Appearing in TickTick App?

1. **Check content for problematic characters**:
   ```bash
   # Check your task content for:
   - Newlines (\n or actual line breaks)
   - Backslashes (\\ or \")
   - Escape sequences
   ```

2. **Verify API response**:
   - Look for 200 status in API response
   - Check if task ID is returned
   - Verify project ID is correct

3. **Test with minimal content**:
   ```bash
   echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_create_task","arguments":{"title":"Debug Test","content":"Simple test"}}}' | node src/index.js
   ```

4. **Check TickTick app sync**:
   - Force refresh in TickTick app
   - Check both mobile and web versions
   - Verify you're logged into same account

### Environment Issues

1. **Token problems**:
   ```bash
   # Verify token is set
   echo "Token length: ${#TICKTICK_ACCESS_TOKEN}"
   
   # Test authentication
   echo '{"jsonrpc":"2.0","id":1,"method":"tools/call","params":{"name":"ticktick_get_projects","arguments":{}}}' | node src/index.js
   ```

2. **Node.js version**:
   ```bash
   node --version  # Should be 18+
   npm list        # Check dependencies
   ```

## 🔬 Testing Methodology

Our testing approach for discovering these limitations:

1. **Systematic Character Testing**: Created 10 test tasks with different character types
2. **API Response Analysis**: All returned 200 success codes
3. **App Verification**: Manually checked which tasks appeared in TickTick app
4. **Pattern Recognition**: Identified specific characters causing sync failures
5. **Validation**: Confirmed patterns with additional focused tests

## 📝 Best Practices

### Content Guidelines
- **Use emojis freely**: They enhance readability and work perfectly
- **Use arrows for workflows**: → ← ↑ ↓ are safe and clear
- **Avoid line breaks**: Use bullets • or arrows → instead
- **Avoid file paths**: Don't include backslash-heavy paths in content
- **Test new patterns**: Before bulk operations, test with single tasks

### Development Workflow
1. **Always test task creation** with a simple example first
2. **Use character compatibility test** before complex content
3. **Monitor TickTick app** to verify sync success
4. **Keep content simple** when possible
5. **Report new patterns** if you discover additional limitations

## 🐛 Reporting Issues

If you discover new sync patterns or API limitations:

1. **Document the exact command** that fails
2. **Include API response** (status codes, error messages)
3. **Test with simplified content** to isolate the issue
4. **Create GitHub issue** with reproduction steps

## 🔗 Resources

- [GitHub Issues](https://github.com/liadgez/ticktick-mcp-server/issues)
- [TickTick API Documentation](https://developer.ticktick.com/)
- [MCP Protocol Specification](https://github.com/modelcontextprotocol)

---

**Last Updated**: 2025-07-07  
**Testing Environment**: TickTick Open API v1, Node.js 24.3.0