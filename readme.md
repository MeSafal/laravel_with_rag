# 🧠 RAG Chatbot: Proof of Concept & Architecture Guide

This document outlines the architecture and design patterns used to build a robust, production-ready RAG (Retrieval-Augmented Generation) system. It serves as a blueprint for developers looking to understand or replicate this system.

## 🎯 Core Objectives

1.  **Hybrid Search**: Combine semantic search (embeddings) with keyword search for maximum accuracy.
2.  **Intent Classification**: Smartly route queries to either the database (for facts) or the LLM (for casual chat).
3.  **Real-time Feedback**: Provide a "Thinking..." UI to show users the AI's process without blocking the interface.
4.  **Scalability**: Modular design that can be dropped into any Laravel application.

---

## 🏗️ System Architecture

The system is built on **Laravel 11** using a modular approach (`nwidart/laravel-modules`).

### 1. The "Brain": Intent Classifier
**File**: `Services/IntentClassifier.php`

Instead of sending every message to the LLM (which is slow and expensive), we first classify the user's intent:
-   **`db_needed`**: The user is asking for facts (e.g., "How many services?", "Who is the CEO?"). -> **Trigger RAG Pipeline**.
-   **`casual`**: The user is saying "Hello" or "Thanks". -> **Trigger Casual LLM Response**.
-   **`blocked`**: The user is asking about forbidden topics. -> **Block Request**.

**Key Innovation**: We use a **Hybrid Classification** strategy:
1.  **Hardcoded Keywords**: Instant detection for critical terms (`classes`, `services`, `price`).
2.  **LLM Fallback**: If no keywords match, a small, fast LLM classifies the intent.

### 2. The "Search Engine": Hybrid Retrieval
**File**: `Services/DatabaseService.php` & `Services/EmbeddingService.php`

Standard vector search often fails on exact terms (e.g., "classes" vs "coaching"). We solved this with a 3-step process:

1.  **Query Expansion**:
    -   User asks: *"classes"*
    -   System expands to: *"classes coaching courses training workshops"*
    -   *Why?* This bridges the gap between user slang and database terminology.

2.  **Vector Search (Semantic)**:
    -   We generate an embedding for the expanded query using `openai/text-embedding-3-small`.
    -   We search `data_embeddings` table using cosine similarity.
    -   **Threshold**: `0.10` (tuned for high recall).

3.  **Multi-Table Aggregation**:
    -   We don't just search one table. We search `services`, `teams`, `blogs`, etc., simultaneously.
    -   Results are merged and ranked by relevance.

### 3. The "Voice": Response Generation
**File**: `Services/ResponseService.php`

Once data is retrieved, we don't just dump it. We feed it to the LLM with a strict **System Prompt**:
-   *"Answer using ONLY this data."*
-   *"If asked for a count, COUNT the items explicitly."*
-   *"Do not hallucinate."*

This ensures the AI is **grounded** in truth.

### 4. The "Nervous System": Event-Driven Architecture
**File**: `Jobs/ProcessOpenRouterMessage.php`

To keep the UI snappy, everything happens in the background:
1.  **User sends message** -> Saved to DB -> Job dispatched -> Return "200 OK".
2.  **Job runs**:
    -   Classifies intent.
    -   Broadcasts "Thinking..." events via **Reverb (WebSockets)**.
    -   Retrieves data.
    -   Generates response.
3.  **Broadcast**: Final response is pushed to the frontend.

---

## 🛠️ Database Schema

### `data_embeddings` Table
The backbone of our search. It stores vector representations of all content.

```sql
create_table 'data_embeddings' (
    id: bigInteger,
    table_name: string,      -- e.g., 'services'
    entity_id: bigInteger,   -- ID in the original table
    embedding_type: string,  -- 'title', 'description', 'combined'
    raw_text: text,          -- The actual text content
    embedding: vector(1536)  -- The vector data
);
```

---

## 🚀 Key Learnings & "Gotchas"

1.  **Embeddings aren't magic**: They fail on specific jargon. **Query Expansion** is mandatory for production reliability.
2.  **Context is tricky**: Don't rely on conversation context for database queries. If a user asks "What are they?" after "How many?", force a new DB search.
3.  **Feedback is UX**: A static loading spinner feels slow. A "Thinking..." log that shows *what* the AI is doing ("Searching services...", "Found 5 items") makes the wait feel shorter and builds trust.

---

## 📦 How to Replicate

1.  **Setup**: Laravel 11 + Reverb + Queue Worker.
2.  **Models**: Create models for your data (`Service`, `Team`, etc.).
3.  **Embed**: Write a script to loop through your data and generate embeddings via OpenRouter/OpenAI.
4.  **Search**: Implement the Cosine Similarity search in SQL.
5.  **Prompt**: rigorous prompt engineering is the final line of defense against hallucinations.

---

*This architecture is designed to be "Model Agnostic". You can swap OpenRouter for OpenAI, Anthropic, or local Llama models without changing the core logic.*
