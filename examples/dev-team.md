# Dev Team: Architect + Frontend + Backend + QA

**AGENT TEAMS (requires flag)**

This is a true Agent Teams example. It requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` and Opus 4.6. Teammates run as peers with independent context windows, a shared task list, and direct lateral messaging. This is not the subagent pattern.

If you want the subagent equivalent (Task tool, no flag needed), the structure is the same but teammates cannot message each other directly. See the note at the bottom.

---

## The Team Prompt

Paste this into Claude Code (Opus 4.6, flag enabled):

```
Create an agent team to build a user authentication system.

Spawn four teammates:

- "architect" to design the full technical spec: API endpoints, data models, auth flow, and file structure. Architect starts immediately. When the spec is complete, message frontend and backend simultaneously with the full spec document. Architect should NOT write any implementation code.

- "frontend" to implement the React TypeScript components for auth: login form, registration form, password reset form, and session management. Frontend is blocked until architect sends the spec. If frontend finds an ambiguity in the spec, message architect directly to resolve it rather than guessing. Frontend should NOT implement API endpoints or database logic.

- "backend" to implement the FastAPI Python endpoints and SQLAlchemy models. Backend is blocked until architect sends the spec. If backend finds a gap in the spec, message architect directly. Backend should NOT implement frontend components.

- "qa" to write a comprehensive test suite covering all API endpoints and frontend components. QA is blocked until architect sends the spec (QA can write tests from the spec without waiting for implementation). When QA finds a test failure against the actual implementation, message backend or frontend directly with the specific failure and expected behavior.

The feature requirements:
- Email/password registration
- JWT-based login (tokens expire in 1 hour)
- Password reset via email
- Session management: logout and token refresh
- Tech stack: React TypeScript frontend, Python FastAPI backend, PostgreSQL database, bcrypt password hashing

Coordinate the team. Tell me when all four teammates have marked their tasks complete.
```

---

## Expected Team Structure

When the team is running, the shared task list at `~/.claude/tasks/dev-team/` will look like this:

```
Task #1: Design auth system spec     [architect]   status: claimed
Task #2: Implement frontend          [frontend]    status: blocked (by #1)
Task #3: Implement backend           [backend]     status: blocked (by #1)
Task #4: Write test suite            [qa]          status: blocked (by #1)
```

After architect completes:

```
Task #1: Design auth system spec     [architect]   status: complete
Task #2: Implement frontend          [frontend]    status: claimed
Task #3: Implement backend           [backend]     status: claimed
Task #4: Write test suite            [qa]          status: claimed
```

All three unblocked tasks self-claim and run simultaneously. This is the fan-out from a single completed task.

---

## What the tmux Split-Pane View Looks Like

When Agent Teams is running correctly, you see four active panes. Each pane is a separate Claude Code instance.

```
+---------------------------+---------------------------+
| PANE 1: team-lead         | PANE 2: architect         |
|                           |                           |
| Team created.             | Claiming Task #1...       |
| Teammates spawned: 4      | Reading requirements...   |
| Watching task list...     | Designing API endpoints:  |
|                           | POST /auth/register       |
|                           | POST /auth/login          |
|                           | POST /auth/logout         |
|                           | POST /auth/refresh        |
|                           | POST /auth/password-reset |
|                           |                           |
|                           | Sending spec to frontend  |
|                           | and backend...            |
+---------------------------+---------------------------+
| PANE 3: frontend          | PANE 4: backend + QA      |
|                           |                           |
| Waiting for architect...  | backend: waiting...       |
| @architect spec received  | qa: waiting...            |
| Claiming Task #2...       |                           |
| Building LoginForm.tsx... | @architect spec received  |
|                           | backend: claiming Task #3 |
|                           | qa: claiming Task #4      |
|                           |                           |
|                           | backend: implementing     |
|                           | POST /auth/register...    |
|                           |                           |
|                           | qa: writing pytest for    |
|                           | /auth/register...         |
+---------------------------+---------------------------+
```

Press `ctrl+t` to toggle a team status overlay showing all teammates and their current task.

Navigate panes with `Shift+Up` and `Shift+Down`.

---

## How Lateral Messaging Flows in This Team

### Architect to Frontend and Backend (Phase 1 handoff)

When architect completes the spec, it sends a lateral message to both peers simultaneously. This is the key coordination event. The team lead does not need to relay the spec.

Architect's outbound messages (you see these in architect's pane):

```
@frontend) Spec complete. Here is the full architecture document:

API Endpoints:
  POST /auth/register - body: {email, password} - returns: {token, user_id}
  POST /auth/login - body: {email, password} - returns: {access_token, refresh_token}
  POST /auth/logout - body: {} - requires: Authorization header
  POST /auth/refresh - body: {refresh_token} - returns: {access_token}
  POST /auth/password-reset - body: {email} - returns: {message}

Components to build:
  RegisterForm.tsx - fields: email, password, confirm_password
  LoginForm.tsx - fields: email, password
  PasswordResetForm.tsx - fields: email
  SessionManager.tsx - handles: token storage, refresh logic, logout

Task #1 marked complete. Your Task #2 is now unblocked.
```

```
@backend) Spec complete. Same architecture document as I sent frontend:
[same spec content]
Task #1 marked complete. Your Task #3 is now unblocked.
```

### QA to Backend (test failures)

When QA runs tests against backend's actual implementation and finds failures, it messages backend directly. The team lead does not mediate.

QA's outbound message (you see this in QA's pane):

```
@backend) Test failure on POST /auth/register:

Expected: 409 Conflict when email already exists
Got: 500 Internal Server Error

Test: test_register_duplicate_email
File: /tmp/dev-team/tests/test_auth.py line 47

The endpoint is not catching the IntegrityError from PostgreSQL. You need a try/except around the INSERT and return a proper 409 response. I cannot mark my task complete until this passes.
```

Backend receives this in its pane, fixes the issue, and replies:

```
@qa) Fixed. Added IntegrityError handling with 409 response. Re-run your test against the updated endpoint at /auth/register.
```

### Frontend to Architect (spec ambiguity)

If frontend encounters an ambiguity in the spec, it messages architect directly:

```
@architect) Spec question: the auth flow shows token storage but does not specify where. LocalStorage has XSS exposure risk; httpOnly cookies require server changes. Which do you want for the access_token? This blocks my SessionManager implementation.
```

Architect replies:

```
@frontend) Use httpOnly cookies for access_token. I will add a Set-Cookie header to the login endpoint response in the spec. Backend will need to handle this too — I am messaging them now.
```

---

## When It Is Working vs. Not Working

**Working correctly:**
- Four panes active simultaneously after architect completes Task #1
- You see `@frontend` and `@backend` appear in architect's pane output
- The task list updates without you doing anything
- QA's failing test messages appear in backend's pane, not in the team lead pane

**Not working (flag not set or misconfigured):**
- Only one pane active at a time
- No `@teammate` messages visible anywhere
- Task list does not update automatically
- You have to manually hand off the architect's spec to frontend and backend

---

## Output Location

Write files to `/tmp/dev-team/` before spawning the team. Each teammate writes to their own subdirectory:

```bash
mkdir -p /tmp/dev-team/frontend/
mkdir -p /tmp/dev-team/backend/
mkdir -p /tmp/dev-team/tests/
mkdir -p /tmp/dev-team/spec/
```

Architect writes the spec to `/tmp/dev-team/spec/auth-spec.json` and includes the path in its lateral messages. Teammates read from that file rather than receiving the full spec inline.

---

## Adapting This for Other Features

The structure works for any feature build. To adapt:

1. Replace the authentication requirements with your feature requirements
2. Keep the four roles (architect, frontend, backend, qa) or adjust names for your stack
3. Keep the lateral message triggers: architect to frontend+backend when spec done, QA to relevant implementer when tests fail
4. Add a fifth "devops" teammate if the feature requires infrastructure changes (blocked by backend)

The architect-first sequencing is intentional and should not be removed. Without a shared spec from architect, frontend and backend make incompatible assumptions and produce work that cannot be integrated.

---

## Subagent Equivalent (no flag needed)

**SUBAGENTS (no flag needed)**

If you cannot use the Agent Teams flag, you can approximate this with the Task tool. The structure is the same but lateral messaging is replaced by the orchestrator relaying messages manually.

The orchestrator pattern:

1. Spawn architect as a subagent. Wait for it to return the spec.
2. Read the spec. Pass it explicitly to frontend, backend, and QA as parallel subagents.
3. Collect all three outputs. Check for failures.
4. If QA reports failures, spawn a new backend subagent with the failure details.

The key limitation: you (the orchestrator) must relay all coordination. This adds latency and can introduce information loss if you summarize rather than pass the full output.
