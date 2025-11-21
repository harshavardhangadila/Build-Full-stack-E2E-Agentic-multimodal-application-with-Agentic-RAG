## Personal Expense Assistant - Multimodal with Google ADK, Gemini 2.5, Firestore & Cloud Run

Build a multimodal personal expense assistant that accepts chat + receipt images, extracts and stores receipt data, and lets you search and analyze expenses via natural language. The stack uses the **Agent Development Kit (ADK)** with **Gemini 2.5**, **Firestore (vector search)**, **Cloud Storage artifacts**, a **FastAPI** backend, and a **Gradio** chat frontend. Deploy the whole thing to **Cloud Run**.

Youtube: [Demo](https://www.youtube.com/playlist?list=PLps8its2VEvkKg03Fm1RMN974EZMPA8k0)
---

## ✨ Features

- **Multimodal chat**: Upload receipts & chat via a web UI (Gradio).
- **Receipt parsing & storage**: Extracts metadata, stores docs in Firestore, files in GCS.
- **Agentic RAG**: Natural-language search over receipts using Firestore vector search + `text-embedding-004`.
- **Metadata search**: Filter by date ranges and amounts.
- **Callbacks for efficiency**: Compact conversation history by replacing old inline images with `[IMAGE-ID ...]` placeholders.
- **Cloud-ready**: Containerized and deployable to Cloud Run (single container runs both frontend & backend via `supervisord`).

---

## 🧱 Architecture

<img width="968" height="644" alt="image" src="https://github.com/user-attachments/assets/7ca86c36-277b-4aad-9ec9-0172b45c8557" />


---

## 🧰 Tech Stack

- **LLM & Agent**: Google **Gemini 2.5 Flash** via **Vertex AI**, **Google ADK** (BuiltInPlanner + callbacks)
- **Backend**: **FastAPI**, **Uvicorn**
- **Frontend**: **Gradio** chat UI
- **Storage**: **Google Cloud Storage** (artifacts), **Firestore** (metadata + vectors)
- **Embeddings**: `text-embedding-004`
- **Deploy**: **Cloud Run**, Docker, `supervisord`

---

## 📁 Repository Structure

```
personal-expense-assistant/
├─ expense_manager_agent/
│  ├─ __init__.py
│  ├─ .env
│  ├─ agent.py
│  ├─ tools.py
│  ├─ callbacks.py
│  └─ task_prompt.md
├─ backend.py
├─ frontend.py
├─ schema.py
├─ utils.py
├─ settings.py
├─ settings.yaml                  # env for Cloud Run
├─ Dockerfile
├─ supervisord.conf
├─ logger.py
├─ requirements.txt / uv.lock     # depending on your Python toolchain
└─ README.md
```

> **Note:** ADK expects the `expense_manager_agent` package to export a `root_agent` (done in `agent.py` + `__init__.py`).

---

## ✅ Prerequisites

- Google Cloud project with **billing enabled**
- **Vertex AI** and **Firestore** APIs enabled; **GCS** bucket created
- Python 3.10+ and **uv** or **pip**
- Logged in to gcloud: `gcloud auth application-default login`

---

## 🔐 Environment & Settings

This project pulls configuration from **`settings.py`** and a YAML file for deployment.

### `settings.yaml` (used for Cloud Run)

```yaml
GCLOUD_PROJECT_ID: "your-gcp-project-id"
GCLOUD_LOCATION: "us-central1"
STORAGE_BUCKET_NAME: "your-gcs-bucket"
DB_COLLECTION_NAME: "receipts"
BACKEND_URL: "http://0.0.0.0:8081/chat"   # Frontend -> Backend URL
```

> Locally, you can also export these as environment variables.

### Required Services

Enable these once per project:
```bash
gcloud services enable aiplatform.googleapis.com   firestore.googleapis.com   run.googleapis.com   storage.googleapis.com
```

Initialize Firestore (Native mode) in your region.

---

## 🧭 Firestore Vector Search Setup

The tools store an embedding under field `embedding` (dimension **768**, model `text-embedding-004`).

Create a **vector index** on your `receipts` collection (names may vary by console/CLI). Example gcloud (update names to match your environment as needed):

```bash
# Example: Create a vector index (adjust --collection-group if you use a collection group)
gcloud firestore indexes composite create   --collection-group=receipts   --field-config field-path=embedding,vector-config='{"dimension":768,"flat":{}}'
```

> If using the Firestore UI, add a **Vector Field** named `embedding` with dimension 768 and a Flat index.  
> The code uses EUCLIDEAN distance.

---

## 🚀 Local Development

> Use **two terminals**: one for backend (8081) and one for frontend (8080).

### 1) Install deps

Using **uv**:
```bash
uv sync
```

Or pip:
```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
```

### 2) Start the Backend (FastAPI + ADK Runner)

```bash
uv run backend.py
# or
python backend.py
```

Expected:
```
Uvicorn running on http://0.0.0.0:8081
```

### 3) Start the Frontend (Gradio)

```bash
uv run frontend.py
# or
python frontend.py
```

Expected:
```
* Running on local URL:  http://0.0.0.0:8080
```

Open the UI and upload 1–2 receipts to get started.

---

## 🧪 ADK Dev UI (Optional)

You can also run the ADK web dev UI to inspect events, tools, and logs:

```bash
uv run adk web --port 8080
```

> If you use this, run the backend on a different port or stop the Gradio frontend.

---

## 🗃️ Data Flow & Tools

- `store_receipt_data(image_id, store_name, transaction_time, total_amount, purchased_items, currency)`
  - Writes normalized receipt record into Firestore.
  - Generates an embedding via `text-embedding-004` and stores `Vector(embedding)` under `embedding`.
  - Rejects duplicates by `receipt_id` (image id).

- `search_receipts_by_metadata_filter(start_time, end_time, min_total_amount=-1, max_total_amount=-1)`
  - Date range + amount filtering (ISO 8601 string timestamps).

- `search_relevant_receipts_by_natural_language_query(query_text, limit=5)`
  - Vector similarity search using Firestore’s nearest-neighbor API.

- `get_receipt_data_by_image_id(image_id)`
  - Exact lookup by image ID (the `[IMAGE-ID ...]` inserted by the callback).

**Callback**: `modify_image_data_in_history`  
Keeps image bytes only for the last **3** user messages; replaces older inline images with `[IMAGE-ID <hash>]` placeholders to keep context light while preserving linkage.

---

## 🧩 API (Backend)

`POST /chat` — Chat with the agent.

**Request body** (`schema.ChatRequest`):
```json
{
  "text": "Store these receipts and summarize my coffee spend.",
  "files": [
    { "serialized_image": "<base64>", "mime_type": "image/png" }
  ],
  "session_id": "default_session",
  "user_id": "default_user"
}
```

**Response body** (`schema.ChatResponse`):
```json
{
  "response": "Here is your 2023 coffee breakdown...",
  "thinking_process": " ... optional agent reasoning snippet ... ",
  "attachments": [
    { "serialized_image": "<base64>", "mime_type": "image/png" }
  ],
  "error": null
}
```

**cURL Example**
```bash
curl -X POST http://localhost:8081/chat   -H "Content-Type: application/json"   -d '{"text":"Give me receipts from Indomaret","files":[],"session_id":"default_session","user_id":"default_user"}'
```

---

## 🌩️ Deploy to Cloud Run

From the repo root:

```bash
gcloud config set project YOUR_PROJECT_ID

gcloud run deploy personal-expense-assistant   --source .   --port=8080   --allow-unauthenticated   --env-vars-file=settings.yaml   --memory 1024Mi   --region us-central1
```

When it finishes, you’ll get a service URL like:

```
https://personal-expense-assistant-*******.us-central1.run.app
```

> **How it runs on Cloud Run**: The container uses `supervisord` to run **frontend (8080)** and **backend (8081)** in one image. The frontend calls the backend via `BACKEND_URL` in `settings.yaml`.

---

## ⚙️ Configuration Reference

| Variable               | Description                                               | Example                          |
|------------------------|-----------------------------------------------------------|----------------------------------|
| `GCLOUD_PROJECT_ID`    | GCP project id                                            | `my-project`                     |
| `GCLOUD_LOCATION`      | Vertex AI / region                                        | `us-central1`                    |
| `STORAGE_BUCKET_NAME`  | GCS bucket for artifacts                                  | `my-receipts-bucket`             |
| `DB_COLLECTION_NAME`   | Firestore collection name                                 | `receipts`                       |
| `BACKEND_URL`          | Frontend → Backend HTTP endpoint                          | `http://0.0.0.0:8081/chat`       |

> The code also sets `GOOGLE_GENAI_USE_VERTEXAI=TRUE` internally. Ensure your runtime has ADC credentials (`gcloud auth application-default login`) and Vertex AI access.

---

## 🔍 Usage Ideas

- “Store these receipts and summarize my coffee spend in 2023.”
- “Give monthly expense breakdown for 2023–2024.”
- “Show me the receipt file from **Yakiniku Like**.”
- “Find groceries receipts under 200k between 2024-01-01 and 2024-06-30.”

---

## 🧯 Troubleshooting

- **429 rate limit** when uploading too many images: keep batches to **2–3** images.
- **Empty search results**:
  - Confirm Firestore vector index exists for `embedding (768)`.
  - Verify `DB_COLLECTION_NAME` and region match.
- **Auth errors**:
  - Run `gcloud auth application-default login`.
  - Ensure Vertex AI & Firestore are enabled.
- **Backend unreachable from frontend**:
  - Check `BACKEND_URL` value and ports (frontend `8080`, backend `8081`).
- **Long responses**:
  - The callback prunes image bytes from older turns; keep uploads small per turn.

---

## 🔐 Security & Cost Notes

- This demo is configured as **public** when deployed. For production, add auth (IAP, identity-aware proxy, signed cookies, etc.).
- Vertex AI, Firestore, and GCS incur costs. Set budgets & alerts.

---

## 🗺️ Roadmap / Improvements

- Persist sessions (e.g., Firestore/Datastore) instead of in-memory.
- Per-user multi-tenant isolation in queries.
- More robust duplicate detection & idempotency.
- Add auth + rate limiting.
- Batch embeddings + streaming responses.

---

## 🙌 Acknowledgements

- Google **Agent Development Kit (ADK)**
- **Gemini 2.5 Flash** via **Vertex AI**
- **Firestore Vector Search** and **GCS**
- **FastAPI**, **Gradio**
