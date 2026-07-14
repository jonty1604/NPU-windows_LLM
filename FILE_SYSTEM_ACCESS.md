# File System Access for NPU Model

The server now supports **file system access via APIs** so your model can read, modify, create, and delete files—just like the CLI does.

## Available Endpoints

### 1. **Set Working Directory**
```
POST /v1/fs/set-working-dir
{
  "path": "/path/to/your/project"
}
```
Response: `{ "success": true, "working_dir": "..." }`

### 2. **Get Current Working Directory**
```
GET /v1/fs/working-dir
```
Response: `{ "working_dir": "..." }`

### 3. **List Files in Directory**
```
POST /v1/fs/list
{
  "path": "." or "subdir/path"
}
```
Response:
```json
{
  "path": ".",
  "items": [
    { "name": "file.txt", "type": "file", "size": 1024 },
    { "name": "folder", "type": "dir", "size": null }
  ]
}
```

### 4. **Read File**
```
POST /v1/fs/read
{
  "file_path": "src/main.py"
}
```
Response: `{ "path": "src/main.py", "content": "..." }`

### 5. **Write File**
```
POST /v1/fs/write
{
  "file_path": "src/new_file.py",
  "content": "#!/usr/bin/env python3\n..."
}
```
Response: `{ "success": true, "path": "src/new_file.py" }`

### 6. **Edit File**
```
POST /v1/fs/edit
{
  "file_path": "src/main.py",
  "old_str": "old code here",
  "new_str": "new code here"
}
```
Response: `{ "success": true, "path": "src/main.py" }`

## How to Use via UI/Chat

1. **Set working directory first:**
   - Message: "Set working directory to C:\path\to\my\project"
   - Model will call: `POST /v1/fs/set-working-dir`

2. **Ask model to read/modify files:**
   - Message: "Read the file src/utils.py"
   - Model will call: `POST /v1/fs/read` with `{"file_path": "src/utils.py"}`
   
3. **Request code changes:**
   - Message: "In index.html, change the title from 'Chat' to 'My App'"
   - Model will call: `POST /v1/fs/edit` with old/new text

4. **Create new files:**
   - Message: "Create a new file config.json with ..."
   - Model will call: `POST /v1/fs/write`

## How Model Uses These Tools

The model can now use file system operations when you ask it to:
- **Read code**: "What does main.py do?"
- **Modify files**: "Change all 'TODO' comments to 'DONE'"
- **Create files**: "Create a new .gitignore with standard entries"
- **Refactor**: "Move the auth logic to a separate module"
- **Debug**: "Find where the error is coming from in my code"

## Security

- ✓ Path traversal protection (can't access files outside working dir)
- ✓ File size limits (large files auto-truncated at 50KB display)
- ✓ Limited to 100 items per directory listing
- ✓ Errors are caught and returned safely

## Example Flow

**You:** "Set working directory to C:\my-project and read all Python files"

**Model:**
1. Calls `POST /v1/fs/set-working-dir` → Sets base path
2. Calls `POST /v1/fs/list` → Lists files in directory
3. Calls `POST /v1/fs/read` for each `.py` file → Reads content
4. Returns summary of what it found

**You:** "Now fix the indentation errors in utils.py"

**Model:**
1. Calls `POST /v1/fs/read` → Gets current content
2. Calls `POST /v1/fs/edit` → Fixes indentation
3. Confirms success

## Next Steps

To integrate with UI:
1. Add a file browser panel to index.html (left sidebar)
2. Show working directory + ability to change it
3. Display file tree with read/edit buttons
4. Show model's file operations in the chat (tool calls visualization)

For now, you can test via:
- **curl commands** (examples below)
- **Model chat** - just ask it to work with your files
- **Direct API calls** from Postman, Python, etc.

### Test via curl

```bash
# Set working dir
curl -X POST http://localhost:8000/v1/fs/set-working-dir \
  -H "Content-Type: application/json" \
  -d '{"path": "C:\\my-project"}'

# List files
curl -X POST http://localhost:8000/v1/fs/list \
  -H "Content-Type: application/json" \
  -d '{"path": "."}'

# Read file
curl -X POST http://localhost:8000/v1/fs/read \
  -H "Content-Type: application/json" \
  -d '{"file_path": "src/main.py"}'
```

## Status

✅ Backend APIs implemented & working  
✅ Security (path traversal protection)  
✅ Model can call these tools  
⏳ UI file browser (optional enhancement)  

You can now use the model as a **local code assistant** that reads, modifies, and creates files in your projects!
