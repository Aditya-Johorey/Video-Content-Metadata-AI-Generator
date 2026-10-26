# pulseflow.ai — Implementation Plan

An AI agent that analyses a video, generates a separate content package per platform, lets the user review or request changes, and publishes to connected accounts.

This plan is executed **one stage at a time**. Each stage ends with passing tests and a short checkpoint before the next begins. Update the status table as stages complete.

## Status

| Stage | Name | Status |
|---|---|---|
| 0 | Setup and foundations | Not started |
| 1 | Authentication | Not started |
| 2 | Video upload and storage | Not started |
| 3 | Job system and progress polling | Not started |
| 4 | LangGraph pipeline | Not started |
| 5 | LangSmith evaluation | Not started |
| 6 | Review UI and revision loop | Not started |
| 7 | Connected accounts (OAuth) | Not started |
| 8 | Publishing | Not started |
| 9 | Cleanup, hardening, end-to-end | Not started |

---

## Decisions (locked)

| Area | Decision |
|---|---|
| Name | pulseflow.ai (UI mockups were exported as "PulseDrop"; rebrand during Stage 1 UI work) |
| Platforms at launch | YouTube Shorts, Instagram Reels, TikTok. X, Reddit, Pinterest deferred. |
| Publishing | Fully real: actual OAuth and live posting, no mocks in production paths |
| Auth | Google sign-in or email and password; login required |
| Video | Never modified (no crop, trim, re-frame). Accepts short-form and horizontal, up to 5 minutes and 1 GB. |
| Incompatible formats | Warn and disable the platforms the video doesn't fit |
| Content | Separate package per platform, each editable on its own |
| Thumbnails | Several candidates: AI-generated plus video frames. User chooses. TikTok supports frame selection only. |
| Revisions | Reuse the saved video analysis; rerun only content agent and thumbnails |
| Publish failures | Per-platform status; successes stay live; failures show the error and a manual Retry |
| Timing | Immediate publish only |
| History | None. One video per session. Assets deleted after publishing or inactivity. |
| Billing | None |
| Progress updates | Polling |
| Stack | React (Vite) + FastAPI + Firebase (Auth, Storage, Firestore) |
| Pipeline | LangGraph |
| Evaluation / tracing | LangSmith |
| AI providers | Gemini by default for video analysis and image generation, behind swappable interfaces. Content agent: Gemini or Claude. |
| UI | Stitch mockups as a style guide ("Electric Velocity"), not a rigid layout. Cut features removed. |

## Platform rules (defaults, kept in one config file)

| Platform | Shape | Length | Notes |
|---|---|---|---|
| YouTube Shorts | Square or vertical | Up to 3 min | Horizontal becomes a regular video, not a Short |
| Instagram Reels | 9:16 | 3 to 90 s default (verify) | 300 MB default (verify); Business/Creator account required |
| TikTok | Vertical recommended | Per creator, fetched live from `creator_info` | File size cap to verify |

Re-verify these against official documentation before Stage 8.

## External approvals (start in Stage 0)

- **TikTok Content Posting audit.** Unaudited apps can post privately only and are capped at a few users per day. The audit can take weeks.
- **Meta app review** for Instagram publishing, plus a Business or Creator account linked to a Facebook Page.
- **YouTube API audit** may be needed for public uploads from new projects.

Until approvals land, develop and test against your own accounts in private/test mode.

---

## Stage 0 — Setup and foundations

**Goal:** A running skeleton for frontend and backend, with tooling and secrets in place.

**Tasks**
- Create the monorepo: `/frontend` (React + Vite), `/backend` (FastAPI), `/evals`.
- Set up the Firebase project (Auth, Firestore, Storage) and a service account.
- Add config loading from environment variables and `.env.example` files.
- Create LangSmith and Gemini API keys.
- Add linting and formatting (ruff, eslint, prettier) and test runners (pytest, vitest).
- Submit TikTok audit and Meta app review applications.

**Test checkpoint**
- `GET /health` returns 200.
- The React app renders.
- One pytest and one vitest smoke test pass.

---

## Stage 1 — Authentication

**Goal:** Users can sign up and sign in; the backend trusts only verified users.

**Tasks**
- Firebase Auth with Google and email/password.
- React login and sign-up page in the Electric Velocity style (rebrand to pulseflow.ai).
- Route guards and an auth context.
- FastAPI dependency that verifies Firebase ID tokens.

**Test checkpoint**
- Protected endpoint returns 401 with no or invalid token and 200 with a valid one.
- Manual: sign up, sign in, sign out in the browser.

---

## Stage 2 — Video upload and storage

**Goal:** Users upload a valid video that is stored and described, untouched.

**Tasks**
- Upload to Firebase Storage with progress.
- Create a session record in Firestore (user, file path, status).
- Server-side validation: size up to 1 GB, duration up to 5 minutes, supported container.
- Read-only metadata extraction with ffprobe (duration, resolution, aspect ratio, codec, size).
- Prompt input and the upload page UI.

**Test checkpoint**
- Unit tests for valid and invalid files (too large, too long, wrong type).
- Metadata extraction test on a fixture video.
- Manual browser upload.

---

## Stage 3 — Job system and progress polling

**Goal:** Long-running work runs in the background and the UI shows progress.

**Tasks**
- Background job runner and a job document (stage, percent, error, result reference).
- `POST /jobs` to start, `GET /jobs/{id}` to poll.
- React polling hook and progress screen with stage list.
- A fake pipeline that advances through stages, to prove the plumbing.

**Test checkpoint**
- Job state transitions are tested, including failure and cancel.
- Progress bar advances through the stages.
- Refreshing the page resumes polling the same job.

---

## Stage 4 — LangGraph pipeline

**Goal:** The real AI pipeline, traced end to end.

**Tasks**
- Define a graph with a typed shared state.
- **Video analysis node** (Gemini): key moments with timestamps and keywords. Runs once per video.
- **Content agent node:** structured output with a separate title, description and hashtags per target platform, plus a thumbnail prompt. Enforces each platform's limits.
- **Thumbnail node:** AI image generation from the thumbnail prompt, plus frame extraction candidates from the video.
- Model access behind interfaces so providers can be swapped.
- LangGraph checkpointer so the analysis result persists for revisions.
- LangSmith tracing enabled from the start.
- Wire the graph into the job system, replacing the fake pipeline.

**Test checkpoint**
- Unit tests for each node with mocked models.
- Schema tests for structured output.
- One live end-to-end run with the trace visible in LangSmith.

---

## Stage 5 — LangSmith evaluation

**Goal:** Measure content quality and catch regressions.

**Tasks**
- Build a dataset of sample videos and prompts with expected properties.
- Evaluators: schema validity, platform-limit compliance (caption length, hashtag count), prompt adherence (LLM judge), thumbnail relevance.
- A runnable experiment script and a baseline score report.

**Test checkpoint**
- The experiment runs and reports scores.
- Evaluators are tested on known-good and known-bad examples.

---

## Stage 6 — Review UI and revision loop

**Goal:** Users review, edit, and request changes.

**Tasks**
- Per-platform editable packages (title, description, hashtags).
- Thumbnail picker with AI and video-frame candidates.
- Compatibility check using the platform rules config; incompatible platforms are disabled with a reason.
- Change-request prompt box that starts a revision job.
- Revision reruns only the content and thumbnail nodes, reusing the saved analysis.

**Test checkpoint**
- Compatibility logic unit tests across shape and length cases.
- A revision does not call video analysis again (verified in traces and with a test).
- UI edits persist and the new results display.

---

## Stage 7 — Connected accounts (OAuth)

**Goal:** Users connect their YouTube, Instagram, and TikTok accounts securely.

**Tasks**
- OAuth flows for each platform with state/CSRF protection.
- Encrypted token storage and refresh logic (TikTok tokens expire quickly).
- Connect/disconnect UI.
- Gate publishing on connected accounts.

**Test checkpoint**
- Tokens are encrypted at rest (verified in a test).
- Refresh works for an expired token.
- Manual: connect and disconnect each platform with your own test accounts.

---

## Stage 8 — Publishing

**Goal:** Approved packages are posted, with clear per-platform results.

**Tasks**
- One publisher per platform behind a common interface.
- TikTok: query `creator_info`, check the maximum duration, upload, and poll publish status.
- Instagram: create a container, upload, wait for processing, and publish.
- YouTube: resumable upload with thumbnail.
- Per-platform status, live links, error reasons, and manual retry.
- Run in private/test mode until audits pass.

**Test checkpoint**
- Mocked-API tests for success, partial failure, and retry.
- One real private post per platform using your test accounts.

---

## Stage 9 — Cleanup, hardening, end-to-end

**Goal:** Production readiness.

**Tasks**
- Delete videos and generated assets after publishing or inactivity.
- Rate limiting, input validation review, and clear error states.
- Logging and basic monitoring.
- Full end-to-end test: sign in, upload, generate, review, revise, publish.
- Deployment (frontend and backend) and environment setup docs.

**Test checkpoint**
- The end-to-end test passes.
- A cleanup test confirms no leftover files or records.

---

## Risks

- **Platform approvals** may delay public posting; keep development in private mode.
- **API limits change.** Platform rules live in one config file and are re-verified before publishing work.
- **Token security.** OAuth tokens for three platforms must be encrypted and handled server-side only.
- **Cost.** Video analysis and image generation cost money per run; revisions reuse analysis to limit this.
- **Source disagreement** on Instagram limits; confirm against Meta's official reference.

## Working agreement

- Each stage starts with a short summary of what will be built and how.
- If requirements for a stage change, the stage's requirements are rewritten before building.
- If a decision changes mid-stage, this plan's Decisions table and the stage section are updated.
- Every section ends with tests; stages finish only when their checkpoint passes.
