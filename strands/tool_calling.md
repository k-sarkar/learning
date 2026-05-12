# Comprehensive Guide to Tool Calling with Strands and AWS Bedrock

## Core Concepts

Tool calling is the ability for AI models to execute functions or call external APIs based on the user's request. This enables AI agents to perform real-world actions like retrieving data, making API calls, or executing business logic.

## 1. Understanding Tool Calling Architecture

### What is Tool Calling?
Tool calling allows AI models to:
- **Identify when to use tools** based on user requests
- **Generate tool parameters** in structured format
- **Execute tools** and return results to the user
- **Handle tool errors** gracefully

### Key Components
1. **Tool Definitions** - JSON schemas describing available tools
2. **Tool Execution** - Actual function calls to external services
3. **Result Processing** - Handling tool responses for model continuation
4. **Error Handling** - Managing failed tool calls

## 2. Setting Up Tool Calling Infrastructure

### Basic Setup with Strands

```python
import boto3
import json
from strands import Strands
import uuid

# Initialize clients
bedrock_client = boto3.client(
    service_name='bedrock-runtime',
    region_name='us-east-1'
)

strands = Strands()

# Tool definitions
tools = [
    {
        "name": "get_customer_info",
        "description": "Get customer information by customer ID",
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_id": {
                    "type": "string",
                    "description": "Unique customer identifier"
                }
            },
            "required": ["customer_id"]
        }
    },
    {
        "name": "search_products",
        "description": "Search for products by category or keyword",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Search query"
                },
                "category": {
                    "type": "string",
                    "description": "Product category"
                }
            },
            "required": []
        }
    },
    {
        "name": "create_order",
        "description": "Create a new order for a customer",
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_id": {
                    "type": "string",
                    "description": "Customer identifier"
                },
                "items": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {
                            "product_id": {"type": "string"},
                            "quantity": {"type": "integer"}
                        },
                        "required": ["product_id", "quantity"]
                    }
                },
                "shipping_address": {
                    "type": "object",
                    "properties": {
                        "street": {"type": "string"},
                        "city": {"type": "string"},
                        "zip_code": {"type": "string"}
                    }
                }
            },
            "required": ["customer_id", "items"]
        }
    }
]
```

## 3. Creating Tool Definitions with Strands

```python
def create_tool_calling_prompt():
    """Create a prompt that enables tool calling"""
    return """
    You are an intelligent assistant with access to the following tools:

    {tool_definitions}

    When a user makes a request, determine if you need to use any of these tools to fulfill the request.
    If you need to use a tool, respond with exactly this JSON format:
    
    {
        "tool_call": {
            "name": "tool_name",
            "arguments": {
                "parameter1": "value1",
                "parameter2": "value2"
            }
        }
    }

    If no tool is needed, respond with:
    
    {
        "response": "Your natural language response here"
    }

    Important rules:
    1. Only use tools when specifically required to complete the task
    2. Always respond in the exact JSON format shown above
    3. Never explain your reasoning in natural language outside of the JSON
    4. If a tool fails, you must handle the error gracefully
    5. The tool names must match exactly what's listed above

    User request: {user_request}
    """

def create_tool_definitions_string(tools):
    """Convert tool definitions to readable string format"""
    tool_str = ""
    for i, tool in enumerate(tools):
        tool_str += f"{i+1}. {tool['name']}: {tool['description']}\n"
        if 'input_schema' in tool:
            tool_str += f"   Parameters: {json.dumps(tool['input_schema'], indent=2)}\n"
        tool_str += "\n"
    return tool_str

# Create the tool calling template
tool_calling_template = strands.create_prompt(
    name="tool_calling",
    template=create_tool_calling_prompt(),
    variables=["tool_definitions", "user_request"]
)
```

## 4. Implementing Tool Execution Functions

```python
# Mock database for demonstration
customer_database = {
    "CUST001": {
        "id": "CUST001",
        "name": "John Smith",
        "email": "john.smith@email.com",
        "phone": "555-0101",
        "address": "123 Main St, Anytown, ST 12345"
    },
    "CUST002": {
        "id": "CUST002",
        "name": "Jane Doe",
        "email": "jane.doe@email.com",
        "phone": "555-0102",
        "address": "456 Oak Ave, Somewhere, ST 67890"
    }
}

product_catalog = [
    {"id": "PROD001", "name": "Laptop", "category": "electronics", "price": 999.99},
    {"id": "PROD002", "name": "Book", "category": "books", "price": 19.99},
    {"id": "PROD003", "name": "Phone", "category": "electronics", "price": 699.99}
]

def get_customer_info(customer_id):
    """Mock function to get customer information"""
    return customer_database.get(customer_id, {"error": "Customer not found"})

def search_products(query=None, category=None):
    """Mock function to search products"""
    results = product_catalog
    if category:
        results = [p for p in results if p['category'] == category]
    if query:
        results = [p for p in results if query.lower() in p['name'].lower()]
    return results

def create_order(customer_id, items, shipping_address=None):
    """Mock function to create an order"""
    order_id = f"ORDER-{uuid.uuid4().hex[:8].upper()}"
    return {
        "order_id": order_id,
        "customer_id": customer_id,
        "items": items,
        "status": "created",
        "shipping_address": shipping_address
    }

# Tool registry
tool_registry = {
    "get_customer_info": get_customer_info,
    "search_products": search_products,
    "create_order": create_order
}
```

## 5. Complete Tool Calling Implementation

```python
import re
import json

class ToolCallingAgent:
    def __init__(self, tools, strands_client):
        self.tools = tools
        self.strands = strands_client
        self.tool_registry = tool_registry
    
    def generate_tool_definitions(self):
        """Generate tool definitions string for the prompt"""
        tool_defs = []
        for tool in self.tools:
            tool_def = {
                "name": tool["name"],
                "description": tool["description"],
                "parameters": tool["input_schema"]
            }
            tool_defs.append(tool_def)
        return json.dumps(tool_defs, indent=2)
    
    def parse_tool_call(self, response_text):
        """Parse tool call from model response"""
        try:
            # Try to parse JSON response
            response = json.loads(response_text)
            if "tool_call" in response:
                return response["tool_call"]
            elif "response" in response:
                return {"response": response["response"]}
            return None
        except json.JSONDecodeError:
            # If JSON parsing fails, try to extract tool call from text
            tool_call_pattern = r'\{.*?"tool_call".*?\}'
            match = re.search(tool_call_pattern, response_text, re.DOTALL)
            if match:
                try:
                    return json.loads(match.group())
                except:
                    return None
            return None
    
    def execute_tool_call(self, tool_call):
        """Execute the tool call"""
        tool_name = tool_call.get("name")
        arguments = tool_call.get("arguments", {})
        
        if tool_name not in self.tool_registry:
            return {"error": f"Tool '{tool_name}' not found"}
        
        try:
            tool_function = self.tool_registry[tool_name]
            result = tool_function(**arguments)
            return {
                "tool_name": tool_name,
                "arguments": arguments,
                "result": result
            }
        except Exception as e:
            return {
                "tool_name": tool_name,
                "arguments": arguments,
                "error": str(e)
            }
    
    def process_request(self, user_request):
        """Process user request with tool calling"""
        # Generate tool definitions string
        tool_definitions = self.generate_tool_definitions()
        
        # Create prompt using Strands
        prompt_data = {
            "tool_definitions": tool_definitions,
            "user_request": user_request
        }
        
        # Get the prompt template
        prompt_template = self.strands.get_prompt("tool_calling")
        
        # Generate the prompt
        final_prompt = prompt_template.format(**prompt_data)
        
        # Call Bedrock with tool calling enabled
        tool_call_response = self.call_bedrock_with_tools(final_prompt)
        
        # Parse the response
        tool_call = self.parse_tool_call(tool_call_response)
        
        if tool_call and "response" in tool_call:
            return tool_call["response"]
        elif tool_call and "tool_call" in tool_call:
            # Execute the tool call
            tool_result = self.execute_tool_call(tool_call["tool_call"])
            
            # Create a follow-up prompt with tool results
            followup_prompt = self.create_followup_prompt(user_request, tool_result)
            
            # Get final response
            final_response = self.call_bedrock_with_tools(followup_prompt)
            return final_response
        else:
            return "I couldn't process your request properly."

    def call_bedrock_with_tools(self, prompt):
        """Call Bedrock with tool calling enabled"""
        try:
            response = bedrock_client.invoke_model(
                modelId="anthropic.claude-3-haiku-20240307-v1:0",
                body=json.dumps({
                    "anthropic_version": "bedrock-2023-05-31",
                    "max_tokens": 1000,
                    "messages": [
                        {
                            "role": "user",
                            "content": prompt
                        }
                    ],
                    "tools": self.tools,
                    "temperature": 0.0
                })
            )
            
            response_body = json.loads(response['body'].read())
            return response_body['content'][0]['text']
        except Exception as e:
            return f"Error calling Bedrock: {str(e)}"

    def create_followup_prompt(self, user_request, tool_result):
        """Create prompt for follow-up response with tool results"""
        return f"""
        User request: {user_request}
        
        Tool execution result:
        {json.dumps(tool_result, indent=2)}
        
        Please provide a natural language response to the user based on the tool result above.
        """
```

## 6. Advanced Tool Calling with Error Handling

```python
class AdvancedToolCallingAgent(ToolCallingAgent):
    def __init__(self, tools, strands_client):
        super().__init__(tools, strands_client)
        self.max_retries = 3
        self.retry_delay = 1
    
    def execute_tool_call_with_retry(self, tool_call):
        """Execute tool call with retry logic"""
        tool_name = tool_call.get("name")
        arguments = tool_call.get("arguments", {})
        
        for attempt in range(self.max_retries):
            try:
                tool_function = self.tool_registry[tool_name]
                result = tool_function(**arguments)
                return {
                    "tool_name": tool_name,
                    "arguments": arguments,
                    "result": result,
                    "success": True,
                    "attempt": attempt + 1
                }
            except Exception as e:
                if attempt == self.max_retries - 1:
                    return {
                        "tool_name": tool_name,
                        "arguments": arguments,
                        "error": str(e),
                        "success": False,
                        "attempt": attempt + 1
                    }
                # Wait before retrying
                import time
                time.sleep(self.retry_delay)
        
        return {
            "tool_name": tool_name,
            "arguments": arguments,
            "error": "Max retries exceeded",
            "success": False,
            "attempt": self.max_retries
        }
    
    def process_request_with_context(self, user_request, conversation_history=None):
        """Process request with conversation context"""
        # Build context for the tool calling
        context = ""
        if conversation_history:
            context = "Previous conversation:\n"
            for msg in conversation_history[-3:]:  # Last 3 messages
                context += f"{msg['role']}: {msg['content']}\n"
        
        tool_definitions = self.generate_tool_definitions()
        
        # Enhanced prompt with context
        enhanced_prompt = f"""
        {context}
        
        Available tools:
        {tool_definitions}
        
        User request: {user_request}
        
        Instructions:
        1. Determine if any tools are needed to fulfill the request
        2. If tools are needed, format your response as JSON with a tool_call
        3. If no tools are needed, provide a natural language response
        4. Always respond in JSON format when using tools
        """
        
        # Get response
        response = self.call_bedrock_with_tools(enhanced_prompt)
        
        # Parse and execute
        tool_call = self.parse_tool_call(response)
        
        if tool_call and "tool_call" in tool_call:
            tool_result = self.execute_tool_call_with_retry(tool_call["tool_call"])
            return self.handle_tool_result(user_request, tool_result)
        elif tool_call and "response" in tool_call:
            return tool_call["response"]
        else:
            return "I couldn't understand your request properly."
    
    def handle_tool_result(self, original_request, tool_result):
        """Handle tool execution results"""
        if tool_result.get("success"):
            return f"Tool execution successful: {json.dumps(tool_result['result'], indent=2)}"
        else:
            return f"Tool execution failed: {tool_result['error']}"
```

## 7. Usage Examples

```python
# Initialize the agent
tools = [
    {
        "name": "get_customer_info",
        "description": "Get customer information by customer ID",
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_id": {
                    "type": "string",
                    "description": "The customer ID"
                }
            },
            "required": ["customer_id"]
        }
    },
    {
        "name": "search_products",
        "description": "Search for products by category or query",
        "input_schema": {
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "Search query"
                },
                "category": {
                    "type": "string",
                    "description": "Product category"
                }
            }
        }
    },
    {
        "name": "create_order",
        "description": "Create a new order",
        "input_schema": {
            "type": "object",
            "properties": {
                "customer_id": {
                    "type": "string",
                    "description": "Customer ID"
                },
                "items": {
                    "type": "array",
                    "items": {
                        "type": "object",
                        "properties": {
                            "product_id": {"type": "string"},
                            "quantity": {"type": "number"}
                        }
                    }
                },
                "shipping_address": {
                    "type": "object",
                    "properties": {
                        "street": {"type": "string"},
                        "city": {"type": "string"},
                        "state": {"type": "string"},
                        "zip": {"type": "string"}
                    }
                }
            },
            "required": ["customer_id", "items"]
        }
    }
]

# Create agent
agent = ToolCallingAgent(tools, strands)

# Example usage
print("Example 1: Get customer info")
response1 = agent.process_request("Get information for customer CUST001")
print(response1)

print("\nExample 2: Search products")
response2 = agent.process_request("Find all electronics products")
print(response2)

print("\nExample 3: Create order")
response3 = agent.process_request("Create an order for John Smith with 2 laptops")
print(response3)
```

This implementation provides a complete tool calling system that:

1. **Integrates with Strands** for prompt management
2. **Uses Bedrock's tool calling capabilities** with proper JSON formatting
3. **Handles tool execution** with error handling and retry logic
4. **Manages conversation context** for more sophisticated interactions
5. **Provides robust parsing** of tool call responses
6. **Supports complex workflows** with multiple tool calls

The system is designed to be extensible and can easily accommodate new tools by adding them to the tool registry and updating the tool definitions.