---
title: "AI Engineering Learning: LangGraph Workflows"
author: pravin_tripathi
date: 2025-10-16 00:00:00 +0530
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
1. [Introduction to LangGraph](#introduction-to-langgraph)
2. [Core Concepts & Theory](#core-concepts--theory)
3. [Week 4: Graph Fundamentals](#week-4-graph-fundamentals)
4. [Week 5: Advanced Workflows](#week-5-advanced-workflows)
5. [Real-World Projects](#real-world-projects)
6. [Production Deployment](#production-deployment)



## Introduction to LangGraph

### What is LangGraph?

**LangGraph** is a framework for building **stateful, controllable AI workflows**. It structures multi-step AI processes as directed graphs where:

- **Nodes** = Individual tasks/functions
- **Edges** = Connections between tasks defining execution flow
- **State** = Shared data flowing through the workflow
- **Conditional Logic** = Dynamic branching based on state

Think of LangGraph as a way to choreograph complex AI workflows the same way a conductor orchestrates an orchestra.

### LangGraph vs LangChain Chains

| Feature | Chains | LangGraph |
||--|--|
| **Flow Type** | Linear/Sequential | Graph with branching/loops |
| **State Management** | Limited | Full persistent state |
| **Conditions** | Simple if-else | Rich conditional routing |
| **Debugging** | Hard to trace | Built-in visualization |
| **Use Case** | Simple pipelines | Complex multi-step workflows |

### When to Use LangGraph

Use LangGraph when you need:
- ✅ Multiple steps with different logic
- ✅ Conditional branching based on intermediate results
- ✅ Loops and iterative processes
- ✅ Persistent state across steps
- ✅ Multi-agent orchestration
- ✅ Complex error handling

Don't use LangGraph when:
- ❌ Simple sequential chains (use LangChain chains)
- ❌ Single-step transformations
- ❌ Real-time streaming needed
- ❌ Ultra-high latency sensitivity

### Real-World Examples

**Document Processing Pipeline**
```
Upload Document → Classify → Extract Info → Validate → Store
        ↓                ↓
    Wrong Type?    Invalid Format?
        ↓                ↓
    Reject          Return to Extract
```

**Customer Support Workflow**
```
Receive Query → Classify Intent → Route to Specialist → 
    Response → Satisfied? → Store in KB / Escalate
```

**Content Creation**
```
Topic → Research → Draft → Review → Quality Check → 
    Needs Revision? → Refine → Publish
```



## Core Concepts & Theory

### 1. TypedDict - State Definition

**TypedDict** is a Python construct that defines a strongly-typed dictionary. LangGraph uses it to define what data flows through your graph.

#### Why TypedDict?

1. **Type Safety**: Catch errors early
2. **Documentation**: Clear what data exists
3. **Validation**: Ensure correct types at runtime
4. **IDE Support**: Better autocomplete

#### Basic Example

```python
from typing import TypedDict, Optional, List

class ChatState(TypedDict):
    """State flowing through the chat graph."""
    messages: List[str]          # List of conversation messages
    user_id: str                 # Current user
    context: Optional[str]       # Optional additional context
    confidence_score: float      # Float between 0-1
```

#### Important: TypedDict Structure

- **Keys** = Variable names that nodes can update
- **Values** = Type annotations (str, int, List, Optional, etc.)
- **Optional** = Can be None
- **All fields** = Must be provided in initial state (unless Optional)

#### With Reducers (Advanced)

Reducers combine values when multiple nodes write to the same key:

```python
from typing import TypedDict, Annotated
import operator

class GraphState(TypedDict):
    """State with reducer for messages list."""
    messages: Annotated[List[str], operator.add]  # Appends, doesn't overwrite
    count: int                                      # Normal overwrite
```

**What happens:**
- `messages`: New messages APPEND to the list
- `count`: New value REPLACES the old one

### 2. Nodes - The Tasks

A **Node** is a function that:
1. Takes current state as input
2. Performs some action
3. Returns a dictionary with updated state fields

#### Node Function Structure

```python
def my_node(state: GraphState) -> dict:
    """
    A node function.
    
    Args:
        state: Current graph state (TypedDict)
        
    Returns:
        Dictionary with state updates
    """
    # Read from state
    current_value = state["field_name"]
    
    # Do something
    result = process(current_value)
    
    # Return updates (only the changed fields)
    return {
        "result": result,
        "status": "completed"
    }
```

#### Key Rules

1. **Parameter**: State must be typed with TypedDict
2. **Return Type**: Dictionary (not TypedDict) - only changed fields
3. **State Immutability**: Don't modify state in-place, return new dict
4. **Side Effects**: Nodes can call APIs, LLMs, databases, etc.

#### Example Nodes

```python
class DocumentState(TypedDict):
    document: str
    classification: str
    extracted_info: dict
    validation_passed: bool

def classify_document_node(state: DocumentState) -> dict:
    """Classify document type."""
    doc = state["document"]
    
    # Classify logic
    if "invoice" in doc.lower():
        classification = "invoice"
    elif "contract" in doc.lower():
        classification = "contract"
    else:
        classification = "other"
    
    return {"classification": classification}

def extract_info_node(state: DocumentState) -> dict:
    """Extract information from document."""
    doc = state["document"]
    classification = state["classification"]
    
    # Extraction logic based on type
    if classification == "invoice":
        info = extract_invoice_fields(doc)
    elif classification == "contract":
        info = extract_contract_terms(doc)
    else:
        info = {}
    
    return {"extracted_info": info}

def validate_node(state: DocumentState) -> dict:
    """Validate extracted information."""
    info = state["extracted_info"]
    
    # Validation logic
    is_valid = len(info) > 0 and all_required_fields_present(info)
    
    return {"validation_passed": is_valid}
```

### 3. Edges - The Connections

Edges define how nodes connect and data flows between them.

#### Types of Edges

**1. Normal Edges (Always Connect)**
```python
workflow.add_edge("node_a", "node_b")  # Always go from A to B
```

**2. Conditional Edges (Smart Routing)**
```python
def route_function(state: GraphState) -> str:
    """Decide which node to go to next."""
    if state["confidence"] > 0.8:
        return "confident_path"
    else:
        return "uncertain_path"

workflow.add_conditional_edges(
    "analyze",                    # From this node
    route_function,              # Use this function to decide
    {
        "confident_path": "respond",
        "uncertain_path": "escalate"
    }
)
```

**3. Entry Points (Starting Node)**
```python
from langgraph.graph import START

workflow.add_edge(START, "first_node")  # Start here
```

**4. Exit Points (Ending Node)**
```python
from langgraph.graph import END

workflow.add_edge("final_node", END)    # End here
```

### 4. StateGraph - The Orchestra

**StateGraph** is the container that organizes nodes and edges into a complete workflow.

#### StateGraph Lifecycle

```
1. Create StateGraph with TypedDict
2. Add nodes (register functions)
3. Add edges (connect nodes)
4. Compile (validate graph structure)
5. Invoke (run the graph)
```

#### Basic Structure

```python
from langgraph.graph import StateGraph, START, END
from typing import TypedDict

# Step 1: Define state
class MyState(TypedDict):
    input: str
    output: str

# Step 2: Define nodes
def process_node(state: MyState) -> dict:
    return {"output": state["input"].upper()}

# Step 3: Create graph
workflow = StateGraph(MyState)

# Step 4: Add nodes
workflow.add_node("process", process_node)

# Step 5: Add edges
workflow.add_edge(START, "process")
workflow.add_edge("process", END)

# Step 6: Compile
app = workflow.compile()

# Step 7: Run
result = app.invoke({"input": "hello"})
print(result)  # {'input': 'hello', 'output': 'HELLO'}
```

### 5. Conditional Routing (The Brain of LangGraph)

Conditional edges are what make LangGraph powerful - they enable **dynamic, intelligent workflows**.

#### Simple Conditional Example

```python
def should_escalate(state: GraphState) -> str:
    """Decide whether to escalate to human."""
    confidence = state["confidence_score"]
    
    if confidence > 0.9:
        return "confident"
    elif confidence > 0.7:
        return "medium"
    else:
        return "escalate_to_human"

workflow.add_conditional_edges(
    "analyze",
    should_escalate,
    {
        "confident": "respond",
        "medium": "double_check",
        "escalate_to_human": "escalation_queue"
    }
)
```

#### Complex Conditional (Multiple Conditions)

```python
def route_based_on_state(state: GraphState) -> str:
    """Complex routing logic."""
    doc_type = state.get("document_type")
    validation_passed = state.get("validation_passed")
    priority = state.get("priority")
    
    # Multi-level decision
    if not validation_passed:
        return "fix_validation"
    elif priority == "high" and doc_type == "contract":
        return "review_with_expert"
    elif doc_type == "invoice":
        return "process_payment"
    else:
        return "archive"
```



## Week 4: Graph Fundamentals

### 1. Installation & Setup

```bash
pip install langgraph
pip install langchain-openai
pip install python-dotenv
```

### 2. Your First Graph

Create `first_graph.py`:

```python
import os
from dotenv import load_dotenv
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

load_dotenv()

# Step 1: Define state
class SimpleState(TypedDict):
    """State for simple workflow."""
    value: int

# Step 2: Define nodes
def increment_node(state: SimpleState) -> dict:
    """Increment the value."""
    return {"value": state["value"] + 1}

def double_node(state: SimpleState) -> dict:
    """Double the value."""
    return {"value": state["value"] * 2}

def print_node(state: SimpleState) -> dict:
    """Print the final value."""
    print(f"Final value: {state['value']}")
    return state

# Step 3: Build graph
workflow = StateGraph(SimpleState)

# Step 4: Add nodes
workflow.add_node("increment", increment_node)
workflow.add_node("double", double_node)
workflow.add_node("print", print_node)

# Step 5: Add edges (linear flow)
workflow.add_edge(START, "increment")
workflow.add_edge("increment", "double")
workflow.add_edge("double", "print")
workflow.add_edge("print", END)

# Step 6: Compile
app = workflow.compile()

# Step 7: Run
result = app.invoke({"value": 5})
print(f"Result: {result}")
# Output:
# Final value: 12
# Result: {'value': 12}
```

**What happens:**
```
5 → +1 → 6 → ×2 → 12 → Print → 12
```

### 3. Conditional Branching

Create `conditional_graph.py`:

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END

class ClassificationState(TypedDict):
    """State for classification workflow."""
    text: str
    category: str
    response: str

def classify_node(state: ClassificationState) -> dict:
    """Classify the input text."""
    text = state["text"].lower()
    
    if any(word in text for word in ["hello", "hi", "hey", "good morning"]):
        category = "greeting"
    elif any(word in text for word in ["help", "support", "issue", "problem"]):
        category = "support"
    elif any(word in text for word in ["price", "cost", "fee", "payment"]):
        category = "billing"
    else:
        category = "general"
    
    return {"category": category}

def greeting_handler(state: ClassificationState) -> dict:
    """Handle greeting."""
    return {"response": "Hello! How can I help you today?"}

def support_handler(state: ClassificationState) -> dict:
    """Handle support request."""
    return {"response": "I'll connect you with our support team. Please describe the issue."}

def billing_handler(state: ClassificationState) -> dict:
    """Handle billing question."""
    return {"response": "For billing inquiries, please check your account or contact billing@company.com"}

def general_handler(state: ClassificationState) -> dict:
    """Handle general inquiry."""
    return {"response": "Thank you for your inquiry. How can we assist you?"}

# Routing function
def route_by_category(state: ClassificationState) -> str:
    """Route to appropriate handler based on category."""
    return state["category"]

# Build graph
workflow = StateGraph(ClassificationState)

# Add nodes
workflow.add_node("classify", classify_node)
workflow.add_node("greeting", greeting_handler)
workflow.add_node("support", support_handler)
workflow.add_node("billing", billing_handler)
workflow.add_node("general", general_handler)

# Add edges
workflow.add_edge(START, "classify")

# Conditional edge from classify to handlers
workflow.add_conditional_edges(
    "classify",
    route_by_category,
    {
        "greeting": "greeting",
        "support": "support",
        "billing": "billing",
        "general": "general"
    }
)

# All handlers go to END
workflow.add_edge("greeting", END)
workflow.add_edge("support", END)
workflow.add_edge("billing", END)
workflow.add_edge("general", END)

# Compile and run
app = workflow.compile()

# Test different inputs
test_queries = [
    "Hello! How are you?",
    "I have a problem with my account",
    "What are your prices?",
    "Tell me about your services"
]

for query in test_queries:
    result = app.invoke({"text": query, "category": "", "response": ""})
    print(f"Query: {query}")
    print(f"Category: {result['category']}")
    print(f"Response: {result['response']}\n")
```

### 4. Graph with LLM

Create `llm_graph.py`:

```python
import os
from dotenv import load_dotenv
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

load_dotenv()

class LLMState(TypedDict):
    """State for LLM-based workflow."""
    topic: str
    draft: str
    review: str
    final_text: str

# Initialize LLM
llm = ChatOpenAI(
    model="gpt-3.5-turbo-0125",
    api_key=os.getenv("OPENAI_API_KEY")
)

def draft_node(state: LLMState) -> dict:
    """Generate initial draft."""
    prompt = ChatPromptTemplate.from_template(
        "Write a short paragraph about {topic}"
    )
    
    chain = prompt | llm
    response = chain.invoke({"topic": state["topic"]})
    
    return {"draft": response.content}

def review_node(state: LLMState) -> dict:
    """Review the draft."""
    prompt = ChatPromptTemplate.from_template(
        "Review this text and provide feedback: {draft}"
    )
    
    chain = prompt | llm
    response = chain.invoke({"draft": state["draft"]})
    
    return {"review": response.content}

def should_revise(state: LLMState) -> str:
    """Decide whether to revise."""
    review = state["review"].lower()
    
    if "need" in review or "should" in review or "improve" in review:
        return "revise"
    else:
        return "finalize"

def revise_node(state: LLMState) -> dict:
    """Revise based on feedback."""
    prompt = ChatPromptTemplate.from_template(
        """Revise this text based on the feedback:
        
Original: {draft}
Feedback: {review}

Revised text:"""
    )
    
    chain = prompt | llm
    response = chain.invoke({
        "draft": state["draft"],
        "review": state["review"]
    })
    
    return {"draft": response.content}

def finalize_node(state: LLMState) -> dict:
    """Finalize the text."""
    return {"final_text": state["draft"]}

# Build graph
workflow = StateGraph(LLMState)

# Add nodes
workflow.add_node("draft", draft_node)
workflow.add_node("review", review_node)
workflow.add_node("revise", revise_node)
workflow.add_node("finalize", finalize_node)

# Add edges
workflow.add_edge(START, "draft")
workflow.add_edge("draft", "review")
workflow.add_conditional_edges(
    "review",
    should_revise,
    {
        "revise": "revise",
        "finalize": "finalize"
    }
)
workflow.add_edge("revise", "review")  # Loop back for another review
workflow.add_edge("finalize", END)

# Compile and run
app = workflow.compile()

result = app.invoke({
    "topic": "artificial intelligence",
    "draft": "",
    "review": "",
    "final_text": ""
})

print("=== Writing Workflow Result ===")
print(f"Final Text:\n{result['final_text']}")
```



## Week 5: Advanced Workflows

### 1. Multi-Agent Coordination

Create `multi_agent_graph.py`:

```python
from typing import TypedDict, Annotated
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
import operator

class ResearchState(TypedDict):
    """State for multi-agent research workflow."""
    topic: str
    research_findings: Annotated[list, operator.add]  # Accumulate findings
    draft: str
    edited_draft: str
    final_content: str

llm = ChatOpenAI(model="gpt-3.5-turbo-0125")

def researcher_node(state: ResearchState) -> dict:
    """Research agent gathers information."""
    prompt = ChatPromptTemplate.from_template(
        "Research and provide key points about {topic}. Format as bullet points."
    )
    
    chain = prompt | llm
    response = chain.invoke({"topic": state["topic"]})
    
    # Append to research_findings (not replace)
    return {"research_findings": [response.content]}

def writer_node(state: ResearchState) -> dict:
    """Writer agent creates content."""
    findings = "\n".join(state["research_findings"])
    
    prompt = ChatPromptTemplate.from_template(
        """Based on these research findings, write an engaging article:
        
{findings}

Article:"""
    )
    
    chain = prompt | llm
    response = chain.invoke({"findings": findings})
    
    return {"draft": response.content}

def editor_node(state: ResearchState) -> dict:
    """Editor agent refines content."""
    prompt = ChatPromptTemplate.from_template(
        "Edit and improve this article for clarity and flow: {draft}"
    )
    
    chain = prompt | llm
    response = chain.invoke({"draft": state["draft"]})
    
    return {"edited_draft": response.content}

def finalize_node(state: ResearchState) -> dict:
    """Finalize the content."""
    return {"final_content": state["edited_draft"]}

# Build graph
workflow = StateGraph(ResearchState)

# Add all agent nodes
workflow.add_node("researcher", researcher_node)
workflow.add_node("writer", writer_node)
workflow.add_node("editor", editor_node)
workflow.add_node("finalize", finalize_node)

# Sequential workflow: Researcher → Writer → Editor → Finalize
workflow.add_edge(START, "researcher")
workflow.add_edge("researcher", "writer")
workflow.add_edge("writer", "editor")
workflow.add_edge("editor", "finalize")
workflow.add_edge("finalize", END)

# Compile and run
app = workflow.compile()

result = app.invoke({
    "topic": "machine learning",
    "research_findings": [],
    "draft": "",
    "edited_draft": "",
    "final_content": ""
})

print("=== Multi-Agent Content Creation ===")
print(f"Final Content:\n{result['final_content']}")
```

### 2. Document Processing Workflow

Create `document_workflow.py`:

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

class DocumentState(TypedDict):
    """State for document processing."""
    document: str
    doc_type: str
    extracted_data: dict
    validation_status: str
    processing_result: dict

llm = ChatOpenAI(model="gpt-3.5-turbo-0125")

def classify_document(state: DocumentState) -> dict:
    """Step 1: Classify document type."""
    prompt = ChatPromptTemplate.from_template(
        """Classify this document as: invoice, receipt, contract, report, or other.
        
Document: {doc}

Classification:"""
    )
    
    chain = prompt | llm
    response = chain.invoke({"doc": state["document"]})
    doc_type = response.content.strip()
    
    return {"doc_type": doc_type}

def extract_information(state: DocumentState) -> dict:
    """Step 2: Extract relevant information."""
    prompt = ChatPromptTemplate.from_template(
        """Extract key information from this {doc_type}:

Document: {doc}

Extract as JSON with appropriate fields."""
    )
    
    chain = prompt | llm
    response = chain.invoke({
        "doc_type": state["doc_type"],
        "doc": state["document"]
    })
    
    # Parse response as dict (simplified)
    return {"extracted_data": {"content": response.content}}

def validate_data(state: DocumentState) -> dict:
    """Step 3: Validate extracted data."""
    data = state["extracted_data"]
    
    # Simple validation
    if data and "content" in data:
        status = "valid"
    else:
        status = "invalid"
    
    return {"validation_status": status}

def should_process(state: DocumentState) -> str:
    """Decide whether to process or reject."""
    if state["validation_status"] == "valid":
        return "process"
    else:
        return "reject"

def process_document(state: DocumentState) -> dict:
    """Step 4: Process valid document."""
    return {
        "processing_result": {
            "status": "processed",
            "type": state["doc_type"],
            "data": state["extracted_data"]
        }
    }

def reject_document(state: DocumentState) -> dict:
    """Reject invalid document."""
    return {
        "processing_result": {
            "status": "rejected",
            "reason": "Validation failed"
        }
    }

# Build graph
workflow = StateGraph(DocumentState)

# Add nodes
workflow.add_node("classify", classify_document)
workflow.add_node("extract", extract_information)
workflow.add_node("validate", validate_data)
workflow.add_node("process", process_document)
workflow.add_node("reject", reject_document)

# Add edges: linear flow for valid path
workflow.add_edge(START, "classify")
workflow.add_edge("classify", "extract")
workflow.add_edge("extract", "validate")

# Conditional edge: process or reject
workflow.add_conditional_edges(
    "validate",
    should_process,
    {
        "process": "process",
        "reject": "reject"
    }
)

# Both paths go to END
workflow.add_edge("process", END)
workflow.add_edge("reject", END)

# Compile and run
app = workflow.compile()

# Test with sample document
sample_doc = """
INVOICE #12345
Date: 2025-01-15
Customer: John Doe
Items:
- Widget A: $50
- Widget B: $75
Total: $125
"""

result = app.invoke({
    "document": sample_doc,
    "doc_type": "",
    "extracted_data": {},
    "validation_status": "",
    "processing_result": {}
})

print("=== Document Processing Result ===")
print(f"Status: {result['processing_result']['status']}")
print(f"Type: {result['processing_result']['type']}")
```

### 3. Looping Workflows (Iterative Refinement)

Create `loop_workflow.py`:

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate

class RefinementState(TypedDict):
    """State for iterative refinement."""
    topic: str
    current_version: str
    feedback: str
    iteration_count: int
    final_version: str

llm = ChatOpenAI(model="gpt-3.5-turbo-0125")

def initial_draft(state: RefinementState) -> dict:
    """Create initial draft."""
    prompt = ChatPromptTemplate.from_template(
        "Write a short description of {topic}"
    )
    
    chain = prompt | llm
    response = chain.invoke({"topic": state["topic"]})
    
    return {
        "current_version": response.content,
        "iteration_count": 1
    }

def get_feedback(state: RefinementState) -> dict:
    """Get feedback on current version."""
    prompt = ChatPromptTemplate.from_template(
        """Review this text and provide specific feedback:

Text: {version}

Feedback:"""
    )
    
    chain = prompt | llm
    response = chain.invoke({"version": state["current_version"]})
    
    return {"feedback": response.content}

def should_continue_refining(state: RefinementState) -> str:
    """Decide whether to refine further."""
    # Stop after 3 iterations
    if state["iteration_count"] >= 3:
        return "finalize"
    # Check if feedback suggests improvements
    elif any(word in state["feedback"].lower() for word in ["improve", "could", "should", "better"]):
        return "refine"
    else:
        return "finalize"

def refine_version(state: RefinementState) -> dict:
    """Refine based on feedback."""
    prompt = ChatPromptTemplate.from_template(
        """Improve this text based on feedback:

Current: {version}
Feedback: {feedback}

Improved version:"""
    )
    
    chain = prompt | llm
    response = chain.invoke({
        "version": state["current_version"],
        "feedback": state["feedback"]
    })
    
    return {
        "current_version": response.content,
        "iteration_count": state["iteration_count"] + 1
    }

def finalize(state: RefinementState) -> dict:
    """Finalize the version."""
    return {"final_version": state["current_version"]}

# Build graph
workflow = StateGraph(RefinementState)

# Add nodes
workflow.add_node("draft", initial_draft)
workflow.add_node("feedback", get_feedback)
workflow.add_node("refine", refine_version)
workflow.add_node("finalize", finalize)

# Add edges with loop
workflow.add_edge(START, "draft")
workflow.add_edge("draft", "feedback")
workflow.add_conditional_edges(
    "feedback",
    should_continue_refining,
    {
        "refine": "refine",
        "finalize": "finalize"
    }
)

# Loop: refine → feedback → decide
workflow.add_edge("refine", "feedback")
workflow.add_edge("finalize", END)

# Compile and run
app = workflow.compile()

result = app.invoke({
    "topic": "quantum computing",
    "current_version": "",
    "feedback": "",
    "iteration_count": 0,
    "final_version": ""
})

print("=== Iterative Refinement Result ===")
print(f"Final Version:\n{result['final_version']}")
print(f"Total Iterations: {result['iteration_count']}")
```



## Real-World Projects

### Project 1: Resume Screening Pipeline

```python
# Complete resume screening workflow with classification,
# extraction, and decision making
class ResumeState(TypedDict):
    resume_text: str
    qualification_score: float
    extracted_skills: list
    recommendations: str
    final_decision: str

# Nodes:
# 1. Extract information (skills, experience)
# 2. Score qualifications
# 3. Check against job requirements
# 4. Generate recommendation (conditional routing)
# 5. Return decision

# Conditional routes:
# - Score > 0.8: "Accept"
# - Score > 0.5: "Review"
# - Score < 0.5: "Reject"
```

### Project 2: Support Ticket Routing

```python
# Multi-step ticket processing
class TicketState(TypedDict):
    ticket_text: str
    category: str
    priority: str
    assigned_team: str
    response_template: str

# Workflow:
# 1. Classify ticket category
# 2. Determine priority (conditional)
# 3. Route to appropriate team (conditional)
# 4. Generate response template
# 5. Update database
```

### Project 3: Content Moderation

```python
# Multi-level content review
class ModerationState(TypedDict):
    content: str
    safety_score: float
    violation_type: str
    action: str
    notification: str

# Workflow with multiple conditional branches:
# - Safe content → Approve
# - Questionable → Send for human review
# - Unsafe → Quarantine and notify user
```



## Production Deployment

### 1. Graph Visualization

```python
from langgraph.graph import StateGraph

app = workflow.compile()

# Generate visualization
image_data = app.get_graph().draw_mermaid_png()
with open("workflow_diagram.png", "wb") as f:
    f.write(image_data)
```

### 2. Debugging & Monitoring

```python
# Enable tracing
from langsmith.wrappers import wrap_openai

# Run with debug output
config = {"configurable": {"debug": True}}
result = app.invoke(state, config)

# Check execution steps
for event in app.stream(state):
    print(event)
```

### 3. Error Handling

```python
def safe_node(state: GraphState) -> dict:
    """Node with error handling."""
    try:
        result = risky_operation(state)
        return {"result": result, "error": None}
    except Exception as e:
        return {"result": None, "error": str(e)}

def handle_errors(state: GraphState) -> str:
    """Route based on errors."""
    if state.get("error"):
        return "error_handler"
    return "success_handler"
```



## Summary: Week 4-5 Learning

### Week 4 Accomplishments
✅ Understood StateGraph and TypedDict  
✅ Created nodes and edges  
✅ Implemented conditional branching  
✅ Built first graphs with LLMs  
✅ Mastered dynamic routing  

### Week 5 Accomplishments
✅ Created multi-agent workflows  
✅ Built complex document processing  
✅ Implemented iterative refinement loops  
✅ Coordinated multiple specialized agents  
✅ Deployed production-ready graphs  



## Key Takeaways

1. **TypedDict is Foundation**: Defines all data flowing through workflow
2. **Nodes are Functions**: Do work, return state updates
3. **Edges are Controllers**: Define flow between nodes
4. **Conditional Routing is Power**: Makes workflows intelligent
5. **State is Shared**: All nodes can read/update same state
6. **Loops Enable Refinement**: Iterative processes for quality
7. **Agents Collaborate**: Multi-agent systems solve complex problems
8. **Visualization Helps**: See your workflow structure clearly



## Real-World Impact

Organizations using LangGraph report:
- **50% reduction** in workflow development time
- **40% improvement** in error handling
- **30% faster** iteration cycles
- **Better maintainability** of complex processes
- **Clearer debugging** of AI workflows



## Next Steps

1. **Memory Management**: Add persistent state across sessions
2. **Checkpointing**: Save workflow progress
3. **Streaming**: Real-time output of graph execution
4. **Custom Nodes**: Integrate your own business logic
5. **Production Scaling**: Deploy multi-graph systems



## Resources

- [LangGraph Official Documentation](https://langgraph.dev/)
- [LangChain Documentation](https://docs.langchain.com/)
- [LangGraph GitHub Repository](https://github.com/langchain-ai/langgraph)
- [LangSmith for Debugging](https://smith.langchain.com/)

Main Page: [AI Engineering Learning](/posts/ai-engineering-learning-1)

**Congratulations!** You now understand how to build sophisticated, stateful AI workflows with LangGraph. You're ready to build complex production applications.