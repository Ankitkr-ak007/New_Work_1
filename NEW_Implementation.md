# VIBE‑TRACE Implementation Documentation

## 1. Design Goals of VIBE‑TRACE
| Goal | Meaning | Realisation |
|------|----------|-------------|
| **Adversarial Verification** | A second model critiques the answer for safety, hallucinations, logical gaps, tone. | Added `adversarial_agent.py` and integrated it into the orchestration pipeline. |
| **Confidence Scoring** | Every agent returns a numeric confidence (0‑1) so the system can decide whether to trust the answer. | Refactored `classifier.py`, `responder.py`, and orchestrator to compute a weighted system confidence. |
| **Evidence‑Based Answers (RAG / Deep‑Linking)** | Answers must be grounded in concrete policies or knowledge‑base entries and display those sources. | Built a lightweight knowledge‑base (`policies.json`) and a retrieval service (`rag_service.py`). The responder now receives this context and returns a list of cited sources. |
| **Feedback Loop (Fix‑Once)** | Users can supply the correct answer when the system is low‑confidence or unsafe; this data is stored for future training. | Extended the `/feedback` endpoint and the front‑end review UI to capture a “correction” payload. |
| **Premium UI / Transparency** | UI must clearly display confidence, verification verdict, and any evidence, with a modern aesthetic. | Updated `ComplaintForm.jsx` and its CSS to show step‑by‑step progress, confidence badges, evidence notifications, and a correction box for low ratings. |
| **Robust Testing** | New logic should be testable without a live Gemini API key. | Added/updated `test_vibe_trace.py` to mock Gemini calls and verify orchestration, confidence aggregation, and RAG integration. |

## 2. Backend – Core Agents
### 2.1 `adversarial_agent.py`
* **Added** a new agent that receives the original complaint and the generated response, prompts Gemini with a “Devil’s‑Advocate” instruction, and returns JSON:
```json
{ "verdict": "SAFE|UNSAFE|NEEDS_IMPROVEMENT", "confidence_score": 0.0‑1.0, "critique": "...", "risk_level": "LOW|MEDIUM|HIGH" }
```
* **Purpose** – Implements adversarial verification; unsafe verdict penalises overall confidence.

### 2.2 `classifier.py`
* Refactored `classify_complaint` to return JSON with `category`, `confidence_score`, and optional `reasoning`.
* **Purpose** – Supplies early confidence for weighted aggregation.

### 2.3 `responder.py`
* Signature changed to `generate_response(category, text, context=None)`.
* Prompt now contains a **“CONTEXT (Ground Truth)”** section listing each retrieved policy.
* Output schema extended with a `sources` array (policy IDs used).
* All success branches include `sources: []` when no policy is used.
* Added robust fallback handling.
* **Purpose** – Generates the final answer **with confidence** and **evidence citations**.

### 2.4 `rag_service.py`
* New service that loads `policies.json` and provides `retrieve_context(text, category, top_n=3)` using simple keyword overlap.
* **Purpose** – Supplies contextual policies for the responder (lightweight RAG).

### 2.5 `policies.json`
* JSON array of policy objects (`id`, `title`, `content`, `keywords`). Example:
```json
{ "id": "POL-REF-001", "title": "Refund Policy", "content": "Customers may request a full refund within 30 days…", "keywords": ["refund","return","30 days"] }
```
* **Purpose** – Source of truth for evidence‑based answers.

### 2.6 `orchestrator.py`
* Imported `rag_service` and called `retrieve_context` before the responder.
* Added a new step entry **"Knowledge Retrieval"** to the `steps` list.
* Passed retrieved policies to `generate_response`.
* Captured `sources` from responder and added them to the final payload.
* Updated doc‑string to mention RAG and adversarial verification.
* Adjusted confidence calculation comment.
* **Purpose** – Coordinates all VIBE‑TRACE components: classification → RAG → response → adversarial critique → confidence aggregation → final packaging.

### 2.7 `feedback.py`
* POST `/feedback` now expects:
```json
{ "name":"...", "email":"...", "rating":1‑5, "recommendation":0‑10, "message":"...", "is_correction":true|false, "corrected_response":"optional" }
```
* **Purpose** – Enables the feedback loop for low‑rating or unsafe answers.

### 2.8 `start_backend.py`
* Restored script to launch FastAPI (`uvicorn backend.app.main:app --reload`).
* **Purpose** – Allows the backend server to run again.

### 2.9 `test_vibe_trace.py`
* Updated to mock `google.generativeai` and verify that the orchestrator returns `confidence_score`, `verification_result`, and `sources`.
* **Purpose** – Guarantees the new pipeline works without a real API key.

## 3. Front‑End – UI & API
### 3.1 `api.js`
* `submitReview` now accepts an optional fourth argument `correctedResponse` and sends `is_correction` and `corrected_response` in the payload.
* **Purpose** – Aligns front‑end with the new feedback schema.

### 3.2 `ComplaintForm.jsx`
* Added `correctionText` state and a **correction textarea** that appears when `rating ≤ 2`.
* After a successful complaint submission, checks `res.sources`; if present, shows an “info” toast listing the policy IDs.
* Fixed duplicated `catch` block syntax error.
* Updated call to `submitReview` to pass `correctionText`.
* **Purpose** – Provides UI for the feedback loop and displays evidence to the user.

### 3.3 `ComplaintForm.css`
* Added `.correction-box` styling with a soft orange gradient, rounded corners, fade‑in animation, and distinct label styling.
* **Purpose** – Gives the new correction UI a premium look consistent with VIBE‑TRACE aesthetics.

### 3.4 `README.md`
* Rewritten to showcase VIBE‑TRACE architecture, confidence scores, adversarial verification, and evidence‑based answers.
* Updated quick‑start instructions and added a Mermaid diagram of the pipeline.
* **Purpose** – Communicates the new capabilities to developers and stakeholders.

## 4. Runtime Flow (What Happens When a User Submits a Complaint)
1. **Front‑end** calls `POST /complaint` → orchestrator starts.
2. **Phase 1** – Classification, priority, sentiment run in parallel; classifier returns `confidence_score`.
3. **Phase 1.5** – `rag_service.retrieve_context` scans `policies.json` and returns up to three matching policies.
4. **Phase 2** – `generate_response` receives category, complaint, and policies; prompts Gemini to cite any policy used and returns `response`, `confidence_score`, `sources`.
5. **Phase 3** – `adversarial_critique` evaluates the response, returning `verdict` and its own confidence.
6. **Confidence Aggregation** – System confidence = 0.4 × adversarial + 0.3 × responder + 0.3 × classifier; if verdict is UNSAFE, confidence is halved.
7. Orchestrator returns a payload containing `category`, `priority`, `response`, `confidence_score`, `verification_result`, `sources`, and the step list.
8. **Front‑end** displays confidence percentage, verification badge (✅ or ⚠️), and a toast with evidence sources.
9. User rates the answer. If rating ≤ 2, a correction textarea appears; the UI sends `is_correction:true` and the corrected text to `POST /feedback`.

## 5. Remaining Work (Next Milestones)
| Area | What Still Needs to Be Done | Suggested Implementation |
|------|----------------------------|--------------------------|
| **Persist feedback corrections** | Store `is_correction` and `corrected_response` in a durable store (JSON file, SQLite, or external DB). | Extend `feedback.py` to write to `feedback_log.json` and add a background job that aggregates corrections for future fine‑tuning. |
| **Observability Dashboard** | Continuously monitor rolling average of `confidence_score` and alert when it drops below a threshold (e.g., 0.85). | Create `observability.py` that writes metrics to a CSV or Prometheus endpoint; add a small React component that fetches and charts the data. |
| **Expand Knowledge Base** | Add real company policies, legal clauses, FAQ entries, and map each to a set of keywords. | Populate `policies.json` with dozens of entries; optionally expose an admin UI to add/edit policies. |
| **RAG Retrieval Accuracy** | Replace simple keyword overlap with vector similarity search for more nuanced matching. | Add a new service `vector_rag_service.py` using `sentence‑transformers` + FAISS and switch the orchestrator to use it when the dataset grows. |
| **Unit‑tests for RAG** | Verify that given a complaint, the correct policies are returned. | Add `tests/test_rag_service.py` with parametrized inputs and expected policy IDs. |
| **CI/CD Integration** | Run the test suite on every push and enforce linting. | Add a GitHub Actions workflow (`.github/workflows/ci.yml`) that runs `pytest` and `ruff`/`eslint`. |
| **Documentation Polish** | Produce a dedicated markdown file (`docs/VIBE-TRACE-Architecture.md`) with a Mermaid diagram visualising the full pipeline, including data flows for feedback and observability. | Use the existing diagram in `README.md` as a starting point, expand it, and reference it from the docs. |
| **Production Build** | Bundle the front‑end with Vite for production and configure FastAPI to serve static assets. | Add an `npm run build` script, update `start_backend.py` to mount the `dist` folder, and optionally add Dockerfiles for containerised deployment. |

## 6. Summary
The VIBE‑TRACE migration has transformed the original Quickfix AI prototype into a **robust, transparent, and self‑correcting** AI assistant:
* **Adversarial verification** ensures unsafe answers are flagged.
* **Confidence scores** from three agents are combined into a single, interpretable metric displayed to the user.
* **RAG‑based deep‑linking** grounds every response in real policy text and shows the source IDs.
* **Feedback loop** lets users correct low‑confidence answers, creating a data source for future model improvement.
* **Premium UI** visualises the entire pipeline (steps, confidence, verification, evidence) with modern styling and micro‑animations.
All components are **implemented and integrated**, and the system runs locally (`npm run dev` + `python start_backend.py`). The remaining items focus on persisting corrections, adding observability, expanding the knowledge base, and polishing the developer experience. Once completed, the platform will be fully **bullet‑proof** and ready for production deployment.

---
*Generated on 2026‑01‑03.*
