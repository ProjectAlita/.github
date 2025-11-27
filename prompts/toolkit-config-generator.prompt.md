---
agent: 'sdk-dev'
description: 'Generate complete toolkit configuration schema with UI metadata and validation'
---

# Generate Alita SDK Toolkit Configuration

Create a complete toolkit configuration schema for the Alita SDK with the following specifications:

## Requirements

1. **Configuration Schema**: Create `toolkit_config_schema()` static method
2. **UI Metadata**: Include label, icon, categories for UI rendering
3. **Field Validation**: Proper Pydantic Field definitions with constraints
4. **Tool Selection**: Dynamic selected_tools field with available tool schemas
5. **Connection Testing**: Optional `check_connection()` method for validation

## Implementation Details

### Basic Structure

```python
from typing import List, Literal, Optional
from pydantic import create_model, BaseModel, ConfigDict, Field
from ..utils import get_max_toolkit_length, check_connection_response
import requests

name = "mytoolkit"

@staticmethod
def toolkit_config_schema() -> BaseModel:
    # Get available tools from API wrapper
    selected_tools = {
        x['name']: x['args_schema'].schema() 
        for x in MyToolkitApiWrapper.model_construct().get_available_tools()
    }
    
    # Calculate max toolkit name length
    MyToolkit.toolkit_max_length = get_max_toolkit_length(selected_tools)
    
    # Create configuration model
    model = create_model(
        name,
        # Required fields
        base_url=(str, Field(description="Base URL of the service")),
        
        # Optional fields with defaults
        timeout=(int, Field(description="Request timeout in seconds", default=30, ge=1, le=300)),
        verify_ssl=(bool, Field(description="Verify SSL certificates", default=True)),
        
        # Configuration objects (reference to configuration classes)
        mytoolkit_configuration=(MyToolkitConfiguration, Field(
            description="Authentication Configuration",
            json_schema_extra={'configuration_types': ['mytoolkit']}
        )),
        
        # Vector store configuration (optional)
        pgvector_configuration=(Optional[PgVectorConfiguration], Field(
            default=None,
            description="PgVector Configuration for indexing",
            json_schema_extra={'configuration_types': ['pgvector']}
        )),
        
        # Embedding model (optional)
        embedding_model=(Optional[str], Field(
            default=None,
            description="Embedding configuration",
            json_schema_extra={'configuration_model': 'embedding'}
        )),
        
        # Selected tools with schemas
        selected_tools=(List[Literal[tuple(selected_tools)]], Field(
            default=[],
            json_schema_extra={'args_schemas': selected_tools}
        )),
        
        # UI metadata
        __config__=ConfigDict(json_schema_extra={
            'metadata': {
                "label": "My Toolkit",
                "icon_url": "mytoolkit-icon.svg",
                "max_length": MyToolkit.toolkit_max_length,
                "categories": ["integration", "project management"],
                "extra_categories": ["api", "rest", "external service"],
            }
        })
    )
    
    return model
```

### Connection Testing (Optional)

```python
@check_connection_response
def check_connection(self):
    """Test connection to the service."""
    try:
        response = requests.get(
            f'{self.base_url}/api/health',
            headers={'Authorization': f'Bearer {self.api_key}'},
            timeout=5,
            verify=self.verify_ssl
        )
        return response
    except Exception as e:
        raise ConnectionError(f"Failed to connect: {str(e)}")

# Attach to model
model.check_connection = check_connection
```

## Field Types and Validation

### Basic Fields
```python
# String with constraints
name=(str, Field(description="Name", min_length=1, max_length=100))

# Integer with range
port=(int, Field(description="Port number", ge=1, le=65535, default=443))

# Boolean flag
enabled=(bool, Field(description="Enable feature", default=True))

# Email validation
email=(str, Field(description="Email address", pattern=r'^[\w\.-]+@[\w\.-]+\.\w+$'))

# URL validation
url=(str, Field(description="Service URL", pattern=r'^https?://'))
```

### Optional Fields
```python
# Optional string
description=(Optional[str], Field(description="Optional description", default=None))

# Optional list
tags=(Optional[List[str]], Field(description="Tags", default=None))

# Optional with example
api_version=(Optional[str], Field(
    description="API version",
    default="v1",
    examples=["v1", "v2", "latest"]
))
```

### Configuration References
```python
# Reference to configuration class
auth_config=(MyAuthConfiguration, Field(
    description="Authentication settings",
    json_schema_extra={'configuration_types': ['myauth']}
))

# Multiple configuration types
connection_config=(ConnectionConfiguration, Field(
    description="Connection settings",
    json_schema_extra={'configuration_types': ['oauth', 'apikey', 'basic']}
))
```

### Complex Fields
```python
# List with constraints
items=(List[str], Field(
    description="Items",
    min_items=1,
    max_items=10,
    default=[]
))

# Dictionary field
headers=(dict, Field(
    description="Custom HTTP headers",
    default={},
    examples=[{"X-Custom-Header": "value"}]
))

# Enum-like using Literal
mode=(Literal['cloud', 'server'], Field(
    description="Deployment mode",
    default='cloud'
))
```

## UI Metadata Guidelines

### Required Metadata
```python
'metadata': {
    "label": "Display Name",           # Shown in UI
    "icon_url": "icon.svg",            # Icon file name
    "max_length": 50,                  # Max toolkit name length
    "categories": ["primary"],         # Primary categories
    "extra_categories": ["tags"]       # Additional tags
}
```

### Category Options
- **Primary Categories**: integration, project management, testing, documentation, communication, analytics, cloud, database
- **Extra Categories**: Any relevant tags (api, rest, graphql, oauth, etc.)

## Configuration Class Pattern

Create separate configuration class in `alita_sdk/configurations/{name}.py`:

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

Register in `configurations/__init__.py`:
```python
_safe_import_configuration('mytoolkit', 'mytoolkit', 'MyToolkitConfiguration')
```

## Examples from Existing Toolkits

### Jira Toolkit
```python
jira_configuration=(JiraConfiguration, Field(
    description="Jira Configuration",
    json_schema_extra={'configuration_types': ['jira']}
)),
cloud=(bool, Field(description="Hosting Option", json_schema_extra={'configuration': True})),
limit=(int, Field(description="Limit issues", gt=0, default=5)),
labels=(Optional[str], Field(
    description="Comma separated labels",
    default=None,
    examples=["alita,elitea;another-label"]
))
```

### TestRail Toolkit
```python
testrail_configuration=(Optional[TestRailConfiguration], Field(
    description="TestRail Configuration",
    json_schema_extra={'configuration_types': ['testrail']}
)),
pgvector_configuration=(Optional[PgVectorConfiguration], Field(
    default=None,
    description="PgVector Configuration",
    json_schema_extra={'configuration_types': ['pgvector']}
)),
embedding_model=(Optional[str], Field(
    default=None,
    description="Embedding configuration",
    json_schema_extra={'configuration_model': 'embedding'}
))
```

## Testing Configuration (MANDATORY CLI TESTING)

### **REQUIRED: CLI Testing**

**ALL configurations MUST be tested with alita-cli:**

```bash
# 1. Validate schema structure
alita-cli toolkit schema mytoolkit

# 2. Test with valid configuration
cat > mytoolkit-config.json <<EOF
{
  "base_url": "https://api.example.com",
  "api_key": "test_key",
  "timeout": 30,
  "verify_ssl": true,
  "selected_tools": ["get_items"]
}
EOF

alita-cli toolkit tools mytoolkit --config mytoolkit-config.json

# 3. Test connection (if implemented)
alita-cli toolkit test mytoolkit \
    --tool get_items \
    --config mytoolkit-config.json \
    --param test="value"

# 4. Verify JSON schema output
alita-cli --output json toolkit schema mytoolkit | jq '.properties'
```

### Additional Testing (Optional)

1. **Validate Schema**: Ensure model can be instantiated
   ```python
   config = MyToolkit.toolkit_config_schema()
   assert config.__name__ == name
   ```

2. **Test Connection**: If `check_connection()` is implemented
   ```python
   config_instance = config(
       base_url="https://api.example.com",
       api_key="test-key"
   )
   response = config_instance.check_connection()
   assert response.status_code == 200
   ```

3. **UI Rendering**: Verify metadata is present
   ```python
   schema = config.model_json_schema()
   assert 'metadata' in schema.get('json_schema_extra', {})
   ```

## Best Practices

- **Use SecretStr**: For API keys, passwords, tokens
- **Provide Defaults**: For optional configuration fields
- **Clear Descriptions**: Help users understand each field
- **Examples**: Show valid values for complex fields
- **Validation**: Use Field constraints to catch errors early
- **UI Metadata**: Complete metadata for good UX
- **Connection Test**: Validate configuration before use
- **Configuration Classes**: Separate auth/connection configs
- **Type Hints**: Full type annotations everywhere
- **Documentation**: Docstrings for configuration classes

## Common Patterns

### API Key Authentication
```python
api_key=(str, Field(description="API Key for authentication"))
# or with SecretStr in configuration class
```

### OAuth Configuration
```python
oauth_configuration=(OAuthConfiguration, Field(
    description="OAuth 2.0 Configuration",
    json_schema_extra={'configuration_types': ['oauth']}
))
```

### Multi-Option Authentication
```python
auth_type=(Literal['apikey', 'oauth', 'basic'], Field(
    description="Authentication type",
    default='apikey'
)),
auth_configuration=(Union[ApiKeyConfig, OAuthConfig, BasicAuthConfig], Field(
    description="Authentication configuration"
))
```

### Vector Store Support
```python
# Always include both for indexing support
pgvector_configuration=(Optional[PgVectorConfiguration], ...),
embedding_model=(Optional[str], ...)
```

## Checklist

### Development
- [ ] Configuration schema created with `toolkit_config_schema()`
- [ ] All required fields defined with descriptions
- [ ] Optional fields have sensible defaults
- [ ] SecretStr used for sensitive data
- [ ] Field validation with constraints where needed
- [ ] UI metadata complete (label, icon, categories)
- [ ] Selected tools field with tool schemas
- [ ] Configuration class created if needed
- [ ] Configuration registered in `configurations/__init__.py`
- [ ] Connection test implemented (optional)
- [ ] Follows SDK naming conventions

### Testing (MANDATORY)
- [ ] **✅ REQUIRED: Schema validated with alita-cli toolkit schema**
- [ ] **✅ REQUIRED: Config file tested with alita-cli**
- [ ] **✅ REQUIRED: Tools list verified with alita-cli**
- [ ] **✅ REQUIRED: JSON schema output validated**

Generate a complete, production-ready toolkit configuration schema with proper validation, UI metadata, and comprehensive field definitions.
