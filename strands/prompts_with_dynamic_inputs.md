# Comprehensive Guide to Dynamic Prompt Generation with Strands and AWS Bedrock

## Core Concepts

Dynamic prompt generation allows you to create flexible, reusable prompts that can adapt to different inputs, contexts, and user needs. This is essential for building intelligent AI applications that can handle diverse scenarios.

## 1. Understanding Dynamic Prompts

### What are Dynamic Prompts?
Dynamic prompts are templates that contain placeholders for variables that get replaced with actual values at runtime. These prompts can:
- Adapt to different user inputs
- Handle varying contexts
- Support multiple languages or formats
- Enable personalization

### Key Components
1. **Prompt Templates** - Base structure with placeholders
2. **Variables/Parameters** - Values that fill the placeholders
3. **Template Engine** - System that replaces placeholders with values
4. **Context Management** - Handling conversation history and state

## 2. Setting Up Strands for Dynamic Prompts

### Basic Strands Setup

```python
import json
from strands import StrandsClient

# Initialize Strands client
strands = StrandsClient(api_key="your-api-key")

# Create a dynamic prompt template
prompt_template = {
    "name": "customer_service_template",
    "description": "Dynamic customer service prompt",
    "template": """
    You are a customer service assistant for {company_name}.
    
    Customer: {customer_name} (ID: {customer_id})
    Issue: {issue_description}
    Priority: {priority_level}
    
    Please respond to the customer's inquiry in a helpful and professional manner.
    
    Customer's previous interactions:
    {previous_interactions}
    
    Current date: {current_date}
    
    Response format:
    - Acknowledgment of their issue
    - Proposed solution or next steps
    - Follow-up actions
    """,
    "variables": ["company_name", "customer_name", "customer_id", 
                 "issue_description", "priority_level", "previous_interactions", "current_date"]
}

# Register the template with Strands
strands.register_prompt(prompt_template)
```

## 3. Creating Dynamic Prompt Templates

### Template Structure

```python
# More complex example with nested structures
complex_template = {
    "name": "content_generation_template",
    "description": "Template for generating marketing content",
    "template": """
    Create marketing content for {product_name} in {target_audience} market.
    
    Product Details:
    - Price: ${product_price}
    - Features: {product_features}
    - Benefits: {product_benefits}
    
    Marketing Objective: {marketing_objective}
    
    Tone: {tone_style}
    Length: {content_length} words
    Format: {content_format}
    
    Additional Instructions:
    {additional_instructions}
    
    Content should include:
    1. Hook: {hook_example}
    2. Problem statement: {problem_example}
    3. Solution: {solution_example}
    4. Call to action: {cta_example}
    """,
    "variables": ["product_name", "target_audience", "product_price", 
                 "product_features", "product_benefits", "marketing_objective",
                 "tone_style", "content_length", "content_format",
                 "additional_instructions", "hook_example", "problem_example",
                 "solution_example", "cta_example"]
}

strands.register_prompt(complex_template)
```

## 4. Using Variables in Prompts

### Simple Variable Substitution

```python
# Simple example with basic variables
def create_simple_prompt(strands_client):
    # Define your variables
    prompt_vars = {
        "company_name": "TechCorp Solutions",
        "customer_name": "John Smith",
        "customer_id": "CUST-12345",
        "issue_description": "Product not working after 2 weeks",
        "priority_level": "High",
        "previous_interactions": "No previous interactions",
        "current_date": "2024-01-15"
    }
    
    # Get the template
    template = strands_client.get_prompt("customer_service_template")
    
    # Generate the prompt
    generated_prompt = template.format(**prompt_vars)
    
    return generated_prompt

# Usage
simple_prompt = create_simple_prompt(strands)
print(simple_prompt)
```

### Complex Variable Handling

```python
# Handle complex data structures
def create_complex_prompt(strands_client):
    # Complex variables
    complex_vars = {
        "product_name": "SmartWatch Pro",
        "target_audience": "Tech-savvy millennials",
        "product_price": 299.99,
        "product_features": ["Heart rate monitoring", "GPS tracking", "Water resistant"],
        "product_benefits": ["Health tracking", "Fitness coaching", "Safety monitoring"],
        "marketing_objective": "Increase sales by 20% in Q1",
        "tone_style": "Professional yet approachable",
        "content_length": 300,
        "content_format": "Social media post",
        "additional_instructions": "Include a discount code for first-time buyers",
        "hook_example": "Track your fitness journey with our smart watch",
        "problem_example": "Are you tired of missing important health metrics?",
        "solution_example": "Our SmartWatch Pro tracks everything you need",
        "cta_example": "Get 15% off your first purchase"
    }
    
    # Get template and generate
    template = strands_client.get_prompt("content_generation_template")
    generated_prompt = template.format(**complex_vars)
    
    return generated_prompt

# Usage
complex_prompt = create_complex_prompt(strands)
print(complex_prompt)
```

## 5. Advanced Dynamic Prompt Features

### Conditional Prompts

```python
# Template with conditional logic
conditional_template = {
    "name": "conditional_prompt_template",
    "description": "Template with conditional content",
    "template": """
    Customer: {customer_name}
    Status: {customer_status}
    
    {if_customer_status == 'VIP' then 'Welcome back, valued customer!' else 'Thank you for your purchase'}
    
    {if_customer_status == 'VIP' then 'You have access to exclusive offers' else 'Check our latest deals'}
    
    {if_customer_status == 'VIP' then 
        'Special benefits include: 20% discount, early access to new products, free shipping'
     else 'Regular benefits include: 10% discount, standard shipping'}
    """,
    "variables": ["customer_name", "customer_status"]
}

# Function to handle conditional logic
def create_conditional_prompt(strands_client, customer_data):
    # Add conditional logic processing
    if customer_data.get("customer_status") == "VIP":
        customer_data["conditional_message"] = "Welcome back, valued customer!"
        customer_data["benefits"] = "Special benefits include: 20% discount, early access to new products, free shipping"
    else:
        customer_data["conditional_message"] = "Thank you for your purchase"
        customer_data["benefits"] = "Regular benefits include: 10% discount, standard shipping"
    
    template = strands_client.get_prompt("conditional_prompt_template")
    return template.format(**customer_data)
```

### Dynamic Content Generation

```python
# Template for dynamic content
dynamic_content_template = {
    "name": "dynamic_content_template",
    "description": "Template for generating dynamic content",
    "template": """
    Report Summary for {report_period}
    
    Key Metrics:
    {metrics_table}
    
    {if metrics_table contains 'revenue' then 
        'Revenue increased by {revenue_change}% compared to last period' 
     else ''}
    
    {if metrics_table contains 'users' then 
        'User base grew by {user_growth}% this period' 
     else ''}
    
    Recommendations:
    {recommendations}
    """,
    "variables": ["report_period", "metrics_table", "revenue_change", 
                 "user_growth", "recommendations"]
}

def create_dynamic_report_prompt(strands_client, report_data):
    # Process metrics table
    metrics_table = "\n".join([f"- {metric}: {value}" for metric, value in report_data.get("metrics", {}).items()])
    
    # Prepare recommendations
    recommendations = "\n".join([f"- {rec}" for rec in report_data.get("recommendations", [])])
    
    # Build final data
    prompt_data = {
        "report_period": report_data.get("period", "current period"),
        "metrics_table": metrics_table,
        "recommendations": recommendations,
        "revenue_change": report_data.get("revenue_change", 0),
        "user_growth": report_data.get("user_growth", 0)
    }
    
    # Add conditional logic
    if "revenue" in metrics_table:
        prompt_data["has_revenue"] = True
    if "users" in metrics_table:
        prompt_data["has_users"] = True
    
    template = strands_client.get_prompt("dynamic_content_template")
    return template.format(**prompt_data)
```

## 6. Integration with AWS Bedrock

### Complete Integration Example

```python
import boto3
import json
from datetime import datetime

class DynamicPromptService:
    def __init__(self, strands_client, bedrock_client=None):
        self.strands = strands_client
        self.bedrock = bedrock_client or boto3.client('bedrock-runtime')
        
    def generate_and_execute_prompt(self, template_name, variables, model_id="anthropic.claude-3-haiku-20240307-v1:0"):
        """
        Generate dynamic prompt and execute it with Bedrock
        """
        # Get the template
        template = self.strands.get_prompt(template_name)
        
        # Generate the prompt with variables
        if isinstance(template, str):
            # Simple string template
            generated_prompt = template.format(**variables)
        else:
            # Template object with format method
            generated_prompt = template.format(**variables)
        
        # Execute with Bedrock
        response = self.call_bedrock(generated_prompt, model_id)
        
        return response
    
    def call_bedrock(self, prompt, model_id):
        """
        Call Bedrock with the generated prompt
        """
        try:
            response = self.bedrock.invoke_model(
                modelId=model_id,
                body=json.dumps({
                    "anthropic_version": "bedrock-2023-05-31",
                    "max_tokens": 1000,
                    "messages": [
                        {
                            "role": "user",
                            "content": prompt
                        }
                    ]
                })
            )
            
            response_body = json.loads(response['body'].read())
            return response_body['content'][0]['text']
            
        except Exception as e:
            return f"Error calling Bedrock: {str(e)}"

# Usage example
def example_usage():
    # Initialize services
    strands = StrandsClient(api_key="your-api-key")
    bedrock = boto3.client('bedrock-runtime')
    
    # Create service
    service = DynamicPromptService(strands, bedrock)
    
    # Define variables
    customer_vars = {
        "company_name": "TechCorp Solutions",
        "customer_name": "John Smith",
        "customer_id": "CUST-12345",
        "issue_description": "Product not working after 2 weeks",
        "priority_level": "High",
        "previous_interactions": "No previous interactions",
        "current_date": datetime.now().strftime("%Y-%m-%d")
    }
    
    # Generate and execute
    result = service.generate_and_execute_prompt(
        "customer_service_template", 
        customer_vars
    )
    
    print(result)
```

## 7. Real-World Examples

### Customer Support System

```python
def create_customer_support_prompt(strands_client, customer_info):
    """
    Create a customer support prompt based on customer data
    """
    # Prepare customer data
    customer_data = {
        "customer_name": customer_info.get("name", "Customer"),
        "customer_id": customer_info.get("id", "Unknown"),
        "issue_type": customer_info.get("issue_type", "General inquiry"),
        "issue_description": customer_info.get("description", "No description provided"),
        "priority": customer_info.get("priority", "Medium"),
        "support_level": customer_info.get("support_level", "Standard"),
        "current_time": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "company_name": "TechCorp Solutions"
    }
    
    # Get template
    template = strands_client.get_prompt("customer_service_template")
    
    # Generate prompt
    prompt = template.format(**customer_data)
    
    return prompt

# Example usage
customer_data = {
    "name": "Alice Johnson",
    "id": "CUST-67890",
    "issue_type": "Billing inquiry",
    "description": "Charged incorrectly on last month's bill",
    "priority": "High",
    "support_level": "VIP"
}

support_prompt = create_customer_support_prompt(strands, customer_data)
print(support_prompt)
```

### Content Marketing Generator

```python
def create_marketing_prompt(strands_client, campaign_data):
    """
    Create marketing content based on campaign data
    """
    # Prepare campaign data
    campaign_vars = {
        "product_name": campaign_data.get("product_name", "Product"),
        "target_audience": campaign_data.get("target_audience", "General audience"),
        "product_price": campaign_data.get("price", 0),
        "product_features": ", ".join(campaign_data.get("features", [])),
        "product_benefits": ", ".join(campaign_data.get("benefits", [])),
        "marketing_objective": campaign_data.get("objective", "General promotion"),
        "tone_style": campaign_data.get("tone", "Professional"),
        "content_length": campaign_data.get("length", 200),
        "content_format": campaign_data.get("format", "Email"),
        "additional_instructions": campaign_data.get("instructions", ""),
        "hook_example": campaign_data.get("hook", "Discover the difference"),
        "problem_example": campaign_data.get("problem", "Struggling with current solutions"),
        "solution_example": campaign_data.get("solution", "Our product solves this"),
        "cta_example": campaign_data.get("cta", "Learn more today")
    }
    
    # Get template
    template = strands_client.get_prompt("content_generation_template")
    
    # Generate prompt
    prompt = template.format(**campaign_vars)
    
    return prompt

# Example usage
campaign = {
    "product_name": "SmartHome Hub",
    "target_audience": "Homeowners",
    "price": 199.99,
    "features": ["Voice control", "Smart device integration", "Mobile app"],
    "benefits": ["Convenience", "Energy savings", "Security"],
    "objective": "Increase product awareness",
    "tone": "Friendly and informative",
    "length": 300,
    "format": "Social media post",
    "instructions": "Include a 10% discount code",
    "hook": "Transform your home with smart technology",
    "problem": "Tired of managing multiple devices?",
    "solution": "Our SmartHome Hub controls everything",
    "cta": "Get 10% off your first purchase"
}

marketing_prompt = create_marketing_prompt(strands, campaign)
print(marketing_prompt)
```

This comprehensive approach to dynamic prompt generation allows you to create flexible, reusable templates that can be customized with different variables for various use cases while seamlessly integrating with AWS Bedrock for AI-powered responses.