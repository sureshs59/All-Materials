# Generative AI — Complete Notes
## From Basics to Production Architecture

> Five diagrams explained — covering what Gen AI is, how Transformers work, RAG pipelines, the LLM training pipeline, and full production architecture.

---

## Table of Contents

1. [What is Generative AI?](#1-what-is-generative-ai)
2. [The Transformer Architecture](#2-the-transformer-architecture)
3. [RAG — Retrieval-Augmented Generation](#3-rag--retrieval-augmented-generation)
4. [LLM Training Pipeline](#4-llm-training-pipeline)
5. [Production Gen AI Architecture](#5-production-gen-ai-architecture)
6. [Key Concepts Quick Reference](#6-key-concepts-quick-reference)
7. [Interview Answer Guide](#7-interview-answer-guide)

---

## 1. What is Generative AI?

### Definition

Generative AI is a category of artificial intelligence model that is trained on massive amounts of existing content — text, code, images — and learns the statistical patterns well enough to **generate new, original content** it has never seen before.

### The three pillars

| Pillar | What it is | Examples |
|--------|-----------|---------|
| **Training data** | What the model learned from | Books, articles, code repos, web pages, conversations, scientific papers |
| **LLM model** | Compressed knowledge as billions of mathematical weights | GPT-4, Claude, Gemini, LLaMA |
| **Application layer** | What engineers build on top | Chatbots, code generators, RAG pipelines, image generation, AI agents |

### Traditional AI vs Generative AI

| Traditional AI | Generative AI |
|---------------|---------------|
| Classifies or predicts from fixed categories | Generates new content — text, code, images, audio |
| Rule-based or narrow supervised learning | Trained on internet-scale data via self-supervised learning |
| Example: spam classifier, fraud detector | Example: write an essay, generate code, answer any question |

### The one-sentence definition

> Generative AI = a model that generates new content by learning patterns from existing data at scale.

---

## 2. The Transformer Architecture

### What is a Transformer?

A Transformer is the neural network architecture that powers every major LLM. Introduced in the 2017 paper "Attention is All You Need" (Vaswani et al.), it replaced RNNs by processing all tokens in parallel via **self-attention**.

### How your prompt flows through a Transformer

```
Step 1 — Tokenisation
"Hello world how are you" → ["Hello", "world", "how", "are", "you"]
Each word (or sub-word) becomes a token with a unique integer ID.

Step 2 — Embedding
Each token ID → high-dimensional vector (e.g. 4,096 numbers for GPT-4)
Similar words have similar vectors. "king" - "man" + "woman" ≈ "queen"

Step 3 — Self-attention (repeated N times — e.g. 96 layers in GPT-4)
Every token looks at every other token and decides how much to attend to it.
Computed via Q (Query), K (Key), V (Value) matrices.

Step 4 — Feed-forward network
Each token transformed independently using learned weights.

Step 5 — Output (Softmax)
Final vector → probability distribution over the entire vocabulary (~50,000 tokens)
Pick the highest probability token → that is the next word generated.
```

### The Q, K, V mechanism — the heart of attention

Every token simultaneously plays three roles:

- **Query (Q):** "What am I looking for?" — what this token needs from others
- **Key (K):** "What do I offer?" — what this token advertises to others
- **Value (V):** "What information do I carry?" — the actual content passed forward

Attention weight = softmax(Q × K^T / sqrt(d_k)) × V

The result: the word "bank" in "river bank" attends strongly to "river" and weakly to financial words. The same word in "bank account" does the opposite. This is how context is understood.

### Key vocabulary

| Term | Plain English meaning |
|------|----------------------|
| **Token** | Smallest unit of text — roughly a word or sub-word |
| **Embedding** | A token converted to a list of numbers (a vector) |
| **Context window** | Maximum tokens the model can hold in memory at once |
| **Temperature** | Controls randomness — 0 = deterministic, 1 = creative, 2 = chaotic |
| **Hallucination** | Model generates confident but factually wrong output |
| **Fine-tuning** | Re-training a pre-trained model on domain-specific data |
| **Parameters** | The billions of numbers (weights) that encode the model's knowledge |

---

## 3. RAG — Retrieval-Augmented Generation

### Why RAG exists

LLMs only know what they were trained on. They cannot answer questions about:
- Your company's internal documents
- Events after their training cutoff date
- Real-time data (prices, news, status)

RAG solves this by retrieving relevant text at query time and injecting it into the prompt.

### The two phases

#### Phase 1 — Offline indexing (runs once)

```
Your documents (PDFs, databases, APIs)
        ↓
Chunker — split into ~500 token pieces
        ↓
Embedding model — convert each chunk to a vector
(e.g. OpenAI text-embedding-ada-002)
        ↓
Vector database — store all vectors
(Pinecone, Weaviate, pgvector, FAISS, Qdrant)
```

#### Phase 2 — Online query (runs on every user question)

```
User question: "What is our refund policy?"
        ↓
Embed the question using the same embedding model
        ↓
Semantic search — find top-K most similar chunks in vector DB
(cosine similarity between question vector and chunk vectors)
        ↓
Build prompt:
  System: "Answer based only on the provided context."
  Context: [retrieved chunk 1] [retrieved chunk 2] [retrieved chunk 3]
  Question: "What is our refund policy?"
        ↓
LLM generates grounded answer with citations
"Our refund policy allows 30 days... [source: policy.pdf p.4]"
```

### Why RAG beats fine-tuning for most use cases

| | RAG | Fine-tuning |
|--|-----|-------------|
| **Cost** | Low — no re-training | High — GPU hours |
| **Freshness** | Real-time — update the vector DB | Stale — re-train to update |
| **Transparency** | Cites sources | Black box |
| **Hallucination** | Low — grounded in documents | Can still hallucinate |
| **Best for** | Q&A over documents, knowledge bases | Changing model style/behaviour |

### RAG in Python — minimal example

```python
from openai import OpenAI
import numpy as np

client = OpenAI()

# Phase 1: Embed a document chunk
def embed(text):
    response = client.embeddings.create(
        model="text-embedding-ada-002",
        input=text
    )
    return response.data[0].embedding

# Phase 2: Retrieve and answer
def rag_answer(question, chunks):
    # Embed the question
    q_vector = embed(question)

    # Find most similar chunk (simplified cosine similarity)
    best_chunk = max(chunks, key=lambda c:
        np.dot(q_vector, embed(c)) /
        (np.linalg.norm(q_vector) * np.linalg.norm(embed(c)))
    )

    # Build prompt with retrieved context
    prompt = f"""Answer based only on this context:
Context: {best_chunk}
Question: {question}"""

    response = client.chat.completions.create(
        model="gpt-4",
        messages=[{"role": "user", "content": prompt}]
    )
    return response.choices[0].message.content
```

### LangChain RAG implementation

```python
from langchain.chains import RetrievalQA
from langchain.vectorstores import Pinecone
from langchain.embeddings import OpenAIEmbeddings
from langchain.llms import OpenAI

# Build the RAG chain
vectorstore = Pinecone.from_existing_index("my-index", OpenAIEmbeddings())
qa_chain = RetrievalQA.from_chain_type(
    llm=OpenAI(model="gpt-4"),
    retriever=vectorstore.as_retriever(search_kwargs={"k": 4})
)

# Answer a question
answer = qa_chain.run("What is our refund policy?")
```

---

## 4. LLM Training Pipeline

Every major LLM — GPT-4, Claude, Gemini, LLaMA — goes through the same four stages:

### Stage 1 — Pre-training

- **Input:** Trillions of tokens from the internet, books, code, scientific papers
- **Task:** Next-token prediction — given the previous tokens, predict the next one
- **Scale:** Months of training on thousands of GPUs
- **Output:** A base model that understands language but does not follow instructions

```
The cat sat on the [???]
Model learns: "mat" is most likely
```

### Stage 2 — Supervised Fine-Tuning (SFT)

- **Input:** Human-written instruction + answer pairs (thousands to millions)
- **Task:** Learn to follow instructions in a helpful format
- **Scale:** Days on fewer GPUs
- **Output:** A model that responds to prompts in a structured, helpful way

```
Instruction: "Explain quantum computing in simple terms"
Answer: "Quantum computing uses quantum bits (qubits)..."
```

### Stage 3 — RLHF (Reinforcement Learning from Human Feedback)

This is what makes a model safe, helpful, and aligned:

1. **Generate multiple responses** to the same prompt
2. **Human raters rank** the responses from best to worst
3. **Train a reward model** to predict human preferences
4. **PPO optimisation** — update the LLM to score higher on the reward model

- **Output:** A model that is helpful, harmless, and honest (Anthropic's Constitutional AI extends this)

### Stage 4 — Deployment

- **Quantisation** — compress model from float32 to int8/4-bit (reduces size 4-8x)
- **API serving** — vLLM, TensorRT, or managed endpoints (OpenAI, Anthropic, Azure)
- **Rate limiting** — tokens per minute, requests per day
- **Guardrails** — content moderation, prompt injection defence
- **Monitoring** — latency, token cost, error rates, hallucination detection

---

## 5. Production Gen AI Architecture

### Full stack layers

```
┌─────────────────────────────────────────────────────────┐
│              User Interface Layer                        │
│  Angular / React — streams tokens, renders citations    │
└─────────────────────────────┬───────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────┐
│           API Gateway (Spring Boot / FastAPI)            │
│  Auth · rate limiting · request validation · SSE        │
└─────────────────────────────┬───────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────┐
│       Orchestration Layer (LangChain / LlamaIndex)       │
│  retrieve docs → build prompt → call LLM → parse        │
│  agents · tool calling · memory · multi-step chains     │
└──────────┬──────────────────┬──────────────────┬────────┘
           │                  │                  │
    ┌──────▼──────┐    ┌──────▼──────┐    ┌──────▼──────┐
    │ Vector DB   │    │ LLM Provider│    │  Memory     │
    │ Pinecone /  │    │ OpenAI /    │    │  Redis /    │
    │ pgvector    │    │ Anthropic / │    │  DynamoDB   │
    │ Semantic    │    │ Azure OAI   │    │  Session    │
    │ search      │    │             │    │  context    │
    └─────────────┘    └─────────────┘    └─────────────┘
┌─────────────────────────────────────────────────────────┐
│         Observability + Safety Layer                     │
│  Token cost · latency · hallucination detection         │
│  Content moderation · prompt injection defence          │
│  A/B testing model versions · drift detection          │
└─────────────────────────────────────────────────────────┘
```

### Spring Boot integration example

```java
@Service
public class GenAIService {

    private final OpenAiChatClient chatClient;
    private final VectorStore vectorStore;

    // RAG: retrieve relevant documents then answer
    public String ragAnswer(String userQuestion) {

        // Step 1: Semantic search in vector store
        List<Document> relevantDocs = vectorStore
            .similaritySearch(userQuestion, 4);

        // Step 2: Build context from retrieved docs
        String context = relevantDocs.stream()
            .map(Document::getContent)
            .collect(Collectors.joining("\n\n"));

        // Step 3: Build prompt with context
        String prompt = String.format("""
            Answer based only on this context:
            %s

            Question: %s
            """, context, userQuestion);

        // Step 4: Call LLM
        return chatClient.call(new Prompt(prompt))
                         .getResult()
                         .getOutput()
                         .getContent();
    }
}
```

### Python FastAPI + LangChain example (KraftHeinz-style)

```python
from fastapi import FastAPI
from langchain.chains import RetrievalQA
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_pinecone import PineconeVectorStore
import os

app = FastAPI()

# Initialise once at startup
embeddings = OpenAIEmbeddings(model="text-embedding-ada-002")
vectorstore = PineconeVectorStore(index_name="compass-docs", embedding=embeddings)
llm = ChatOpenAI(model="gpt-4o", temperature=0.1, streaming=True)

qa_chain = RetrievalQA.from_chain_type(
    llm=llm,
    retriever=vectorstore.as_retriever(search_kwargs={"k": 4}),
    return_source_documents=True
)

@app.post("/api/ai/query")
async def query(question: str):
    result = qa_chain.invoke({"query": question})
    return {
        "answer": result["result"],
        "sources": [doc.metadata["source"] for doc in result["source_documents"]]
    }
```

---

## 6. Key Concepts Quick Reference

### Core vocabulary

| Term | Definition | Interview one-liner |
|------|-----------|---------------------|
| **LLM** | Large Language Model — transformer trained on billions of tokens | "A statistical model that predicts the most likely next token given context" |
| **Token** | Smallest unit — roughly a word or sub-word | "GPT-4 processes ~4 chars per token; 1000 words ≈ 750 tokens" |
| **Embedding** | A token converted to a vector of numbers | "Words with similar meanings have similar vectors — measurable with cosine similarity" |
| **Attention** | Mechanism where every token looks at every other token | "Self-attention captures context: 'bank' means something different near 'river' vs 'money'" |
| **RAG** | Retrieve relevant docs and inject into the prompt | "Grounds LLM answers in real documents, eliminates hallucination on domain knowledge" |
| **Fine-tuning** | Re-training a model on domain-specific data | "Changes model behaviour — expensive but powerful for domain adaptation" |
| **Prompt engineering** | Crafting inputs to guide model output | "System prompt, few-shot examples, chain-of-thought all affect output quality" |
| **Temperature** | Randomness control (0 = deterministic, 1 = balanced, 2 = chaotic) | "Use 0 for factual Q&A, 0.7 for creative tasks" |
| **Hallucination** | Confident but factually wrong output | "Model fills gaps with plausible-sounding but incorrect information" |
| **Context window** | Max tokens model can process at once | "GPT-4: 128K tokens ≈ 100,000 words — entire codebases fit in context" |
| **RLHF** | Reinforcement Learning from Human Feedback | "Humans rank outputs → reward model → PPO optimises LLM to score higher" |
| **Vector database** | Stores embeddings, enables semantic search | "Query with a meaning, get back semantically similar results — not keyword match" |

### Prompt engineering patterns

```python
# Pattern 1: System + User structure
messages = [
    {"role": "system", "content": "You are a helpful Java expert. Be concise."},
    {"role": "user", "content": "Explain Optional<T> in Java 8"}
]

# Pattern 2: Few-shot examples (show before ask)
prompt = """
Convert these sentences to formal English:
Informal: "gonna fix it later" → Formal: "The issue will be resolved subsequently."
Informal: "super fast" → Formal: "Exceptionally efficient performance."
Informal: "can't figure it out" → Formal: ???
"""

# Pattern 3: Chain-of-thought (force reasoning)
prompt = "Solve this step by step: If a train leaves at 9am..."

# Pattern 4: Role assignment
prompt = "You are a senior software architect with 20 years experience. Review this design..."

# Pattern 5: Output format specification
prompt = """
Analyse this code and return ONLY a JSON object:
{
  "issues": [...],
  "severity": "low|medium|high",
  "suggestions": [...]
}
Code: [code here]
"""
```

---

## 7. Interview Answer Guide

### "What is Generative AI and how does it work?"

> "Generative AI is a type of model trained on massive datasets to learn statistical patterns and generate new content. The key architecture is the Transformer — it converts text into token embeddings, passes them through many self-attention layers where every token attends to every other token, and predicts the next most likely token. Modern LLMs like GPT-4 have billions of parameters, are pre-trained on trillions of tokens, then fine-tuned with RLHF to be helpful and safe. What makes it useful for enterprise applications is the RAG pattern — you retrieve relevant documents at query time and inject them into the prompt, which grounds the model in your real data and eliminates hallucination."

### "What is RAG and when would you use it?"

> "RAG stands for Retrieval-Augmented Generation. It solves the core problem that LLMs only know what they were trained on — they cannot answer questions about your company's internal documents or real-time data. In RAG you have two phases. In the offline phase you chunk your documents, embed each chunk as a vector, and store them in a vector database like Pinecone or pgvector. At query time you embed the user's question, semantically search for the most relevant chunks, inject them into the LLM prompt, and the model generates a grounded answer with citations. I built this at KraftHeinz — we had a RAG pipeline that let sales teams query the Compass policy documents in natural language, with the LLM returning answers backed by specific document sources."

### "What is hallucination and how do you prevent it?"

> "Hallucination is when a model generates confident but factually incorrect output — it fills knowledge gaps with statistically plausible but wrong information. The three main mitigations are: RAG (ground answers in real documents), temperature 0 for factual queries (forces the most likely token rather than creative sampling), and output validation (check responses against known facts or have the model cite its sources so you can verify). Monitoring response quality over time via LLM-as-judge patterns also catches drift."

### "How would you integrate an LLM into a Spring Boot application?"

> "I would use Spring AI — it provides a consistent abstraction layer over OpenAI, Anthropic, and Azure OpenAI. The architecture would be: Angular frontend streams tokens via SSE, Spring Boot API gateway handles auth and rate limiting, LangChain4j or Spring AI orchestrates the RAG chain — semantic search in pgvector, prompt construction, LLM call, response parsing. For conversation memory I would use Redis to store the session context between turns. CloudWatch or Datadog monitors token usage and latency. I implemented exactly this pattern at KraftHeinz where I built a FastAPI Python service for high-throughput AI-powered data extraction alongside the Spring Boot backend."

---

*Generative AI Notes — Angular 18 · Spring Boot · Python · LangChain · OpenAI · RAG · Transformers · May 2025*
