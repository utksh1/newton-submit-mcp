# AGENTS.md — newton-submit-mcp

Instructions for any AI agent working in this repository. Read this fully before
changing code or calling the Newton School API. It records the verified API
workflows so they do not need to be rediscovered.

## 1. What this project is

A Model Context Protocol (MCP) server (TypeScript, Node 16 modules, ESM) that
wraps Newton School platform APIs, GitLab repository operations, and the
Newton evaluation runner. It exposes 14 tools over stdio.

Layout:

- `src/index.ts` — the entire server: tool definitions, request handlers, all HTTP calls.
- `build/index.js` — compiled, executable output. This is the MCP entry point.
- `dist/server.js` — older artifact; not the current entry point. Do not target it.
- `package.json`, `tsconfig.json` — scripts and compiler settings.
- `README.md` — user-facing install/config reference.

Build/verify:

```bash
npm install
npm run build      # tsc && chmod +x build/index.js
test -x build/index.js
```

Do not edit `build/index.js` directly — it is regenerated from `src/index.ts`.

## 2. Credentials — hard rules

- The Newton token is the user's credential. Obtain it from the user; never echo
  it into chat, logs, commits, MCP config files, or documentation.
- Canonical location: `~/.newton-submit-mcp/auth.json` with mode `0600`:
  `{"token": "<value>"}`.
- `src/index.ts` auto-loads that file at startup, so MCP host config does NOT
  need an `env` entry. Prefer this over putting a token in config.
- Every API call uses `Authorization: Bearer <token>`.
- Verify a token with `GET /api/v1/user/me/` — HTTP 200 and a profile means it works.

Host wiring:

Add a stdio server entry to your MCP client configuration:

```json
{
  "mcpServers": {
    "newton-submit": {
      "command": "node",
      "args": ["/absolute/path/to/build/index.js"]
    }
  }
}
```

Use an absolute path to `build/index.js`. Restart the client to load the tools;
some hosts prefix tool names with the configured server name.

## 3. Repository and course discovery

- `GET /api/v2/course/all/applied/?pagination=false` → enrolled parent courses.
- A parent course's `children_courses.admin_unit_courses[]` maps semesters to
  component courses via `learning_unit_courses[]`. Semester 3 of
  `Newton School of Technology'25 (CS+AI) (RU)` is admin unit `70hmqbf3t250`,
  with 9 component courses (ADA, ADA Lab 2, AP, AP Lab 2, DE, DE Lab 2, LHL,
  Maths III, Maths III Lab 2).
- `GET /api/v1/course/h/{ch}/assignment/all/?pagination=false&completed=false`
  appears to return assignments for ALL enrolled courses, not just `{ch}`.
  Always filter the result by `assignment.course.hash` for the courses you want.
- `completed=false` does NOT mean unsolved. Inspect each question's
  `attempt_status`: `1` = unattempted, `2` = attempted, `3` = solved. Treat only
  `1` and `2` as work to do. (This distinction is the single most common mistake.)

## 4. Question discovery — VERIFIED endpoint chain

1. `GET /api/v1/course/h/{ch}/assignment/h/{ahash}/details/`
   - `assignment.hash`, `assignment.assignment_questions[]` with `hash`,
     `question_title`, `question_text`, `input`, `output`, `example`,
     `constraints`, `question_type_text`, and
     `assignment_question_language_mappings[]` (`language_id`, `language_text`,
     `pre_function_code`, `function_code`, `post_function_code`).
   - The shorter `/assignment/h/{ahash}/` (no `/details/`) returns 404.
2. `GET /api/v1/course/h/{ch}/assignment/h/{ahash}/question/h/{qhash}/details/`
   - Top-level `hash` IS the coding-playground hash. This is the missing link
     between an assignment question and a submittable playground.
   - Transient HTTP 409 occurs under concurrency: retry with backoff.
   - `type` field identifies the question kind: `code`, `database`, `newton-box`,
     `subjective`, `project`.
3. Alternative broad listing: `GET /api/v2/course/h/{ch}/assignment/all/?pagination=false&completed=false`.

## 5. Submission workflows — VERIFIED

### 5.1 Coding and database playgrounds

`PATCH https://my.newtonschool.co/api/v1/playground/coding/h/{playgroundHash}/?run_hidden_test_cases=true`

Body: `{"source_code": "...", "language_id": N}`

- Coding questions: submit ONLY the function definition matching
  `function_code` exactly (name, parameters, return). Do not include
  `pre_function_code`, `post_function_code`, `if __name__ == "__main__"`, or a
  test harness. Python is language_id `71`, C++ `54`. All Sam/ADA coding
  questions offered Python; prefer it.
- Database questions: submit ONE statement/expression, never several
  semicolon-separated ones. MongoDB shell is `2003` (e.g.
  `db.coll.aggregate([...])`); PostgreSQL 16 is `2004` (a SELECT). Read
  `pre_function_code` for exact collection/table names and seed data.
- Poll `GET /api/v1/playground/coding/h/{hash}/latest_submission/?username=<user>`
  until `current_status != 1`. `current_status: 3` = Accepted; check
  `number_of_test_cases_passing` and `wrong_submission`.
- The MCP `submit_solution` tool wraps this endpoint but expects a PLAYGROUND
  hash, not an assignment/question hash. Passing an assignment hash gives 404.

### 5.2 Subjective (text) questions

`PATCH /api/v1/playground/subjective/h/{playgroundHash}/`

Body: `{"text_field": "<answer>", "create_evaluable_submission": true}`

Without `create_evaluable_submission` the answer is only saved as a draft.
Poll `/api/v1/playground/subjective/h/{hash}/latest_submission/`; results land
in `ai_evaluation_json` (correctness/completeness scores).

### 5.3 Newton Box (project) questions

`GET /api/v1/playground/project/h/{hash}/` returns `git_lab_project_id` and a
per-project `access_token` (GitLab PAT), plus `question_text`, `boilerplate`.

1. `GET https://gitlab.newtonschool.co/api/v4/projects/{id}/repository/tree?recursive=true&per_page=200`
   with header `PRIVATE-TOKEN: <access_token>`.
2. Read a file: `/repository/files/{urlencoded_path}?ref=main`; `content` is base64.
3. Write: `PUT /repository/files/{path}` with
   `{"branch":"main","content":"<full file>","commit_message":"..."}`. If GitLab
   returns 400 for a missing file, retry as `POST` to create it.
4. Do not modify `.env`, `package.json`, `package-lock.json`, `.gitignore`,
   `nodemon.json` unless the question explicitly requires it (some Prisma seed
   questions require a `prisma.seed` entry in `package.json`).
5. Trigger: `PATCH /api/v1/playground/project/h/{hash}/?create_and_run_build=true`.
6. Poll `GET /api/v1/playground/project/h/{hash}/latest_submission/?username=<user>`
   until `build_status != 1`. `build_status: 3` + `all_test_cases_passing: true`
   is success. `5` = Internal Error (often runner-side, retry once).

### 5.4 GitHub View projects — known gap

When `type == "project"` and `sub_type_text == "Git Hub View"`, the project
endpoint has no `git_lab_project_id`/`access_token`. Submission requires an
external GitHub repository or hosted URL through the web UI
(`/playground/project/{hash}` → "Clone Repository" / "Hosted Project Link").
The `POST /api/v1/playground/project/h/{hash}/clone-repo/` call needs an
`assignment_question_git_repo_mapping_hash`. Fully automated submission for
this type is NOT established — treat it as a manual/browser workflow.

## 6. Batch solving pattern (large assignment sets)

Proven approach for hundreds of questions:

1. Collect once: fetch assignment details + playground hashes for every
   unsolved question with a bounded thread pool; persist to a JSON file.
   Checkpoint progress to disk so retries resume instead of restarting.
2. Generate solutions in parallel batches (separate workers per ~20 questions).
   Give workers the full question text, seed data, function template, and
   output-format rules; have them write JSON keyed by question hash.
3. Submit with low concurrency (4–5 workers), poll each result, and record
   per-question status to a results JSON. Skip already-accepted items on retry.
4. Re-feed failures into a second, smaller fix batch with test-case feedback.
   Common causes: multiple SQL statements, wrong function signature, wrong
   response body shape, strict header parsing.
5. Never fabricate results — only report accepted after reading runner status.

Reference implementations (outside this repo, in the scratch workspace):
`collect_s3.py`, `submit_s3.py`, `submit_subjective.py`, `collect_nb.py`,
`apply_nb.py`. They demonstrate the checkpoint-and-resubmit pattern.

## 7. Pitfalls

- Assignment hash ≠ question hash ≠ playground hash. Always resolve via the
  `/question/h/{qhash}/details/` endpoint.
- v1 assignment list 404s for some course types; v2 is the reliable listing,
  but v2 assignment detail is not — use the v1 `/details/` route above.
- HTTP 409 on question details is transient; retry.
- Runner marks some files read-only in the browser editor even when the GitLab
  API can write them; only deviate when the question requires it.
- `get_file`/`update_file` MCP tools expect a GitLab numeric project ID, not a
  playground hash.
- `wait_for_submission` polls the *project* endpoint; coding submissions need
  the *coding* endpoint.
- Keep generated answers and scratch JSON out of git; only source and docs change.

## 8. Adding or changing an MCP tool

1. Add the tool descriptor to `ListToolsRequestSchema` in `src/index.ts`.
2. Add a `case` in `CallToolRequestSchema` that resolves the token, builds the
   URL, calls `fetch`, and returns `{ content: [{ type: 'text', text }] }`.
3. Return `isError: true` with the error message on failure; never throw raw
   token-bearing URLs.
4. Rebuild with `npm run build` and re-test with a stdio JSON-RPC `tools/call`
   request, for example `get_user_info`.

## 9. Style and safety

- TypeScript strict mode; match existing code style in `src/index.ts`.
- No secrets in source, docs, config, or git history.
- Do not commit generated solution files, tokens, or scratch artifacts.
- Verify real execution (build + at least one live API call) before claiming success.
