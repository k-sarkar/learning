# Comprehensive Guide to Using Strands with AWS Bedrock

## Table of Contents
1. [Introduction to Strands and AWS Bedrock](#introduction)
2. [Architecture Overview](#architecture)
3. [Installation and Setup](#installation)
4. [Core Concepts](#core-concepts)
5. [Detailed Implementation Examples](#implementation)
6. [Advanced Features](#advanced)
7. [Best Practices](#best-practices)
8. [Troubleshooting](#troubleshooting)

## Introduction to Strands and AWS Bedrock <a name="introduction"></a>

### What is Strands?
Strands is a workflow orchestration platform that enables developers to create, manage, and execute complex multi-agent AI systems. It provides:
- Task-based workflow management
- Agent collaboration frameworks
- Automated dependency handling
- Monitoring and logging capabilities

### What is AWS Bedrock?
AWS Bedrock is Amazon's fully managed service that provides access to foundation models from leading AI companies. It offers:
- Access to multiple pre-trained models (Claude, Llama, Titan)
- Serverless inference
- Built-in guardrails and security
- Integration with AWS ecosystem

## Architecture Overview <a name="architecture"></a>

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Client App    │    │   Strands API   │    │   Bedrock API   │
│                 │    │                 │    │                 │
│  ┌───────────┐  │    │  ┌───────────┐  │    │  ┌───────────┐  │
│  │   Agent   │  │    │  │  Workflow │  │    │  │  Model    │  │
│  │   Class   │  │    │  │  Manager  │  │    │  │  Inference│  │
│  └───────────┘  │    │  └───────────┘  │    │  └───────────┘  │
│                 │    │                 │    │                 │
│  ┌───────────┐  │    │  ┌───────────┐  │    │  ┌───────────┐  │
│  │   Task    │  │    │  │  Task     │  │    │  │  Prompt   │  │
│  │  Engine   │  │    │  │  Executor │  │    │  │  Generation│  │
│  └───────────┘  │    │  └───────────┘  │    │  └───────────┘  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────────────────────┐
                    │      Strands Core Engine        │
                    │  ┌─────────────────────────┐    │
                    │  │  Agent Manager          │    │
                    │  │  Task Scheduler         │    │
                    │  │  Dependency Resolver     │    │
                    │  │  Output Aggregator      │    │
                    │  └─────────────────────────┘    │
                    └─────────────────────────────────┘
```

## Installation and Setup <a name="installation"></a>

### Prerequisites
```bash
# Install required packages
pip install strands boto3 awscli

# Configure AWS credentials
aws configure
# or set environment variables
export AWS_ACCESS_KEY_ID=your_access_key
export AWS_SECRET_ACCESS_KEY=your_secret_key
export AWS_DEFAULT_REGION=us-east-1
```

### Basic Setup
```python
import boto3
import json
from strands import Agent, Workflow, Task

# Initialize Bedrock client
bedrock = boto3.client(
    service_name='bedrock-runtime',
    region_name='us-east-1'
)
```

## Core Concepts <a name="core-concepts"></a>

### 1. Agents
Agents are the building blocks of your workflow - they represent different AI capabilities:

```python
class ContentAgent(Agent):
    def __init__(self, name, model_id="anthropic.claude-3-sonnet-20240229-v1:0"):
        super().__init__(name)
        self.model_id = model_id
        self.bedrock = boto3.client(
            service_name='bedrock-runtime',
            region_name='us-east-1'
        )
    
    def process(self, task_data):
        # This method is called by the workflow engine
        prompt = task_data.get('prompt', '')
        
        response = self.bedrock.invoke_model(
            modelId=self.model_id,
            body=json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": 1000,
                "messages": [{"role": "user", "content": prompt}]
            })
        )
        
        result = json.loads(response['body'].read())
        return result['content'][0]['text']
```

### 2. Tasks
Tasks represent individual units of work within a workflow:

```python
# Simple task
task = Task(
    name="Generate Summary",
    agent=content_agent,
    input_data={
        "prompt": "Summarize the key points of quantum computing"
    }
)

# Task with dependencies
task_with_deps = Task(
    name="Create Article",
    agent=writer_agent,
    input_data={
        "research": "{{research_task.output}}",
        "topic": "AI Ethics"
    },
    depends_on=["research_task"]
)
```

### 3. Workflows
Workflows orchestrate multiple tasks with dependencies:

```python
workflow = Workflow("Content Creation Pipeline")
workflow.add_task(research_task)
workflow.add_task(write_task, depends_on=[research_task])
workflow.add_task(analyze_task, depends_on=[write_task])
```

## Detailed Implementation Examples <a name="implementation"></a>

### Example 1: Simple Multi-Agent Content Pipeline

```python
import boto3
import json
from strands import Agent, Workflow, Task
from datetime import datetime

class ResearchAgent(Agent):
    def __init__(self, name):
        super().__init__(name)
        self.bedrock = boto3.client(
            service_name='bedrock-runtime',
            region_name='us-east-1'
        )
        self.model_id = "anthropic.claude-3-sonnet-20240229-v1:0"
    
    def process(self, task_data):
        topic = task_data.get('topic', 'AI')
        prompt = f"""
        Research the following topic and provide:
        1. Main concepts and definitions
        2. Current applications
        3. Future predictions
        4. Key challenges
        
        Topic: {topic}
        """
        
        response = self.bedrock.invoke_model(
            modelId=self.model_id,
            body=json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": 1500,
                "messages": [{"role": "user", "content": prompt}],
                "temperature": 0.7,
                "top_p": 0.9
            })
        )
        
        result = json.loads(response['body'].read())
        return result['content'][0]['text']

class WritingAgent(Agent):
    def __init__(self, name):
        super().__init__(name)
        self.bedrock = boto3.client(
            service_name='bedrock-runtime',
            region_name='us-east-1'
        )
        self.model_id = "anthropic.claude-3-sonnet-20240229-v1:0"
    
    def process(self, task_data):
        research = task_data.get('research', '')
        audience = task_data.get('audience', 'General')
        
        prompt = f"""
        Write a comprehensive article based on the research provided.
        Audience: {audience}
        
        Research Data:
        {research}
        
        Format:
        - Introduction (150 words)
        - Main Content (600 words)
        - Conclusion (100 words)
        """
        
        response = self.bedrock.invoke_model(
            modelId=self.model_id,
            body=json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": 2000,
                "messages": [{"role": "user", "content": prompt}],
                "temperature": 0.8,
                "top_p": 0.8
            })
        )
        
        result = json.loads(response['body'].read())
        return result['content'][0]['text']

class AnalysisAgent(Agent):
    def __init__(self, name):
        super().__init__(name)
        self.bedrock = boto3.client(
            service_name='bedrock-runtime',
            region_name='us-east-1'
        )
        self.model_id = "anthropic.claude-3-sonnet-20240229-v1:0"
    
    def process(self, task_data):
        content = task_data.get('content', '')
        
        prompt = f"""
        Analyze the following content and provide:
        1. Key strengths
        2. Areas for improvement
        3. SEO recommendations
        4. Readability score (1-10)
        5. Overall quality assessment
        
        Content:
        {content}
        """
        
        response = self.bedrock.invoke_model(
            modelId=self.model_id,
            body=json.dumps({
                "anthropic_version": "bedrock-2023-05-31",
                "max_tokens": 1000,
                "messages": [{"role": "user", "content": prompt}],
                "temperature": 0.3,
                "top_p": 0.5
            })
        )
        
        result = json.loads(response['body'].read())
        return result['content'][0]['text']

# Create workflow
def create_content_workflow():
    # Create agents
    researcher = ResearchAgent("Researcher")
    writer = WritingAgent("Writer")
    analyzer = AnalysisAgent("Analyzer")
    
    # Create tasks
    research_task = Task(
        name="Research Topic",
        agent=researcher,
        input_data={"topic": "Artificial Intelligence in Healthcare"}
    )
    
    write_task = Task(
        name="Write Article",
        agent=writer,
        input_data={
            "research": "{{research_task.output}}",
            "audience": "Healthcare Professionals"
        }
    )
    
    analyze_task = Task(
        name="Analyze Content",
        agent=analyzer,
        input_data={"content": "{{write_task.output}}"}
    )
    
    # Create workflow
    workflow = Workflow("Healthcare AI Content Pipeline")
    workflow.add_task(research_task)
    workflow.add_task(write_task, depends_on=[research_task])
    workflow.add_task(analyze_task, depends_on=[write_task])
    
    return workflow

# Execute workflow
if __name__ == "__main__":
    workflow = create_content_workflow()
    workflow.execute()
    
    print("Research Results:")
    print(workflow.tasks[0].output)
    print("\n" + "="*50 + "\n")
    
    print("Article Draft:")
    print(workflow.tasks[1].output)
    print("\n" + "="*50 + "\n")
    
    print("Analysis Results:")
    print(workflow.tasks[2].output)
```

### Example 2: Advanced Multi-Agent System with Error Handling

```python
import boto3
import json
import logging
from strands import Agent, Workflow, Task
from typing import Dict, Any, Optional

# Configure logging
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class BedrockAgent(Agent):
    def __init__(self, name: str, model_id: str, max_retries: int = 3):
        super().__init__(name)
        self.bedrock = boto3.client(
            service_name='bedrock-runtime',
            region_name='us-east-1'
        )
        self.model_id = model_id
        self.max_retries = max_retries
    
    def _invoke_model_with_retry(self, prompt: str, **kwargs) -> Dict[str, Any]:
        """Invoke Bedrock model with retry logic"""
        for attempt in range(self.max_retries):
            try:
                body = {
                    "anthropic_version": "bedrock-2023-05-31",
                    "max_tokens": 2000,
                    "messages": [{"role": "user", "content": prompt}],
                    **kwargs
                }
                
                response = self.bedrock.invoke_model(
                    modelId=self.model_id,
                    body=json.dumps(body)
                )
                
                result = json.loads(response['body'].read())
                return result
                
            except Exception as e:
                logger.warning(f"Attempt {attempt + 1} failed: {str(e)}")
                if attempt == self.max_retries - 1:
                    raise e
                continue
    
    def process(self, task_data: Dict[str, Any]) -> str:
        prompt = task_data.get('prompt', '')
        temperature = task_data.get('temperature', 0.7)
        max_tokens = task_data.get('max_tokens', 2000)
        
        try:
            response = self._invoke_model_with_retry(
                prompt=prompt,
                temperature=temperature,
                max_tokens=max_tokens
            )
            
            return response['content'][0]['text']
            
        except Exception as e:
            logger.error(f"Failed to process task: {str(e)}")
            return f"Error processing task: {str(e)}"

class DocumentSummarizerAgent(BedrockAgent):
    def __init__(self, name: str = "DocumentSummarizer"):
        super().__init__(name, "anthropic.claude-3-sonnet-20240229-v1:0")
    
    def process(self, task_data: Dict[str, Any]) -> str:
        document = task_data.get('document', '')
        summary_type = task_data.get('summary_type', 'brief')
        
        prompt = f"""
        Please provide a {summary_type} summary of the following document:
        
        {document}
        
        Summary should be clear, concise, and capture the main points.
        """
        
        return super().process({
            "prompt": prompt,
            "temperature": 0.3,
            "max_tokens": 1000
        })

class SentimentAnalyzerAgent(BedrockAgent):
    def __init__(self, name: str = "SentimentAnalyzer"):
        super().__init__(name, "anthropic.claude-3-sonnet-20240229-v1:0")
    
    def process(self, task_data: Dict[str, Any]) -> str:
        text = task_data.get('text', '')
        
        prompt = f"""
        Analyze the sentiment of the following text:
        
        "{text}"
        
        Provide:
        1. Overall sentiment (positive/negative/neutral)
        2. Confidence score (0-100)
        3. Key emotional indicators
        4. Recommendations based on sentiment
        """
        
        return super().process({
            "prompt": prompt,
            "temperature": 0.5,
            "max_tokens": 500
        })

class ContentValidatorAgent(BedrockAgent):
    def __init__(self, name: str = "ContentValidator"):
        super().__init__(name, "anthropic.claude-3-sonnet-20240229-v1:0")
    
    def process(self, task_data: Dict[str, Any]) -> str:
        content = task_data.get('content', '')
        validation_rules = task_data.get('validation_rules', [])
        
        prompt = f"""
        Validate the following content against the specified rules:
        
        Content:
        {content}
        
        Validation Rules:
        {json.dumps(validation_rules, indent=2)}
        
        For each rule, indicate if it's met, and if not, provide specific feedback.
        """
        
        return super().process({
            "prompt": prompt,
            "temperature": 0.2,
            "max_tokens": 1000
        })

# Advanced workflow with validation
def create_advanced_workflow():
    # Create agents
    summarizer = DocumentSummarizerAgent()
    analyzer = SentimentAnalyzerAgent()
    validator = ContentValidatorAgent()
    
    # Create tasks with complex dependencies
    summarize_task = Task(
        name="Summarize Document",
        agent=summarizer,
        input_data={
            "document": "The company's quarterly report shows significant growth in AI research and development. The new machine learning models have improved accuracy by 15% compared to previous versions.",
            "summary_type": "detailed"
        }
    )
    
    analyze_sentiment_task = Task(
        name="Analyze Sentiment",
        agent=analyzer,
        input_data={"text": "{{summarize_task.output}}"}
    )
    
    validate_task = Task(
        name="Validate Content",
        agent=validator,
        input_data={
            "content": "{{summarize_task.output}}",
            "validation_rules": [
                "Content must be between 100 and 500 words",
                "Must include at least one technical term",
                "Should be written in professional tone"
            ]
        }
    )
    
    # Create workflow
    workflow = Workflow("Advanced Content Pipeline")
    workflow.add_task(summarize_task)
    workflow.add_task(analyze_sentiment_task, depends_on=[summarize_task])
    workflow.add_task(validate_task, depends_on=[summarize_task])
    
    return workflow

# Execute advanced workflow
if __name__ == "__main__":
    try:
        workflow = create_advanced_workflow()
        workflow.execute()
        
        print("=== Summary ===")
        print(workflow.tasks[0].output)
        print("\n=== Sentiment Analysis ===")
        print(workflow.tasks[1].output)
        print("\n=== Validation Results ===")
        print(workflow.tasks[2].output)
        
    except Exception as e:
        logger.error(f"Workflow execution failed: {str(e)}")
```

## Configuration and Best Practices <a name="best-practices"></a>

### 1. Model Selection
```python
# Different models for different tasks
MODEL_CONFIGS = {
    "research": "anthropic.claude-3-sonnet-20240229-v1:0",  # Good balance
    "writing": "anthropic.claude-3-sonnet-20240229-v1:0",   # Creative tasks
    "analysis": "anthropic.claude-3-sonnet-20240229-v1:0",  # Analytical tasks
    "coding": "anthropic.claude-3-sonnet-20240229-v1:0"     # Technical tasks
}
```

### 2. Rate Limiting and Error Handling
```python
import time
import random
from functools import wraps

def rate_limit(calls_per_second: int = 1):
    """Simple rate limiting decorator"""
    min_interval = 1.0 / calls_per_second
    last_called = [0.0]
    
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            elapsed = time.time() - last_called[0]
            left_to_wait = min_interval - elapsed
            if left_to_wait > 0:
                time.sleep(left_to_wait)
            ret = func(*args, **kwargs)
            last_called[0] = time.time()
            return ret
        return wrapper
    return decorator
```

### 3. Monitoring and Logging
```python
import logging
from datetime import datetime

class WorkflowMonitor:
    def __init__(self):
        self.logger = logging.getLogger(__name__)
    
    def log_task_execution(self, task_name: str, duration: float, success: bool):
        self.logger.info(
            f"Task '{task_name}' completed in {duration:.2f}s - "
            f"{'SUCCESS' if success else 'FAILED'}"
        )
    
    def log_workflow_start(self, workflow_name: str):
        self.logger.info(f"Starting workflow: {workflow_name}")
    
    def log_workflow_end(self, workflow_name: str, duration: float):
        self.logger.info(f"Workflow '{workflow_name}' completed in {duration:.2f}s")
```

This comprehensive guide provides you with everything needed to build robust multi-agent workflows using Bedrock and Strands. The examples demonstrate various patterns from simple pipelines to complex systems with error handling and monitoring.