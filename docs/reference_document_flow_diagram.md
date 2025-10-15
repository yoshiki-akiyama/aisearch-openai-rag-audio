# RAG Reference Document Display - Visual Flow Diagram

## Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           USER INTERACTION                               │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ▼
                    User asks a question (voice)
                                 │
┌────────────────────────────────┴────────────────────────────────────────┐
│                        BACKEND PROCESSING                                │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Step 1: AI Model receives question                                     │
│          │                                                               │
│          ▼                                                               │
│  Step 2: AI calls "search" tool                                         │
│          │                                                               │
│          ├─────► _search_tool() in ragtools.py                          │
│          │       │                                                       │
│          │       ├─────► Azure AI Search (hybrid search)                │
│          │       │       Returns: [chunk_id]: content                   │
│          │       │                                                       │
│          │       └─────► Returns results to AI                          │
│          │                                                               │
│          ▼                                                               │
│  Step 3: AI generates answer using search results                       │
│          │                                                               │
│          ▼                                                               │
│  Step 4: AI calls "report_grounding" tool                               │
│          │       args: { sources: ["chunk_id1", "chunk_id2", ...] }     │
│          │                                                               │
│          ├─────► _report_grounding_tool() in ragtools.py                │
│          │       │                                                       │
│          │       ├── Step 4.1: Filter source IDs (security)             │
│          │       │    sources = [s for s in args["sources"]             │
│          │       │                if KEY_PATTERN.match(s)]              │
│          │       │                                                       │
│          │       ├── Step 4.2: Search Azure AI Search                   │
│          │       │    search_text=list,                                 │
│          │       │    search_fields=[identifier_field],                 │
│          │       │    select=[identifier_field, title_field,            │
│          │       │            content_field]                            │
│          │       │                                                       │
│          │       ├── Step 4.3: Package results                          │
│          │       │    docs = [{                                         │
│          │       │      "chunk_id": r[identifier_field],                │
│          │       │      "title": r[title_field],                        │
│          │       │      "chunk": r[content_field]                       │
│          │       │    }, ...]                                           │
│          │       │                                                       │
│          │       └── Step 4.4: Send to frontend                         │
│          │           ToolResult({"sources": docs},                      │
│          │                      ToolResultDirection.TO_CLIENT)          │
│          │                                                               │
└──────────┼───────────────────────────────────────────────────────────────┘
           │
           │ WebSocket Message
           │ {
           │   type: "extension.middle_tier_tool.response",
           │   tool_result: JSON.stringify({
           │     sources: [
           │       {chunk_id: "...", title: "...", chunk: "..."},
           │       ...
           │     ]
           │   })
           │ }
           │
           ▼
┌──────────┴───────────────────────────────────────────────────────────────┐
│                        FRONTEND PROCESSING                               │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Step 5: Receive tool response (App.tsx)                                │
│          │                                                               │
│          ├── onReceivedExtensionMiddleTierToolResponse callback         │
│          │                                                               │
│          ├── Step 5.1: Parse JSON string                                │
│          │   const result: ToolResult =                                 │
│          │     JSON.parse(message.tool_result);                         │
│          │                                                               │
│          ├── Step 5.2: Transform data structure                         │
│          │   const files: GroundingFile[] =                             │
│          │     result.sources.map(x => ({                               │
│          │       id: x.chunk_id,      // chunk_id → id                 │
│          │       name: x.title,        // title → name                  │
│          │       content: x.chunk      // chunk → content               │
│          │     }));                                                      │
│          │                                                               │
│          └── Step 5.3: Update state (accumulative)                      │
│              setGroundingFiles(prev => [...prev, ...files]);            │
│                                                                          │
└──────────┬───────────────────────────────────────────────────────────────┘
           │
           ▼
┌──────────┴───────────────────────────────────────────────────────────────┐
│                           UI RENDERING                                   │
├──────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  Step 6: GroundingFiles component (grounding-files.tsx)                 │
│          │                                                               │
│          ├── Checks if files.length > 0                                 │
│          │   If empty, returns null (no display)                        │
│          │                                                               │
│          ├── Renders Card with title and description                    │
│          │                                                               │
│          └── Maps over files array                                      │
│              │                                                           │
│              └── For each file, renders GroundingFile component         │
│                  (grounding-file.tsx)                                    │
│                  │                                                       │
│                  └── Button with file icon and name                     │
│                      onClick → setSelectedFile(file)                    │
│                                                                          │
│  Step 7: User clicks a grounding file button                            │
│          │                                                               │
│          └── selectedFile state is updated                              │
│                                                                          │
│  Step 8: GroundingFileView component (grounding-file-view.tsx)          │
│          │                                                               │
│          ├── Modal overlay appears (if groundingFile !== null)          │
│          │                                                               │
│          ├── Displays document title (groundingFile.name)               │
│          │                                                               │
│          ├── Displays full content (groundingFile.content)              │
│          │   in scrollable <pre><code> element                          │
│          │                                                               │
│          └── Close button → setSelectedFile(null)                       │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## Data Structure Transformation

### Backend (Python) → Frontend (TypeScript)

```
Backend Structure (_report_grounding_tool output):
┌─────────────────────────────────────────┐
│ ToolResult {                            │
│   sources: [                            │
│     {                                   │
│       chunk_id: "doc1_chunk_5",         │
│       title: "Company Overview",        │
│       chunk: "Contoso Electronics..."   │
│     },                                  │
│     ...                                 │
│   ]                                     │
│ }                                       │
└─────────────────────────────────────────┘
                    │
                    │ JSON.stringify() → WebSocket
                    ▼
                    │ JSON.parse()
                    ▼
Frontend Structure (GroundingFile):
┌─────────────────────────────────────────┐
│ GroundingFile[] = [                     │
│   {                                     │
│     id: "doc1_chunk_5",                 │
│     name: "Company Overview",           │
│     content: "Contoso Electronics..."   │
│   },                                    │
│   ...                                   │
│ ]                                       │
└─────────────────────────────────────────┘
```

## Component Hierarchy

```
App.tsx
│
├── State: groundingFiles: GroundingFile[]
├── State: selectedFile: GroundingFile | null
│
├── GroundingFiles (if groundingFiles.length > 0)
│   │
│   └── For each file:
│       GroundingFile (button)
│       │
│       └── onClick → setSelectedFile(file)
│
└── GroundingFileView (if selectedFile !== null)
    │
    ├── Modal overlay
    ├── Document title (selectedFile.name)
    ├── Document content (selectedFile.content)
    └── Close button → setSelectedFile(null)
```

## Key Configuration Points

### Environment Variables (Backend)

```
.env or Azure App Settings:
├── AZURE_SEARCH_ENDPOINT=https://your-service.search.windows.net
├── AZURE_SEARCH_INDEX=your-index-name
├── AZURE_SEARCH_IDENTIFIER_FIELD=chunk_id  ← Maps to GroundingFile.id
├── AZURE_SEARCH_TITLE_FIELD=title          ← Maps to GroundingFile.name
└── AZURE_SEARCH_CONTENT_FIELD=chunk        ← Maps to GroundingFile.content
```

### Tool Registration (app.py)

```python
attach_rag_tools(
    rtmt,
    credentials=search_credential,
    search_endpoint=os.environ["AZURE_SEARCH_ENDPOINT"],
    search_index=os.environ["AZURE_SEARCH_INDEX"],
    identifier_field=os.environ.get("AZURE_SEARCH_IDENTIFIER_FIELD") or "chunk_id",
    title_field=os.environ.get("AZURE_SEARCH_TITLE_FIELD") or "title",
    content_field=os.environ.get("AZURE_SEARCH_CONTENT_FIELD") or "chunk",
    ...
)
```

## Security Considerations

1. **Source ID Validation**: Only alphanumeric IDs are allowed
   ```python
   KEY_PATTERN = re.compile(r'^[a-zA-Z0-9_=\-]+$')
   sources = [s for s in args["sources"] if KEY_PATTERN.match(s)]
   ```

2. **Search vs Filter**: Using search API instead of filter to avoid SQL-like injection
   ```python
   search_results = await search_client.search(
       search_text=list,  # Safe: search text
       search_fields=[identifier_field],  # Limited to identifier field
       query_type="full"
   )
   ```

## Performance Notes

- **Accumulative Display**: All grounding files from the conversation are kept in state
  ```typescript
  setGroundingFiles(prev => [...prev, ...files]);
  ```
  
- **Lazy Loading**: Modal content only rendered when file is selected

- **Animation**: Framer Motion provides smooth transitions without blocking

## Future Enhancements (from TODO comment)

```python
# TODO: move from sending all chunks used for grounding eagerly to only sending links to 
# the original content in storage, it'll be more efficient overall
```

Current: Full chunk content sent immediately
Proposed: Send only links/IDs, fetch content on-demand when user clicks
