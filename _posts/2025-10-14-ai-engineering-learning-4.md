---
title: "AI Engineering Learning: Tool Integration with LLMs"
author: pravin_tripathi
date: 2025-10-14 00:00:00 +0530
readtime: true
media_subpath: /assets/img/ai-engineering-learning/
categories: [Blogging, Article]
mermaid: true
tags: [softwareengineering, backenddevelopment, aiengineering, langchain, multiagentsystems]
image:
  path: header.png
  width: 1600   # in pixels
  height: 900   # in pixels
  alt: Generated using Gemini AI
---

## Table of Contents
1. [Introduction to Tool Integration](#introduction-to-tool-integration)
2. [Core Concepts & Theory](#core-concepts--theory)
3. [Week 3: Tool Fundamentals](#week-3-tool-fundamentals)
4. [Week 4: Advanced Tool Integration](#week-4-advanced-tool-integration)
5. [Real-World Projects](#real-world-projects)
6. [Testing & Production](#testing--production)



## Introduction to Tool Integration

### What is Tool Integration (Function Calling)?

**Tool Integration**, also called **Function Calling**, enables LLMs to decide when and which external tools to use to accomplish tasks. Instead of only generating text responses, the LLM can:

1. **Recognize** when a task requires external action
2. **Select** the appropriate tool/function
3. **Extract** necessary arguments from user input
4. **Return** a structured request to call that tool
5. **Incorporate** the tool's response into its reasoning

**Simple Analogy:**
Think of it like giving a smart assistant access to your company's systems. When you ask "What's my order status?", the assistant:
- Recognizes it needs real data
- Decides to use the "check_order_status" function
- Extracts your order ID
- Calls the function
- Uses the result to answer you

### Why Tool Integration Matters

Traditional LLMs have critical limitations that tool integration solves:

| Problem | Tool Integration Solution |
||-|
| **No real-time data** | Tools provide current information |
| **Can't access systems** | Tools can call APIs, databases, CRMs |
| **Can't take actions** | Tools execute tasks (send emails, create tickets) |
| **No transactional capability** | Tools can modify state (process refunds, book appointments) |
| **Limited to knowledge cutoff** | Tools bypass training data limitations |

### Real-World Use Cases

**Customer Support AI**
- Check order status via e-commerce API
- Process refunds automatically
- Search knowledge bases for solutions
- Create support tickets in ticketing systems

**E-Commerce Chatbots**
- Search product inventory
- Add items to cart
- Process payments
- Track shipments

**HR Assistants**
- Check employee policies
- Process time-off requests
- Submit expense reports
- Search internal documentation

**Enterprise Automation**
- Query databases
- Generate reports
- Send notifications
- Update CRM records



## Core Concepts & Theory

### 1. How Function Calling Works

#### The Function Calling Flow

```
User Request
    ↓
[LLM Evaluates Request]
    ↓
LLM decides: "Does this need a tool?"
    ├─ No → Generate text response
    └─ Yes → Select tool and extract arguments
                    ↓
            [Return tool call specification]
                    ↓
            (Your app receives: tool_name, arguments)
                    ↓
            [Your app executes the tool]
                    ↓
            [Pass result back to LLM]
                    ↓
            [LLM generates final response]
```

#### Critical Point: The LLM Doesn't Execute Tools

This is the most important concept to understand:

> **The LLM DECIDES to call a tool. It does NOT execute the tool.**

Your application is responsible for:
1. Receiving the LLM's decision
2. Validating the arguments
3. Executing the actual function
4. Returning the result to the LLM

Think of it like a delivery system:
- LLM is the dispatcher
- You are the delivery driver
- The tool is the package

### 2. Understanding Tools/Functions

#### Tool Definition Components

Every tool needs:

1. **Name**: Unique identifier for the tool
   - Example: `get_order_status`, `search_kb`, `process_refund`
   - Must be descriptive and unambiguous

2. **Description**: When and how to use the tool
   - Should explain the purpose clearly
   - Can include examples or counter-examples
   - Example: "Retrieves the current status of a customer order. Use this when the user asks about their order status or tracking information."

3. **Parameters/Arguments**: What information the tool needs
   - Define the exact inputs required
   - Specify data types (string, number, boolean, etc.)
   - Mark which parameters are required
   - Example: `order_id` (required, string), `include_tracking` (optional, boolean)

4. **Return Type**: What the tool outputs
   - Specify the format of results
   - Examples: JSON object, string, list, etc.
   - Example: "Returns a JSON object with fields: status, tracking_number, estimated_delivery"

#### Tool Definition Example

```
Tool: get_current_weather

Name: get_current_weather

Description: "Retrieves the current weather for a specific city. 
Use this when the user asks about weather conditions, 
temperature, or weather forecasts. Always use the city name, 
not abbreviations (e.g., 'New York' not 'NYC')."

Parameters:
  - city (required, string): The city name
  - units (optional, string): Temperature units ('celsius' or 'fahrenheit')

Returns: JSON object with fields:
  - temperature (number)
  - condition (string)
  - humidity (number)
  - wind_speed (number)

Example:
  Input: {"city": "London", "units": "celsius"}
  Output: {
    "temperature": 15,
    "condition": "Cloudy",
    "humidity": 65,
    "wind_speed": 12
  }
```

### 3. The ReAct Pattern (Reasoning + Acting)

The **ReAct** pattern is the foundation of most agent systems. It alternates between:

1. **Reasoning (Thought)**: LLM thinks about what to do
2. **Acting (Action)**: LLM calls a tool
3. **Observing (Observation)**: Tool returns results
4. **Loop**: Go back to step 1 with new information

#### ReAct Loop Example

```
User: "Is it sunny in London? If so, I'd like to plan a picnic."

Step 1 - Thought:
"The user is asking about weather in London. I need to check 
the current weather using the get_current_weather tool."

Step 2 - Action:
Tool Call: get_current_weather
Arguments: {"city": "London"}

Step 3 - Observation:
Result: {"temperature": 15, "condition": "Sunny", "humidity": 60}

Step 4 - Thought (New):
"Great! It's sunny in London. The user wants to plan a picnic. 
I have enough information to provide recommendations."

Step 5 - Final Answer:
"Yes, it's sunny in London with a temperature of 15°C! 
Here are some picnic tips: bring a light jacket, choose a 
location with good views, pack water and snacks..."
```

#### Multi-Step ReAct Loop

Tools don't have to be called just once. Complex tasks use multiple tool calls:

```
User: "What's the weather like in Paris, Berlin, and Barcelona? 
Should I visit any of them tomorrow?"

Step 1: Call get_current_weather("Paris")
Step 2: Call get_current_weather("Berlin")  
Step 3: Call get_current_weather("Barcelona")
Step 4: Observe all three results
Step 5: Synthesize: "Based on all three cities, I recommend 
Barcelona because it's the sunniest..."
```

### 4. Structured Tools vs Simple Tools

#### Simple Tools (Deprecated)

Simple tools accept free-form text input and output:

```python
@tool
def get_weather(location):
    """Get weather for a location."""
    # Returns text
    return f"Weather in {location}: Sunny"
```

**Problems:**
- LLM might pass wrong data types
- No validation of arguments
- Unclear what parameters are required
- Difficult to ensure consistency

#### Structured Tools (Recommended)

Structured tools use Pydantic schemas to validate inputs:

```python
from pydantic import BaseModel, Field

class GetWeatherInput(BaseModel):
    """Input schema for get_weather tool."""
    city: str = Field(description="City name (e.g., 'London')")
    units: str = Field(
        default="celsius",
        description="Temperature units: 'celsius' or 'fahrenheit'"
    )

@tool(args_schema=GetWeatherInput)
def get_weather(city: str, units: str = "celsius") -> dict:
    """Get current weather for a specific city."""
    # Returns structured JSON
    return {
        "city": city,
        "temperature": 15,
        "condition": "Sunny",
        "units": units
    }
```

**Benefits:**
- Pydantic validates inputs before execution
- Clear required vs optional parameters
- Type safety
- Better error messages
- LLM better understands what's needed



## Week 3: Tool Fundamentals

### 1. Creating Your First Tool

#### Step 1: Installation

```bash
pip install langchain-openai
pip install langgraph
pip install pydantic
```

#### Step 2: Define a Simple Tool

Create `simple_tools.py`:

```python
import os
from dotenv import load_dotenv
from langchain_core.tools import tool

load_dotenv()

# Method 1: Using @tool decorator (Simple)
@tool
def add(a: float, b: float) -> float:
    """Add two numbers together."""
    return a + b

@tool
def multiply(a: float, b: float) -> float:
    """Multiply two numbers together."""
    return a * b

# Verify tools are created correctly
print("Tools created:")
print(f"- {add.name}: {add.description}")
print(f"- {multiply.name}: {multiply.description}")

# Use them directly
result1 = add.invoke({"a": 5, "b": 3})
print(f"\n5 + 3 = {result1}")

result2 = multiply.invoke({"a": 4, "b": 2})
print(f"4 × 2 = {result2}")
```

**Output:**
```
Tools created:
- add: Add two numbers together.
- multiply: Multiply two numbers together.

5 + 3 = 8
4 × 2 = 8
```

#### Step 3: Create Structured Tools

Create `structured_tools.py`:

```python
from pydantic import BaseModel, Field
from langchain_core.tools import tool
import requests
from typing import Optional

# Define input schema
class GetWeatherInput(BaseModel):
    """Input for get_weather tool."""
    city: str = Field(description="The city name (e.g., 'London', 'Paris')")
    units: str = Field(
        default="celsius",
        description="Temperature units: 'celsius' or 'fahrenheit'"
    )

class SearchKBInput(BaseModel):
    """Input for knowledge base search."""
    query: str = Field(description="Search query")
    limit: Optional[int] = Field(
        default=5,
        description="Maximum number of results"
    )

# Create structured tools
@tool(args_schema=GetWeatherInput)
def get_weather(city: str, units: str = "celsius") -> dict:
    """
    Get the current weather for a specific city.
    
    Use this tool when users ask about:
    - Current weather conditions
    - Temperature
    - Weather forecasts
    """
    # Simulate weather API call
    weather_data = {
        "London": {"temp": 15, "condition": "Cloudy"},
        "Paris": {"temp": 18, "condition": "Sunny"},
        "Barcelona": {"temp": 22, "condition": "Sunny"},
    }
    
    data = weather_data.get(city, {"temp": "unknown", "condition": "unknown"})
    
    # Convert to Fahrenheit if needed
    temp = data["temp"]
    if units == "fahrenheit" and temp != "unknown":
        temp = (temp * 9/5) + 32
    
    return {
        "city": city,
        "temperature": temp,
        "condition": data["condition"],
        "units": units
    }

@tool(args_schema=SearchKBInput)
def search_knowledge_base(query: str, limit: int = 5) -> list:
    """
    Search the company knowledge base for relevant articles.
    
    Use this tool when users ask about:
    - Company policies
    - How-to questions
    - Product information
    """
    # Simulate KB search
    articles = {
        "password": [
            "How to reset your password",
            "Password requirements",
            "Two-factor authentication setup"
        ],
        "refund": [
            "Refund policy",
            "How to request a refund",
            "Refund status tracking"
        ],
        "shipping": [
            "Shipping information",
            "Tracking your order",
            "International shipping rates"
        ]
    }
    
    # Find relevant articles
    results = []
    for keyword, docs in articles.items():
        if keyword.lower() in query.lower():
            results.extend(docs[:limit])
    
    return results if results else ["No articles found"]

# Test the tools
if __name__ == "__main__":
    # Test weather tool
    weather = get_weather.invoke({
        "city": "Paris",
        "units": "celsius"
    })
    print(f"Weather: {weather}")
    
    # Test KB search
    kb_results = search_knowledge_base.invoke({
        "query": "I forgot my password",
        "limit": 3
    })
    print(f"KB Results: {kb_results}")
```

### 2. Binding Tools to LLMs

#### Understanding bind_tools()

The `bind_tools()` method tells the LLM which tools are available:

Create `bind_tools_demo.py`:

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

load_dotenv()

# Step 1: Create tools
@tool
def get_order_status(order_id: str) -> str:
    """Get the status of a customer order."""
    # Simulated order data
    orders = {
        "ORD001": "Shipped - Arriving tomorrow",
        "ORD002": "Processing",
        "ORD003": "Delivered"
    }
    return orders.get(order_id, "Order not found")

@tool
def check_inventory(product: str) -> dict:
    """Check if a product is in stock."""
    inventory = {
        "laptop": {"in_stock": True, "quantity": 5},
        "phone": {"in_stock": True, "quantity": 12},
        "tablet": {"in_stock": False, "quantity": 0}
    }
    return inventory.get(product.lower(), {"in_stock": False, "quantity": 0})

# Step 2: Create LLM
model = ChatOpenAI(
    model="gpt-3.5-turbo-0125",
    temperature=0,
    api_key=os.getenv("OPENAI_API_KEY")
)

# Step 3: Bind tools to model
tools = [get_order_status, check_inventory]
model_with_tools = model.bind_tools(tools)

# Step 4: Test the binding
print("=== Testing Tool Binding ===\n")

# The model now "knows" about these tools
response = model_with_tools.invoke(
    "What's the status of order ORD001?"
)

print("LLM Response Type:", type(response))
print("LLM Response Content:", response.content)

# Check if model decided to call a tool
if response.tool_calls:
    print("\nTool Calls Detected:")
    for tool_call in response.tool_calls:
        print(f"  Tool: {tool_call['name']}")
        print(f"  Arguments: {tool_call['args']}")
else:
    print("\nNo tool calls detected - LLM generated text response")
```

**Output:**
```
=== Testing Tool Binding ===

LLM Response Type: <class 'langchain_core.messages.ai.AIMessage'>
LLM Response Content: 

Tool Calls Detected:
  Tool: get_order_status
  Arguments: {'order_id': 'ORD001'}
```

### 3. Building Your First Tool-Calling Agent

#### Agent Architecture

```
┌─────────────────────────────────────┐
│         User Request                 │
└────────────────────┬────────────────┘
                     │
        ┌────────────▼────────────┐
        │  LLM with Bound Tools   │
        │  (Decides to call tool) │
        └────────────┬────────────┘
                     │
        ┌────────────▼────────────────┐
        │  Extract Tool + Arguments   │
        └────────────┬────────────────┘
                     │
        ┌────────────▼────────────────┐
        │  Validate Arguments         │
        │  (Using Pydantic schema)    │
        └────────────┬────────────────┘
                     │
        ┌────────────▼────────────────┐
        │  Execute Tool               │
        └────────────┬────────────────┘
                     │
        ┌────────────▼────────────────┐
        │  Return Result to LLM       │
        └────────────┬────────────────┘
                     │
        ┌────────────▼────────────────┐
        │  LLM Synthesizes Answer     │
        └────────────┬────────────────┘
                     │
        ┌────────────▼────────────────┐
        │  Final Response to User     │
        └────────────────────────────┘
```

#### Creating a Tool-Calling Agent

Create `first_agent.py`:

```python
import os
import json
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain.prompts import ChatPromptTemplate
from langchain_core.tools import tool

load_dotenv()

# Step 1: Define tools
@tool
def get_customer_info(customer_id: str) -> dict:
    """Get customer information by ID."""
    customers = {
        "C001": {
            "name": "Alice Johnson",
            "email": "alice@example.com",
            "account_type": "Premium",
            "balance": 1500
        },
        "C002": {
            "name": "Bob Smith",
            "email": "bob@example.com",
            "account_type": "Standard",
            "balance": 500
        }
    }
    return customers.get(customer_id, {"error": "Customer not found"})

@tool
def process_refund(customer_id: str, amount: float, reason: str) -> dict:
    """Process a refund for a customer."""
    return {
        "status": "Success",
        "customer_id": customer_id,
        "refund_amount": amount,
        "reason": reason,
        "confirmation_number": "REF123456"
    }

@tool
def check_order_history(customer_id: str, limit: int = 5) -> list:
    """Get customer's order history."""
    orders = {
        "C001": [
            {"order_id": "ORD001", "date": "2025-01-10", "amount": 250},
            {"order_id": "ORD002", "date": "2025-01-05", "amount": 180},
        ],
        "C002": [
            {"order_id": "ORD003", "date": "2025-01-15", "amount": 450},
        ]
    }
    return orders.get(customer_id, [])

tools = [get_customer_info, process_refund, check_order_history]

# Step 2: Create LLM
llm = ChatOpenAI(
    model="gpt-3.5-turbo-0125",
    temperature=0,
    api_key=os.getenv("OPENAI_API_KEY")
)

# Step 3: Create prompt
prompt = ChatPromptTemplate.from_messages([
    (
        "system",
        """You are a helpful customer service AI assistant. 
You have access to tools to help customers with their inquiries.
Use the tools to gather information and assist customers.
Be polite and professional."""
    ),
    ("placeholder", "{chat_history}"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])

# Step 4: Create agent
agent = create_tool_calling_agent(llm, tools, prompt)

# Step 5: Create agent executor
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True  # Shows the agent's thinking
)

# Step 6: Test the agent
print("=== Customer Service Agent ===\n")

queries = [
    "What's the account status for customer C001?",
    "Process a refund of $100 for customer C001 because they're unhappy",
    "Show me the order history for customer C002"
]

for query in queries:
    print(f"Customer: {query}")
    result = agent_executor.invoke({"input": query})
    print(f"\nAgent: {result['output']}\n")
    print("-" * 60 + "\n")
```



## Week 4: Advanced Tool Integration

### 1. Error Handling in Tools

Tools can fail. Smart error handling is critical:

Create `error_handling_tools.py`:

```python
from pydantic import BaseModel, Field
from langchain_core.tools import tool
from langchain_core.exceptions import ToolException

class GetUserInput(BaseModel):
    user_id: str = Field(description="The user ID")

@tool(args_schema=GetUserInput, handle_tool_error=True)
def get_user_secure(user_id: str) -> dict:
    """
    Get user information safely with error handling.
    
    Handles:
    - Invalid user IDs
    - Database errors
    - Missing data
    """
    # Validate input
    if not user_id:
        raise ToolException("User ID cannot be empty")
    
    if not user_id.startswith("U"):
        raise ToolException("Invalid user ID format. Must start with 'U'")
    
    # Simulate database lookup
    database = {
        "U001": {"name": "Alice", "email": "alice@example.com"},
        "U002": {"name": "Bob", "email": "bob@example.com"},
    }
    
    if user_id not in database:
        raise ToolException(f"User {user_id} not found in database")
    
    return database[user_id]

# Test error handling
print("=== Error Handling Test ===\n")

# Test 1: Valid user
try:
    result = get_user_secure.invoke({"user_id": "U001"})
    print(f"Success: {result}")
except Exception as e:
    print(f"Error: {e}")

# Test 2: Invalid format
try:
    result = get_user_secure.invoke({"user_id": "123"})
    print(f"Success: {result}")
except Exception as e:
    print(f"Error: {e}")

# Test 3: User not found
try:
    result = get_user_secure.invoke({"user_id": "U999"})
    print(f"Success: {result}")
except Exception as e:
    print(f"Error: {e}")
```

### 2. Multiple Tool Calls and Sequencing

Tools often need to be called in sequence:

Create `sequential_tools.py`:

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain.prompts import ChatPromptTemplate
from langchain_core.tools import tool

load_dotenv()

# Create tools that build on each other
@tool
def search_inventory(product_name: str) -> dict:
    """Search inventory for product availability."""
    inventory = {
        "laptop": {"product_id": "P001", "in_stock": True},
        "mouse": {"product_id": "P002", "in_stock": False},
        "keyboard": {"product_id": "P003", "in_stock": True}
    }
    return inventory.get(product_name.lower(), {"product_id": None, "in_stock": False})

@tool
def get_product_details(product_id: str) -> dict:
    """Get detailed information about a product."""
    products = {
        "P001": {"name": "Laptop", "price": 1200, "warranty": "2 years"},
        "P002": {"name": "Mouse", "price": 25, "warranty": "1 year"},
        "P003": {"name": "Keyboard", "price": 75, "warranty": "2 years"}
    }
    return products.get(product_id, {})

@tool
def check_price_match(product_id: str, competitor_price: float) -> dict:
    """Check if we can match a competitor's price."""
    our_prices = {
        "P001": 1200,
        "P002": 25,
        "P003": 75
    }
    our_price = our_prices.get(product_id, 0)
    can_match = our_price >= competitor_price
    
    return {
        "our_price": our_price,
        "competitor_price": competitor_price,
        "can_match": can_match
    }

tools = [search_inventory, get_product_details, check_price_match]

llm = ChatOpenAI(
    model="gpt-3.5-turbo-0125",
    temperature=0,
    api_key=os.getenv("OPENAI_API_KEY")
)

prompt = ChatPromptTemplate.from_messages([
    ("system", """You are a product assistant. When users ask about products:
1. Search inventory first
2. Get detailed product info
3. Check prices if mentioned
4. Provide comprehensive recommendations"""),
    ("placeholder", "{chat_history}"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])

agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(agent=agent, tools=tools, verbose=True)

# Complex query requiring multiple tool calls
query = "Is the laptop in stock? If so, can you match a price of $1100?"
print(f"Query: {query}\n")
result = agent_executor.invoke({"input": query})
print(f"\nFinal Answer: {result['output']}")
```

### 3. Real-Time Data Integration

Integrate with actual APIs:

Create `api_integration_tools.py`:

```python
from pydantic import BaseModel, Field
from langchain_core.tools import tool
import requests
from typing import Optional

class GetWeatherInput(BaseModel):
    city: str = Field(description="City name")
    country: Optional[str] = Field(default=None, description="Country code (optional)")

@tool(args_schema=GetWeatherInput)
def get_real_weather(city: str, country: Optional[str] = None) -> dict:
    """
    Get real weather data from Open-Meteo API (free, no API key needed).
    """
    try:
        # Geocode the city
        geo_url = f"https://geocoding-api.open-meteo.com/v1/search"
        geo_params = {
            "name": city,
            "count": 1,
            "language": "en",
            "format": "json"
        }
        
        geo_response = requests.get(geo_url, params=geo_params, timeout=5)
        geo_data = geo_response.json()
        
        if not geo_data.get("results"):
            return {"error": f"City '{city}' not found"}
        
        location = geo_data["results"][0]
        latitude = location["latitude"]
        longitude = location["longitude"]
        
        # Get weather
        weather_url = "https://api.open-meteo.com/v1/forecast"
        weather_params = {
            "latitude": latitude,
            "longitude": longitude,
            "current": "temperature_2m,weather_code,wind_speed_10m",
            "temperature_unit": "celsius"
        }
        
        weather_response = requests.get(weather_url, params=weather_params, timeout=5)
        weather_data = weather_response.json()
        
        current = weather_data["current"]
        
        return {
            "city": city,
            "country": location.get("country"),
            "temperature": current["temperature_2m"],
            "wind_speed": current["wind_speed_10m"],
            "weather_code": current["weather_code"]
        }
        
    except requests.exceptions.RequestException as e:
        return {"error": f"Failed to fetch weather: {str(e)}"}

@tool
def get_exchange_rates(base_currency: str = "USD") -> dict:
    """Get current exchange rates (requires internet connection)."""
    try:
        url = f"https://api.exchangerate-api.com/v4/latest/{base_currency}"
        response = requests.get(url, timeout=5)
        data = response.json()
        
        return {
            "base": data["base"],
            "rates": data["rates"]
        }
    except requests.exceptions.RequestException as e:
        return {"error": f"Failed to fetch rates: {str(e)}"}

# Test the tools
if __name__ == "__main__":
    # Test weather
    weather = get_real_weather.invoke({"city": "London"})
    print("Weather in London:")
    print(f"  Temperature: {weather.get('temperature')}°C")
    print(f"  Wind Speed: {weather.get('wind_speed')} km/h")
    
    # Test exchange rates
    rates = get_exchange_rates.invoke({"base_currency": "USD"})
    print("\nUSD Exchange Rates:")
    if "rates" in rates:
        print(f"  USD to EUR: {rates['rates'].get('EUR')}")
        print(f"  USD to GBP: {rates['rates'].get('GBP')}")
```



## Real-World Projects

### Project 1: Customer Support Chatbot

Complete implementation of a real support system:

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor
from langchain.prompts import ChatPromptTemplate
from langchain_core.tools import tool
from pydantic import BaseModel, Field
from typing import Optional

load_dotenv()

# ============ TOOL DEFINITIONS ============

class OrderLookupInput(BaseModel):
    order_id: str = Field(description="Customer order ID")

class RefundInput(BaseModel):
    order_id: str = Field(description="Order ID for refund")
    reason: str = Field(description="Reason for refund")
    amount: Optional[float] = Field(default=None, description="Refund amount")

class KBSearchInput(BaseModel):
    query: str = Field(description="Search query")
    max_results: int = Field(default=3, description="Max results to return")

# Mock database
orders_db = {
    "ORD001": {"status": "Delivered", "amount": 99.99, "date": "2025-01-10"},
    "ORD002": {"status": "Shipped", "amount": 149.99, "date": "2025-01-12"},
    "ORD003": {"status": "Processing", "amount": 49.99, "date": "2025-01-15"},
}

kb_articles = {
    "shipping": [
        "Standard shipping takes 5-7 business days",
        "Express shipping takes 2-3 business days",
        "Free shipping on orders over $50"
    ],
    "refund": [
        "30-day money-back guarantee",
        "Full refund for unused items",
        "Refunds processed within 5-7 business days"
    ],
    "returns": [
        "Free returns within 30 days",
        "No questions asked return policy",
        "Use prepaid shipping label"
    ]
}

@tool(args_schema=OrderLookupInput)
def get_order_status(order_id: str) -> dict:
    """Get the status of a customer order."""
    if order_id not in orders_db:
        return {"error": f"Order {order_id} not found"}
    return {"order_id": order_id, **orders_db[order_id]}

@tool(args_schema=RefundInput)
def process_refund_request(order_id: str, reason: str, amount: Optional[float] = None) -> dict:
    """Process a refund request for a customer."""
    if order_id not in orders_db:
        return {"error": f"Order {order_id} not found"}
    
    order = orders_db[order_id]
    refund_amount = amount or order["amount"]
    
    if refund_amount > order["amount"]:
        return {"error": "Refund amount exceeds order total"}
    
    return {
        "status": "Approved",
        "order_id": order_id,
        "refund_amount": refund_amount,
        "reason": reason,
        "confirmation": f"REF{order_id}",
        "message": "Refund will be processed within 5-7 business days"
    }

@tool(args_schema=KBSearchInput)
def search_knowledge_base(query: str, max_results: int = 3) -> list:
    """Search company knowledge base for help articles."""
    results = []
    for category, articles in kb_articles.items():
        if any(word in query.lower() for word in category.split()):
            results.extend(articles[:max_results])
    return results if results else ["No articles found for your query"]

tools = [get_order_status, process_refund_request, search_knowledge_base]

# ============ AGENT SETUP ============

llm = ChatOpenAI(
    model="gpt-3.5-turbo-0125",
    temperature=0,
    api_key=os.getenv("OPENAI_API_KEY")
)

prompt = ChatPromptTemplate.from_messages([
    (
        "system",
        """You are a helpful customer support AI assistant. Your goals:
1. Help customers track orders
2. Process refunds when appropriate
3. Answer questions using the knowledge base
4. Be empathetic and professional

Guidelines:
- Always ask for order ID if not provided
- For refunds: gather details before processing
- Use knowledge base for common questions
- Escalate complex issues appropriately"""
    ),
    ("placeholder", "{chat_history}"),
    ("human", "{input}"),
    ("placeholder", "{agent_scratchpad}"),
])

agent = create_tool_calling_agent(llm, tools, prompt)
agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    verbose=True,
    max_iterations=5
)

# ============ TEST THE CHATBOT ============

if __name__ == "__main__":
    print("=== Customer Support AI ===\n")
    
    support_queries = [
        "Hi, I need to check on my order ORD001",
        "The product arrived damaged. I'd like a refund for order ORD002",
        "How long does shipping usually take?",
        "Can you process a $50 partial refund for order ORD003? It wasn't what I expected."
    ]
    
    for query in support_queries:
        print(f"Customer: {query}")
        result = agent_executor.invoke({"input": query})
        print(f"Support AI: {result['output']}\n")
        print("-" * 70 + "\n")
```



## Testing & Production

### 1. Unit Testing Tools

Create `test_tools.py`:

```python
import pytest
from unittest.mock import patch, MagicMock
from your_tools import (
    get_order_status,
    process_refund_request,
    search_knowledge_base
)

class TestOrderTools:
    
    def test_get_order_status_success(self):
        """Test successful order lookup."""
        result = get_order_status.invoke({"order_id": "ORD001"})
        assert result["order_id"] == "ORD001"
        assert "status" in result
    
    def test_get_order_status_not_found(self):
        """Test order not found."""
        result = get_order_status.invoke({"order_id": "NONEXISTENT"})
        assert "error" in result
    
    def test_process_refund_success(self):
        """Test successful refund processing."""
        result = process_refund_request.invoke({
            "order_id": "ORD001",
            "reason": "Defective item"
        })
        assert result["status"] == "Approved"
        assert "confirmation" in result
    
    def test_process_refund_exceeds_total(self):
        """Test refund amount validation."""
        result = process_refund_request.invoke({
            "order_id": "ORD001",
            "reason": "Testing",
            "amount": 999.99
        })
        assert "error" in result

# Run tests
# pytest test_tools.py -v
```

### 2. Agent Testing

Create `test_agent.py`:

```python
import pytest
from your_agent import agent_executor

class TestCustomerServiceAgent:
    
    def test_order_lookup_integration(self):
        """Test agent can look up orders."""
        result = agent_executor.invoke({
            "input": "What's the status of order ORD001?"
        })
        assert "Delivered" in result["output"].lower() or "order" in result["output"].lower()
    
    def test_knowledge_base_search(self):
        """Test agent searches knowledge base."""
        result = agent_executor.invoke({
            "input": "How long does shipping take?"
        })
        output_lower = result["output"].lower()
        assert "shipping" in output_lower or "business days" in output_lower
    
    def test_refund_processing(self):
        """Test agent processes refunds."""
        result = agent_executor.invoke({
            "input": "I want a refund for order ORD001 because it's defective"
        })
        assert "refund" in result["output"].lower() or "approved" in result["output"].lower()
```

### 3. Production Deployment Best Practices

```python
# ============ PRODUCTION-READY CONFIGURATION ============

import logging
from langchain.callbacks.tracers import LangChainTracer
from langchain_core.callbacks import BaseCallbackHandler

# Setup logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('agent.log'),
        logging.StreamHandler()
    ]
)
logger = logging.getLogger(__name__)

class ProductionConfig:
    """Production settings for the agent."""
    
    # Temperature: Lower for consistency
    TEMPERATURE = 0.1
    
    # Timeout for tool execution
    TOOL_TIMEOUT = 30
    
    # Max iterations to prevent infinite loops
    MAX_ITERATIONS = 10
    
    # Retry policy
    MAX_RETRIES = 3
    RETRY_DELAY = 1
    
    # Rate limiting
    MAX_REQUESTS_PER_MINUTE = 100
    
    # Error handling
    SILENT_FAILURES = False
    LOG_ALL_INTERACTIONS = True

# Usage
from langchain_openai import ChatOpenAI
from langchain.agents import create_tool_calling_agent, AgentExecutor

llm = ChatOpenAI(
    model="gpt-3.5-turbo-0125",
    temperature=ProductionConfig.TEMPERATURE,
    api_key=os.getenv("OPENAI_API_KEY")
)

agent = create_tool_calling_agent(llm, tools, prompt)

agent_executor = AgentExecutor(
    agent=agent,
    tools=tools,
    max_iterations=ProductionConfig.MAX_ITERATIONS,
    verbose=ProductionConfig.LOG_ALL_INTERACTIONS,
    handle_parsing_errors=True
)

# Monitoring
def log_agent_interaction(input_text, output_text, tools_used):
    """Log agent interactions for monitoring."""
    logger.info(f"Input: {input_text}")
    logger.info(f"Tools used: {tools_used}")
    logger.info(f"Output: {output_text}")

# Example usage with monitoring
result = agent_executor.invoke({"input": "Check order status"})
log_agent_interaction(
    input_text="Check order status",
    output_text=result['output'],
    tools_used=["get_order_status"]
)
```



## Summary: Week 3-4 Learning Path

### Week 3 Accomplishments
✅ Understood function calling and ReAct pattern  
✅ Created simple and structured tools  
✅ Bound tools to LLMs  
✅ Built your first tool-calling agent  
✅ Executed basic tool-based workflows  

### Week 4 Accomplishments
✅ Implemented error handling for tools  
✅ Created sequential multi-tool workflows  
✅ Integrated real-time APIs  
✅ Built production-ready chatbots  
✅ Implemented testing and monitoring  



## Key Takeaways

1. **LLMs Don't Execute Tools**: They only decide to call them
2. **Structured > Unstructured**: Always use Pydantic schemas
3. **ReAct Pattern**: Reason → Act → Observe → Loop
4. **Error Handling Matters**: Tools will fail; handle gracefully
5. **Sequential Calls**: Complex tasks need multiple tool calls
6. **Testing Required**: Validate agent behavior extensively
7. **Production Setup**: Use logging, monitoring, rate limiting
8. **Real APIs**: Integrate with actual business systems



## Real-World Impact

Companies using agent-based tool calling see:
- **40% reduction** in customer support response time
- **60% fewer** escalations to human support
- **24/7 availability** for routine tasks
- **Consistent** application of business rules
- **Audit trails** of all actions taken



## Next Steps

1. **Advanced Agents**: Multi-agent collaboration
2. **Tool Chaining**: Tools that call other tools
3. **State Management**: Persistent agent memory
4. **Async Tools**: Parallel execution
5. **Custom LLMs**: Fine-tune for specific domains



## Resources

- [LangChain Tool Documentation](https://docs.langchain.com/oss/python/langchain/tools)
- [OpenAI Function Calling Guide](https://platform.openai.com/docs/guides/function-calling)
- [LangGraph Documentation](https://langgraph.dev/)
- [Agent Best Practices](https://blog.langchain.com/tool-calling-with-langchain/)


Main Page: [AI Engineering Learning](/posts/ai-engineering-learning-1)

**Congratulations!** You now understand how to build sophisticated AI agents that can interact with real systems and APIs. The next step is building your own production application.