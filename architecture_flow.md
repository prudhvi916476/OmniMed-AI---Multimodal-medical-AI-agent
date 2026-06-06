# Multimodal Clinical Decision Support System Architecture Flow

This document details the architecture, component interaction, and data flow of the Multimodal Clinical Decision Support System.

## Architecture Overview

The application is a Streamlit-based web tool that integrates **Retrieval-Augmented Generation (RAG)**, **Vision-Language Models (VLM)**, and **external APIs** to assist clinical decision-making. 

```mermaid
graph TD
    %% Core Components
    User([User]) <--> UI[Streamlit UI / Frontend]
    UI -->|1. PDF Upload| PDFProc[PDF Reader pypdf]
    UI -->|2. Image Upload| VLMProc[Gemini 2.5 Flash Vision]
    UI -->|3. Trigger| RAGChain[LangChain Retrieval Chain]
    
    %% RAG & Vector Database
    subgraph Vector DB Pipeline
        FAISSStore[(FAISS Vector Store)]
        Embeddings[Gemini Embeddings models/gemini-embedding-001]
        Guidelines[Mock Clinical Guidelines]
        Guidelines --> Embeddings --> FAISSStore
    end
    
    %% RAG Execution
    PDFProc -->|Extract Text| RAGChain
    VLMProc -->|Scan Assessment| RAGChain
    FAISSStore -->|Retrieve Relevant Guidelines| RAGChain
    
    %% Inference and API
    RAGChain <-->|Query / Context & Prompts| LLM[Gemini 2.5 Flash LLM]
    RAGChain -->|Clinical Assessment| FDACheck{Drug Match Check}
    
    %% External API
    FDACheck -->|Yes: Query Side Effects| openFDA[openFDA API api.fda.gov]
    FDACheck -->|No| Hist[History & State Manager]
    openFDA -->|Append Side Effects| Hist
    
    %% Output
    Hist -->|Update UI & Chat History| UI
    
    %% Styling
    style UI fill:#3a4750,stroke:#f8f9fa,stroke-width:2px,color:#fff
    style FAISSStore fill:#1b2a4a,stroke:#3a86c8,stroke-width:2px,color:#fff
    style LLM fill:#4a1c40,stroke:#d81b60,stroke-width:2px,color:#fff
    style openFDA fill:#1a5f7a,stroke:#57c5b6,stroke-width:2px,color:#fff
```

---

## Detailed Step-by-Step Flow

The table below breaks down the execution process chronologically:

| Phase | Step | Component | Action / Data Flow Description |
|:---|:---|:---|:---|
| **Initialization** | 1 | **Streamlit App** | Renders the Glassmorphism UI, requests the Google Gemini API key via the sidebar, and initializes the session state (`chat_history`). |
| | 2 | **Embeddings & FAISS** | Initializes `initialize_rag_database()` (cached via `@st.cache_resource`): embeds the 9 mock medical guidelines using `gemini-embedding-001` and loads them into a local `FAISS` vector store. |
| **Ingestion** | 3 | **PDF Document Uploader** | If a PDF is uploaded, `extract_text_from_pdf` reads pages using `pypdf.PdfReader` and extracts raw clinical/blood test text. |
| | 4 | **Image Scan Uploader** | If a medical scan image (X-Ray/MRI/CT) is uploaded, it is base64-encoded into a Data URI to be sent multimodally. |
| **Vision Analysis** | 5 | **Vision Model (VLM)** | Sends the base64 scan to `gemini-2.5-flash` with a system prompt instructions role ("You are a radiologist..."). It returns a text-based scan abnormality analysis. |
| **RAG Retrieval** | 6 | **Vector Store Retriever** | Queries FAISS vector store to retrieve guidelines relevant to the query input ("Analyze the patient data"). |
| **Clinical Reasoning**| 7 | **Stuff Documents Chain** | Combines retrieved guidelines, extracted PDF report text, and image scan analysis into a consolidated prompt template. |
| | 8 | **Inference** | Invokes `gemini-2.5-flash` to evaluate the combined context and generate clinical decision recommendations. |
| **Drug Safety Check** | 9 | **Demo Drug Matcher** | Scans the generated response text for any of the monitored demo drugs (Amoxicillin, Azithromycin, Lisinopril, Dexamethasone, Ibuprofen). |
| | 10 | **openFDA API Query** | If a drug matches, queries the FDA events endpoint `https://api.fda.gov/drug/event.json` to fetch the top 3 adverse drug reactions. |
| **Rendering** | 11 | **History & Presentation** | Appends the FDA warning info to the assessment, pushes it to the front of `st.session_state.chat_history`, and renders the result cards on screen. |

---

## Sequence Execution Flow

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Streamlit UI
    participant PDF as PDF Parser (pypdf)
    participant VDB as FAISS Vector Store
    participant VLM as Gemini 2.5 (Vision)
    participant LLM as Gemini 2.5 (LLM RAG)
    participant FDA as openFDA API
    
    User->>UI: Enter API Key & Upload Files (PDF / Image)
    User->>UI: Click "Generate Diagnostic Insights"
    
    alt PDF Uploaded
        UI->>PDF: extract_text_from_pdf(file)
        PDF-->>UI: Return raw report text
    end
    
    alt Image Uploaded
        UI->>UI: Base64 Encode Image
        UI->>VLM: invoke(HumanMessage with image data URI)
        VLM-->>UI: Return visual abnormality assessment text
    end
    
    UI->>VDB: Retrieve guidelines matching query
    VDB-->>UI: Return relevant guidelines context
    
    UI->>LLM: invoke(Combined Prompt: Guidelines + PDF Text + Scan Assessment)
    LLM-->>UI: Return Diagnostic Insight response
    
    alt Response mentions tracked drug (e.g. Ibuprofen)
        UI->>FDA: GET openFDA endpoint search for reactions
        FDA-->>UI: Return top reactions list
        UI->>UI: Format FDA drug warning text
    end
    
    UI->>UI: Prepend result to session chat history
    UI-->>User: Render updated results on UI with Glassmorphic blocks
```

---

## Key Design Principles & Features

1. **Caching (`@st.cache_resource`)**:
   Prevents re-initializing the local FAISS database and downloading embeddings on every Streamlit widget rerun, drastically speeding up response times.
2. **Vision-to-Text Pipeline Integration**:
   Rather than passing raw image bytes direct into a complex single RAG chain, the model utilizes a two-step inference process:
   - *Step 1*: Isolate visual features (e.g., detecting shadows or midline shifts on brain MRIs) using a specialized visual prompt.
   - *Step 2*: Feed the textual visual assessment along with clinical notes to the primary RAG coordinator.
3. **Safety Guardrail instructions**:
   The prompt template explicitly restricts the LLM from forcing alignments to irrelevant guidelines (e.g., matching brain MRIs to community pneumonia standard treatments).
4. **FDA Drug Verification**:
   Uses real-time external data APIs (`openFDA`) to enrich the LLM suggestion with actual adverse reactions reported to the FDA.
