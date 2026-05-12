# Comprehensive Guide to Structured Output with Strands and AWS Bedrock

## Core Concepts

Structured output refers to generating AI responses in a specific, predictable format (JSON, XML, CSV, etc.) rather than natural language text. This is crucial for downstream processing, data integration, and maintaining data quality.

## 1. Understanding Structured Output Requirements

### What Makes Output Structured?
- **Predictable Format**: Always returns the same schema
- **Valid Data Types**: Numbers, strings, booleans, arrays, objects
- **Consistent Naming**: Same field names across all responses
- **No Ambiguity**: Clear, unambiguous responses

### Why Use Structured Output?
1. **Data Processing**: Easy to parse and store in databases
2. **Integration**: Compatible with APIs and other systems
3. **Validation**: Can validate against schemas
4. **Automation**: Enables programmatic workflows

## 2. Setting Up the Foundation

```python
import boto3
import json
from strands import Strands
import re

# Initialize clients
bedrock_client = boto3.client(
    service_name='bedrock-runtime',
    region_name='us-east-1'
)

strands = Strands()
```

## 3. Core Prompt Engineering for Structured Output

### Basic Template Structure

```python
def create_structured_output_prompt():
    """Create a prompt that forces structured output"""
    return """
    You are an expert data extractor and formatter. Your task is to analyze the input text and return 
    ONLY valid JSON that matches the exact schema provided below.

    Schema:
    {
        "customer_id": "string",
        "customer_name": "string",
        "feedback_category": "string",
        "sentiment_score": "number",
        "key_issues": ["string"],
        "resolution_suggestion": "string",
        "priority": "string"
    }

    Input Text:
    {input_text}

    Return ONLY the JSON object that matches this schema. Do not include any explanations, 
    markdown, or additional text. Only valid JSON.
    """

# Create template with Strands
template = strands.create_prompt(
    name="structured_feedback_analysis",
    template=create_structured_output_prompt(),
    variables=["input_text"]
)
```

## 4. Implementation Examples

### Example 1: Customer Feedback Analysis

```python
def analyze_feedback_structured(feedback_text):
    """Analyze feedback and return structured JSON"""
    
    # Prepare the prompt with dynamic input
    inputs = {
        "input_text": feedback_text
    }
    
    # Generate prompt
    prompt = strands.fill_prompt(template, inputs)
    
    # Call Bedrock with structured output instruction
    response = call_bedrock_structured(prompt)
    
    # Parse and validate the response
    try:
        result = json.loads(response)
        return validate_structured_output(result)
    except json.JSONDecodeError as e:
        print(f"Failed to parse structured output: {e}")
        return {"error": "Invalid structured output", "raw_response": response}

def call_bedrock_structured(prompt, model_id="anthropic.claude-3-haiku-20240307-v1:0"):
    """Call Bedrock with structured output constraints"""
    
    payload = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1000,
        "messages": [
            {
                "role": "user",
                "content": prompt
            }
        ],
        "temperature": 0.1,  # Lower temperature for more consistent output
        "top_p": 0.9
    }
    
    body = json.dumps(payload)
    
    response = bedrock_client.invoke_model(
        modelId=model_id,
        body=body
    )
    
    response_body = json.loads(response['body'].read())
    return response_body['content'][0]['text']

def validate_structured_output(data):
    """Validate that the output matches expected schema"""
    required_fields = ["customer_id", "customer_name", "feedback_category", 
                      "sentiment_score", "key_issues", "resolution_suggestion", "priority"]
    
    # Check if all required fields exist
    for field in required_fields:
        if field not in data:
            data[field] = None  # Set default value
    
    # Validate data types
    if not isinstance(data.get("sentiment_score"), (int, float)):
        data["sentiment_score"] = 0
    
    if not isinstance(data.get("key_issues"), list):
        data["key_issues"] = []
    
    return data
```

### Example 2: Invoice Data Extraction

```python
def create_invoice_extraction_template():
    """Template for extracting invoice data"""
    return """
    Extract the following information from the invoice text below and return ONLY valid JSON.

    Required Fields:
    - invoice_number: string (the invoice number)
    - vendor_name: string (the company that issued the invoice)
    - invoice_date: string (in YYYY-MM-DD format)
    - due_date: string (in YYYY-MM-DD format)
    - total_amount: number (the total invoice amount)
    - currency: string (the currency code)
    - line_items: array of objects with {description: string, quantity: number, unit_price: number, total: number}
    - tax_amount: number (the tax amount)
    - payment_terms: string (payment terms)

    Invoice Text:
    {invoice_text}

    Return ONLY valid JSON that matches this exact schema.
    """

def extract_invoice_data(invoice_text):
    """Extract structured data from invoice text"""
    
    invoice_template = strands.create_prompt(
        name="invoice_extraction",
        template=create_invoice_extraction_template(),
        variables=["invoice_text"]
    )
    
    inputs = {"invoice_text": invoice_text}
    prompt = strands.fill_prompt(invoice_template, inputs)
    
    response = call_bedrock_structured(prompt)
    
    try:
        result = json.loads(response)
        return result
    except json.JSONDecodeError:
        return {"error": "Failed to extract structured data", "raw_output": response}
```

## 5. Advanced Techniques for Better Structured Output

### Technique 1: Output Constraints and Examples

```python
def create_constrained_prompt():
    """Create a prompt with explicit constraints and examples"""
    return """
    You must return ONLY valid JSON that matches this exact schema:
    {
        "product_id": "string",
        "product_name": "string",
        "category": "string",
        "price": "number",
        "rating": "number",
        "review_count": "number",
        "features": ["string"],
        "is_available": "boolean",
        "tags": ["string"]
    }

    Here's an example of correct output:
    {
        "product_id": "P12345",
        "product_name": "Wireless Headphones",
        "category": "Electronics",
        "price": 199.99,
        "rating": 4.5,
        "review_count": 128,
        "features": ["Noise cancellation", "Bluetooth 5.0", "30-hour battery"],
        "is_available": true,
        "tags": ["wireless", "audio", "premium"]
    }

    Input Product Description:
    {product_description}

    Return ONLY the JSON object that matches the schema exactly.
    """

def process_product_description(description):
    """Process product description into structured data"""
    
    template = strands.create_prompt(
        name="product_structured",
        template=create_constrained_prompt(),
        variables=["product_description"]
    )
    
    inputs = {"product_description": description}
    prompt = strands.fill_prompt(template, inputs)
    
    response = call_bedrock_structured(prompt)
    
    try:
        return json.loads(response)
    except json.JSONDecodeError:
        return {"error": "Failed to generate structured output"}
```

### Technique 2: Iterative Refinement

```python
def get_structured_output_with_refinement(prompt, max_attempts=3):
    """Get structured output with refinement attempts"""
    
    for attempt in range(max_attempts):
        try:
            response = call_bedrock_structured(prompt)
            
            # Try to parse JSON
            parsed = json.loads(response)
            
            # Validate structure
            if validate_structure(parsed):
                return parsed
                
        except (json.JSONDecodeError, ValueError) as e:
            # If parsing fails, ask for correction
            if attempt < max_attempts - 1:
                prompt = prompt + f"\n\nPrevious attempt failed with error: {str(e)}. Please return ONLY valid JSON matching the schema."
                continue
            else:
                return {"error": "Failed to generate valid structured output", "raw": response}
    
    return {"error": "Max attempts exceeded"}

def validate_structure(data):
    """Validate that data matches required structure"""
    required_fields = ["product_id", "product_name", "category", "price", "rating"]
    
    for field in required_fields:
        if field not in data:
            return False
    
    if not isinstance(data["price"], (int, float)):
        return False
    
    if not isinstance(data["rating"], (int, float)):
        return False
    
    return True
```

## 6. Complete Working Example

```python
def comprehensive_structured_output_example():
    """Complete example showing structured output workflow"""
    
    # Sample customer feedback
    feedback = """
    I purchased the wireless headphones last month and I'm very disappointed. 
    The battery life is terrible - only lasts 2 hours, and the sound quality 
    is poor. Customer service was unhelpful when I tried to return it. 
    I would not recommend this product to anyone. The price was too high for 
    the quality received.
    """
    
    # Process the feedback
    result = analyze_feedback_structured(feedback)
    
    print("Structured Output:")
    print(json.dumps(result, indent=2))
    
    return result

# Run the example
if __name__ == "__main__":
    # Test with sample data
    sample_feedback = "The product is excellent. Fast delivery, great quality, and very satisfied with the purchase."
    result = analyze_feedback_structured(sample_feedback)
    print("Sample Output:")
    print(json.dumps(result, indent=2))
```

## 7. Best Practices for Structured Output

### Practice 1: Clear Schema Definition

```python
def create_clear_schema_prompt():
    """Prompt with explicit schema definition"""
    return """
    Your task is to extract data into a specific JSON structure. 
    The output must be valid JSON that exactly matches this schema:

    SCHEMA:
    {
        "timestamp": "string",
        "event_type": "string",
        "user_id": "string",
        "metadata": {
            "device_type": "string",
            "browser": "string",
            "location": "string"
        },
        "properties": {
            "value": "number",
            "category": "string"
        }
    }

    Input Data:
    {input_data}

    Return ONLY valid JSON matching this schema exactly. No explanations, no extra text.
    """
```

### Practice 2: Temperature and Constraints

```python
def call_bedrock_with_constraints(prompt, model_id="anthropic.claude-3-haiku-20240307-v1:0"):
    """Call Bedrock with optimal settings for structured output"""
    
    payload = {
        "anthropic_version": "bedrock-2023-05-31",
        "max_tokens": 1000,
        "messages": [
            {
                "role": "user",
                "content": prompt
            }
        ],
        "temperature": 0.0,  # Zero temperature for consistent output
        "top_p": 1.0,
        "top_k": 0
    }
    
    body = json.dumps(payload)
    
    response = bedrock_client.invoke_model(
        modelId=model_id,
        body=body
    )
    
    response_body = json.loads(response['body'].read())
    return response_body['content'][0]['text']
```

## 8. Error Handling and Validation

```python
class StructuredOutputValidator:
    """Validator for structured output"""
    
    def __init__(self, schema):
        self.schema = schema
    
    def validate(self, data):
        """Validate data against schema"""
        errors = []
        
        for field, field_type in self.schema.items():
            if field not in data:
                errors.append(f"Missing required field: {field}")
            elif not self._validate_type(data[field], field_type):
                errors.append(f"Invalid type for {field}: expected {field_type}")
        
        return len(errors) == 0, errors
    
    def _validate_type(self, value, expected_type):
        """Validate individual field type"""
        if expected_type == "string":
            return isinstance(value, str)
        elif expected_type == "number":
            return isinstance(value, (int, float))
        elif expected_type == "boolean":
            return isinstance(value, bool)
        elif expected_type == "array":
            return isinstance(value, list)
        elif expected_type == "object":
            return isinstance(value, dict)
        return True

# Usage example
schema = {
    "customer_id": "string",
    "sentiment_score": "number",
    "key_issues": "array"
}

validator = StructuredOutputValidator(schema)
```

## Key Takeaways

1. **Always specify the exact output format** in your prompts
2. **Use low temperature settings** (0.0-0.3) for consistent output
3. **Provide examples** of correct structured output
4. **Implement validation** to ensure output quality
5. **Use Strands for prompt management** and reusability
6. **Handle parsing errors gracefully** with fallback mechanisms
7. **Test with multiple inputs** to ensure robustness

This approach ensures you get reliable, consistent, structured output that can be easily integrated into downstream systems and applications.