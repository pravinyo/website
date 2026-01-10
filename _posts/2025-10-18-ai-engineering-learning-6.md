---
title: "AI Engineering Learning: Memory Integration for LLMs"
author: pravin_tripathi
date: 2025-10-18 00:00:00 +0530
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
1. [Introduction to Memory Systems](#introduction-to-memory-systems)
2. [Core Concepts & Theory](#core-concepts--theory)
3. [Week 5: Short-Term Memory](#week-5-short-term-memory)
4. [Week 6: Long-Term Memory & Integration](#week-6-long-term-memory--integration)
5. [Real-World Projects](#real-world-projects)
6. [Production Implementation](#production-implementation)



## Introduction to Memory Systems

### What is Memory in AI?

**Memory** in AI systems is how applications **remember information across multiple conversations** to provide personalized, context-aware responses. Without memory, every conversation starts fresh—the AI forgets everything the user said before.

Think of it like the difference between:
- **Without memory**: Talking to a new person every single time
- **With memory**: Having an ongoing relationship where someone remembers your preferences

### Why Memory Matters

Traditional LLMs have zero memory:
```
Conversation 1:
User: "My name is Sarah"
AI: "Nice to meet you, Sarah!"

Conversation 2 (later):
User: "What's my name?"
AI: "I don't know your name. What is it?"
```

With memory systems:
```
Session Start → Store: "User name: Sarah"
Later Query: "What's my name?" 
→ Retrieve from memory → "Sarah"
```

### The Two Memory Types

| Aspect | Short-Term | Long-Term |
|--|--|-|
| **Duration** | Current conversation only | Across many conversations |
| **Storage** | In memory/state | Vector DB or external storage |
| **Data Type** | Message history | Extracted facts, preferences |
| **Example** | Last 10 messages | "User lives in NYC, likes coffee" |
| **Use Case** | Current chat context | User profile information |



## Core Concepts & Theory

### 1. Short-Term Memory (Conversation Context)

**Short-term memory** maintains the current conversation history so the AI can reference what was said earlier in this conversation.

#### How It Works

```
User Message 1: "I love Python"
  ↓
[Store in Memory]
  ↓
User Message 2: "What language do I love?"
  ↓
[Retrieve past messages]
  ↓
AI: "You love Python" ← Uses short-term memory
```

#### Memory Storage Approaches

**1. Buffer Memory (Store Everything)**
```python
# Entire conversation:
[
    "User: Hello",
    "AI: Hi there!",
    "User: I'm working on ML",
    "AI: That's interesting!",
    ...
]
```
- **Pro**: Complete context
- **Con**: Gets huge quickly, costs tokens

**2. Window Memory (Keep Last K Messages)**
```python
# Only last 4 messages:
[
    "User: I'm working on ML",
    "AI: That's interesting!",
    "User: Can you help?",
    "AI: Of course!"
]
```
- **Pro**: Bounded size, recent context
- **Con**: Loses older context

**3. Summary Memory (Compress History)**
```python
# Instead of storing messages, store summary:
Summary: "User is working on ML and wants help with Python"
```
- **Pro**: Compact, efficient
- **Con**: May lose details

**4. Hybrid Memory (Summary + Recent)**
```python
# Combine both:
- Summary of older messages
- Full recent messages (last 5)
```
- **Pro**: Balance of context and efficiency
- **Con**: More complex

### 2. Long-Term Memory (Knowledge Storage)

**Long-term memory** persists important information across different conversations and sessions.

#### What Gets Stored

1. **User Profile**: Name, preferences, history
2. **Facts**: "User worked at Google", "Prefers email communication"
3. **Preferences**: "Likes detailed answers", "Dislikes small talk"
4. **Episodic Memory**: "User had trouble with Docker last week"
5. **Semantic Memory**: General knowledge about the user's domain

#### Storage Methods

**Vector Store (Semantic Search)**
```
User: "I like hiking in mountains"
  ↓
[Extract fact and embed]
  ↓
Store in vector DB
  ↓
Later: "What are my hobbies?"
  ↓
[Search vector DB]
  ↓
"I like hiking in mountains"
```

**Knowledge Graph (Relationships)**
```
User (Alice) → works_at → Google
User (Alice) → likes_activity → Hiking
User (Alice) → prefers_communication → Email

Query: "Where does Alice work?"
Answer: Google (via relationship traversal)
```

**Database (Structured)**
```
User Table:
- ID: 1
- Name: Alice
- Company: Google
- Hobbies: Hiking, Photography
```

### 3. Entity Extraction (Pulling Meaning)

**Entity Extraction** automatically identifies important information from conversations to store as memories.

#### Process

```
Raw Conversation:
"I'm Alice, I work at Google in the ML team, 
and I love hiking on weekends"

↓ [LLM extracts entities]

Extracted Facts:
- Name: Alice
- Employer: Google
- Department: ML Team
- Hobby: Hiking
- Frequency: Weekends

↓ [Store these in long-term memory]
```

#### LangMem Framework

LangMem provides tools to:
1. Extract meaningful facts from conversations
2. Store them persistently
3. Update when information changes
4. Retrieve contextually

```python
from langmem import create_memory_manager

manager = create_memory_manager(
    model="gpt-4",
    instructions="Extract all important facts"
)

# Process conversation
facts = manager.invoke({"messages": conversation})
# Returns: [Extracted facts with IDs and timestamps]
```

### 4. ConversationalRetrievalChain

**ConversationalRetrievalChain** combines:
- **Retrieval**: Fetch relevant documents from vector store
- **Conversation Memory**: Maintain chat history
- **Generation**: Answer based on both

#### Architecture

```
User Query
    ↓
[1. Retrieve documents from vector store]
    ↓
[2. Fetch conversation memory]
    ↓
[3. Construct augmented prompt]
    ↓
Retrieved Docs + Chat History + Query
    ↓
[4. Send to LLM]
    ↓
Response
```



## Week 5: Short-Term Memory

### 1. Installation & Setup

```bash
pip install langchain
pip install langchain-openai
pip install langchain-community
pip install python-dotenv
```

### 2. ConversationBufferMemory (Store Everything)

Create `buffer_memory.py`:

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.memory import ConversationBufferMemory
from langchain.chains import ConversationChain
from langchain.prompts import PromptTemplate

load_dotenv()

# Step 1: Create LLM
llm = ChatOpenAI(
    model="gpt-3.5-turbo-0125",
    api_key=os.getenv("OPENAI_API_KEY")
)

# Step 2: Create buffer memory
memory = ConversationBufferMemory(
    memory_key="chat_history",
    return_messages=True  # Return as message objects
)

# Step 3: Create custom prompt
prompt_template = """
You are a helpful AI assistant. Use the conversation history 
to understand context and provide personalized responses.

Chat History:
{chat_history}

Current Input: {input}

Response:
"""

prompt = PromptTemplate(
    input_variables=["chat_history", "input"],
    template=prompt_template
)

# Step 4: Create conversation chain
conversation = ConversationChain(
    llm=llm,
    memory=memory,
    prompt=prompt,
    verbose=True
)

# Step 5: Test the conversation
print("=== Conversation with Buffer Memory ===\n")

responses = [
    "Hi, my name is Alice and I work in data science",
    "I've been learning Python for 2 years",
    "What's my name and what do I work on?",
    "Can you suggest projects for my field?",
    "Do you remember what I told you about Python?"
]

for user_input in responses:
    print(f"User: {user_input}")
    response = conversation.run(user_input=user_input)
    print(f"AI: {response}\n")

# See what's in memory
print("\n=== Memory Contents ===")
print(memory.buffer)
```

**Key Points:**
- Stores EVERY message
- Perfect for short conversations
- Gets expensive quickly (many tokens)
- Best for <100 message conversations

### 3. ConversationBufferWindowMemory (Keep Last K)

Create `window_memory.py`:

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.memory import ConversationBufferWindowMemory
from langchain.chains import ConversationChain
from langchain.prompts import PromptTemplate

load_dotenv()

llm = ChatOpenAI(model="gpt-3.5-turbo-0125")

# Window memory: Only keep last k=4 messages
memory = ConversationBufferWindowMemory(
    k=4,  # Keep only last 4 messages
    memory_key="chat_history",
    return_messages=True
)

prompt = PromptTemplate(
    input_variables=["chat_history", "input"],
    template="""
You are a helpful assistant.

Recent Conversation:
{chat_history}

User: {input}

Response:"""
)

conversation = ConversationChain(
    llm=llm,
    memory=memory,
    prompt=prompt
)

# Long conversation
print("=== Window Memory Demo (k=4) ===\n")

messages = [
    "Message 1: My name is Bob",
    "Message 2: I love coffee",
    "Message 3: I work at Microsoft",
    "Message 4: I speak Spanish",
    "Message 5: What's my name?",  # At this point, message 1 is dropped
    "Message 6: What do I do for work?",
    "Message 7: Do you remember my name?"  # Name is lost (message 1 was dropped)
]

for msg in messages:
    print(f"User: {msg}")
    response = conversation.run(input=msg)
    print(f"AI: {response}\n")
```

**Key Points:**
- Only remembers last K messages
- Bounded memory size
- Cheaper than buffer memory
- Good for ongoing conversations

### 4. ConversationSummaryMemory (Compress)

Create `summary_memory.py`:

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.memory import ConversationSummaryMemory
from langchain.chains import ConversationChain

load_dotenv()

llm = ChatOpenAI(model="gpt-3.5-turbo-0125")

# Summary memory: Compress conversation
memory = ConversationSummaryMemory(
    llm=llm,  # Uses LLM to create summaries
    memory_key="chat_history",
    return_messages=True
)

conversation = ConversationChain(
    llm=llm,
    memory=memory,
    verbose=True
)

# Test with long conversation
print("=== Summary Memory Demo ===\n")

long_conversation = [
    "My name is Charlie and I'm a software engineer",
    "I specialize in Python and JavaScript",
    "I've been coding for 10 years",
    "My favorite framework is React for frontend",
    "I also like Django for backend",
    "I work at a startup called TechCorp",
    "We build AI applications",
    "I lead a team of 5 engineers",
    "What have I told you so far?",
    "What's my name and what do I do?"
]

for msg in long_conversation:
    print(f"User: {msg}")
    response = conversation.run(input=msg)
    print(f"AI: {response}\n")

print("\n=== Memory Summary ===")
print(memory.buffer)
```

**Key Points:**
- Automatically compresses history
- Uses LLM for intelligent summarization
- Better for very long conversations
- May lose some details

### 5. ConversationSummaryBufferMemory (Hybrid)

Create `hybrid_memory.py`:

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.memory import ConversationSummaryBufferMemory
from langchain.chains import ConversationChain

load_dotenv()

llm = ChatOpenAI(model="gpt-3.5-turbo-0125")

# Hybrid: Keep recent messages + summary of older ones
memory = ConversationSummaryBufferMemory(
    llm=llm,
    max_token_limit=500,  # Keep recent 500 tokens, summarize rest
    memory_key="chat_history",
    return_messages=True
)

conversation = ConversationChain(
    llm=llm,
    memory=memory
)

print("=== Hybrid Memory Demo ===\n")

# Very long conversation
long_messages = [
    "My name is Diana",
    "I'm learning machine learning",
    "I completed an Andrew Ng course",
    "Now I'm working on a CV project",
    "I'm building an image classifier",
    "Using PyTorch as my framework",
    "I've spent 3 months on this project",
    "My accuracy is currently 92%",
    "I'm trying to improve to 95%",
    "What's my name and what project am I working on?"
]

for msg in long_messages:
    print(f"User: {msg}")
    response = conversation.run(input=msg)
    print(f"AI: {response}\n")
```

**Key Points:**
- Combines summaries with recent context
- Balances efficiency and recall
- Good for production systems
- Best for most use cases



## Week 6: Long-Term Memory & Integration

### 1. Vector Store for Long-Term Memory

Create `long_term_memory.py`:

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain.vectorstores import FAISS
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.schema import Document
from datetime import datetime

load_dotenv()

# Step 1: Create embedding model
embeddings = OpenAIEmbeddings(
    api_key=os.getenv("OPENAI_API_KEY")
)

# Step 2: Extract facts from conversation
extracted_facts = [
    "User's name is Eva",
    "Eva works at Netflix",
    "Eva is on the ML team",
    "Eva likes Python and Scala",
    "Eva prefers morning meetings",
    "Eva worked previously at Amazon",
]

# Step 3: Convert to documents with metadata
documents = []
for i, fact in enumerate(extracted_facts):
    doc = Document(
        page_content=fact,
        metadata={
            "fact_id": i,
            "timestamp": datetime.now().isoformat(),
            "user_id": "eva_001",
            "importance": "high"
        }
    )
    documents.append(doc)

# Step 4: Create vector store
vectorstore = FAISS.from_documents(
    documents,
    embeddings
)

# Save for later use
vectorstore.save_local("eva_memory")

# Step 5: Retrieve memories
print("=== Long-Term Memory Retrieval ===\n")

queries = [
    "Where does Eva work?",
    "What's Eva's name?",
    "What programming languages does Eva like?",
    "When does Eva prefer meetings?"
]

for query in queries:
    results = vectorstore.similarity_search(query, k=2)
    print(f"Query: {query}")
    for result in results:
        print(f"  Memory: {result.page_content}")
    print()
```

### 2. Entity Extraction with LLM

Create `entity_extraction.py`:

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI
from langchain.prompts import ChatPromptTemplate
from langchain_core.output_parsers import JsonOutputParser
from pydantic import BaseModel, Field
from typing import List

load_dotenv()

# Define entity schema
class UserEntity(BaseModel):
    name: str = Field(description="User's name")
    organization: str = Field(description="Company/organization")
    role: str = Field(description="Job title or role")
    skills: List[str] = Field(description="Technical skills")
    preferences: List[str] = Field(description="Communication or work preferences")

# Create extraction chain
llm = ChatOpenAI(model="gpt-3.5-turbo-0125")

prompt = ChatPromptTemplate.from_template("""
Extract entities from this conversation:

{conversation}

Return JSON with these fields: name, organization, role, skills, preferences
""")

parser = JsonOutputParser(pydantic_object=UserEntity)

chain = prompt | llm | parser

# Test extraction
conversation = """
User: Hi, I'm Frank. I work at Google in the Cloud team.
AI: Nice to meet you, Frank! What's your role?
User: I'm a Senior Software Engineer. I specialize in Kubernetes and Go.
AI: That's great!
User: I prefer written communication and async work.
"""

print("=== Entity Extraction ===\n")

result = chain.invoke({"conversation": conversation})

print(f"Name: {result.get('name')}")
print(f"Organization: {result.get('organization')}")
print(f"Role: {result.get('role')}")
print(f"Skills: {', '.join(result.get('skills', []))}")
print(f"Preferences: {', '.join(result.get('preferences', []))}")
```

### 3. ConversationalRetrievalChain

Create `conversational_retrieval.py`:

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain.vectorstores import FAISS
from langchain.chains import ConversationalRetrievalChain
from langchain.memory import ConversationBufferMemory
from langchain.text_splitter import RecursiveCharacterTextSplitter
from langchain.schema import Document

load_dotenv()

# Step 1: Create knowledge base (documents to retrieve from)
knowledge_base_text = """
Product A: $50, available in blue and red
Product B: $100, has warranty for 2 years
Product C: $200, includes free shipping
Our return policy: 30 days for all products
Warranty covers manufacturing defects only
Customer support: support@company.com or call 1-800-HELP
"""

# Split into chunks
splitter = RecursiveCharacterTextSplitter(
    chunk_size=100,
    chunk_overlap=20
)
documents = [Document(page_content=chunk) for chunk in splitter.split_text(knowledge_base_text)]

# Step 2: Create vector store
embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_documents(documents, embeddings)
retriever = vectorstore.as_retriever()

# Step 3: Create memory
memory = ConversationBufferMemory(
    memory_key="chat_history",
    return_messages=True
)

# Step 4: Create LLM
llm = ChatOpenAI(model="gpt-3.5-turbo-0125")

# Step 5: Create conversational retrieval chain
qa_chain = ConversationalRetrievalChain.from_llm(
    llm=llm,
    retriever=retriever,
    memory=memory,
    verbose=True
)

# Step 6: Test the chain
print("=== Conversational Retrieval Chain ===\n")

queries = [
    "What products do you have?",
    "What's the price of Product B?",
    "Does it have warranty?",
    "How long is your return policy?",
    "What's your support phone number?"
]

for query in queries:
    print(f"Customer: {query}")
    response = qa_chain.run(query)
    print(f"Support Bot: {response}\n")
```



## Real-World Projects

### Project 1: Customer Service Bot with Memory

```python
# Complete customer service implementation featuring:
# - Short-term memory for current interaction
# - Long-term memory for customer profile
# - Order history retrieval
# - Personalized responses

class CustomerServiceBot:
    def __init__(self):
        self.short_term = ConversationBufferWindowMemory(k=10)
        self.vectorstore = load_customer_profiles()
        self.order_db = load_orders()
    
    def respond(self, customer_id, query):
        # 1. Get customer profile
        profile = self.vectorstore.search(customer_id)
        
        # 2. Get conversation context
        context = self.short_term.load_memory_variables({})
        
        # 3. Get relevant orders
        orders = self.order_db.get_recent(customer_id)
        
        # 4. Generate response using all context
        response = llm.generate(
            profile=profile,
            context=context,
            orders=orders,
            query=query
        )
        
        return response
```

### Project 2: Personal Assistant with LongTerm Memory

```python
# AI assistant that remembers:
# - User preferences
# - Past interactions
# - Tasks and goals
# - Important dates

class PersonalAssistant:
    def __init__(self):
        self.memory_manager = create_memory_manager()
        self.vectorstore = VectorStore()
    
    def process_conversation(self, messages):
        # Extract important facts
        extracted = self.memory_manager.invoke({
            "messages": messages,
            "instructions": "Extract user preferences, goals, and important info"
        })
        
        # Store facts in vector store
        for fact in extracted:
            self.vectorstore.add(fact)
        
        # Generate personalized response
        return self.generate_response()
```

### Project 3: Support Ticket Router with Memory

```python
# Route tickets while remembering:
# - Customer history
# - Previous issues
# - Resolution approaches
# - Priority patterns

class TicketRouter:
    def __init__(self):
        self.short_term = ConversationSummaryBufferMemory()
        self.customer_history = load_history_vectorstore()
        self.issue_patterns = load_patterns()
    
    def route_ticket(self, ticket):
        # Get customer history
        history = self.customer_history.search(ticket.customer_id)
        
        # Get conversation context
        context = self.short_term.load_memory_variables({})
        
        # Decide routing
        route = self.decide_route(
            ticket=ticket,
            history=history,
            context=context
        )
        
        return route
```



## Production Implementation

### 1. Persistent Storage

```python
from langchain.storage import LocalFileStore
from langchain.schema.messages import BaseMessage

# Store memory to disk
file_store = LocalFileStore(".langchain_storage")

# Save conversation after each turn
def save_conversation(user_id, messages):
    key = f"conversation_{user_id}"
    file_store.mset([(key, str(messages))])

# Load conversation for returning user
def load_conversation(user_id):
    key = f"conversation_{user_id}"
    return file_store.mget([key])
```

### 2. Memory Pruning

```python
def prune_old_memories(vectorstore, days_old=30):
    """Remove memories older than X days."""
    from datetime import datetime, timedelta
    
    cutoff = datetime.now() - timedelta(days=days_old)
    
    # Get all memories with timestamps
    memories = vectorstore.get_all()
    
    # Remove old ones
    for memory in memories:
        timestamp = datetime.fromisoformat(
            memory.metadata["timestamp"]
        )
        if timestamp < cutoff:
            vectorstore.delete([memory.id])
```

### 3. Memory Consolidation

```python
def consolidate_memories(manager, memories):
    """Merge redundant or contradictory memories."""
    
    # Identify similar memories
    similar_groups = cluster_by_similarity(memories)
    
    # Consolidate each group
    consolidated = []
    for group in similar_groups:
        merged = manager.consolidate_memories(group)
        consolidated.append(merged)
    
    return consolidated
```

### 4. Privacy & Security

```python
def anonymize_memory(memory_text):
    """Remove sensitive information."""
    import re
    
    # Remove emails
    memory_text = re.sub(r'\S+@\S+', '[EMAIL]', memory_text)
    
    # Remove phone numbers
    memory_text = re.sub(r'\d{3}-\d{3}-\d{4}', '[PHONE]', memory_text)
    
    # Remove SSN
    memory_text = re.sub(r'\d{3}-\d{2}-\d{4}', '[SSN]', memory_text)
    
    return memory_text

def encrypt_memory(memory, encryption_key):
    """Encrypt sensitive memories."""
    from cryptography.fernet import Fernet
    
    cipher = Fernet(encryption_key)
    encrypted = cipher.encrypt(memory.encode())
    return encrypted
```



## Summary: Week 5-6 Learning

### Week 5 Accomplishments
✅ Understood memory types and use cases  
✅ Implemented ConversationBufferMemory  
✅ Implemented ConversationBufferWindowMemory  
✅ Used ConversationSummaryMemory  
✅ Combined with hybrid approaches  
✅ Created conversational chains with memory  

### Week 6 Accomplishments
✅ Implemented long-term memory with vector stores  
✅ Extracted entities from conversations  
✅ Used LangMem for automated extraction  
✅ Created ConversationalRetrievalChain  
✅ Integrated short and long-term memory  
✅ Built personalized conversational systems  



## Key Takeaways

1. **Memory Type Selection**: Choose based on conversation length and token budget
2. **Short vs Long-Term**: Use both for complete personalization
3. **Entity Extraction**: Automate fact extraction with LLMs
4. **Vector Stores**: Persist knowledge for retrieval
5. **ConversationalRetrievalChain**: Powerful combination pattern
6. **Privacy**: Always anonymize and encrypt sensitive data
7. **Pruning**: Remove old/irrelevant memories regularly
8. **Consolidation**: Merge redundant information



## Real-World Impact

Organizations implementing memory systems report:
- **40% improvement** in customer satisfaction
- **50% reduction** in repeat questions
- **35% faster** issue resolution
- **Better personalization** leading to higher engagement
- **Reduced support load** through better context



## Next Steps

1. **Multi-User Systems**: Manage memory for many users
2. **Cross-Session Memory**: Remember between app restarts
3. **Forgetting Mechanisms**: Intelligently remove outdated info
4. **Memory Analytics**: Track what's remembered
5. **Integration with Databases**: Connect to production systems



## Resources

- [LangChain Memory Documentation](https://docs.langchain.com/oss/python/langchain/modules/memory)
- [LangMem Documentation](https://langchain-ai.github.io/langmem/)
- [Conversational Retrieval Guide](https://docs.langchain.com/oss/python/langchain/rag)
- [Vector Store Options](https://www.pinecone.io/learn/vector-database/)

Main Page: [AI Engineering Learning](/posts/ai-engineering-learning-1)

**Congratulations!** You now understand how to build intelligent AI systems that remember, learn, and provide personalized experiences. You're ready to create production applications with sophisticated memory management.