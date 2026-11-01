# Banking Codebase Onboarding Assistant

A proposal for a web application that helps new engineers learn a large Java banking codebase by asking questions and getting answers grounded in the actual code and documentation.

---

## Step 1: Project Overview

**Core idea.** A chat-style web app where a new engineer asks questions like "How does a payment reversal flow through the services?" or "Which module owns KYC checks?" and gets an answer with citations to the exact files, classes, and docs it came from.

**Problem.** In banking, the Java systems are big, layered, and old. In the system I work on, there are 50+ microservices, and fully onboarding a new engineer can take up to three months. That time goes into understanding the services, the architecture diagrams, and the edge-case scenarios. Knowledge transfer (KT) sessions are the main way this knowledge is passed on, but they can't fill a full 8-hour working day, so they are spread out and incomplete. The knowledge lives in a few senior people's heads, so those seniors get interrupted constantly.

**Opportunity.** Make the existing code and docs searchable in plain language, so new hires get answers faster and seniors get their time back, without letting the AI invent answers in a regulated domain.

---

## Step 2: Use Case & Workflow

**Workflow improved:** onboarding and day-to-day code discovery for new engineers.

**Today:** read outdated wiki pages, grep the repo, then ask a senior engineer and wait.

**With the assistant:**
1. The engineer asks a question in the web app.
2. The system retrieves the most relevant code and docs they are allowed to see.
3. The AI drafts an answer using only that material, with citations.
4. A verification step checks that the answer is supported by the sources. If not, it says "I don't know."
5. For high-risk areas (payments, authentication, compliance), the answer is marked "needs senior review" and goes to a senior engineer to approve before it is shown as verified.
6. The engineer can rate the answer, which feeds evaluation.

AI automates the search-and-explain part. Humans stay responsible for the high-risk judgment.

---

## Step 3: AI Features to Be Implemented

| Feature | What it does here | Why it matters |
|---|---|---|
| **Prompt engineering** | System prompt tells the model to answer only from the provided context, cite sources, and refuse when evidence is missing. Separate prompts for answering and for verifying. | Reduces made-up answers, which is the biggest risk in banking. |
| **Structured outputs** | Answers are returned as a defined schema: `answer`, `citations[]` (file, lines), `confidence`, `risk_area`, `needs_review`. | Lets the UI show citations, enforce the review rule, and makes outputs testable. |
| **RAG + vector database** | Code and docs are chunked (by class/method for Java, by section for docs), embedded, and stored in a vector DB. Questions retrieve the top matching chunks. | The model can't know a private codebase. Retrieval gives it real, current context. |
| **Prompt chaining** | Retrieve → draft answer → verify answer against sources → classify risk. | Smaller steps are easier to check than one big prompt. |
| **Human-in-the-loop** | Senior engineers approve answers touching payments, auth, or compliance. | Wrong answers in these areas carry real financial and regulatory cost. |
| **Evaluation framework** | A fixed question set with expected sources, run on every change. | Shows whether changes improve or hurt quality. |
| **Observability** | Traces of each request, plus cost, latency, and feedback. | Lets the team find failures and track quality over time. |

---

## Step 4: Technical Approach

**High-level architecture**

```
Web UI  ->  API (Java Spring Boot)
                |
                +-> Auth / access control (only repos the user may see)
                +-> Redaction layer (mask PII, secrets before any model call)
                +-> Retriever -> Vector DB
                +-> LLM (draft answer, structured output)
                +-> Verifier (checks answer against sources)
                +-> Review queue (senior approval for high-risk areas)
                +-> Tracing / metrics
```

**Indexing pipeline.** A scheduled job pulls the repos and docs, splits Java by class and method and docs by heading, creates embeddings, and writes them to the vector DB with metadata (repo, path, module, owner, access group).

**Tools and services (proposed)**
- **LLM:** a model approved by the bank, ideally a private or self-hosted deployment. A public API is only used with redaction and approval.
- **Embeddings and vector store:** an embeddings model plus a vector DB such as Pinecone or pgvector.
- **Backend:** Java Spring Boot (matches our existing stack). **Frontend:** React.
- **Evaluation and tracing:** LangSmith or a similar tool.
- **Development:** Cursor with project rules for consistent code (cite sources, no real customer data, `BigDecimal` for money).

---

## Step 5: Example Prompts & Expected Outputs

**Example 1**

*Prompt:* "How does a payment reversal flow through the services?"

*Expected response (structured):*
```json
{
  "answer": "A reversal request enters through PaymentController, is validated in PaymentService, then ReversalService creates a compensating ledger entry and writes an audit record.",
  "citations": [
    {"file": "payments/PaymentController.java", "lines": "40-72"},
    {"file": "payments/ReversalService.java", "lines": "15-88"}
  ],
  "confidence": "medium",
  "risk_area": "payments",
  "needs_review": true
}
```
The UI shows the answer with clickable citations and a "Pending senior review" badge.

**Example 2**

*Prompt:* "Which module owns KYC checks?"

*Expected response:* The module name with citations to its package and docs. If nothing relevant is retrieved, the app replies: "I couldn't find evidence for this in the indexed code or docs," instead of guessing.

**Format strategy:** the model must return the JSON schema above. If the output doesn't match the schema, the request is retried once, then fails safely.

---

## Step 6: Evaluation Strategy

- **Golden question set:** 50 to 100 real onboarding questions, each with the expected files and a short reference answer, written with senior engineers.
- **Retrieval quality:** recall@k, meaning how often the expected source is in the retrieved chunks.
- **Answer quality:** groundedness (is every claim supported by a citation?), correctness against the reference, and citation accuracy. Scored by an LLM judge and spot-checked by humans.
- **Safety checks:** the assistant must refuse when no evidence exists, and must never output secrets or PII.
- **Human review:** senior engineers rate a sample each sprint.
- **Regression gating:** the suite runs whenever prompts, chunking, or models change.

---

## Step 7: Observability Plan

- **Tracing:** log each request's retrieved chunks, prompts, model, output, and verifier result (with sensitive data redacted).
- **Metrics:** latency, token cost, error rate, "I don't know" rate, and share of answers sent to review.
- **User feedback:** thumbs up/down with a reason, linked back to the trace.
- **Dashboards and alerts:** alert on rising error rates, falling feedback scores, or retrieval returning nothing.
- **Audit log:** who asked what and who approved which answer, since banking needs auditability.

---

## Risks and Limitations

| Risk | Mitigation |
|---|---|
| Wrong or invented answers | Citations required, verifier step, refuse without evidence |
| Leaking confidential code or PII | Redaction, private model, access control, no real customer data |
| Stale index | Scheduled re-indexing, show last-indexed time |
| Over-trust by new engineers | Review badges, clear "AI-generated" label |

## Future Improvements

- Auto-generated onboarding paths per team or module
- IDE plugin
- Detecting outdated docs by comparing them to the code
