---
agent: 'sdk-dev'
description: 'Specialized skill for building API wrapper classes in Alita SDK toolkits'
---

# API Wrapper Builder Skill

## Purpose

This skill focuses on building robust API wrapper classes (`api_wrapper.py`) for Alita SDK toolkits. It guides you through creating the wrapper that connects to external services and implements tool methods.

## When to Use This Skill

- Starting a new toolkit and need to create the API wrapper
- Adding new tool methods to an existing wrapper
- Implementing authentication for API calls
- Setting up HTTP client configuration
- Handling API responses and errors
- Implementing pagination or rate limiting

## Key Concepts

### Base Class Selection

Choose the appropriate base class based on toolkit requirements:

1. **BaseToolApiWrapper**: Basic toolkit without vector store functionality
   - Simple API integrations (weather, currency, etc.)
   - Tools that don't need document indexing

2. **BaseVectorStoreToolApiWrapper**: Toolkit with document indexing
   - Document management systems (Confluence, SharePoint)
   - Issue trackers with indexing (Jira, ADO)
   - Any system where you need to index and search content

3. **BaseCodeToolApiWrapper**: Code repository toolkit
   - Source code management (GitHub, GitLab, Bitbucket)
   - Code analysis tools
   - Systems that index and search code files

## API Wrapper Structure

### Basic Pattern

```python
from pydantic import Field, SecretStr, create_model
from typing import Optional, List, Dict, Any
from ..elitea_base import BaseToolApiWrapper
import requests
from langchain_core.tools import ToolException

class MyServiceApiWrapper(BaseToolApiWrapper):
    """API wrapper for MyService integration."""
    
    # Configuration fields (must match configuration schema)
    base_url: str = Field(description="Base URL of the service")
    api_key: SecretStr = Field(description="API key for authentication")
    timeout: int = Field(default=30, description="Request timeout in seconds")
    verify_ssl: bool = Field(default=True, description="Verify SSL certificates")
    
    # Optional: Store session or client
    _session: Optional[requests.Session] = None
    
    def __init__(self, **kwargs):
        """Initialize the API wrapper."""
        super().__init__(**kwargs)
        # Setup HTTP session
        self._session = requests.Session()
        self._session.headers.update({
            'Authorization': f'Bearer {self.api_key.get_secret_value()}',
            'Content-Type': 'application/json',
            'User-Agent': 'Alita-SDK'
        })
    
    def _make_request(self, method: str, endpoint: str, **kwargs) -> Dict[str, Any]:
        """
        Make HTTP request to the API.
        
        Args:
            method: HTTP method (GET, POST, PUT, DELETE)
            endpoint: API endpoint path
            **kwargs: Additional arguments for requests
            
        Returns:
            Response data as dictionary
            
        Raises:
            ToolException: If request fails
        """
        url = f"{self.base_url.rstrip('/')}/{endpoint.lstrip('/')}"
        
        try:
            response = self._session.request(
                method=method,
                url=url,
                timeout=self.timeout,
                verify=self.verify_ssl,
                **kwargs
            )
            response.raise_for_status()
            
            # Handle different content types
            content_type = response.headers.get('Content-Type', '')
            if 'application/json' in content_type:
                return response.json()
            return {'content': response.text}
            
        except requests.exceptions.HTTPError as e:
            status_code = e.response.status_code
            error_message = f"HTTP {status_code} error"
            
            # Try to extract error details from response
            try:
                error_data = e.response.json()
                error_message = error_data.get('message', error_message)
            except:
                error_message = e.response.text or error_message
            
            raise ToolException(f"API request failed: {error_message}")
            
        except requests.exceptions.Timeout:
            raise ToolException(f"Request timed out after {self.timeout} seconds")
            
        except requests.exceptions.ConnectionError:
            raise ToolException(f"Failed to connect to {self.base_url}")
            
        except Exception as e:
            raise ToolException(f"Unexpected error: {str(e)}")
    
    # Tool methods
    def get_items(self, project_id: str, limit: int = 10) -> dict:
        """
        Retrieve items from a project.
        
        Args:
            project_id: The unique identifier of the project
            limit: Maximum number of items to return (default: 10)
            
        Returns:
            Dictionary containing list of items and metadata
        """
        self._log_tool_event(f"Fetching items for project {project_id}", "get_items")
        
        params = {
            'project_id': project_id,
            'limit': limit
        }
        
        result = self._make_request('GET', '/api/items', params=params)
        
        items = result.get('items', [])
        self._log_tool_event(f"Retrieved {len(items)} items", "get_items")
        
        return {
            'items': items,
            'count': len(items),
            'project_id': project_id
        }
    
    def create_item(self, project_id: str, title: str, description: Optional[str] = None) -> dict:
        """
        Create a new item in a project.
        
        Args:
            project_id: The unique identifier of the project
            title: Title of the item (required)
            description: Optional description of the item
            
        Returns:
            Dictionary containing the created item details
        """
        self._log_tool_event(f"Creating item in project {project_id}", "create_item")
        
        data = {
            'project_id': project_id,
            'title': title
        }
        
        if description:
            data['description'] = description
        
        result = self._make_request('POST', '/api/items', json=data)
        
        self._log_tool_event(f"Item created with ID: {result.get('id')}", "create_item")
        
        return result
    
    def get_available_tools(self):
        """Return list of available tools with their schemas."""
        return [
            {
                "name": "get_items",
                "ref": self.get_items,
                "description": self.get_items.__doc__,
                "args_schema": create_model(
                    "GetItemsModel",
                    project_id=(str, Field(description="Project identifier", min_length=1)),
                    limit=(int, Field(description="Maximum number of items to return", default=10, ge=1, le=100))
                )
            },
            {
                "name": "create_item",
                "ref": self.create_item,
                "description": self.create_item.__doc__,
                "args_schema": create_model(
                    "CreateItemModel",
                    project_id=(str, Field(description="Project identifier", min_length=1)),
                    title=(str, Field(description="Item title", min_length=1, max_length=200)),
                    description=(Optional[str], Field(description="Optional item description", default=None))
                )
            }
        ]
```

## Implementation Patterns

### 1. Authentication

#### API Key in Header
```python
self._session.headers.update({
    'Authorization': f'Bearer {self.api_key.get_secret_value()}'
})
```

#### Basic Auth
```python
from requests.auth import HTTPBasicAuth

def _make_request(self, method: str, endpoint: str, **kwargs):
    auth = HTTPBasicAuth(self.username, self.password.get_secret_value())
    response = self._session.request(method, url, auth=auth, **kwargs)
```

#### OAuth Token
```python
self._session.headers.update({
    'Authorization': f'Bearer {self.oauth_token.get_secret_value()}'
})
```

#### Custom Headers
```python
if self.custom_headers:
    self._session.headers.update(self.custom_headers)
```

### 2. Pagination

#### Offset-based
```python
def get_all_items(self, project_id: str) -> list:
    """Retrieve all items using pagination."""
    all_items = []
    offset = 0
    limit = 100
    
    while True:
        result = self._make_request('GET', '/api/items', params={
            'project_id': project_id,
            'offset': offset,
            'limit': limit
        })
        
        items = result.get('items', [])
        if not items:
            break
            
        all_items.extend(items)
        offset += limit
        
        if len(items) < limit:
            break
    
    return all_items
```

#### Cursor-based
```python
def get_all_items(self, project_id: str) -> list:
    """Retrieve all items using cursor pagination."""
    all_items = []
    cursor = None
    
    while True:
        params = {'project_id': project_id}
        if cursor:
            params['cursor'] = cursor
            
        result = self._make_request('GET', '/api/items', params=params)
        
        items = result.get('items', [])
        all_items.extend(items)
        
        cursor = result.get('next_cursor')
        if not cursor:
            break
    
    return all_items
```

### 3. Rate Limiting

```python
import time
from datetime import datetime, timedelta

class MyServiceApiWrapper(BaseToolApiWrapper):
    _last_request_time: Optional[datetime] = None
    _min_request_interval: float = 0.1  # 100ms between requests
    
    def _make_request(self, method: str, endpoint: str, **kwargs):
        # Implement rate limiting
        if self._last_request_time:
            elapsed = (datetime.now() - self._last_request_time).total_seconds()
            if elapsed < self._min_request_interval:
                time.sleep(self._min_request_interval - elapsed)
        
        response = self._session.request(method, url, **kwargs)
        self._last_request_time = datetime.now()
        
        return response
```

### 4. Error Handling

```python
def safe_tool_method(self, param: str) -> dict:
    """Tool method with comprehensive error handling."""
    try:
        # Validate input
        if not param or not param.strip():
            raise ValueError("Parameter cannot be empty")
        
        # Make API call
        result = self._make_request('GET', f'/api/resource/{param}')
        
        # Validate response
        if 'data' not in result:
            raise ValueError("Invalid response format from API")
        
        return result
        
    except ToolException:
        # Re-raise ToolException (already formatted)
        raise
        
    except ValueError as e:
        # Handle validation errors
        raise ToolException(f"Validation error: {str(e)}")
        
    except Exception as e:
        # Catch-all for unexpected errors
        raise ToolException(f"Unexpected error in safe_tool_method: {str(e)}")
```

### 5. Progress Reporting

```python
def batch_operation(self, items: List[str]) -> dict:
    """Process multiple items with progress reporting."""
    results = []
    total = len(items)
    
    self._log_tool_event(f"Starting batch operation on {total} items", "batch_operation")
    
    for idx, item in enumerate(items, 1):
        result = self._process_item(item)
        results.append(result)
        
        # Report progress every 10% or at completion
        if idx % max(1, total // 10) == 0 or idx == total:
            self._log_tool_event(
                f"Processed {idx}/{total} items ({idx*100//total}%)",
                "batch_operation"
            )
    
    return {
        'processed': total,
        'results': results
    }
```

## Examples from Existing Toolkits

### Jira API Wrapper
Reference: `alita_sdk/tools/jira/api_wrapper.py`

Key patterns:
- Uses `atlassian-python-api` library
- Implements both cloud and server versions
- Custom headers support
- Label management for created entities

### Confluence API Wrapper
Reference: `alita_sdk/tools/confluence/api_wrapper.py`

Key patterns:
- Extends `BaseVectorStoreToolApiWrapper` for indexing
- Document processing with metadata extraction
- Space and page management
- CQL (Confluence Query Language) support

### GitHub API Wrapper
Reference: `alita_sdk/tools/github/api_wrapper.py`

Key patterns:
- Extends `BaseCodeToolApiWrapper` for code indexing
- Uses PyGitHub library
- Repository and file management
- Branch operations

## Checklist for API Wrapper

- [ ] Inherits from appropriate base class
- [ ] All configuration fields defined with proper types
- [ ] SecretStr used for sensitive data (API keys, passwords)
- [ ] HTTP session initialized in `__init__`
- [ ] Generic `_make_request()` method for API calls
- [ ] Comprehensive error handling in all methods
- [ ] Clear, LLM-friendly docstrings for all tool methods
- [ ] Progress reporting for long-running operations
- [ ] Pagination implemented where needed
- [ ] Rate limiting if required by API
- [ ] `get_available_tools()` returns complete tool definitions
- [ ] Tool schemas use Pydantic Field with validation
- [ ] Optional parameters have sensible defaults
- [ ] All tool methods return dictionaries or structured data
- [ ] No bare exceptions (all wrapped in ToolException)
- [ ] Connection cleanup (if using resources)
- [ ] **✅ MANDATORY: Tested with alita-cli toolkit test**
- [ ] **✅ MANDATORY: All tools verified with CLI**
- [ ] **✅ MANDATORY: JSON output validated**
- [ ] Tested with various error scenarios
- [ ] Follows SDK naming conventions (snake_case)

## Common Pitfalls to Avoid

1. **Not using SecretStr for sensitive data**: Always use `SecretStr` for API keys, passwords, tokens
2. **Missing error handling**: Every API call can fail - handle it gracefully
3. **Poor docstrings**: LLMs rely on docstrings to understand tools
4. **No progress reporting**: Long operations should report progress
5. **Hardcoded values**: Use configuration fields instead
6. **Ignoring rate limits**: Can get your API key banned
7. **Not validating responses**: API can return unexpected formats
8. **Leaking exceptions**: Wrap all exceptions in ToolException
9. **No SSL verification option**: Some environments need this
10. **Missing timeout**: Requests can hang indefinitely

## Testing Your API Wrapper (MANDATORY)

### Required: CLI Testing

**ALL toolkits MUST be tested with CLI before considering them complete.**

```bash
# 1. Check toolkit schema
alita-cli toolkit schema mytoolkit

# 2. Create test config
cat > mytoolkit-config.json <<EOF
{
  "api_key": "test_key",
  "base_url": "https://api.example.com"
}
EOF

# 3. List available tools
alita-cli toolkit tools mytoolkit --config mytoolkit-config.json

# 4. Test each tool
alita-cli toolkit test mytoolkit \
    --tool my_tool_method \
    --config mytoolkit-config.json \
    --param param1="test_value"

# 5. Verify JSON output for automation
alita-cli --output json toolkit test mytoolkit \
    --tool my_tool_method \
    --config mytoolkit-config.json \
    --param param1="test_value"
```

### Additional Testing (Recommended)

1. **Unit tests**: Test individual methods with mocked responses
2. **Integration tests**: Test against real API (with test credentials)
3. **Error scenarios**: Test with invalid credentials, bad inputs, timeouts
4. **Edge cases**: Empty responses, large datasets, special characters
5. **Streamlit testing** (optional): Use `alita_local.py` for interactive GUI testing

## Next Steps

After completing the API wrapper:
1. Create the toolkit class (`__init__.py`)
2. Design configuration schema
3. Register the toolkit
4. **✅ MANDATORY: Test with alita-cli (all tools must pass)**
5. Write documentation
6. (Optional) Test with Streamlit for interactive validation

**DO NOT consider toolkit complete until CLI testing passes for all tools.**
