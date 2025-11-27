---
agent: 'sdk-dev'
description: 'Generate a complete Alita SDK toolkit from scratch with API wrapper, configuration, and registration'
---

# Generate Complete Alita SDK Toolkit

Create a complete, production-ready toolkit for the Alita SDK with the following specifications:

## Requirements

1. **Project Structure**: Create toolkit directory with proper files
2. **API Wrapper**: Implement API wrapper with tool methods
3. **Configuration**: Define configuration schema with UI metadata
4. **Tool Schemas**: Create Pydantic models for tool arguments
5. **Registration**: Register toolkit in SDK
6. **Error Handling**: Comprehensive error handling
7. **Testing Support**: Compatible with Streamlit testing

## Project Structure

```
alita_sdk/tools/{toolkit_name}/
├── __init__.py              # Toolkit class and configuration
└── api_wrapper.py           # API wrapper with tool methods
```

Optional configuration:
```
alita_sdk/configurations/{toolkit_name}.py
```

## Step-by-Step Implementation

### Step 1: Create API Wrapper (`api_wrapper.py`)

```python
from typing import Optional, List, Dict, Any, Generator
from pydantic import Field, SecretStr, create_model
from langchain_core.documents import Document
from langchain_core.tools import ToolException
import requests

# Choose base class based on requirements:
# - BaseToolApiWrapper for basic toolkit
# - BaseVectorStoreToolApiWrapper for indexing support
# - BaseCodeToolApiWrapper for code repositories
from ..elitea_base import BaseToolApiWrapper

class MyToolkitApiWrapper(BaseToolApiWrapper):
    """API wrapper for MyToolkit integration."""
    
    # Configuration fields
    base_url: str = Field(description="Base URL of the service")
    api_key: SecretStr = Field(description="API key for authentication")
    timeout: int = Field(default=30, description="Request timeout in seconds")
    verify_ssl: bool = Field(default=True, description="Verify SSL certificates")
    
    # Optional: HTTP session
    _session: Optional[requests.Session] = None
    
    def __init__(self, **kwargs):
        """Initialize API wrapper and setup HTTP session."""
        super().__init__(**kwargs)
        self._session = requests.Session()
        self._session.headers.update({
            'Authorization': f'Bearer {self.api_key.get_secret_value()}',
            'Content-Type': 'application/json'
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
            return response.json()
        except requests.exceptions.HTTPError as e:
            raise ToolException(f"HTTP error: {e.response.status_code}")
        except requests.exceptions.Timeout:
            raise ToolException(f"Request timed out after {self.timeout}s")
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
            limit: Maximum number of items to return
            
        Returns:
            Dictionary containing list of items and metadata
        """
        self._log_tool_event(f"Fetching items for project {project_id}", "get_items")
        
        result = self._make_request('GET', '/api/items', params={
            'project_id': project_id,
            'limit': limit
        })
        
        items = result.get('items', [])
        self._log_tool_event(f"Retrieved {len(items)} items", "get_items")
        
        return {
            'items': items,
            'count': len(items),
            'project_id': project_id
        }
    
    def create_item(
        self,
        project_id: str,
        title: str,
        description: Optional[str] = None
    ) -> dict:
        """
        Create a new item in a project.
        
        Args:
            project_id: The unique identifier of the project
            title: Title of the item
            description: Optional description of the item
            
        Returns:
            Dictionary containing the created item details
        """
        self._log_tool_event(f"Creating item in project {project_id}", "create_item")
        
        data = {'project_id': project_id, 'title': title}
        if description:
            data['description'] = description
        
        result = self._make_request('POST', '/api/items', json=data)
        
        self._log_tool_event(f"Item created with ID: {result.get('id')}", "create_item")
        return result
    
    def get_available_tools(self):
        """Return list of available tools with schemas."""
        return [
            {
                "name": "get_items",
                "ref": self.get_items,
                "description": self.get_items.__doc__,
                "args_schema": create_model(
                    "GetItemsModel",
                    project_id=(str, Field(
                        description="Project identifier",
                        min_length=1
                    )),
                    limit=(int, Field(
                        description="Maximum items to return",
                        default=10,
                        ge=1,
                        le=100
                    ))
                )
            },
            {
                "name": "create_item",
                "ref": self.create_item,
                "description": self.create_item.__doc__,
                "args_schema": create_model(
                    "CreateItemModel",
                    project_id=(str, Field(
                        description="Project identifier",
                        min_length=1
                    )),
                    title=(str, Field(
                        description="Item title",
                        min_length=1,
                        max_length=200
                    )),
                    description=(Optional[str], Field(
                        description="Optional item description",
                        default=None
                    ))
                )
            }
        ]
```

### Step 2: Create Toolkit Class (`__init__.py`)

```python
from typing import List, Optional, Literal
from langchain_core.tools import BaseTool, BaseToolkit
from pydantic import create_model, BaseModel, ConfigDict, Field
import requests

from .api_wrapper import MyToolkitApiWrapper
from ..base.tool import BaseAction
from ..elitea_base import filter_missconfigured_index_tools
from ..utils import clean_string, TOOLKIT_SPLITTER, get_max_toolkit_length, check_connection_response
from ...configurations.mytoolkit import MyToolkitConfiguration

name = "mytoolkit"

def get_tools(tool):
    """Entry point for loading toolkit tools."""
    return MyToolkit().get_toolkit(
        selected_tools=tool['settings'].get('selected_tools', []),
        mytoolkit_configuration=tool['settings']['mytoolkit_configuration'],
        toolkit_name=tool.get('toolkit_name'),
        # Add other configuration fields
    ).get_tools()

class MyToolkit(BaseToolkit):
    """Toolkit for MyService integration."""
    
    tools: List[BaseTool] = []
    toolkit_max_length: int = 0
    
    @staticmethod
    def toolkit_config_schema() -> BaseModel:
        """Generate configuration schema for UI."""
        # Get available tools
        selected_tools = {
            x['name']: x['args_schema'].schema()
            for x in MyToolkitApiWrapper.model_construct().get_available_tools()
        }
        
        # Calculate max toolkit name length
        MyToolkit.toolkit_max_length = get_max_toolkit_length(selected_tools)
        
        # Create configuration model
        model = create_model(
            name,
            base_url=(str, Field(description="Base URL of the service")),
            timeout=(int, Field(
                description="Request timeout in seconds",
                default=30,
                ge=1,
                le=300
            )),
            verify_ssl=(bool, Field(
                description="Verify SSL certificates",
                default=True
            )),
            mytoolkit_configuration=(MyToolkitConfiguration, Field(
                description="MyToolkit Configuration",
                json_schema_extra={'configuration_types': ['mytoolkit']}
            )),
            selected_tools=(List[Literal[tuple(selected_tools)]], Field(
                default=[],
                json_schema_extra={'args_schemas': selected_tools}
            )),
            __config__=ConfigDict(json_schema_extra={
                'metadata': {
                    "label": "My Toolkit",
                    "icon_url": "mytoolkit-icon.svg",
                    "max_length": MyToolkit.toolkit_max_length,
                    "categories": ["integration"],
                    "extra_categories": ["api", "rest"],
                }
            })
        )
        
        # Optional: Add connection testing
        @check_connection_response
        def check_connection(self):
            response = requests.get(
                f'{self.base_url}/api/health',
                timeout=5,
                verify=self.verify_ssl
            )
            return response
        
        model.check_connection = check_connection
        return model
    
    @classmethod
    @filter_missconfigured_index_tools
    def get_toolkit(
        cls,
        selected_tools: Optional[List[str]] = None,
        toolkit_name: Optional[str] = None,
        **kwargs
    ):
        """Instantiate toolkit with selected tools."""
        if selected_tools is None:
            selected_tools = []
        
        # Merge configuration
        wrapper_payload = {
            **kwargs,
            **kwargs.get('mytoolkit_configuration', {}),
        }
        
        # Create API wrapper
        api_wrapper = MyToolkitApiWrapper(**wrapper_payload)
        
        # Generate tool name prefix
        prefix = (
            clean_string(toolkit_name, cls.toolkit_max_length) + TOOLKIT_SPLITTER
            if toolkit_name else ''
        )
        
        # Create tools
        available_tools = api_wrapper.get_available_tools()
        tools = []
        
        for tool in available_tools:
            if selected_tools and tool["name"] not in selected_tools:
                continue
            
            tools.append(BaseAction(
                api_wrapper=api_wrapper,
                name=prefix + tool["name"],
                description=tool["description"],
                args_schema=tool["args_schema"]
            ))
        
        return cls(tools=tools)
    
    def get_tools(self):
        """Return list of tool instances."""
        return self.tools
```

### Step 3: Create Configuration Class (Optional)

`alita_sdk/configurations/mytoolkit.py`:

```python
from pydantic import BaseModel, Field, SecretStr

class MyToolkitConfiguration(BaseModel):
    """Configuration for MyToolkit authentication."""
    
    api_key: SecretStr = Field(description="API Key for authentication")
    username: Optional[str] = Field(description="Username", default=None)
    
    class Config:
        json_schema_extra = {
            "required": ["api_key"]
        }
```

### Step 4: Register Toolkit

In `alita_sdk/tools/__init__.py`:

```python
# Add safe import
_safe_import_tool('mytoolkit', 'mytoolkit', 'get_tools', 'MyToolkit')

# In get_tools() function, add handler:
if tool_type == 'mytoolkit' and 'mytoolkit' in AVAILABLE_TOOLS:
    try:
        tools.extend(AVAILABLE_TOOLS['mytoolkit']['get_tools'](tool))
    except Exception as e:
        logger.error(f"Error getting tools for mytoolkit: {e}")
        raise ToolException(f"Error getting tools for mytoolkit: {e}")
    continue
```

In `alita_sdk/configurations/__init__.py` (if config class created):

```python
_safe_import_configuration('mytoolkit', 'mytoolkit', 'MyToolkitConfiguration')
```

## Testing (MANDATORY CLI TESTING)

### **REQUIRED: CLI Testing**

**ALL toolkits MUST be tested with alita-cli before being considered complete.**

```bash
# 1. Check toolkit schema
alita-cli toolkit schema mytoolkit

# 2. Create test configuration
cat > mytoolkit-config.json <<EOF
{
  "base_url": "https://api.example.com",
  "mytoolkit_configuration": {
    "api_key": "test-key"
  },
  "selected_tools": ["get_items", "create_item"]
}
EOF

# 3. List available tools
alita-cli toolkit tools mytoolkit --config mytoolkit-config.json

# 4. Test each tool
alita-cli toolkit test mytoolkit \
    --tool get_items \
    --config mytoolkit-config.json \
    --param project_id="test-project" \
    --param limit=10

alita-cli toolkit test mytoolkit \
    --tool create_item \
    --config mytoolkit-config.json \
    --param project_id="test-project" \
    --param title="Test Item"

# 5. Verify JSON output
alita-cli --output json toolkit test mytoolkit \
    --tool get_items \
    --config mytoolkit-config.json \
    --param project_id="test-project" \
    | jq '.success, .result'

# 6. Test error scenarios
alita-cli toolkit test mytoolkit \
    --tool get_items \
    --config mytoolkit-config.json \
    --param project_id="" \
    # Should fail with validation error
```

**Toolkit is NOT complete until:**
- ✅ All tools pass CLI testing
- ✅ JSON output is valid and parseable
- ✅ Error scenarios are handled properly
- ✅ Tool schemas validated with `alita-cli toolkit schema`

### Additional Testing (Optional)

#### Streamlit Testing (for interactive validation)

1. Run: `streamlit run alita_local.py`
2. Login with credentials
3. Go to "Toolkit testing" tab
4. Select your toolkit
5. Configure parameters
6. Test tools in "function mode"

#### Unit Testing (recommended)

```python
from alita_sdk.tools.mytoolkit import MyToolkit

def test_toolkit_creation():
    toolkit = MyToolkit.get_toolkit(
        base_url="https://api.example.com",
        mytoolkit_configuration={
            'api_key': 'test-key'
        },
        selected_tools=['get_items']
    )
    
    tools = toolkit.get_tools()
    assert len(tools) == 1
    assert 'get_items' in tools[0].name

def test_tool_execution():
    # Mock API responses
    # Test tool invocation
    pass
```

## Best Practices Checklist

### Development
- [ ] Inherits from appropriate base class
- [ ] All configuration fields defined with proper types
- [ ] SecretStr used for sensitive data
- [ ] HTTP session initialized in `__init__`
- [ ] Generic `_make_request()` method
- [ ] Comprehensive error handling
- [ ] Clear, LLM-friendly docstrings
- [ ] Progress reporting with `_log_tool_event()`
- [ ] Tool schemas with Pydantic Field validation
- [ ] Optional parameters have defaults
- [ ] Configuration schema with UI metadata
- [ ] Toolkit registered in `tools/__init__.py`
- [ ] Configuration registered in `configurations/__init__.py`
- [ ] Follows SDK naming conventions

### Testing (MANDATORY)
- [ ] **✅ REQUIRED: Tested with alita-cli toolkit test**
- [ ] **✅ REQUIRED: All tools pass CLI testing**
- [ ] **✅ REQUIRED: JSON output validated**
- [ ] **✅ REQUIRED: Error scenarios tested with CLI**
- [ ] **✅ REQUIRED: Tool schemas verified with alita-cli**
- [ ] (Optional) Tested with Streamlit
- [ ] (Optional) Unit tests written

## Common Toolkit Types

### Basic API Integration
- No vector store needed
- Extend `BaseToolApiWrapper`
- Simple CRUD operations
- Examples: Weather, Currency, Public APIs

### Document Management
- Vector store for indexing
- Extend `BaseVectorStoreToolApiWrapper`
- Implement `_base_loader()` and `_process_document()`
- Examples: Confluence, SharePoint, Wiki

### Issue Tracking
- Optional vector store
- Extend `BaseVectorStoreToolApiWrapper`
- Complex schemas for tickets/issues
- Examples: Jira, ADO, GitHub Issues

### Code Repositories
- Code-specific vector store
- Extend `BaseCodeToolApiWrapper`
- Implement `loader()` with file filtering
- Examples: GitHub, GitLab, Bitbucket

## Additional Features to Consider

- **Pagination**: For large result sets
- **Rate Limiting**: Respect API limits
- **Caching**: For frequently accessed data
- **Webhooks**: Real-time updates
- **Batch Operations**: Process multiple items
- **File Upload**: Handle file attachments
- **Search Filters**: Advanced querying
- **Metadata Extraction**: Rich document metadata

Generate a complete, production-ready Alita SDK toolkit with proper structure, error handling, configuration, and testing support.
