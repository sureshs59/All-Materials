# Large Language Models — Complete Beginner's Guide

> All 10 core LLM concepts explained in plain English with diagrams, examples, and practical notes.

---

## Table of Contents

1. [What is a Large Language Model?](#1-what-is-a-large-language-model)
2. [How are LLMs Trained?](#2-how-are-llms-trained)
3. [What is Prompt Engineering?](#3-what-is-prompt-engineering)
4. [Zero-Shot and Few-Shot Prompting](#4-zero-shot-and-few-shot-prompting)
5. [Temperature in LLM Generation](#5-temperature-in-llm-generation)
6. [Hallucination in LLMs](#6-hallucination-in-llms)
7. [Instruction Tuning](#7-instruction-tuning)
8. [RLHF — Reinforcement Learning from Human Feedback](#8-rlhf--reinforcement-learning-from-human-feedback)
9. [Context Windows in LLMs](#9-context-windows-in-llms)
10. [Embeddings and Why They Are Used with LLMs](#10-embeddings-and-why-they-are-used-with-llms)

---

## 1. What is a Large Language Model?

### Definition

A Large Language Model (LLM) is a neural network trained on massive amounts of text to understand and generate human language. It works by predicting the **next most likely word (token)** given everything that came before it.

- **"Large"** refers to the scale: billions to trillions of parameters
- It does **not look things up** — it generates responses based on learned statistical patterns
- Examples: GPT-4, Claude, Gemini, LLaMA, Mistral, Falcon

> "An LLM is an extremely sophisticated autocomplete that has read essentially the entire internet."

### How it generates text — one token at a time

```
Input: "The capital of France is"
    │
    ▼
LLM computes probability for every token in vocabulary
    │
    ▼
"Paris" → 92%    "London" → 3%    "Berlin" → 1%    ...
    │
    ▼
Samples "Paris" → appends to input → predicts next token → repeats
```

### Key components

| Component | What it does |
|-----------|--------------|
| **Tokeniser** | Splits text into tokens (word-pieces). "running" → ["run","ning"]. LLMs think in tokens, not words |
| **Transformer** | The neural network architecture. Uses "attention" to understand relationships between all tokens simultaneously |
| **Parameters (weights)** | Billions of numbers stored inside the model. Training adjusts these to make better predictions |
| **Autoregressive generation** | LLMs generate one token, append it to input, generate next. Text is built left to right |

### Scale reference

| Model | Parameters | Training data |
|-------|-----------|---------------|
| GPT-2 (2019) | 1.5 billion | ~40 GB text |
| GPT-3 (2020) | 175 billion | ~570 GB text |
| GPT-4 (2023) | ~1.8 trillion (est.) | Unknown |
| LLaMA 3 (2024) | 8B – 405B | 15 trillion tokens |

---

## 2. How are LLMs Trained?

### Three-stage training pipeline

```
Stage 1 (months)          Stage 2 (weeks)           Stage 3 (weeks)
  Pre-training        →   Instruction tuning    →      RLHF alignment
(learn the world)       (learn to follow tasks)    (learn to be helpful)
       │                        │                          │
       ▼                        ▼                          ▼
  Base model             Instruction-           Aligned, safe,
  (text completer)       following model        helpful assistant
```

---

### Stage 1 — Pre-training (the expensive part)

- **Data:** Trillions of tokens — internet text, books, Wikipedia, code, scientific papers
- **Task:** Predict the next token. Nothing more. No labels, no human input
- **Cost:** GPT-4 pre-training estimated at ~$100 million in compute
- **Duration:** Months on thousands of GPUs running continuously
- **Result:** A "base model" — knows a lot but cannot follow instructions

**The training loop:**

```
Text corpus (trillions of tokens)
    │
    ▼
Forward pass → predict next token
    │
    ▼
Loss = how wrong was the prediction?
    │
    ▼
Backpropagation → adjust weights to reduce loss
    │
    ▼
Repeat billions of times until loss plateaus
```

---

### Stage 2 — Instruction Tuning

Fine-tune the base model on curated `{instruction → response}` pairs. Teaches the model to follow commands rather than just continue text. See [Section 7](#7-instruction-tuning) for full detail.

### Stage 3 — RLHF

Human raters rank model outputs. A reward model learns human preferences. The LLM is updated to score higher on the reward model. See [Section 8](#8-rlhf--reinforcement-learning-from-human-feedback) for full detail.

> **Key insight:** The base model knows a lot but behaves like an unconstrained text completer. Stages 2 and 3 are what make it helpful, harmless, and honest.

---

## 3. What is Prompt Engineering?

### Definition

Prompt engineering is the practice of **crafting inputs to an LLM to get better outputs** — no code or fine-tuning required. LLMs are extremely sensitive to how a question is phrased; small wording changes can dramatically improve quality.

It works because LLMs pattern-match against their training data. Framing that activates more relevant patterns produces better results.

### Core techniques

#### 1. Be specific and detailed

| Vague | Better |
|-------|--------|
| "Write about Java" | "Write a beginner-friendly explanation of Java generics with two working code examples in Java 17, targeting developers with 1 year of experience" |

#### 2. Assign a role / persona

```
"You are a senior Spring Boot architect with 10 years of enterprise experience.
 Review the following code for performance bottlenecks and suggest improvements."
```

#### 3. Chain of thought — ask it to reason step by step

```
"Solve this problem step by step. Show your reasoning at each stage before
 giving the final answer. Do not skip steps."
```

#### 4. Specify output format and length

```
"Respond as a JSON object with keys: 'summary' (string), 'sentiment'
 (POSITIVE/NEGATIVE/NEUTRAL), 'confidence' (0.0–1.0). No extra text."
```

#### 5. Use delimiters to separate context from instruction

```
Summarise the following code.

Code:
```java
[paste code here]
```

Summary:
```

#### 6. Provide negative examples

```
"Explain this concept simply. Do NOT use jargon. Do NOT use analogies
 involving cooking or recipes."
```

### Prompt engineering is not magic

It works by:
- Activating relevant patterns the model saw during training
- Reducing ambiguity so the model does not have to guess what you want
- Giving the model the "role" that puts it in the right part of its knowledge

---

## 4. Zero-Shot and Few-Shot Prompting

### The spectrum

```
Zero shot          One shot         Few shot          Fine-tuning
(no examples)    (1 example)    (2–10 examples)   (thousands of examples)
     │                │                │                   │
  Fastest           Good           Most powerful       Most expensive
  Less accurate   for format     before fine-tuning    (requires GPU)
```

---

### Zero-shot prompting

Give the task with **no examples** — rely entirely on the model's pre-trained knowledge.

- Works well for common, well-understood tasks
- Fastest — uses the fewest tokens
- Best first attempt for any task

**Example:**
```
Prompt:
"Classify this review as POSITIVE or NEGATIVE:
 'The food was amazing and service was fast.'"

Model output:
"POSITIVE"
```

---

### Few-shot prompting

Provide **2–10 examples** of input/output pairs before your actual question. The model recognises the pattern and replicates it without retraining.

**Example — teaching a custom output format:**
```
Review: "Worst experience ever."
→ Sentiment: NEGATIVE, Score: 1/5, Action: ESCALATE

Review: "Absolutely loved it!"
→ Sentiment: POSITIVE, Score: 5/5, Action: PUBLISH

Review: "It was okay, nothing special."
→
```

**Model output (matches the learned pattern):**
```
Sentiment: NEUTRAL, Score: 3/5, Action: REVIEW
```

The model learned three things from the examples:
1. The output format (`key: value, key: value`)
2. The valid values for each field
3. The scoring scale

> **Rule of thumb:** If zero-shot gives inconsistent or wrong-format output, add 2–3 examples. If few-shot still fails, consider fine-tuning.

---

## 5. Temperature in LLM Generation

### What is temperature?

A number (typically 0 to 2) that controls **how random or creative the model's output is**.

At each generation step the model produces a probability distribution over all possible next tokens. Temperature scales those probabilities before sampling:
- **Low temperature** → peaks become sharper → model almost always picks the most probable token
- **High temperature** → probabilities flatten → model samples more uniformly across options

### Temperature scale

| Temperature | Behaviour | Best for |
|------------|-----------|----------|
| **0.0** | Deterministic — always picks highest-probability token | Code, SQL, JSON extraction, classification |
| **0.1 – 0.3** | Very focused, minimal variation | Technical writing, summarisation, data extraction |
| **0.5 – 0.7** | Balanced — natural variation, still coherent | Conversational AI, emails, general writing |
| **0.8 – 1.0** | Creative — diverse outputs, less predictable | Brainstorming, creative writing, marketing copy |
| **1.5 – 2.0** | Chaotic — often incoherent | Rarely useful in practice |

### Mathematical intuition

```
Raw logits:  [Paris: 10.5,  London: 7.2,  Berlin: 6.8,  Tokyo: 4.1]

Temperature = 0.1 (sharpen)
After scaling: [Paris: 96%,  London: 2%,  Berlin: 1.5%,  Tokyo: 0.5%]
→ Almost always picks Paris

Temperature = 1.0 (neutral)
After scaling: [Paris: 60%,  London: 20%,  Berlin: 15%,  Tokyo: 5%]
→ Usually Paris, sometimes London

Temperature = 2.0 (flatten)
After scaling: [Paris: 35%,  London: 28%,  Berlin: 25%,  Tokyo: 12%]
→ Basically a weighted coin flip
```

> **Practical tip for Spring Boot + LLM API integrations:** Use `temperature: 0` for deterministic, structured outputs (JSON parsing, classification). Use `temperature: 0.7` for user-facing chat responses where naturalness matters.

---

## 6. Hallucination in LLMs

### What is hallucination?

When an LLM **generates confident, plausible-sounding text that is factually wrong**.

The model is not lying. It has no concept of truth. It is doing exactly what it was trained to do: generate statistically likely text. If a wrong answer sounds plausible based on training patterns, it will produce it — confidently.

### Why hallucination happens

```
User: "Who invented the Kafka messaging system?"

LLM process:
  1. Pattern-match "Kafka" + "invented" → finds training patterns
  2. "Kafka" appears near "Jay Kreps, Jun Rao, Neha Narkhede" in training data
  3. Also appears near "Franz Kafka" (the author) in training data
  4. Model generates the highest-probability completion

  Correct answer: Jay Kreps, Jun Rao, Neha Narkhede at LinkedIn (2011)
  Potential hallucination: Confident mix of the two, or invented detail
```

### Types of hallucination

| Type | Description | Example |
|------|-------------|---------|
| **Factual** | Invents incorrect dates, names, statistics | "Java was created in 1991 by James Smith" |
| **Citation** | Invents fake paper titles, DOIs, URLs | Cites a research paper that does not exist |
| **Reasoning** | Gets right answer via wrong logic | Solves a math problem correctly but shows wrong working |
| **Instruction** | Claims inability or false completion | "I cannot access the internet" (when it never tried) |
| **Source** | Invents quotes attributed to real people | Fabricates a Linus Torvalds quote |

### How to reduce hallucination

1. **Retrieval-Augmented Generation (RAG)** — give the model actual documents to reference instead of relying on memory
2. **Temperature 0** for factual tasks — reduces random sampling
3. **Ask for sources** — forces the model to articulate evidence (still verify)
4. **Decompose the task** — break complex questions into smaller verifiable steps
5. **Validate outputs** — always verify critical facts against authoritative sources

> **Warning:** Never trust LLM outputs for medical, legal, financial, or safety-critical decisions without human verification. Hallucination is a fundamental property of all current LLMs — not a bug to be patched.

---

## 7. Instruction Tuning

### What is instruction tuning?

Fine-tuning a pre-trained base model on thousands of **`{instruction → response}`** pairs to teach it to follow commands rather than just continue text.

Also called **SFT (Supervised Fine-Tuning)**.

### The problem it solves

**Base model (before instruction tuning):**
```
Prompt:   "Translate 'hello' to French."
Output:   "Translate 'bonjour' to English. Translate 'goodbye' to French.
           Translate 'cat' to French..."
           ← Continues the pattern instead of answering
```

**Instruction-tuned model (after):**
```
Prompt:   "Translate 'hello' to French."
Output:   "Bonjour"
           ← Follows the instruction correctly
```

### Training data format

```json
{
  "instruction": "Summarise this article in one sentence.",
  "input": "[long article text here]",
  "output": "The article discusses climate change policies..."
}

{
  "instruction": "Write a Java method to reverse a string.",
  "input": "",
  "output": "public String reverse(String s) { return new StringBuilder(s).reverse().toString(); }"
}

{
  "instruction": "Is this customer review positive or negative?",
  "input": "Absolutely loved the product!",
  "output": "Positive"
}
```

Datasets like Stanford Alpaca, OpenAssistant, and FLAN contain hundreds of thousands of these pairs.

### What instruction tuning changes

| Before | After |
|--------|-------|
| Continues given text | Follows explicit commands |
| Ignores instruction intent | Understands "translate", "summarise", "list", "explain" |
| No consistent format | Adapts output format to the task |
| Unsafe outputs possible | Safer (though RLHF is needed for full alignment) |

> **Instruction tuning is what turns a raw language model into something you can actually talk to.** ChatGPT, Claude, and Gemini all start with a pre-trained base and apply instruction tuning as Stage 2.

---

## 8. RLHF — Reinforcement Learning from Human Feedback

### What is RLHF?

A training technique that teaches the model **what "good" means using human preferences** — making it helpful, harmless, and honest.

After instruction tuning the model can follow instructions but may still give verbose, unsafe, or unhelpful responses. RLHF is the alignment step.

### The 3-step process

```
Step 1 — Collect human preference data
─────────────────────────────────────────
Same prompt → model generates Response A and Response B
Human rater reads both and chooses which is better
→ Thousands of (prompt, A, B, preference) tuples collected

              ↓

Step 2 — Train a reward model
─────────────────────────────────────────
Train a separate neural network to predict human preferences
Input: (prompt + response) → Output: a score (how good is this?)
This reward model learns what humans consider "better"

              ↓

Step 3 — Fine-tune the LLM with RL (PPO algorithm)
─────────────────────────────────────────
LLM generates a response
→ Reward model scores it
→ PPO algorithm updates LLM weights to get higher scores
→ Repeat thousands of times
→ LLM learns to generate responses humans prefer
```

### What RLHF teaches

| Behaviour | How RLHF instills it |
|-----------|---------------------|
| Being concise | Humans rate shorter, clearer answers higher |
| Admitting uncertainty | "I don't know" rated higher than confident wrong answers |
| Refusing harmful requests | Raters flag unsafe responses negatively |
| Staying on topic | Off-topic responses rated poorly |
| Using appropriate tone | Raters prefer helpful, non-condescending responses |

### RLHF vs Instruction Tuning

| | Instruction tuning | RLHF |
|--|-------------------|------|
| **Data type** | Human-written instruction/response pairs | Human preference comparisons |
| **What it teaches** | How to follow instructions | What "good" looks like |
| **Algorithm** | Supervised learning | Reinforcement learning (PPO) |
| **Cost** | Lower | Higher (requires ongoing human raters) |

> **Key insight:** RLHF is why Claude refuses harmful requests, admits uncertainty, and tries to be concise. These behaviours were not explicitly programmed — they emerged from learning what humans rate as better responses.

---

## 9. Context Windows in LLMs

### What is a context window?

The **maximum amount of text the model can see and process at one time**, measured in tokens.

- Everything inside the window influences every token the model generates
- Everything **outside the window is completely invisible** to the model — it does not exist
- Both your input (prompt) and the model's output count against the limit
- ~1 token ≈ 0.75 words in English (roughly 4 characters)

### What goes inside a context window

```
┌────────────────────────────────────────────────────────────┐
│                    Context Window                           │
│                                                            │
│  [System prompt — instructions, persona, rules]            │
│  [Conversation history — all previous turns]               │
│  [Retrieved documents — from RAG]                         │
│  [Current user message]                                    │
│  [Model's response so far]                                 │
│                                                            │
│  All of this TOGETHER must fit within the token limit      │
└────────────────────────────────────────────────────────────┘
```

### Context window sizes — 2025

| Model | Context window | Approximate equivalent |
|-------|---------------|----------------------|
| GPT-3.5 (original) | 4,096 tokens | ~3,000 words |
| GPT-4 Turbo | 128,000 tokens | ~96,000 words / a large book |
| Claude 3.5 Sonnet | 200,000 tokens | ~150,000 words / a full novel |
| Gemini 1.5 Pro | 1,000,000 tokens | ~750,000 words / multiple novels |
| LLaMA 3.1 405B | 128,000 tokens | ~96,000 words |

### Key problems with context windows

**1. Cost scales with tokens**
Every token in + every token out = your API bill. A 200K context costs far more than a 4K context.

**2. Long conversation overflow**
When a conversation exceeds the window, older messages are dropped (truncation) or must be summarised. The model has no memory of what was dropped.

**3. "Lost in the middle" problem**
Research shows LLMs attend to the **beginning and end** of long contexts better than the middle. Critical instructions placed in the middle of a 100K token context may effectively be ignored.

```
Memory strength across a long context:

High  ████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░████████
Low        Beginning                                    End
                        Middle (weakest)
```

### Practical implications for Java/Spring Boot developers

```java
// When integrating LLMs via API in Spring Boot:

// 1. Track token usage to avoid exceeding limits
TokenCounter counter = new TokenCounter();
int promptTokens = counter.count(systemPrompt + userMessage + chatHistory);
if (promptTokens > MAX_CONTEXT_TOKENS * 0.8) {
    chatHistory = summarise(chatHistory); // compress old messages
}

// 2. Put the most important instructions FIRST and LAST — not in the middle
String prompt = criticalInstruction     // ← remembered best
             + retrievedDocuments
             + conversationHistory
             + userMessage
             + "Remember: " + criticalInstruction; // ← repeat at end

// 3. Use RAG to avoid stuffing the full document corpus into context
```

---

## 10. Embeddings and Why They Are Used with LLMs

### What are embeddings?

A way to **convert text into a list of numbers (a vector)** that captures its meaning in a mathematical space. Semantically similar text produces numerically similar vectors.

- "dog" and "puppy" → vectors very close together
- "dog" and "spreadsheet" → vectors very far apart
- A typical embedding vector has **768 to 3,072 numbers**

### Visualising semantic similarity

```
Vector space (simplified to 2D):

                    cat ●    ● kitten
              dog ●    ● puppy

                                              ● spreadsheet
                                         ● database

"dog" and "puppy" are neighbours.
"dog" and "spreadsheet" are far apart.
Similarity is measured by cosine distance between vectors.
```

### How embeddings are generated

```
Text: "Spring Boot microservices"
         │
         ▼
  Embedding model (e.g. text-embedding-3-large)
         │
         ▼
  [0.023, -0.187, 0.441, 0.092, -0.330, ...]
  ←────────── 3072 numbers ──────────────→
```

The embedding model is a **separate, smaller model** than the LLM itself. You call it once per document to generate and store its vector.

### Why embeddings are used with LLMs — the RAG pattern

**The problem:** LLMs cannot know your company's private data. They hallucinate when asked about domain-specific or recent information.

**The solution:** Retrieval-Augmented Generation (RAG)

```
SETUP (do once):
  Your documents (PDFs, wikis, code)
       │
       ▼  embed each chunk
  Vector database (pgvector, Pinecone, Weaviate, Chroma)
  Stores: [document_id, text_chunk, embedding_vector]

AT QUERY TIME:
  User question: "How do I reset a password in our system?"
       │
       ▼  embed the question
  Question vector: [0.31, -0.12, 0.88, ...]
       │
       ▼  similarity search in vector database
  Top 3 most similar document chunks retrieved
  (found by cosine similarity — no keywords needed)
       │
       ▼  inject into LLM prompt
  Prompt = "Answer based on these documents: [chunk1] [chunk2] [chunk3]
            Question: How do I reset a password?"
       │
       ▼
  LLM answers using actual company documentation
  → No hallucination — it has the real answer in context
```

### Embedding use cases

| Use case | How embeddings help |
|----------|---------------------|
| **Semantic search** | "Show me docs about cloud deployment" finds results mentioning "AWS Kubernetes" without keyword match |
| **RAG** | Give LLM access to your private documents without hallucination |
| **Duplicate detection** | Two differently-worded support tickets cluster together in embedding space |
| **Recommendation** | Find nearest neighbours — items semantically similar to what the user liked |
| **Clustering** | Group 10,000 customer reviews by topic without keyword rules |
| **Anomaly detection** | Detect messages that are semantically unusual compared to normal inputs |

### Spring Boot — calling an embedding API

```java
// Using OpenAI embeddings API via Spring Boot
@Service
public class EmbeddingService {

    private final OpenAiEmbeddingClient embeddingClient;

    // Embed a single text chunk
    public float[] embed(String text) {
        EmbeddingResponse response = embeddingClient.embedForResponse(
            List.of(text)
        );
        return response.getResults().get(0).getOutput();
    }

    // Store in pgvector (PostgreSQL vector extension)
    public void indexDocument(String docId, String text) {
        float[] vector = embed(text);
        documentRepository.save(new DocumentEmbedding(docId, text, vector));
    }

    // Semantic search — find most similar documents
    public List<String> findSimilar(String query, int topK) {
        float[] queryVector = embed(query);
        // pgvector: SELECT text FROM docs ORDER BY embedding <=> $1 LIMIT $2
        return documentRepository.findSimilar(queryVector, topK);
    }
}
```

> **Bottom line:** Embeddings are the bridge between your private data and the LLM. They let you build systems that answer questions from your own documents accurately, without hallucination, and without ever exposing your data to the LLM during training.

---

## Quick Reference — All 10 Concepts

| Concept | One-line definition |
|---------|---------------------|
| **LLM** | Neural network trained to predict the next token — generates text by pattern-matching against training data |
| **Training** | Pre-training (predict next token) → Instruction tuning (follow commands) → RLHF (be helpful and safe) |
| **Prompt engineering** | Crafting inputs to get better outputs — roles, chain-of-thought, format specs, delimiters |
| **Zero-shot** | Ask without examples — relies on pre-trained knowledge |
| **Few-shot** | Provide 2–10 examples — model replicates the pattern without retraining |
| **Temperature** | Controls randomness: 0 = deterministic, 0.7 = natural, 2.0 = chaotic |
| **Hallucination** | Confident, plausible-sounding text that is factually wrong — the model has no concept of truth |
| **Instruction tuning** | Fine-tuning on instruction/response pairs — turns text completer into instruction follower |
| **RLHF** | Human raters rank outputs → reward model trained → LLM updated to score higher |
| **Context window** | Maximum text the model can see at once — everything outside is invisible |
| **Embeddings** | Text converted to vectors — similar meaning = similar numbers — powers semantic search and RAG |

---

*This guide covers the foundational concepts required to work with LLMs in enterprise Java/Spring Boot applications. Concepts are ordered from fundamentals to advanced application patterns.*
