# Knowledge Chatbot (RAG) — Detailed Functional Requirements

## 1. Overview

Build an embeddable chat widget (a React component) that answers visitor questions using one
company's own knowledge base. The MVP supports text and Markdown documents only. Each deployment
of the widget is configured with exactly one persona and one knowledge base. There is no
end-user login, and the bot has no memory beyond a single visit.

## 2. Deployment Model

- Each deployment has **one persona and one knowledge base**, chosen by the company when it
  embeds the widget. The persona is fixed for the deployment; it cannot switch at chat time.
- A deployment belongs to one company/site and has its own isolated set of documents. A visitor
  on Company A's site can only ever retrieve Company A's documents.
- There are no visitor accounts. The only "user" who configures anything is the company that
  owns the deployment.

## 3. Data Model

| Entity | Key fields | Notes |
|---|---|---|
| `Deployment` | id, company_id, persona_name, persona_system_prompt, embed_key, max_history_turns (default 10), rate_limit_per_minute (default 10) | One per company/site. Persona name and tone are configurable per deployment. |
| `Document` | id, deployment_id, filename, type (txt/md), raw_text, uploaded_at, version | MVP supports `.txt` and `.md` only. `version` increments each time the document is re-uploaded (see §4.1, step 4). |
| `Chunk` | id, document_id, document_version, text, embedding_vector | Created at upload time and scoped to one deployment's documents. Tied to a specific document `version`. |
| `ChatSession` | id, deployment_id, visitor_name (nullable), started_at, ended_at | Exists only for the current visit. No cross-visit identity or persistence. |
| `Message` | id, session_id, role (user/assistant), text, source_document_ids[], cost, feedback (nullable) | `source_document_ids` applies to assistant messages only. `feedback` is `up`, `down`, or null. |

## 4. Core Flow

### 4.1 Setup (company/admin side, once per deployment)

1. A company creates a deployment: it names a persona and uploads one or more `.txt`/`.md`
   documents as its knowledge base.
2. The system chunks each document, creates embeddings, and stores them against that deployment
   only.
3. The company embeds the chat widget on its site using the deployment's embed key.
4. **Re-uploading or editing a document:** the system re-chunks and re-embeds only that document
   and increments its `version`. Old chunks for that document are removed. Retrieval runs fresh
   on every message (see §4.2), so there is no cache to invalidate. The next message in any
   session, new or in progress, uses the updated chunks.

**Upload limits (MVP):** at most 100 documents per deployment, and at most 10 KB per document.
Reject anything larger or any other file type with a clear error message.

### 4.2 Runtime (visitor side, every chat session)

5. A visitor opens the widget. A new `ChatSession` starts. There is no login and no
   returning-visitor memory.
6. The bot opens with one combined prompt asking for identity and intent in a single turn, for
   example: "I am {persona_name}, please tell me your name, also please state your query." This
   is one prompt, not a multi-step form. The visitor's name is stored in `visitor_name` if given.
7. For every visitor message, including follow-ups:
   - **Retrieve:** the system finds the most relevant chunks from **this deployment's own
     knowledge base only**. It uses recent conversation turns as context so follow-ups resolve
     correctly. For example, "what are the sample projects" asked right after a question about a
     specific course must resolve to that course, not run as a fresh unrelated search.
   - **Cap history:** the conversation history sent to the model is limited to the deployment's
     `max_history_turns` (default: last 10 message pairs). Older turns drop out of the context
     sent to the AI but stay in the stored `Message` log untouched.
   - **Generate:** the retrieved chunks, the capped history, and the question are sent to the AI.
     The AI answers **in its own words and cites which document(s) the answer came from**. It
     must paraphrase and synthesize, never quote a document verbatim, so a company's internal
     documents are not exposed word-for-word through the chat.
   - **Fallback:** if the retrieved chunks do not confidently support an answer, the bot must say
     it does not know, for example "I couldn't find that in our documents." It must not answer
     from general knowledge outside the deployment's documents. Treat the answer as unsupported
     when the best-matching chunk's similarity score is below a configurable threshold
     (`MIN_RELEVANCE_SCORE`, a constant in config). Pick the starting value by testing against
     your sample questions, and record the value you chose in your README.
   - **Log:** the system logs the question, answer, source documents, and the per-message cost.
8. Session memory lasts for the current visit only. Closing and reopening the widget (or
   returning another day) starts a fresh session with no memory of any prior visit.
9. **Feedback:** each assistant answer shows a thumbs up and a thumbs down. A visitor can click
   one (or neither). The choice is saved in `Message.feedback` (`up` or `down`). The visitor can
   change their choice, but each message holds only one value.
10. **Rate limiting:** each deployment enforces a per-visitor-session limit of
    `rate_limit_per_minute` messages (default: 10). When a visitor exceeds it, return HTTP 429
    and show a friendly "please wait a moment" message in the widget. This protects a public,
    anonymous, cost-incurring endpoint from abuse and is in MVP scope.

## 5. Business Rules Summary

- Retrieval is strictly scoped to the active deployment's own documents, never across companies.
- One persona and one knowledge base per deployment; no runtime switching.
- Every assistant answer cites its source document(s), in the AI's own synthesized words, never
  as a verbatim quote of the source text.
- If the documents do not support an answer, the bot says it does not know instead of guessing.
- Session identity and memory never outlive the visit that created them.
- History sent to the model is capped per deployment; the full history is still logged.
- A document update takes effect immediately for all retrieval, in new and in-progress sessions.
- Every deployment is rate-limited per visitor session.

## 6. Non-Functional Requirements

- **Multi-tenancy isolation** is the most important requirement. Chunk retrieval must be filtered
  by `deployment_id` **in the database/vector-store query itself**, not just in the UI or in
  application code after the fact. Write a test that proves Company A's question can never
  return Company B's chunks.
- **Cost logging is internal only in the MVP.** Store `cost` on every assistant `Message`, but do
  not build a company-facing usage dashboard. The data is logged from day one so pricing
  decisions can use it later.
- **Abuse prevention** in the MVP means the per-session rate limit in §4.2. IP-level throttling,
  CAPTCHA, and similar measures are extension scope.

## 7. Out of Scope for MVP (Extension Ideas)

- PDF and Docx document support.
- More than one document collection active per deployment at once.
- Per-end-user private knowledge bases.
- Memory that persists across a visitor's separate visits.
- A company-facing usage/cost dashboard.
- Stronger abuse prevention beyond simple per-session rate limiting.

## 8. Test Knowledge Base

A shared set of real documents for testing and demos will be provided by your instructor later.
Until then, build and test with your own sample `.txt`/`.md` files within the §4.1 limits, and
name the set you used in your README.

## 9. Acceptance Checklist

Your MVP is done when all of these are true:

- [ ] A company can create a deployment, set a persona, and upload `.txt`/`.md` files within the
      size and count limits (§4.1).
- [ ] Oversized or unsupported files are rejected with a clear message.
- [ ] The widget embeds on a plain HTML page using only the deployment's embed key.
- [ ] The bot opens with the combined name-and-query prompt (§4.2, step 6).
- [ ] A follow-up question that depends on the previous turn is answered correctly.
- [ ] Answers are paraphrased and name their source document(s).
- [ ] A question the documents cannot answer gets the "I don't know" response.
- [ ] Only the last `max_history_turns` message pairs are sent to the model, while all messages
      are stored.
- [ ] Re-uploading a document changes answers on the very next message, with no restart.
- [ ] A test proves one deployment cannot retrieve another deployment's chunks.
- [ ] Thumbs up/down is saved per message.
- [ ] Exceeding the rate limit returns HTTP 429 and a friendly message in the widget.
- [ ] Question, answer, sources, and cost are logged for every assistant message.
- [ ] Your README states the `MIN_RELEVANCE_SCORE` you chose and the knowledge base you tested with.
