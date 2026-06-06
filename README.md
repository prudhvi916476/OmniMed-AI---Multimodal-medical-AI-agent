# Multimodal Clinical Decision Support System 🩺

A Streamlit-based prototype that demonstrates **Retrieval-Augmented Generation (RAG)**, **Vision-Language Models (VLM)**, and external API integration to assist with clinical decision-making.

> [!WARNING]
> **DISCLAIMER**: This application is a prototype intended solely for demonstration purposes. It is **NOT** for real clinical or medical use.

---

## 🚀 Features

*   **PDF Report Analysis**: Extracts text from clinical notes or blood test reports using `pypdf`.
*   **Medical Scan Vision Assessment**: Leverages Google Gemini 2.5 Flash Vision capabilities to analyze X-Rays, MRIs, and CT scans for visual abnormalities.
*   **Retrieval-Augmented Generation (RAG)**: Integrates a local `FAISS` vector database containing clinical guidelines embedded with `gemini-embedding-001`.
*   **openFDA Drug Verification**: Automatically checks generated clinical assessments for specific medications and fetches real-time adverse reactions from the official FDA events API.
*   **Premium Glassmorphic UI**: Beautiful dark-mode UI with blur effects and session state history tracking.

---

## 🛠️ Tech Stack & Requirements

*   **Streamlit**: Frontend interface.
*   **LangChain**: Orchestration of RAG chains, prompts, and memory.
*   **FAISS (CPU)**: In-memory vector store for guidelines lookup.
*   **Google Gemini (via langchain-google-genai)**:
    *   Embeddings: `models/gemini-embedding-001`
    *   LLM & Multimodal: `gemini-2.5-flash`
*   **openFDA API**: Querying adverse drug events.

---

## 📥 Setup & Installation

### 1. Install Dependencies
Make sure you have Python 3.8+ installed. Install the required libraries using:

```bash
pip install -r requirements.txt
```

### 2. Run the App
Launch the Streamlit app locally:

```bash
streamlit run multimodel.py
```

### 3. Provide API Key
Once the application loads in your browser:
1. Open the sidebar configuration panel.
2. Paste your **Google Gemini API Key**.
3. Upload a PDF clinical note and/or a medical image (X-Ray, MRI, CT) to begin generating diagnostic insights.

---

## 📂 Project Structure

*   `multimodel.py`: Core Streamlit application code.
*   `requirements.txt`: Python package dependencies.
*   `README.md`: Project documentation.
