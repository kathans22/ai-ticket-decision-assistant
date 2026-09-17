# MaxsorLabs Assignment: AI Support-Ticket Decision Assistant

## Overview
Build an end-to-end AI Support-Ticket Decision Assistant:
- **Frontend**: Streamlit (talks to FastAPI exclusively over HTTP)
- **Backend**: FastAPI (Python 3.12, Uvicorn, sync endpoints calling threadpools for LLM calls)
- **Auth**: User registration/login with JWT (pwdlib[argon2] + PyJWT), authorization checks (users cannot access others' tickets)
- **Database**: SQLite with SQLAlchemy 2.0 (typed Mapped style, sync, foreign keys enforced, cascade)
- **RAG Engine**: Heading-aware chunking of policy markdown files, Gemini embeddings, SQLite storage with NumPy cosine/KNN search
- **AI Decision Pipeline**: Gemini model via google-genai SDK, structured Pydantic output, guardrails, handling insufficient information (`NEEDS_MORE_INFORMATION` or exact action from discovery)
- **Testing & Eval**: Offline pytest suite with fake LLM/embedder, plus evaluation runner

## Notes & Discrepancies
- The original brief text may mention `refunds.md`, but the candidate data pack contains 6 specific policy files:
  - `cancellations.md`
  - `damaged_goods.md`
  - `defective_products.md`
  - `returns.md`
  - `shipping.md`
  - `wrong_item.md`
The implementation must build against the actual policy pack files.
