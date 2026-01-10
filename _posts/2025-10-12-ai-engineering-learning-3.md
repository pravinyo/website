---
title: "AI Engineering Learning: RAG Implementation"
author: pravin_tripathi
date: 2025-10-12 00:00:00 +0530
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
1. [Introduction to RAG](#introduction-to-rag)
2. [Core Concepts & Theory](#core-concepts--theory)
3. [Week 2: RAG Fundamentals](#week-2-rag-fundamentals)
4. [Week 3: Advanced RAG Implementation](#week-3-advanced-rag-implementation)
5. [Practical Projects](#practical-projects)
6. [Debugging & Optimization](#debugging--optimization)

## Introduction to RAG

### What is Retrieval Augmented Generation (RAG)?

**Retrieval Augmented Generation (RAG)** is a technique that enhances Large Language Models by integrating them with external data sources. Instead of relying only on the knowledge the model was trained on, RAG allows models to:

1. **Retrieve** relevant information from external knowledge bases
2. **Augment** the user's query with this retrieved context
3. **Generate** accurate responses using both the retrieved context and the LLM's training knowledge

Think of RAG as giving your AI assistant access to a reference library. When you ask a question, the assistant first looks up the relevant books (retrieval), then uses that information (augmentation) to answer your question (generation).

### Why RAG Matters

Traditional LLMs suffer from several limitations that RAG solves:

| Problem | RAG Solution |
||-|
| **Hallucinations**: Making up false information | Retrieved documents provide grounded truth |
| **Outdated Information**: Knowledge cutoff in training data | Can query latest data from knowledge bases |
| **Private Data**: Can't access company-specific documents | Can retrieve from private/internal sources |
| **Context Window Limits**: Can't process very large documents | Retrieves only relevant chunks |
| **Lack of Citations**: Hard to trace where answers come from | Can cite source documents |

### Real-World Use Cases

**Document Processing Systems**
- Company policies chatbots
- Technical manual Q&A systems
- Legal document analysis

**Customer Support**
- Support ticket auto-responses using company docs
- Product knowledge bases
- FAQ systems

**Enterprise Knowledge Management**
- Internal documentation search
- Research paper analysis
- Medical record analysis



## Core Concepts & Theory

### 1. Vector Embeddings - The Foundation

#### What Are Embeddings?

**Embeddings** are numerical representations (vectors) of text that capture semantic meaning. Instead of storing text as raw strings, we convert it to arrays of numbers that machines can process.

**Simple Example:**
```
Text: "dog"
Embedding: [0.25, 0.89, -0.12, 0.45, 0.78, ...]  (typically 300-1536 dimensions)

Text: "puppy"
Embedding: [0.27, 0.85, -0.10, 0.48, 0.75, ...]  (similar to "dog")
```

#### Key Properties

1. **Semantic Similarity**: Words with similar meanings have embeddings close together in vector space
   - "king" and "queen" are close
   - "dog" and "cat" are close
   - "dog" and "car" are far apart

2. **Dimensionality**: 
   - Smaller models: 300 dimensions
   - Modern models (OpenAI): 1,536 dimensions
   - Large models: Up to 4,096 dimensions

3. **Standardization**: Must use the same embedding model for both documents and queries

#### Embedding Models to Know

| Model | Dimensions | Pros | Cons |
|-|--|||
| **OpenAI text-embedding-3-small** | 1,536 | Best quality, production-ready | Paid API |
| **HuggingFace BAAI/bge-small-en-v1.5** | 384 | Free, fast, good quality | Open-source dependencies |
| **Sentence-BERT** | 384 | Fast, lightweight | Less accurate than large models |

### 2. Vector Similarity Metrics

#### Cosine Similarity (Most Common)

**Cosine Similarity** measures the angle between two vectors. It ranges from -1 to 1:
- **1.0** = vectors point in same direction (identical meaning)
- **0.0** = vectors are perpendicular (no relation)
- **-1.0** = vectors point in opposite directions

**Formula:**
```
similarity = (A · B) / (||A|| × ||B||)

Where:
A · B = dot product (multiply matching dimensions, add results)
||A|| and ||B|| = magnitude of vectors
```

**Why It's Perfect for RAG:**
- Computationally efficient
- Works well with high-dimensional data
- Only cares about direction, not magnitude
- Ideal for semantic search

**Example:**
```
Query: "How do I reset my password?"
Document 1: "Password reset instructions: Click forgot password..."  → Similarity: 0.92 ✓
Document 2: "Account security best practices..."                   → Similarity: 0.45 ✗
Document 3: "Password history and policies..."                    → Similarity: 0.68 ✓

System retrieves documents 1 and 3 (top matches)
```

#### Other Metrics

**Euclidean Distance (L2)**
- Measures straight-line distance between vectors
- Considers both direction AND magnitude
- Better when vector magnitude matters

**Dot Product**
- Fast computation
- Works well with normalized vectors
- Less interpretable than cosine

### 3. Document Chunking

#### Why Chunk Documents?

Raw documents (100+ pages) are too large for:
- Embedding models (context window limits)
- LLM context windows
- Semantic search accuracy

**Solution**: Split into smaller, manageable chunks while maintaining semantic meaning.

#### Chunking Strategies

**Strategy 1: Fixed-Size Chunking**
```
Chunk Size: 1000 characters
Overlap: 100 characters

Original: "Machine learning is... [very long text]"
Chunk 1: "Machine learning is... [first 1000 chars]"
Chunk 2: "[last 100 chars of chunk 1]...next content..."
Chunk 3: "[last 100 chars of chunk 2]...final content"
```
- Pros: Simple, fast
- Cons: Cuts mid-sentence, loses semantic meaning

**Strategy 2: Recursive Chunking (Recommended)**
```
Try separators in order:
1. Double newline (\n\n) → Paragraphs
2. Single newline (\n) → Sentences
3. Space ( ) → Words
4. Character level

Stop when chunk_size is reached
```
- Pros: Respects document structure, maintains semantics
- Cons: Slightly slower than fixed-size

**Strategy 3: Semantic Chunking (Advanced)**
```
1. Break text into sentences
2. Create embeddings for each sentence
3. Detect semantic breakpoints (where topic changes)
4. Group sentences before breakpoint into chunks
```
- Pros: Most semantically coherent
- Cons: Expensive (requires many embeddings)

#### Best Practices

- **Chunk Size**: 250-1000 tokens (document and use case dependent)
- **Overlap**: 10-50 tokens (prevents losing context at boundaries)
- **Document Type**: Use specialized splitters for code, Markdown, HTML

### 4. The RAG Pipeline

#### Three Main Phases

**Phase 1: Indexing (Offline)**
```
Raw Documents
    ↓
[Load Documents]
    ↓
[Split into Chunks]
    ↓
[Generate Embeddings]  ← Use embedding model
    ↓
[Store in Vector DB]   ← FAISS, Pinecone, etc.
```

**Phase 2: Retrieval (Runtime)**
```
User Query
    ↓
[Generate Query Embedding]  ← Same embedding model as indexing
    ↓
[Vector Similarity Search]  ← Find top-k similar chunks
    ↓
[Retrieved Chunks]
```

**Phase 3: Generation (Runtime)**
```
User Query + Retrieved Chunks
    ↓
[Augment Prompt]  ← Combine query with context
    ↓
[Send to LLM]
    ↓
[Generate Response]
    ↓
Response (grounded in retrieved documents)
```



## Week 2: RAG Fundamentals

### 1. Vector Databases Comparison

#### FAISS (Facebook AI Similarity Search)

**What it is:** Open-source library for efficient similarity search

**Best for:** Learning, local development, small projects

**Pros:**
- Free and open-source
- Blazing fast in-memory search
- Great for experimentation
- No setup complexity

**Cons:**
- No persistence (manual save/load)
- Single machine only
- Not production-ready for distributed systems

**Install:**
```bash
pip install faiss-cpu
# Or GPU version: pip install faiss-gpu
```

#### Pinecone

**What it is:** Fully managed cloud vector database

**Best for:** Production applications, enterprise use

**Pros:**
- Automatic persistence and backups
- Seamless scaling
- Beautiful dashboard
- Enterprise security (SOC 2, HIPAA)
- Real-time index updates

**Cons:**
- Paid service (free tier limited)
- Internet dependency
- API rate limits

**Install:**
```bash
pip install pinecone-client
```

#### Chroma

**What it is:** Open-source vector database designed for LLM applications

**Best for:** Easy prototyping, local development

**Pros:**
- Simple API
- Local-first approach
- SQLite persistence
- Perfect for small projects

**Cons:**
- Less scalable than Pinecone
- Limited indexing options

**When to Use Each:**
- **FAISS**: Jupyter notebooks, prototyping, learning
- **Chroma**: Local projects with persistence needs
- **Pinecone**: Production systems, enterprise apps

### 2. Setting Up Your First RAG System

#### Step 1: Install Dependencies

```bash
pip install langchain==0.1.0
pip install langchain-openai==0.0.2
pip install langchain-community==0.0.12
pip install faiss-cpu
pip install openai==1.7.2
pip install python-dotenv
pip install pypdf  # For PDF loading
```

#### Step 2: Document Preparation

Create `prepare_documents.py`:

```python
import os
from dotenv import load_dotenv
from langchain.document_loaders import TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter

load_dotenv()

# Step 1: Load document
loader = TextLoader("sample_document.txt")
documents = loader.load()

# Step 2: Split into chunks
text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,        # Max 1000 characters per chunk
    chunk_overlap=100,      # 100 characters overlap between chunks
    length_function=len,
    is_separator_regex=False
)

splits = text_splitter.split_documents(documents)

print(f"Total chunks created: {len(splits)}")
print(f"\nFirst chunk:")
print(splits[0].page_content[:200])
print(f"Metadata: {splits[0].metadata}")
```

#### Step 3: Create Embeddings

Create `create_embeddings.py`:

```python
import os
from dotenv import load_dotenv
from langchain.document_loaders import TextLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import OpenAIEmbeddings
from langchain.vectorstores import FAISS

load_dotenv()

# Load and split documents (same as before)
loader = TextLoader("sample_document.txt")
documents = loader.load()

text_splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=100
)
splits = text_splitter.split_documents(documents)

# Step 1: Create embedding model
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small",
    api_key=os.getenv("OPENAI_API_KEY")
)

# Step 2: Create vector store
vectorstore = FAISS.from_documents(splits, embeddings)

# Step 3: Save vector store for later use
vectorstore.save_local("faiss_index")

print("✓ Vectorstore created and saved!")
print(f"Total vectors stored: {len(splits)}")
```

#### Step 4: Query the Vector Store

Create `query_vectorstore.py`:

```python
import os
from dotenv import load_dotenv
from langchain_openai import OpenAIEmbeddings
from langchain.vectorstores import FAISS

load_dotenv()

# Load the saved vectorstore
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small",
    api_key=os.getenv("OPENAI_API_KEY")
)

vectorstore = FAISS.load_local("faiss_index", embeddings, allow_dangerous_deserialization=True)

# Step 1: Query the vectorstore
query = "What is machine learning?"
results = vectorstore.similarity_search(query, k=3)

# Step 2: Display results
print(f"Query: {query}\n")
for i, doc in enumerate(results, 1):
    print(f"\n Result {i} ")
    print(f"Content: {doc.page_content[:200]}...")
    print(f"Score: High")
```

### 3. Building Your First RAG Chain

Create `first_rag_chain.py`:

```python
import os
from dotenv import load_dotenv
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain.vectorstores import FAISS
from langchain.prompts import ChatPromptTemplate
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain.chains import create_retrieval_chain
from langchain_core.output_parsers import StrOutputParser

load_dotenv()

# Step 1: Load vector store and create retriever
embeddings = OpenAIEmbeddings(
    model="text-embedding-3-small",
    api_key=os.getenv("OPENAI_API_KEY")
)

vectorstore = FAISS.load_local(
    "faiss_index",
    embeddings,
    allow_dangerous_deserialization=True
)

retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

# Step 2: Define prompt template
system_prompt = """You are a helpful assistant answering questions based on provided context.

Use the following context to answer the question:
{context}

If the context doesn't contain relevant information, say "I don't have this information in my knowledge base."
"""

prompt = ChatPromptTemplate.from_messages([
    ("system", system_prompt),
    ("human", "{input}")
])

# Step 3: Create LLM
llm = ChatOpenAI(
    model="gpt-3.5-turbo-0125",
    temperature=0,
    api_key=os.getenv("OPENAI_API_KEY")
)

# Step 4: Create document chain (combines docs with prompt)
document_chain = create_stuff_documents_chain(llm, prompt)

# Step 5: Create retrieval chain (combines retriever with document chain)
retrieval_chain = create_retrieval_chain(retriever, document_chain)

# Step 6: Run the chain
query = "What is the main topic of the document?"
response = retrieval_chain.invoke({"input": query})

print(f"Question: {query}\n")
print(f"Answer: {response['answer']}\n")
print(" Retrieved Context ")
for i, doc in enumerate(response['context'], 1):
    print(f"\nSource {i}:")
    print(doc.page_content[:300] + "...")
```

#### Understanding the Chain Components

```
┌─────────────────────────────────────────────────────────┐
│           User Query: "What is X?"                      │
└────────────────────┬────────────────────────────────────┘
                     │
         ┌───────────▼───────────┐
         │ RETRIEVER             │ (Uses vector similarity)
         │ - Vectorizes query    │
         │ - Searches top-k docs │
         └───────────┬───────────┘
                     │
         ┌───────────▼───────────────────────┐
         │ DOCUMENT CHAIN                    │
         │ - Formats retrieved docs          │
         │ - Creates augmented prompt        │
         │ - Sends to LLM                    │
         └───────────┬───────────────────────┘
                     │
         ┌───────────▼───────────┐
         │ LLM (GPT-3.5-turbo)   │
         │ - Generates response  │
         │ - Uses context        │
         └───────────┬───────────┘
                     │
      ┌──────────────▼──────────────┐
      │ Response + Source Citations │
      └─────────────────────────────┘
```

### 4. Understanding Context Augmentation

#### What is Context Augmentation?

**Context Augmentation** is the process of enhancing the user's query with retrieved documents before sending it to the LLM.

**Before Augmentation:**
```
Prompt to LLM: "What is quantum computing?"
(LLM uses only its training knowledge)
```

**After Augmentation (RAG):**
```
Prompt to LLM:
"Use the following context to answer the question:

CONTEXT:
Quantum computing is a form of computation that harnesses quantum 
mechanical phenomena such as superposition and entanglement...
[more retrieved text]

QUESTION: What is quantum computing?
"
(LLM uses both training knowledge AND provided context)
```

#### How LangChain Does This

The `create_stuff_documents_chain` function automatically:
1. Takes retrieved documents
2. Combines them into a single context string
3. Inserts into prompt template
4. Sends to LLM

```python
# This happens automatically:
augmented_prompt = f"""You are helpful assistant.

Context:
{retrieved_doc_1.page_content}

{retrieved_doc_2.page_content}

{retrieved_doc_3.page_content}

User Question: {query}

Answer:"""
```



## Week 3: Advanced RAG Implementation

### 1. Advanced Document Loading

#### Loading Different File Types

```python
from langchain.document_loaders import (
    PyPDFLoader,
    TextLoader,
    CSVLoader,
    DirectoryLoader
)

# Load PDF
pdf_loader = PyPDFLoader("document.pdf")
pdf_docs = pdf_loader.load()

# Load multiple files from directory
directory_loader = DirectoryLoader(
    "./documents",
    glob="**/*.txt",
    loader_cls=TextLoader
)
all_docs = directory_loader.load()

# Load CSV
csv_loader = CSVLoader(
    file_path="data.csv",
    source_column="source"
)
csv_docs = csv_loader.load()
```

### 2. Optimizing Retrieval

#### Retriever Types

```python
# Basic similarity search
retriever = vectorstore.as_retriever(
    search_type="similarity",
    search_kwargs={"k": 3}  # Top 3 results
)

# Maximum marginal relevance (diverse results)
retriever = vectorstore.as_retriever(
    search_type="mmr",
    search_kwargs={
        "k": 3,
        "fetch_k": 10  # Get 10, return 3 most diverse
    }
)

# Similarity with threshold
retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "score_threshold": 0.8,
        "k": 3
    }
)
```

### 3. Hybrid Search

Combine multiple search strategies for better results:

```python
from langchain.retrievers import EnsembleRetriever
from langchain_community.retrievers import BM25Retriever

# BM25 (keyword-based)
bm25_retriever = BM25Retriever.from_documents(documents)

# Semantic (vector-based)
semantic_retriever = vectorstore.as_retriever()

# Combine both
ensemble_retriever = EnsembleRetriever(
    retrievers=[bm25_retriever, semantic_retriever],
    weights=[0.3, 0.7]  # 30% BM25, 70% semantic
)
```

### 4. Advanced Prompt Engineering for RAG

#### Few-Shot Prompting

```python
from langchain.prompts import FewShotChatMessagePromptTemplate

examples = [
    {
        "input": "How do I reset my password?",
        "output": "To reset your password: 1) Click 'Forgot Password' 2) Enter email..."
    },
    {
        "input": "What's your refund policy?",
        "output": "Our refund policy allows 30 days for returns..."
    }
]

example_prompt = ChatPromptTemplate.from_template(
    "Question: {input}\nAnswer: {output}"
)

few_shot_prompt = FewShotChatMessagePromptTemplate(
    examples=examples,
    example_prompt=example_prompt,
    suffix="Question: {input}\nAnswer:"
)
```

#### Chain-of-Thought Prompting

```python
system_prompt = """You are a helpful assistant that answers questions step-by-step.

Follow this process:
1. Identify the key information needed
2. Find relevant context in the provided documents
3. Reason through the answer
4. Provide a clear, well-sourced response

Context:
{context}

Think step-by-step when answering."""
```

### 5. Handling LLM Hallucinations

#### Problem: Hallucinations

The model generates plausible-sounding but false information not in the retrieved context.

#### Solutions

**Solution 1: Strict Context Adherence**
```python
system_prompt = """You MUST only answer based on the provided context.

If the context doesn't contain the answer, respond with:
"This information is not available in my knowledge base."

Do NOT use your general knowledge if it contradicts the context.

Context:
{context}

Question: {input}"""
```

**Solution 2: Source Citation**
```python
system_prompt = """Always cite your sources using [Document X] notation.

Context:
[Document 1] Machine learning basics...
[Document 2] Deep learning applications...

Example response:
"Machine learning can be applied to image recognition [Document 2]. 
The basic concepts include supervised learning [Document 1]."

Question: {input}"""
```

**Solution 3: Confidence Scoring**
```python
# Add a confidence check step
chain = (
    {
        "context": retriever,
        "input": RunnablePassthrough()
    }
    | prompt
    | llm
    | confidence_output_parser  # Custom parser that adds confidence
)
```

### 6. Conversational RAG

Maintain conversation history:

```python
from langchain.chains import create_history_aware_retriever
from langchain.chains import create_retrieval_chain
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain_core.prompts import ChatPromptTemplate, MessagesPlaceholder

# Contextualize the query using chat history
contextualize_q_system_prompt = """Given a chat history and the latest user question 
which might reference context in the chat history, formulate a standalone question 
which can be understood without the chat history. Don't answer it, just reformulate it."""

contextualize_q_prompt = ChatPromptTemplate.from_messages([
    ("system", contextualize_q_system_prompt),
    MessagesPlaceholder(variable_name="chat_history"),
    ("human", "{input}"),
])

# History-aware retriever
history_aware_retriever = create_history_aware_retriever(
    llm, retriever, contextualize_q_prompt
)

# Question-answering prompt
system_prompt = """Use the retrieved context to answer the question."""

qa_prompt = ChatPromptTemplate.from_messages([
    ("system", system_prompt),
    MessagesPlaceholder(variable_name="chat_history"),
    ("human", "{input}"),
])

# Create the full chain
question_answer_chain = create_stuff_documents_chain(llm, qa_prompt)
rag_chain = create_retrieval_chain(history_aware_retriever, question_answer_chain)

# Usage
chat_history = []
while True:
    user_input = input("You: ")
    result = rag_chain.invoke({
        "input": user_input,
        "chat_history": chat_history
    })
    
    # Add to history
    chat_history.append(("human", user_input))
    chat_history.append(("ai", result["answer"]))
    
    print(f"Assistant: {result['answer']}")
```



## Practical Projects

### Project 1: Company Documentation Chatbot

**Objective**: Create a chatbot that answers questions about company policies

```python
import os
from dotenv import load_dotenv
from pathlib import Path
from langchain.document_loaders import TextLoader, DirectoryLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain.vectorstores import FAISS
from langchain.prompts import ChatPromptTemplate
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain.chains import create_retrieval_chain

load_dotenv()

# Step 1: Load company documents
company_docs_path = "./company_docs"
loader = DirectoryLoader(company_docs_path, glob="**/*.txt")
documents = loader.load()

# Step 2: Split documents
splitter = RecursiveCharacterTextSplitter(
    chunk_size=1000,
    chunk_overlap=100
)
splits = splitter.split_documents(documents)

# Step 3: Create vectorstore
embeddings = OpenAIEmbeddings(api_key=os.getenv("OPENAI_API_KEY"))
vectorstore = FAISS.from_documents(splits, embeddings)
retriever = vectorstore.as_retriever()

# Step 4: Create RAG chain
llm = ChatOpenAI(model="gpt-3.5-turbo-0125", api_key=os.getenv("OPENAI_API_KEY"))

prompt = ChatPromptTemplate.from_template("""
You are a helpful HR assistant. Answer questions about company policies.

Context from company documents:
{context}

Question: {input}

If the information is not in the provided documents, say: 
"This information is not available in our company documentation."
""")

document_chain = create_stuff_documents_chain(llm, prompt)
chain = create_retrieval_chain(retriever, document_chain)

# Step 5: Use the chatbot
while True:
    question = input("\nQuestion: ")
    if question.lower() in ['exit', 'quit']:
        break
    
    result = chain.invoke({"input": question})
    print(f"\nAnswer: {result['answer']}")
```

### Project 2: Research Paper Q&A System

```python
from langchain.document_loaders import PyPDFLoader
from langchain_text_splitters import RecursiveCharacterTextSplitter
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain.vectorstores import FAISS
from langchain.prompts import ChatPromptTemplate
from langchain.chains.combine_documents import create_stuff_documents_chain
from langchain.chains import create_retrieval_chain

# Load PDF paper
pdf_loader = PyPDFLoader("research_paper.pdf")
documents = pdf_loader.load()

# Split with larger chunks for papers
splitter = RecursiveCharacterTextSplitter(
    chunk_size=2000,
    chunk_overlap=200
)
splits = splitter.split_documents(documents)

# Create vectorstore
embeddings = OpenAIEmbeddings()
vectorstore = FAISS.from_documents(splits, embeddings)

# Create chain with academic prompt
llm = ChatOpenAI(model="gpt-3.5-turbo-0125", temperature=0)

academic_prompt = ChatPromptTemplate.from_template("""
You are an academic research assistant analyzing a research paper.

Paper content:
{context}

Research Question: {input}

Provide a detailed, scholarly response grounded in the paper's content.""")

chain = (
    academic_prompt
    | llm
    | StrOutputParser()
)

# Query
questions = [
    "What is the main contribution of this paper?",
    "What methodology did the authors use?",
    "What are the limitations discussed?"
]

for q in questions:
    print(f"Q: {q}")
    result = chain.invoke({"input": q, "context": retriever.get_relevant_documents(q)})
    print(f"A: {result}\n")
```



## Debugging & Optimization

### 1. Enable Debug Mode for RAG

```python
from langchain.globals import set_debug

# Enable full debugging
set_debug(True)

# Run your chain
result = chain.invoke({"input": "Your question"})

# This will show:
# - Retrieval results
# - Formatted prompt sent to LLM
# - LLM response
# - Token usage

set_debug(False)
```

### 2. Evaluate Retrieval Quality

```python
from langchain.evaluation import load_evaluator

# Check if retrieved documents are relevant
retrieval_qa_evaluator = load_evaluator("qa")

# Evaluate for specific query
query = "What is machine learning?"
retrieved_docs = retriever.get_relevant_documents(query)

evaluation = retrieval_qa_evaluator.evaluate_strings(
    input=query,
    prediction=retrieved_docs[0].page_content if retrieved_docs else "No results"
)

print(f"Relevance score: {evaluation['score']}")
```

### 3. Common Issues & Solutions

**Issue 1: Poor Retrieval Results**

```python
# Solution: Adjust chunk size
splitter = RecursiveCharacterTextSplitter(
    chunk_size=500,      # Smaller chunks = more specific results
    chunk_overlap=50
)

# Or try different retriever settings
retriever = vectorstore.as_retriever(
    search_type="mmr",   # Diverse results
    search_kwargs={"k": 5}  # Get more results
)
```

**Issue 2: LLM Ignoring Context**

```python
# Solution: Make instructions clearer
system_prompt = """
IMPORTANT: You MUST answer ONLY based on the provided context.
You CANNOT use external knowledge.

If information is not in the context, respond:
"I don't have this information in the provided documents."

Context:
{context}

Question: {input}

Answer:"""
```

**Issue 3: Too Many False Positives**

```python
# Solution: Use similarity threshold
retriever = vectorstore.as_retriever(
    search_type="similarity_score_threshold",
    search_kwargs={
        "score_threshold": 0.85,  # Only return high-confidence matches
        "k": 3
    }
)
```

### 4. Performance Optimization

#### Batch Processing

```python
queries = [
    "Question 1?",
    "Question 2?",
    "Question 3?"
]

# Batch multiple queries
results = chain.batch([{"input": q} for q in queries])

for query, result in zip(queries, results):
    print(f"Q: {query}")
    print(f"A: {result['answer']}\n")
```

#### Caching Embeddings

```python
from langchain.cache import InMemoryCache
from langchain.globals import set_llm_cache

set_llm_cache(InMemoryCache())

# Now repeated queries use cached embeddings
result1 = chain.invoke({"input": "What is X?"})
result2 = chain.invoke({"input": "What is X?"})  # Uses cache
```



## Summary: Week 2-3 Learning Path

### Week 2 Accomplishments
✅ Understood embeddings and semantic similarity  
✅ Learned vector database concepts  
✅ Set up FAISS vector store  
✅ Built your first RAG chain  
✅ Learned about context augmentation  

### Week 3 Accomplishments
✅ Advanced document loading techniques  
✅ Optimized retrieval strategies  
✅ Implemented hybrid search  
✅ Added conversational context  
✅ Built production-ready chatbots  
✅ Mastered hallucination prevention  



## Real-World RAG Use Cases in Action

### Case 1: Technical Support Chatbot

A software company uses RAG to:
- Index all support documentation and FAQs
- When customers ask "How do I fix error 404?", the system retrieves relevant troubleshooting guides
- Returns citations so users can read the original docs
- Reduces support tickets by 40%

### Case 2: Legal Document Analysis

Law firms use RAG to:
- Index thousands of case documents and contracts
- Query: "What are precedents for breach of contract?"
- System retrieves relevant cases and sections
- Lawyers save days of manual document review

### Case 3: Medical Information System

Hospitals use RAG to:
- Index patient records and medical literature
- Query: "What treatments are recommended for condition X?"
- System retrieves patient data + latest research
- Ensures answers are grounded in real data, not hallucinations



## Next Steps (Week 4+)

After mastering RAG fundamentals:

1. **Advanced Retrieval**: Implement reranking, query expansion
2. **Multiple Data Sources**: Combine databases, SQL, APIs
3. **Real-Time Updates**: Handle dynamic data sources
4. **Evaluation Frameworks**: RAGAS, LangSmith
5. **Fine-Tuning**: Custom embedding models
6. **Scalability**: Production deployment with Pinecone/Weaviate



## Key Takeaways

1. **RAG solves hallucinations**: By providing grounded context
2. **Embeddings are fundamental**: They enable semantic search
3. **Chunking matters**: Right chunk size = better retrieval
4. **Vector databases are essential**: They enable fast similarity search
5. **Retrieval quality first**: A good retriever beats a bad one more than LLM choice
6. **Context is key**: The augmented prompt is where the magic happens
7. **Test and iterate**: RAG systems require evaluation and tuning



## Resources

- [Official LangChain RAG Guide](https://docs.langchain.com/oss/python/langchain/rag)
- [Pinecone RAG Guide](https://docs.pinecone.io/guides/rag)
- [Deep Learning AI - RAG Course](https://learn.deeplearning.ai/courses/retrieval-augmented-generation/)
- [FAISS Documentation](https://github.com/facebookresearch/faiss)
- [Weaviate Vector Database](https://weaviate.io/)

Main Page: [AI Engineering Learning](/posts/ai-engineering-learning-1)

**Congratulations!** You now have a solid foundation in RAG implementation. The next step is to start building projects and experimenting with your own data.Happy coding! 🚀