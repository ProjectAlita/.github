---
agent: 'sdk-dev'
description: 'Specialized skill for adding vector store indexing and search capabilities to Alita SDK toolkits'
---

# Vector Store Integrator Skill

## Purpose

This skill focuses on adding vector store capabilities to toolkits, enabling document indexing, semantic search, and RAG (Retrieval Augmented Generation) patterns. It's essential for toolkits that manage documents, issues, wikis, or any text-based content that needs to be searched.

## When to Use This Skill

- Adding indexing capabilities to a document management toolkit
- Implementing semantic search for issues, tickets, or content
- Building RAG-enabled tools for knowledge bases
- Creating searchable indexes of external system data
- Implementing stepback prompting or summary generation

## Key Concepts

### Vector Store Architecture

```
External System → Loader → Documents → Processing → Chunking → Embedding → Vector Store
                                                                              ↓
                                                                          Search/Query
```

### Base Classes

1. **BaseVectorStoreToolApiWrapper**: For document-based content (Confluence, SharePoint, Jira)
2. **BaseCodeToolApiWrapper**: For code repositories (GitHub, GitLab, Bitbucket)

### Core Methods to Implement

| Method | Purpose | Required |
|--------|---------|----------|
| `_base_loader()` | Load documents from external system | Yes |
| `_process_document()` | Extract metadata and enrich documents | Optional |
| `_index_tool_params()` | Define custom indexing parameters | Optional |
| `get_index_data_tool()` | Customize index_data tool | Optional |

## Implementation Pattern

### Step 1: Choose Base Class and Set Doctype

```python
from typing import Generator, Optional, Dict, Any
from langchain_core.documents import Document
from ..elitea_base import BaseVectorStoreToolApiWrapper

class MyServiceApiWrapper(BaseVectorStoreToolApiWrapper):
    """API wrapper with vector store support."""
    
    # Set document type: 'document' for text, 'code' for source code
    doctype: str = "document"
    
    # Configuration fields
    base_url: str
    api_key: SecretStr
    
    # Vector store fields (inherited, can override defaults)
    # llm: Any                           # LLM for operations
    # embedding_model: str               # Embedding model name
    # vectorstore_type: str              # Type of vector store
    # collection_name: str               # Collection name
    # connection_string: SecretStr       # DB connection string
```

### Step 2: Implement Base Loader

The `_base_loader()` method fetches raw data and yields base documents with minimal metadata:

```python
def _base_loader(self, **kwargs) -> Generator[Document, None, None]:
    """
    Load documents from external system.
    
    This method should:
    1. Fetch data from external API
    2. Create Document objects with page_content
    3. Add minimal metadata: id, created_on
    4. Yield documents (don't process deeply here)
    
    Args:
        **kwargs: Custom parameters from index_data tool
        
    Yields:
        Document objects with basic metadata
    """
    # Get custom parameters
    project_key = kwargs.get('project_key')
    include_archived = kwargs.get('include_archived', False)
    
    self._log_tool_event(
        f"Loading documents from project: {project_key}",
        tool_name="_base_loader"
    )
    
    # Fetch items from API (implement pagination)
    page = 1
    page_size = 100
    total_loaded = 0
    
    while True:
        # API call to get page of items
        response = self._make_request('GET', '/api/items', params={
            'project': project_key,
            'page': page,
            'page_size': page_size,
            'include_archived': include_archived
        })
        
        items = response.get('items', [])
        if not items:
            break
        
        for item in items:
            # Create document with minimal metadata
            doc = Document(
                page_content=self._extract_content(item),
                metadata={
                    'id': item['id'],                    # Required: unique ID
                    'created_on': item['created_at'],    # Required: timestamp
                    'type': item['type'],                # Optional: item type
                    'url': item['url'],                  # Optional: link
                }
            )
            
            yield doc
            total_loaded += 1
        
        # Report progress
        if total_loaded % 100 == 0:
            self._log_tool_event(
                f"Loaded {total_loaded} documents",
                tool_name="_base_loader"
            )
        
        page += 1
        
        # Check if we've reached the last page
        if len(items) < page_size:
            break
    
    self._log_tool_event(
        f"Total documents loaded: {total_loaded}",
        tool_name="_base_loader"
    )

def _extract_content(self, item: dict) -> str:
    """
    Extract text content from item.
    
    Combines title, description, and other text fields.
    """
    parts = []
    
    if item.get('title'):
        parts.append(f"# {item['title']}")
    
    if item.get('description'):
        parts.append(item['description'])
    
    if item.get('comments'):
        parts.append("\n## Comments:")
        for comment in item['comments']:
            parts.append(f"- {comment['text']}")
    
    return "\n\n".join(parts)
```

### Step 3: Implement Document Processing (Optional)

The `_process_document()` method enriches documents after deduplication:

```python
def _process_document(self, base_document: Document) -> Generator[Document, None, None]:
    """
    Process document to extract additional metadata.
    
    This method is called AFTER deduplication, so only process
    documents that will actually be indexed. This is where you:
    1. Extract dependencies and relationships
    2. Parse structured data
    3. Add computed metadata
    4. Split into multiple documents if needed
    
    Args:
        base_document: Document with basic metadata from _base_loader
        
    Yields:
        Processed documents with enriched metadata
    """
    doc_id = base_document.metadata.get('id')
    
    self._log_tool_event(
        f"Processing document: {doc_id}",
        tool_name="_process_document"
    )
    
    # Fetch additional details if needed
    try:
        details = self._make_request('GET', f'/api/items/{doc_id}')
        
        # Extract dependencies
        dependencies = []
        for link in details.get('links', []):
            dependencies.append({
                'id': link['target_id'],
                'type': link['type'],
                'name': link['target_name']
            })
        
        # Extract keywords/tags
        keywords = []
        if details.get('tags'):
            keywords.extend(details['tags'])
        if details.get('labels'):
            keywords.extend(details['labels'])
        
        # Add enriched metadata
        base_document.metadata.update({
            'dependencies': dependencies,
            'keywords': keywords,
            'author': details.get('author', {}).get('name'),
            'status': details.get('status'),
            'priority': details.get('priority'),
            'updated_on': details.get('updated_at'),
        })
        
        # You can yield multiple documents from one base document
        # For example, split by sections
        yield base_document
        
    except Exception as e:
        self._log_tool_event(
            f"Error processing document {doc_id}: {str(e)}",
            tool_name="_process_document"
        )
        # Still yield the base document even if processing fails
        yield base_document
```

### Step 4: Add Custom Index Parameters (Optional)

Define toolkit-specific parameters for the index_data tool:

```python
def _index_tool_params(self, **kwargs) -> dict:
    """
    Define custom parameters for index_data tool.
    
    Returns:
        Dictionary mapping parameter names to (type, Field) tuples
    """
    from pydantic import Field
    from typing import Optional, List
    
    return {
        'project_key': (str, Field(
            description="Project identifier to index documents from",
            min_length=1
        )),
        'include_archived': (bool, Field(
            description="Include archived items in indexing",
            default=False
        )),
        'item_types': (Optional[List[str]], Field(
            description="Filter by specific item types (e.g., ['story', 'bug'])",
            default=None
        )),
        'date_from': (Optional[str], Field(
            description="Index items created after this date (ISO format)",
            default=None,
            examples=['2024-01-01']
        )),
    }
```

### Step 5: Register Vector Store Tools

Add vector store tools to your `get_available_tools()` method:

```python
def get_available_tools(self):
    """Return all available tools including vector store tools."""
    tools = [
        # Your custom tools
        {
            "name": "get_items",
            "ref": self.get_items,
            "description": self.get_items.__doc__,
            "args_schema": create_model(...)
        },
        # ... more custom tools
    ]
    
    # Add vector store tools (search, index, etc.)
    tools.extend(self._get_vector_search_tools())
    
    # Add custom index_data tool
    tools.append(self.get_index_data_tool())
    
    return tools
```

## Advanced Patterns

### Pattern 1: Custom Chunking Strategy

Override chunking methods for specialized content:

```python
def _get_dependencies_chunker(self, document: Optional[Document] = None):
    """Return appropriate chunker for document type."""
    from alita_sdk.tools.chunkers import markdown_chunker, statistical_chunker
    
    # Use different chunker based on document type
    doc_type = document.metadata.get('type') if document else None
    
    if doc_type == 'code':
        from alita_sdk.tools.chunkers.code.codeparser import parse_code_files_for_db
        return parse_code_files_for_db
    elif doc_type == 'markdown':
        return markdown_chunker
    else:
        return statistical_chunker

def _get_dependencies_chunker_config(self, document: Optional[Document] = None):
    """Return chunker configuration."""
    return {
        'embedding': self._embedding,
        'llm': self.llm,
        'chunk_size': 1000,
        'chunk_overlap': 200
    }
```

### Pattern 2: Batch Document Processing

Process documents in batches for efficiency:

```python
def _process_documents(self, documents: List[Document]) -> Generator[Document, None, None]:
    """
    Process multiple documents efficiently.
    
    Override this method to batch process documents instead of one-by-one.
    """
    total_docs = len(documents)
    batch_size = 50
    
    self._log_tool_event(
        f"Processing {total_docs} documents in batches of {batch_size}",
        tool_name="_process_documents"
    )
    
    for i in range(0, total_docs, batch_size):
        batch = documents[i:i + batch_size]
        
        # Batch API call to get details for all docs
        doc_ids = [doc.metadata['id'] for doc in batch]
        details_map = self._batch_get_details(doc_ids)
        
        # Enrich documents with fetched details
        for doc in batch:
            doc_id = doc.metadata['id']
            if doc_id in details_map:
                doc.metadata.update(details_map[doc_id])
            
            # Apply chunking
            chunker = self._get_dependencies_chunker(doc)
            if chunker:
                yield from chunker(
                    file_content_generator=iter([doc]),
                    config=self._get_dependencies_chunker_config(doc)
                )
            else:
                yield doc
        
        self._log_tool_event(
            f"Processed {min(i + batch_size, total_docs)}/{total_docs} documents",
            tool_name="_process_documents"
        )

def _batch_get_details(self, doc_ids: List[str]) -> Dict[str, dict]:
    """Fetch details for multiple documents in one API call."""
    response = self._make_request('POST', '/api/items/batch', json={
        'ids': doc_ids
    })
    
    # Return mapping of id -> details
    return {item['id']: item for item in response.get('items', [])}
```

### Pattern 3: Metadata Extraction with Dependencies

Extract relationships and dependencies:

```python
def _process_document(self, base_document: Document) -> Generator[Document, None, None]:
    """Extract dependencies and create relationship metadata."""
    from alita_sdk.runtime.utils.utils import IndexerKeywords
    
    doc_id = base_document.metadata.get('id')
    details = self._fetch_item_details(doc_id)
    
    # Extract dependencies
    dependencies = []
    for link in details.get('links', []):
        dep = {
            'id': link['related_id'],
            'type': link['link_type'],
            'direction': link.get('direction', 'outbound')
        }
        dependencies.append(dep)
    
    # Store dependencies in metadata
    if dependencies:
        base_document.metadata['dependencies'] = dependencies
        
        # Use IndexerKeywords for standard dependency tracking
        base_document.metadata[IndexerKeywords.DEPENDENCIES.value] = [
            dep['id'] for dep in dependencies
        ]
    
    # Set parent-child relationship if applicable
    if details.get('parent_id'):
        base_document.metadata[IndexerKeywords.PARENT.value] = details['parent_id']
    
    yield base_document
```

### Pattern 4: Custom Search Filters

Implement toolkit-specific search filters:

```python
def search_by_project(
    self,
    query: str,
    project_key: str,
    status: Optional[str] = None,
    search_top: int = 10
) -> str:
    """
    Search documents within a specific project.
    
    Args:
        query: Search query text
        project_key: Project to search within
        status: Optional status filter
        search_top: Number of results to return
        
    Returns:
        Search results as formatted string
    """
    # Build filter
    filter_dict = {
        "metadata.project_key": {"$eq": project_key}
    }
    
    if status:
        filter_dict["metadata.status"] = {"$eq": status}
    
    # Use base search method with custom filter
    return self.search_index(
        query=query,
        index_name="",  # Search all collections
        filter=filter_dict,
        search_top=search_top
    )
```

## Complete Example: Confluence-like Toolkit

```python
from typing import Generator, Optional, List, Dict, Any
from langchain_core.documents import Document
from pydantic import Field, SecretStr, create_model
from ..elitea_base import BaseVectorStoreToolApiWrapper
import requests

class WikiApiWrapper(BaseVectorStoreToolApiWrapper):
    """API wrapper for Wiki with vector store support."""
    
    doctype: str = "document"
    
    # Configuration
    base_url: str = Field(description="Wiki base URL")
    api_token: SecretStr = Field(description="API token")
    
    def _base_loader(self, **kwargs) -> Generator[Document, None, None]:
        """Load pages from wiki."""
        space_key = kwargs.get('space_key', 'ALL')
        
        self._log_tool_event(f"Loading pages from space: {space_key}", "_base_loader")
        
        # Paginate through pages
        start = 0
        limit = 100
        
        while True:
            response = self._make_request('GET', '/api/content', params={
                'spaceKey': space_key,
                'start': start,
                'limit': limit,
                'expand': 'body.storage,version,space'
            })
            
            pages = response.get('results', [])
            if not pages:
                break
            
            for page in pages:
                # Extract HTML content and convert to text
                content = self._html_to_text(page['body']['storage']['value'])
                
                yield Document(
                    page_content=content,
                    metadata={
                        'id': page['id'],
                        'created_on': page['version']['when'],
                        'title': page['title'],
                        'space': page['space']['key'],
                        'url': f"{self.base_url}/pages/viewpage.action?pageId={page['id']}"
                    }
                )
            
            start += limit
            
            if len(pages) < limit:
                break
    
    def _process_document(self, base_document: Document) -> Generator[Document, None, None]:
        """Enrich page with additional metadata."""
        page_id = base_document.metadata['id']
        
        # Fetch child pages (dependencies)
        children = self._make_request('GET', f'/api/content/{page_id}/child/page')
        child_ids = [child['id'] for child in children.get('results', [])]
        
        # Fetch labels
        labels = self._make_request('GET', f'/api/content/{page_id}/label')
        label_names = [label['name'] for label in labels.get('results', [])]
        
        # Update metadata
        base_document.metadata.update({
            'child_pages': child_ids,
            'labels': label_names,
            'has_children': len(child_ids) > 0
        })
        
        yield base_document
    
    def _index_tool_params(self, **kwargs):
        """Custom indexing parameters."""
        return {
            'space_key': (str, Field(
                description="Space key to index (or 'ALL' for all spaces)",
                default="ALL"
            )),
            'include_archived': (bool, Field(
                description="Include archived pages",
                default=False
            ))
        }
    
    def _html_to_text(self, html: str) -> str:
        """Convert HTML content to plain text."""
        from bs4 import BeautifulSoup
        soup = BeautifulSoup(html, 'html.parser')
        return soup.get_text(separator='\n', strip=True)
    
    def get_available_tools(self):
        """Return available tools."""
        tools = [
            {
                "name": "get_page",
                "ref": self.get_page,
                "description": "Retrieve a specific page by ID or title",
                "args_schema": create_model(
                    "GetPageModel",
                    page_id=(str, Field(description="Page ID or title"))
                )
            }
        ]
        
        # Add vector store tools
        tools.extend(self._get_vector_search_tools())
        tools.append(self.get_index_data_tool())
        
        return tools
    
    def get_page(self, page_id: str) -> dict:
        """Get page content."""
        return self._make_request('GET', f'/api/content/{page_id}', params={
            'expand': 'body.storage,version'
        })
```

## Testing Vector Store Integration (MANDATORY CLI TESTING)

### **REQUIRED: CLI Testing**

**ALL vector store integrations MUST be tested with alita-cli:**

```bash
# 1. Check toolkit schema with vector store tools
alita-cli toolkit schema mytoolkit

# 2. Create config with vector store settings
cat > mytoolkit-config.json <<EOF
{
  "base_url": "https://api.example.com",
  "api_token": "test-token",
  "pgvector_configuration": {
    "connection_string": "postgresql://localhost/vectordb"
  },
  "embedding_model": "text-embedding-3-small",
  "collection_name": "test_collection",
  "selected_tools": ["index_data", "search_index"]
}
EOF

# 3. List vector store tools
alita-cli toolkit tools mytoolkit --config mytoolkit-config.json

# 4. Test indexing
alita-cli toolkit test mytoolkit \
    --tool index_data \
    --config mytoolkit-config.json \
    --param index_name="test01" \
    --param project_key="PROJ" \
    --param clean_index=true

# 5. Test search
alita-cli toolkit test mytoolkit \
    --tool search_index \
    --config mytoolkit-config.json \
    --param query="find documentation about API" \
    --param index_name="test01" \
    --param search_top=5

# 6. Verify JSON output
alita-cli --output json toolkit test mytoolkit \
    --tool search_index \
    --config mytoolkit-config.json \
    --param query="test query" \
    --param index_name="test01" \
    | jq '.result'
```

### Additional Testing (Optional)

#### Test Script

```python
# test_vectorstore.py
from alita_sdk.tools.mytoolkit import MyToolkit

# Setup
toolkit = MyToolkit.get_toolkit(
    base_url="https://api.example.com",
    api_token="test-token",
    pgvector_configuration={
        'connection_string': 'postgresql://...',
    },
    embedding_model='text-embedding-3-small',
    collection_name='test_collection',
    selected_tools=['index_data', 'search_index']
)

tools = toolkit.get_tools()

# Find tools
index_tool = next(t for t in tools if 'index_data' in t.name)
search_tool = next(t for t in tools if 'search_index' in t.name)

# Test indexing
result = index_tool.invoke({
    'index_name': 'test01',
    'project_key': 'PROJ',
    'clean_index': True
})
print(f"Indexing result: {result}")

# Test search
results = search_tool.invoke({
    'query': 'find documentation about API',
    'index_name': 'test01',
    'search_top': 5
})
print(f"Search results: {results}")
```

## Checklist for Vector Store Integration

### Development
- [ ] Extends `BaseVectorStoreToolApiWrapper` or `BaseCodeToolApiWrapper`
- [ ] `doctype` field set appropriately ('document' or 'code')
- [ ] `_base_loader()` implemented and yields Documents
- [ ] Documents have required metadata: `id`, `created_on`
- [ ] `_process_document()` implemented if metadata enrichment needed
- [ ] Progress reporting in loader and processor
- [ ] `_index_tool_params()` defined if custom parameters needed
- [ ] Vector store tools added via `_get_vector_search_tools()`
- [ ] Index data tool added via `get_index_data_tool()`
- [ ] Pagination implemented in loader
- [ ] Error handling for API failures
- [ ] Chunking strategy appropriate for content type

### Testing (MANDATORY)
- [ ] **✅ REQUIRED: index_data tested with alita-cli**
- [ ] **✅ REQUIRED: search_index tested with alita-cli**
- [ ] **✅ REQUIRED: Vector store tools listed with alita-cli**
- [ ] **✅ REQUIRED: Indexing completes successfully**
- [ ] **✅ REQUIRED: Search returns relevant results**
- [ ] **✅ REQUIRED: Filters work correctly**
- [ ] **✅ REQUIRED: JSON output validated**
- [ ] (Optional) Tested with small dataset first
- [ ] (Optional) Verified embeddings are created correctly

## Common Pitfalls

1. **Processing too much in _base_loader**: Keep it minimal, use `_process_document` for heavy processing
2. **Missing progress reporting**: Users need feedback during indexing
3. **Not handling pagination**: Can miss documents
4. **Wrong doctype**: Affects how documents are processed and searched
5. **Missing required metadata**: `id` and `created_on` are required
6. **No error handling in processing**: One bad document shouldn't break everything
7. **Inefficient API calls**: Batch when possible
8. **Not testing with large datasets**: Performance issues only show up at scale
9. **Forgetting to call parent methods**: `super().__init__()` is important
10. **Not registering tools properly**: Vector store tools must be added to `get_available_tools()`

## Next Steps

After implementing vector store integration:
1. Test indexing with small dataset
2. Verify search quality
3. Tune chunking strategy if needed
4. Add custom search filters
5. Implement advanced features (reranking, etc.)
6. Document usage examples
