---
agent: 'sdk-dev'
description: 'Test Alita SDK toolkits - CLI testing is MANDATORY, Streamlit optional for debugging'
---

# Test Alita SDK Toolkit

## IMPORTANT: CLI Testing is MANDATORY

**ALL toolkits MUST be tested with alita-cli before being considered complete. Streamlit testing is optional for interactive debugging.**

## Primary Method: CLI Testing (REQUIRED)

### Quick Start

```bash
# 1. Setup environment
cat > .env <<EOF
DEPLOYMENT_URL=https://api.elitea.ai
PROJECT_ID=123
API_KEY=your_api_key
EOF

# 2. Check toolkit schema
alita-cli toolkit schema mytoolkit

# 3. Create config
cat > mytoolkit-config.json <<EOF
{
  "base_url": "https://api.example.com",
  "api_key": "test_key",
  "selected_tools": ["get_items", "create_item"]
}
EOF

# 4. List tools
alita-cli toolkit tools mytoolkit --config mytoolkit-config.json

# 5. Test each tool
alita-cli toolkit test mytoolkit \
    --tool get_items \
    --config mytoolkit-config.json \
    --param project_id="PROJ-123" \
    --param limit=10

# 6. Verify JSON output
alita-cli --output json toolkit test mytoolkit \
    --tool get_items \
    --config mytoolkit-config.json \
    --param project_id="PROJ-123"
```

### Complete CLI Testing Workflow

See `@sdk-dev /cli-testing` for comprehensive CLI testing guide.

### CLI Testing Checklist (MANDATORY)

- [ ] **✅ REQUIRED: toolkit schema verified**
- [ ] **✅ REQUIRED: All tools pass CLI tests**
- [ ] **✅ REQUIRED: JSON output validated**
- [ ] **✅ REQUIRED: Error scenarios tested**
- [ ] **✅ REQUIRED: All parameters tested**
- [ ] **✅ REQUIRED: Success cases documented**
- [ ] **✅ REQUIRED: Failure cases documented**

**Toolkit is NOT complete until all CLI tests pass.**

---

## Secondary Method: Streamlit Testing (OPTIONAL)

Use Streamlit for interactive debugging and visual exploration. This is helpful for development but NOT a replacement for CLI testing.

Test and debug Alita SDK toolkits locally using the Streamlit testing interface.

## Requirements

1. **Streamlit Setup**: Run `alita_local.py` for interactive testing
2. **Environment**: `.env` file with platform credentials
3. **Breakpoints**: Set in API wrapper for debugging
4. **Test Cases**: Prepare test scenarios for each tool

## Testing Workflow

### Step 1: Setup Environment

Create `.env` file with credentials:
```env
DEPLOYMENT_URL=<your_deployment_url>
API_KEY=<your_api_key>
PROJECT_ID=<your_project_id>
```

### Step 2: Launch Streamlit

```bash
# From SDK root directory
streamlit run alita_local.py

# If pytorch issues occur:
streamlit run alita_local.py --server.fileWatcherType none
```

### Step 3: Login to Platform

1. Navigate to "Alita Settings" in sidebar
2. Credentials auto-populate from `.env`
3. Click "Login" to authenticate
4. Verify connection success

### Step 4: Select Testing Mode

Two modes available:

**Agent Testing**:
- Select existing agent
- Test complete agent workflow
- Observe toolkit integration in context

**Toolkit Testing** (Recommended for development):
- Select specific toolkit
- Test individual tools
- Direct tool invocation
- Better for debugging

### Step 5: Configure Toolkit

In Toolkit Testing tab:

1. **Select Toolkit**: Choose from dropdown
2. **Configure Parameters**:
   - Base URL
   - Authentication (API key, OAuth, etc.)
   - Optional settings
   - Vector store config (if needed)
3. **Select Tools**: Choose specific tools to test
4. **Save Configuration**

### Step 6: Test Tools

**Function Mode** (Recommended):
```
Test single tool with specific parameters
Clear input/output visibility
Best for debugging
```

**Example Test Queries**:
```
# For get_items tool
Get items from project "PROJ-123" with limit 5

# For create_item tool
Create item in project "PROJ-123" with title "Test Item" and description "Testing"

# For search tool
Search for "authentication" in index "docs01"
```

### Step 7: Debug with Breakpoints

Set breakpoints in your API wrapper:

```python
# In api_wrapper.py
def get_items(self, project_id: str, limit: int = 10) -> dict:
    """Retrieve items from a project."""
    # Set breakpoint here
    breakpoint()  # or use IDE breakpoint
    
    self._log_tool_event(f"Fetching items for {project_id}", "get_items")
    result = self._make_request('GET', '/api/items', params={
        'project_id': project_id,
        'limit': limit
    })
    return result
```

**Debug Flow**:
1. Run Streamlit in debug mode (IDE)
2. Execute tool from UI
3. Execution pauses at breakpoint
4. Inspect variables, step through code
5. Verify API responses
6. Check error handling

## Testing Scenarios

### Basic Tool Testing

```python
# Test 1: Successful request
project_id: "PROJ-123"
limit: 10
Expected: Returns list of items

# Test 2: Empty results
project_id: "EMPTY-PROJECT"
limit: 10
Expected: Returns empty list or appropriate message

# Test 3: Invalid input
project_id: ""
limit: 10
Expected: Validation error

# Test 4: API error
project_id: "INVALID-999"
limit: 10
Expected: Meaningful error message
```

### Error Handling Testing

```python
# Test connection failures
- Invalid base URL
- Wrong API key
- Timeout scenarios
- Network errors

# Test validation errors
- Missing required parameters
- Invalid parameter types
- Out-of-range values
- Malformed inputs
```

### Vector Store Testing

```python
# Test indexing
1. index_data with small dataset
2. Verify documents are indexed
3. Check collection listing
4. Validate metadata extraction

# Test search
1. search_index with simple query
2. Verify relevant results
3. Test filters
4. Check ranking
```

### Performance Testing

```python
# Test pagination
- Large result sets
- Multiple pages
- Cursor handling

# Test rate limiting
- Multiple rapid requests
- Verify throttling works
- Check retry logic

# Test timeouts
- Long-running operations
- Progress reporting
- Cancellation
```

## Debugging Tips

### Common Issues

**Authentication Fails**:
```python
# Check in debugger:
print(f"API Key: {self.api_key.get_secret_value()[:10]}...")
print(f"Headers: {self._session.headers}")
# Verify correct auth header format
```

**Empty Results**:
```python
# Check API response:
print(f"Status: {response.status_code}")
print(f"Response: {response.text}")
# Verify API endpoint and parameters
```

**Timeout Errors**:
```python
# Check timeout setting:
print(f"Timeout: {self.timeout}")
print(f"URL: {url}")
# Increase timeout or optimize request
```

**Tool Not Found**:
```python
# Check tool registration:
tools = self.get_available_tools()
print(f"Available tools: {[t['name'] for t in tools]}")
# Verify tool name matches
```

### Progress Monitoring

Use `_log_tool_event()` to track execution:

```python
def long_running_operation(self, items: List[str]) -> dict:
    """Process multiple items."""
    total = len(items)
    self._log_tool_event(f"Starting processing of {total} items", "long_running")
    
    results = []
    for idx, item in enumerate(items, 1):
        result = self._process_item(item)
        results.append(result)
        
        # Report progress
        if idx % 10 == 0 or idx == total:
            self._log_tool_event(
                f"Processed {idx}/{total} items ({idx*100//total}%)",
                "long_running"
            )
    
    self._log_tool_event(f"Completed processing {total} items", "long_running")
    return {'processed': total, 'results': results}
```

### Logging

Enable logging for more details:

```python
import logging

# In your test script or api_wrapper
logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

# In methods:
logger.debug(f"Making request to {url}")
logger.info(f"Retrieved {len(items)} items")
logger.warning(f"Slow response: {elapsed}s")
logger.error(f"Request failed: {error}")
```

## Test Data Preparation

### Mock API Responses

For testing without live API:

```python
from unittest.mock import Mock, patch

def test_get_items():
    """Test get_items with mocked API."""
    with patch.object(MyApiWrapper, '_make_request') as mock_request:
        mock_request.return_value = {
            'items': [
                {'id': '1', 'title': 'Item 1'},
                {'id': '2', 'title': 'Item 2'}
            ]
        }
        
        wrapper = MyApiWrapper(base_url='http://test', api_key='test')
        result = wrapper.get_items('PROJ-123', limit=10)
        
        assert len(result['items']) == 2
        mock_request.assert_called_once()
```

### Test Configuration

Save test configurations:

```json
{
  "base_url": "https://api.test.example.com",
  "timeout": 30,
  "verify_ssl": false,
  "test_project_id": "TEST-PROJECT",
  "test_user_id": "test-user"
}
```

## Testing Checklist

### Before Testing
- [ ] `.env` file configured
- [ ] Streamlit running
- [ ] Logged into platform
- [ ] Toolkit visible in dropdown
- [ ] Test data prepared

### During Testing
- [ ] All tools accessible
- [ ] Configuration accepts valid values
- [ ] Required fields validated
- [ ] Optional fields work with defaults
- [ ] Authentication successful
- [ ] API calls complete
- [ ] Responses formatted correctly
- [ ] Errors handled gracefully
- [ ] Progress reported for long operations

### After Testing
- [ ] All test cases passed
- [ ] Error scenarios handled
- [ ] Performance acceptable
- [ ] Documentation updated
- [ ] Known issues documented

## VS Code Launch Configuration

For debugging with VS Code:

`.vscode/launch.json`:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Streamlit Debug",
      "type": "python",
      "request": "launch",
      "module": "streamlit",
      "args": [
        "run",
        "alita_local.py",
        "--server.fileWatcherType",
        "none"
      ],
      "envFile": "${workspaceFolder}/.env",
      "console": "integratedTerminal",
      "justMyCode": false
    }
  ]
}
```

## Integration Testing

Test toolkit with real agents:

1. **Create Test Agent** in platform:
   - Add your toolkit
   - Configure with test credentials
   - Create test prompts

2. **Run Agent Tests**:
   - Use Streamlit agent testing mode
   - Verify toolkit integration
   - Check tool selection by LLM
   - Validate outputs in context

3. **Monitor Execution**:
   - Watch tool invocations
   - Check parameter passing
   - Verify result formatting
   - Observe error handling

## Best Practices

- **Start Simple**: Test one tool at a time
- **Use Function Mode**: Direct tool testing for development
- **Set Breakpoints**: Debug execution flow
- **Log Everything**: Use `_log_tool_event()` liberally
- **Mock When Possible**: Don't hit live APIs for every test
- **Test Edge Cases**: Empty results, errors, timeouts
- **Verify Errors**: Check error messages are helpful
- **Test Performance**: Large datasets, pagination
- **Document Issues**: Keep notes on problems found
- **Iterate Quickly**: Fix and retest immediately

Successfully test your Alita SDK toolkit locally before deployment using comprehensive Streamlit-based testing.
