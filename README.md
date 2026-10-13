# Recall

Recall is a document question-answering application built with retrieval-augmented generation (RAG). Upload a document, let Recall index its contents, and ask questions that are answered using the most relevant parts of that document.

The application is designed to keep answers grounded in the uploaded document:

1. A document is uploaded to the backend.
2. Text is extracted from the file.
3. The text is normalized and split into overlapping chunks.
4. Gemini creates a 768-dimensional embedding for every chunk.
5. The chunks and embeddings are stored in Qdrant.
6. A question is embedded and matched against the stored chunks.
7. The matching context is sent to Gemini.
8. The answer is streamed back to the browser together with the retrieved source chunks.

## Features

- Upload PDF, TXT, Markdown, and CSV documents through the backend API.
- Upload PDF and TXT documents through the current web interface.
- Automatic PDF text extraction.
- Overlapping text chunking for better retrieval across chunk boundaries.
- Gemini embeddings using `gemini-embedding-001`.
- Similarity search with Qdrant and cosine distance.
- Optional filtering so a question searches only one uploaded document.
- Streaming answers using Server-Sent Events (SSE).
- Source information for each answer, including file name, chunk index, and similarity score.
- Automatic creation of the Qdrant collection and payload index at backend startup.
- CORS enabled by the backend for frontend development.

## Tech stack

### Frontend

- React 19
- TypeScript
- Vite
- `oxlint`

### Backend

- NestJS 11
- TypeScript
- Google Gemini API through `@google/genai`
- Qdrant through `@qdrant/js-client-rest`
- Multer for multipart file uploads
- `pdf-parse` for PDF extraction

## Project structure

```text
.
├── backend/
│   ├── src/
│   │   ├── gemini/                 # Gemini client, embeddings, and text generation
│   │   ├── rag/
│   │   │   ├── chunking.service.ts
│   │   │   ├── document-extraction.service.ts
│   │   │   ├── embedding.service.ts
│   │   │   ├── qdrant.service.ts
│   │   │   ├── rag.controller.ts   # Upload and streaming query endpoints
│   │   │   └── rag.service.ts      # RAG orchestration
│   │   ├── app.module.ts
│   │   └── main.ts
│   ├── Dockerfile
│   └── compose.yml
└── frontend/
    ├── src/
    │   ├── api/client.ts            # REST and SSE clients
    │   ├── components/
    │   │   ├── UploadPanel.tsx
    │   │   └── QuestionPanel.tsx
    │   │   ├── App.tsx
    │   │   └── types/rag.ts
    │   └── ...
    └── package.json
```

## Prerequisites

- Node.js 20 or newer
- npm
- A Gemini API key
- A Qdrant instance and its URL
- Docker and Docker Compose, if using the Docker workflow

The backend currently uses these Gemini models:

- `gemini-embedding-001` for 768-dimensional embeddings
- `gemini-3.8-flash` for streamed answers

Make sure the configured Gemini account has access to the models used by the application.

## Configuration

Create `backend/.env`:

```env
GEMINI_API_KEY=your_gemini_api_key
QDRANT_URL=https://your-qdrant-instance.example.com
QDRANT_API_KEY=your_qdrant_api_key
PORT=8000
```

`QDRANT_API_KEY` is optional when Qdrant does not require authentication. `PORT` is optional for the backend itself, but setting it to `8000` matches the frontend's current API URL and the Docker Compose port mapping.

Do not commit `.env` files or API keys. The backend ignores dotenv files by default.

## Running locally

### 1. Start the backend

```bash
cd backend
npm install
npm run start:dev
```

The API listens on `http://localhost:8000` when `PORT=8000` is set. Without `PORT`, NestJS defaults to port `3000`.

The backend initializes the `recall_documents` Qdrant collection with:

- Vector size: `768`
- Distance: `Cosine`
- Payload index: `documentId`

### 2. Start the frontend

In a second terminal:

```bash
cd frontend
npm install
npm run dev
```

Open the URL printed by Vite, usually `http://localhost:5173`.

The current frontend client is configured to call `http://localhost:8000`. If the backend runs on a different host or port, update `API_URL` in `frontend/src/api/client.ts`.

## Running with Docker Compose

The Compose configuration is located in `backend/compose.yml` and builds the backend image from `backend/Dockerfile`.

```bash
cd backend
docker compose up --build
```

This starts the backend on port `8000` and loads environment variables from `backend/.env`. Start the frontend separately:

```bash
cd frontend
npm install
npm run dev
```

The Compose setup mounts the backend source directory and runs NestJS in watch mode, so it is intended primarily for development.

## Using the web application

1. Start the backend and frontend.
2. Select a PDF or TXT file in the **Upload a document** panel.
3. Click **Upload document**.
4. Wait for the document-ready message and generated chunk count.
5. Enter a question in the **Ask a question** panel.
6. Click **Get answer**.
7. Read the streamed answer and inspect the retrieved source chunks.

The browser keeps the returned `documentId` and sends it with subsequent questions, so the question is scoped to the document uploaded in the current page session.

## API reference

### Upload a document

```http
POST /documents/upload
Content-Type: multipart/form-data
```

The multipart field must be named `file`.

Example:

```bash
curl -X POST http://localhost:8000/documents/upload \
  -F "file=@./notes.pdf"
```

Successful response:

```json
{
  "documentId": "generated-uuid",
  "filename": "notes.pdf",
  "chunks": 4,
  "fileName": "notes.pdf",
  "chunkCount": 4,
  "sizeBytes": 18342
}
```

Supported backend file extensions are `.pdf`, `.txt`, `.md`, and `.csv`. The web interface currently limits its file picker to `.pdf` and `.txt`.

### Stream an answer

```http
POST /query/stream
Content-Type: application/json
```

Request body:

```json
{
  "question": "What is the main idea?",
  "documentId": "generated-uuid",
  "topK": 5
}
```

- `question` is required.
- `documentId` is optional at the API level. When provided, search is limited to that document.
- `topK` is optional and defaults to `5`.

The response is an SSE stream. Each event contains JSON in its `data` field:

```text
data: {"type":"sources","sources":[...]}

data: {"type":"token","text":"The"}

data: {"type":"token","text":" document"}

data: {"type":"done"}
```

If processing fails after the stream starts, the backend sends an SSE error event:

```text
event: error
data: {"message":"Unable to stream the answer."}
```

Example:

```bash
curl -N http://localhost:8000/query/stream \
  -H "Content-Type: application/json" \
  -d '{"question":"What is the main idea?","documentId":"generated-uuid","topK":5}'
```

## RAG implementation details

### Extraction and chunking

PDF content is extracted with `pdf-parse`. Text, Markdown, and CSV files are read as UTF-8. Extracted text is normalized by collapsing whitespace.

The default chunking settings are:

- Maximum chunk size: `1200` characters
- Overlap: `150` characters

### Embeddings and retrieval

Each chunk and each question is embedded by Gemini with an output dimensionality of `768`. Chunks are stored in the `recall_documents` Qdrant collection with this payload:

```json
{
  "documentId": "generated-uuid",
  "fileName": "notes.pdf",
  "chunkIndex": 0,
  "text": "The chunk content..."
}
```

Question retrieval uses cosine similarity and returns the requested number of nearest chunks. The retrieved chunk text is combined into the prompt sent to Gemini.

### Answer generation

The answer model receives an instruction to use only the retrieved context and to say clearly when the context does not contain the answer. Generated text is streamed to the client as it becomes available.

## Development commands

Run these commands from the relevant directory.

### Backend

```bash
npm run start:dev  # Start with watch mode
npm run build      # Compile the NestJS application
npm run start:prod # Run compiled output
npm run lint       # Run ESLint and apply fixes
npm test           # Run Jest tests
npm run test:cov   # Run tests with coverage
```

### Frontend

```bash
npm run dev        # Start the Vite development server
npm run build      # Type-check and build for production
npm run preview    # Preview the production build
npm run lint       # Run oxlint
```

## Troubleshooting

### `GEMINI_API_KEY is required`

Create `backend/.env` and set `GEMINI_API_KEY`, then restart the backend.

### `QDRANT_URL is required`

Set `QDRANT_URL` in `backend/.env`. If the Qdrant deployment requires authentication, also set `QDRANT_API_KEY`.

### The frontend cannot connect to the API

Confirm that the backend is running on port `8000` and that `frontend/src/api/client.ts` points to the correct API URL. Also check that the browser is not blocking the request because of an incorrect host or port.

### The answer is not relevant

Confirm that the uploaded document contains extractable text, that the question is about that document, and that Qdrant is reachable. The answer is generated only from the top retrieved chunks, so changing `topK` can change the available context.

### The PDF has no useful text

The current extractor handles text-based PDFs. Scanned PDFs that contain only images require OCR, which is not currently included.

## Current limitations

- There is no authentication or user account system.
- Uploaded files are not stored as original files; extracted chunks and embeddings are stored in Qdrant.
- There is no endpoint for listing or deleting indexed documents.
- The frontend currently supports PDF and TXT selection even though the backend also accepts Markdown and CSV.
- The frontend API URL is hardcoded rather than supplied through a Vite environment variable.
- There are no automated end-to-end tests in the repository.
- The application depends on external Gemini and Qdrant services.
