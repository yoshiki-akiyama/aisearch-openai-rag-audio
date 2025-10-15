# Reference Document Display Logic Documentation

This directory contains detailed documentation about how the VoiceRAG application displays reference documents (grounding sources) used by the AI in generating answers.

## Available Documentation

### Japanese Documentation
- **[reference_document_display_logic.md](./reference_document_display_logic.md)** - Comprehensive Japanese documentation explaining the complete flow of how reference documents are displayed in the RAG system.

### What's Documented

The documentation covers:

1. **Architecture Overview** - High-level flow of how grounding sources are retrieved and displayed
2. **Backend Logic** (`app/backend/ragtools.py`) - Detailed explanation of the `_report_grounding_tool` function
3. **Frontend Logic** (`app/frontend/src/App.tsx` and UI components) - How grounding files are received, transformed, and displayed
4. **Data Flow** - Complete data transformation pipeline from Azure AI Search to UI
5. **Customization Points** - How to customize field names, search methods, and UI components
6. **Troubleshooting** - Common issues and solutions

## Quick Overview

### Flow Summary

```
User asks question
    ↓
AI searches knowledge base (search tool)
    ↓
AI generates answer using retrieved sources
    ↓
AI reports grounding sources (report_grounding tool)
    ↓
Backend retrieves full document details from Azure AI Search
    ↓
Backend sends {chunk_id, title, chunk} to frontend
    ↓
Frontend transforms to {id, name, content}
    ↓
UI displays as clickable document chips
    ↓
User clicks to view full document content
```

### Key Files

**Backend:**
- `app/backend/ragtools.py` - Contains `_report_grounding_tool()` function

**Frontend:**
- `app/frontend/src/App.tsx` - Receives and transforms grounding data
- `app/frontend/src/components/ui/grounding-files.tsx` - Lists grounding documents
- `app/frontend/src/components/ui/grounding-file.tsx` - Individual document button
- `app/frontend/src/components/ui/grounding-file-view.tsx` - Full document modal
- `app/frontend/src/types.ts` - Type definitions for GroundingFile and ToolResult

### Code Comments

All key functions now include inline comments explaining:
- Purpose and functionality
- Step-by-step processing flow
- Data transformations
- Important behavior notes

## Configuration

Reference document display can be customized via environment variables:

```bash
AZURE_SEARCH_IDENTIFIER_FIELD=chunk_id  # Field name for chunk ID
AZURE_SEARCH_TITLE_FIELD=title          # Field name for document title  
AZURE_SEARCH_CONTENT_FIELD=chunk        # Field name for document content
```

These are configured in `app/backend/app.py` when calling `attach_rag_tools()`.

## For Developers

When modifying the reference document display logic:

1. **Backend changes**: Update `_report_grounding_tool()` in `ragtools.py`
2. **Data structure changes**: Update both `ToolResult` type (types.ts) and the mapping in App.tsx
3. **UI changes**: Modify the grounding-*.tsx components in `app/frontend/src/components/ui/`

All changes should maintain the flow: Backend → WebSocket → Frontend → UI Components
