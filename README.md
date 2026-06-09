# 🦷 Clinical Support-Chatbot: RAG Multi-Level Assistant

![n8n](https://img.shields.io/badge/n8n-FF6600?style=for-the-badge&logo=n8n&logoColor=white)
![Redis](https://img.shields.io/badge/redis-%23DD0031.svg?style=for-the-badge&logo=redis&logoColor=white)
![Pinecone](https://img.shields.io/badge/Pinecone-000000?style=for-the-badge&logo=pinecone&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI-412991?style=for-the-badge&logo=openai&logoColor=white)
![WhatsApp](https://img.shields.io/badge/WhatsApp-25D366?style=for-the-badge&logo=whatsapp&logoColor=white)

> A WhatsApp chatbot for dental clinical support, built on a multi-level Retrieval-Augmented Generation (RAG) architecture explicitly designed to **eliminate hallucinations in healthcare contexts**.

*Note: Client details are omitted per confidentiality agreement. Architecture and engineering decisions are documented here for portfolio purposes.*

---

## 📑 Table of Contents
- [Architecture Overview](#%EF%B8%8F-architecture-overview)
- [The 4-Stage Decision Pipeline](#-the-4-stage-decision-pipeline)
- [Deep Dive: Engineering Decisions](#-deep-dive-engineering-decisions)
- [Tech Stack](#-tech-stack)

---

## 🏗️ Architecture Overview

<!-- Coloque uma screenshot bonita do n8n ou diagrama aqui -->

<p align="center">
  <img src="./architecture/flow_diagram.jpeg" width="1000" alt="n8n Workflow" />
  <br>
  <sub><em>n8n Workflow</em></sub>
</p>

The system orchestrates a 4-stage decision pipeline that physically isolates decision logic from text generation logic. The LLM acts **only as a reader and formatter**, never as an ungrounded knowledge source.

---

## 🛤️ The 4-Stage Decision Pipeline

### 1. LGPD Validation (Gatekeeper)
State control is managed via **Google Sheets** and **Redis**. Every incoming user ID is intercepted and checked for Terms-of-Use acceptance before any AI processing begins. Unregistered users are securely locked in a consent flow.

<p align="center">
  <img src="./architecture/LGPD.png" width="350" alt="WhatsApp Demo - LGPD" />
  <br>
  <sub><em>WhatsApp Demo - LGPD</em></sub>
</p>

### 2. Tier 1: Hot Search (Vectorized FAQ)
Semantic retrieval using Pinecone + Redis + OpenAI Embeddings. Queries are vectorized and matched against a pre-approved FAQ knowledge base. 
*If the similarity score exceeds a strict threshold, the answer is served instantly, bypassing the LLM entirely.*

### 3. Tier 2: Router LLM (The "AI Trench")
When the Hot Search score is low, `gpt-4o-mini` operates strictly as a classifier. It reads the incoming message and routes it to exactly one predefined segment:
- 📖 **Clinical Manual:** Routes to curated technical protocols.
- 💬 **FAQ:** Routes to hand-crafted answers for recurring questions.
- 🛠️ **Question Forger:** If the query is ambiguous or incomplete, this trench reconstructs it into a well-formed query before re-injecting it into the pipeline—improving retrieval precision natively.
- 🛑 **OUT_OF_SCOPE:** Triggers a hard block with a fixed response.

### 4. Tier 3: Specialist RAG
Based on the router's decision, the flow queries isolated **Pinecone namespaces per product line**. Each namespace contains curated clinical documentation scoped strictly to its product.

---

## 🧠 Deep Dive: Engineering Decisions

To dig deeper into the technical choices that make this architecture robust, scalable, and safe for healthcare, explore the toggles below.

<details>
<summary><b>📉 FAQ Vector Strategy & Cost Optimization</b> (Click to expand)</summary>
<br>
In conversational AI, passing every single message through an LLM is highly inefficient. 

- **Cost & Latency:** By intercepting common queries with the Tier 1 Hot Search, we return answers instantly and bypass LLM prompt/completion costs.
- **Data Ingestion:** The FAQ dataset is maintained externally by the domain expert (client) via Google Sheets, empowering non-technical users to manage Q&A pairs. An automated workflow syncs these into Pinecone.
- **Threshold Tuning:** A strict threshold prevents false positives. If the query falls below the mathematical threshold it is escalated to the Router LLM.


```mermaid
flowchart TD
    %% GitHub native colors for styling
    classDef dataBase fill:#1f6feb,color:#fff,stroke-width:0px
    classDef success fill:#238636,color:#fff,stroke-width:0px
    classDef escalate fill:#da3633,color:#fff,stroke-width:0px

    A([Incoming User Message]) --> B[Vectorize Query<br/>OpenAI Embeddings]
    B --> C[(Pinecone Vector DB<br/>FAQ Namespace)]:::dataBase
    
    C --> D{Similarity Score<br/>> 0.85 ?}
    
    D -- "Yes (Match Found)" --> E[Return Exact FAQ Answer<br/>Zero LLM Tokens Used]:::success
    D -- "No (Novel Query)" --> F[Escalate to Router LLM<br/>Deep Intent Classification]:::escalate
```



</details>

<details>
<summary><b>✍️ Prompt Engineering & Structuring Notes</b> (Click to expand)</summary>
<br>

- **The Single-Word Router:** Traditional LLMs are verbose. The Router prompt is strictly constrained to output *only a single predefined keyword* (e.g., `Cemente1`, `Cement2`, `OUT_OF_SCOPE`). This eliminates parsing errors and ensures the Switch Node executes exact string matching.
- **Contextual Query Reconstruction:** The "Question Forger" intercepts fragmented follow-up questions and reformulates them against the chat history before vector retrieval, drastically improving semantic search accuracy.
- **Hierarchical Markdown Ingestion:** Clinical documentation was curated manually and structured in Markdown to preserve semantic boundaries (Headers, bullet points). The text splitter respects these boundaries, ensuring critical decision tables and protocols are retrieved as complete, unaltered units (preventing the LLM from summarizing or omitting key clinical data).
- **Injection Surface Minimization:** Out-of-scope queries trigger a hard block. Since the LLM never processes unclassified input directly, the prompt injection surface is drastically minimized.

```mermaid
flowchart TD
    %% Muted / Opaque Palette (GitHub Dark Mode aesthetic)
    classDef default fill:#21262d,stroke:#30363d,color:#c9d1d9,stroke-width:1px
    classDef router fill:#272336,stroke:#493e68,color:#c9d1d9,stroke-width:1px
    classDef forger fill:#2d243e,stroke:#4a3b69,color:#c9d1d9,stroke-width:1px
    classDef block fill:#3d1e1e,stroke:#662c2c,color:#c9d1d9,stroke-width:1px
    classDef db fill:#1b2a40,stroke:#2e4468,color:#c9d1d9,stroke-width:1px
    classDef success fill:#1e3625,stroke:#2f5e3e,color:#c9d1d9,stroke-width:1px

    A[Raw User Input] --> R[Single-Word Router<br/>Outputs EXACTLY 1 Keyword]:::router
    
    %% 1. Question Forger Loop
    R -- "FORGER" --> F[Question Forger<br/>Reformulates using Chat History]:::forger
    F -. "Re-injects fully qualified query" .-> R
    
    %% 2. Security Block
    R -- "OUT_OF_SCOPE" --> B[Static Hard Block<br/>Minimizes Injection Surface]:::block
    
    %% 3. Hierarchical Retrieval
    R -- "Cement1 / Cement2" --> DB[(Pinecone Vector DB<br/>Hierarchical Markdown Chunks)]:::db
    
    %% 4. Final Output
    DB --> S[Final RAG Synthesis<br/>Preserves Critical Clinical Tables]:::success
```

</details>

---

## 💻 Tech Stack
- **Orchestration:** n8n
- **State & Caching:** Redis
- **Vector Database:** Pinecone
- **AI Models:** OpenAI (`gpt-4o-mini` + `text-embedding-3-small`)
- **Messaging:** WhatsApp API
- **Data Management:** Google Sheets
