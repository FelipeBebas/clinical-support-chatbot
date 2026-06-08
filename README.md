# Clinical Support-Chatbot: RAG Multi-Level Clinical Assistant

WhatsApp chatbot for dental clinical support, built on a multi-level RAG 
architecture designed to eliminate hallucinations in healthcare contexts.

## Architecture overview

**4-stage decision pipeline:**

1. **LGPD validation:** State control via Google Sheets. Every user ID is 
   checked for terms-of-use acceptance before any AI processing begins.

2. **Hot search:** Semantic FAQ retrieval using Pinecone + Redis + OpenAI 
   embeddings. Queries are vectorized and matched against a pre-approved FAQ 
   knowledge base. A confidence score threshold determines whether the answer 
   is served immediately or escalated to the next stage.

3. **Router LLM:** GPT-4o-mini operates strictly as a classifier, not a text 
   generator. It reads the incoming message and routes it to exactly one of 
   four predefined segments: Clinical Manual, FAQ, Question Forger, or 
   OUT_OF_SCOPE. This "AI trench" approach isolates decision logic from 
   generation logic, ensuring the pipeline always reaches the correct 
   specialist context.

   - **Clinical Manual:** Routes to the curated Markdown knowledge base for 
     technical protocol questions
   - **FAQ:** Routes to a dentist-curated Q&A base with hand-crafted answers 
     for known recurring questions
   - **Question Forger:** When the incoming query is ambiguous or incomplete, 
     this trench reconstructs it into a well-formed query before re-injecting 
     it into the main pipeline — improving retrieval precision without 
     requiring the user to rephrase
   - **OUT_OF_SCOPE:** Triggers a hard block with a fixed response

4. **Specialist RAG:** Retrieves from isolated Pinecone namespaces per 
   product line. Each namespace contains curated clinical documentation in 
   Markdown, scoped strictly to its product.

## Key engineering decisions

- Clinical documentation manually curated and structured as hierarchical 
  Markdown files before ingestion into Pinecone. Section hierarchy controls 
  chunk boundaries, ensuring critical decision tables and protocols are 
  retrieved as complete, unaltered units preventing the LLM from 
  summarizing or omitting key clinical data
- LLM acts only as a reader and formatter, never as a knowledge source
- Out-of-scope queries trigger a hard block with a fixed response, which also 
  serves as a prompt injection barrier: since the LLM never processes 
  unclassified input directly, injection surface is minimized
- Redis layer adds caching and session state management to reduce latency 
  and redundant vector lookups

## Stack

n8n · Redis · Pinecone · OpenAI (GPT-4o-mini + text-embedding-3-small) 
· WhatsApp API · Google Sheets

## Note

Client details are omitted per confidentiality agreement.  
Architecture and engineering decisions are documented here for portfolio purposes.
