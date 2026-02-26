# Dev Team: Architect + Frontend + Backend + Testing Agents

A software development team that designs and implements a feature in parallel. The architect runs first to define the contract that all other agents share. Frontend, backend, and testing agents then run in parallel against that contract.

---

## Team Structure

```
Phase 1 (Sequential):
  Orchestrator provides feature spec
      |
      v
  Agent 1: Architect
      - Designs system architecture
      - Defines API contracts
      - Specifies data models
      - Returns shared contract for all agents

Phase 2 (Parallel, all receive architect output):
  Agent 2: Frontend Developer
      - Implements UI components
      - Writes frontend code to shared contract
  Agent 3: Backend Developer
      - Implements API endpoints
      - Implements business logic
      - Writes to shared contract
  Agent 4: Test Engineer
      - Writes tests for all layers
      - Uses contract to write accurate tests

Phase 3 (Sequential):
  Orchestrator
      - Reviews all outputs
      - Checks contract compliance
      - Produces integration summary
```

---

## Setup

```bash
# Create shared output directory before spawning agents
mkdir -p /tmp/dev-team/
mkdir -p /tmp/dev-team/frontend/
mkdir -p /tmp/dev-team/backend/
mkdir -p /tmp/dev-team/tests/
```

---

## Feature Spec (Input to Orchestrator)

```json
{
  "feature": "User authentication system",
  "requirements": [
    "Email/password registration",
    "JWT-based login",
    "Password reset via email",
    "Session management (logout, token refresh)"
  ],
  "tech_stack": {
    "frontend": "React with TypeScript",
    "backend": "Python FastAPI",
    "database": "PostgreSQL",
    "auth": "JWT tokens"
  },
  "constraints": [
    "Passwords must be bcrypt hashed",
    "JWT tokens expire in 1 hour",
    "Refresh tokens expire in 30 days"
  ]
}
```

---

## Phase 1: Task Call - Architect Agent

```
description: "Architect agent - design authentication system"

prompt: |
  You are a Software Architect agent.

  ## Your Job
  Design the technical architecture for a user authentication system. Produce a complete, unambiguous contract that frontend, backend, and test engineers can implement independently without further clarification.

  ## Feature Spec
  {
    "feature": "User authentication system",
    "requirements": ["Email/password registration", "JWT-based login", "Password reset via email", "Session management (logout, token refresh)"],
    "tech_stack": {"frontend": "React with TypeScript", "backend": "Python FastAPI", "database": "PostgreSQL", "auth": "JWT tokens"},
    "constraints": ["Passwords must be bcrypt hashed", "JWT tokens expire in 1 hour", "Refresh tokens expire in 30 days"]
  }

  ## Architecture Output Requirements
  Design must include:
  1. All API endpoints (method, path, request body, response body, error codes)
  2. Data models (all fields, types, constraints)
  3. Frontend component list (component name, props, responsibilities)
  4. Authentication flow diagram (as numbered steps)
  5. File structure for both frontend and backend

  ## Tools You May Use
  None needed - use your knowledge to design.

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "architect",
    "result": {
      "api_endpoints": [
        {
          "method": "POST|GET|PUT|DELETE",
          "path": "string",
          "description": "string",
          "request_body": {"field": "type and description"},
          "response_success": {"status_code": number, "body": {}},
          "response_errors": [{"status_code": number, "condition": "string"}]
        }
      ],
      "data_models": [
        {
          "model": "string",
          "fields": [{"name": "string", "type": "string", "constraints": "string"}]
        }
      ],
      "frontend_components": [
        {
          "component": "string",
          "file_path": "string",
          "props": ["string"],
          "responsibilities": ["string"]
        }
      ],
      "auth_flow": [
        {"step": number, "actor": "frontend|backend|database", "action": "string"}
      ],
      "file_structure": {
        "frontend": ["string file paths"],
        "backend": ["string file paths"]
      },
      "security_notes": ["string"]
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Phase 2: Task Call - Frontend Agent (receives architect output)

```
description: "Frontend developer - implement authentication UI"

prompt: |
  You are a Frontend Developer agent. You write React with TypeScript.

  ## Your Job
  Implement the frontend components for the user authentication system according to the architecture contract provided. Write complete, production-quality code.

  ## Architecture Contract
  [INSERT architect_output["result"] as JSON here]

  ## Your Scope
  Implement only frontend components. Do not implement API endpoints. Do not implement database logic.

  ## Output Location
  Write each component file to: /tmp/dev-team/frontend/
  File naming: use the file_path values from the architecture contract.

  ## Implementation Requirements
  - TypeScript with proper type definitions
  - Form validation on the client side before submitting
  - Error state handling (display API errors to user)
  - Loading states during API calls
  - Use fetch() for API calls (no extra libraries)
  - Each component in its own file

  ## Tools You May Use
  Write (to create files at /tmp/dev-team/frontend/)

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "frontend-developer",
    "result": {
      "files_created": [
        {"file_path": "string", "component": "string", "lines": number}
      ],
      "assumptions_made": ["string"],
      "deviations_from_contract": ["string or empty array"],
      "dependencies_required": ["npm package names"]
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Phase 2: Task Call - Backend Agent (receives architect output)

```
description: "Backend developer - implement authentication API"

prompt: |
  You are a Backend Developer agent. You write Python with FastAPI.

  ## Your Job
  Implement the backend API endpoints for the user authentication system according to the architecture contract provided. Write complete, production-quality code.

  ## Architecture Contract
  [INSERT architect_output["result"] as JSON here]

  ## Your Scope
  Implement only API endpoints and business logic. Do not implement frontend components. Implement the database models as SQLAlchemy models.

  ## Output Location
  Write each file to: /tmp/dev-team/backend/
  File naming: use the file_path values from the architecture contract.

  ## Implementation Requirements
  - FastAPI with Pydantic v2 for request/response models
  - SQLAlchemy for database models
  - bcrypt for password hashing
  - python-jose for JWT operations
  - Proper HTTP status codes for all responses
  - Input validation on all endpoints
  - Each router in its own file

  ## Tools You May Use
  Write (to create files at /tmp/dev-team/backend/)

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "backend-developer",
    "result": {
      "files_created": [
        {"file_path": "string", "description": "string", "lines": number}
      ],
      "endpoints_implemented": ["METHOD /path", ...],
      "assumptions_made": ["string"],
      "deviations_from_contract": ["string or empty array"],
      "dependencies_required": ["pip package names"],
      "environment_variables_required": ["string"]
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Phase 2: Task Call - Test Agent (receives architect output)

```
description: "Test engineer - write tests for authentication system"

prompt: |
  You are a Test Engineer agent.

  ## Your Job
  Write a comprehensive test suite for the user authentication system. Use the architecture contract to write tests that validate every endpoint, component behavior, and edge case. Tests are written before the implementation is complete (test-driven approach).

  ## Architecture Contract
  [INSERT architect_output["result"] as JSON here]

  ## Your Scope
  Write both backend API tests (pytest) and frontend component tests (React Testing Library + Jest). Cover happy paths and all specified error conditions.

  ## Output Location
  Write test files to: /tmp/dev-team/tests/
  Backend tests: /tmp/dev-team/tests/backend/
  Frontend tests: /tmp/dev-team/tests/frontend/

  ## Test Requirements
  Backend (pytest):
  - Test every API endpoint
  - Test every error condition from the contract
  - Test authentication flow end-to-end
  - Include fixtures for database setup/teardown

  Frontend (Jest + React Testing Library):
  - Test form validation behavior
  - Test successful form submission
  - Test error state display
  - Test loading state display

  ## Tools You May Use
  Write (to create files at /tmp/dev-team/tests/)

  ## Output Requirements
  Return ONLY a JSON object:
  {
    "status": "complete",
    "agent": "test-engineer",
    "result": {
      "files_created": [
        {"file_path": "string", "test_count": number, "coverage_area": "string"}
      ],
      "total_tests": number,
      "endpoints_covered": ["string"],
      "edge_cases_covered": ["string"],
      "missing_coverage": ["string or empty array"]
    }
  }
  Do not return markdown. Do not add explanation. Return the JSON only.
```

---

## Phase 3: Integration Review

After all Phase 2 agents complete, orchestrator performs integration review:

```python
import json

architect = json.loads(architect_result)["result"]
frontend = json.loads(frontend_result)["result"]
backend = json.loads(backend_result)["result"]
tests = json.loads(tests_result)["result"]

# Check for contract compliance
deviations = []
if frontend.get("deviations_from_contract"):
    deviations.extend([f"Frontend: {d}" for d in frontend["deviations_from_contract"]])
if backend.get("deviations_from_contract"):
    deviations.extend([f"Backend: {d}" for d in backend["deviations_from_contract"]])

# Check that all endpoints have tests
endpoints_designed = [ep["path"] for ep in architect["api_endpoints"]]
endpoints_implemented = backend.get("endpoints_implemented", [])
endpoints_tested = tests.get("endpoints_covered", [])

missing_implementation = [ep for ep in endpoints_designed if not any(ep in impl for impl in endpoints_implemented)]
missing_tests = [ep for ep in endpoints_designed if not any(ep in t for t in endpoints_tested)]

# Produce integration summary
summary = {
    "status": "complete",
    "files_created": {
        "frontend": [f["file_path"] for f in frontend["files_created"]],
        "backend": [f["file_path"] for f in backend["files_created"]],
        "tests": [f["file_path"] for f in tests["files_created"]]
    },
    "total_files": len(frontend["files_created"]) + len(backend["files_created"]) + len(tests["files_created"]),
    "total_tests": tests["total_tests"],
    "contract_deviations": deviations,
    "missing_implementation": missing_implementation,
    "missing_tests": missing_tests,
    "combined_dependencies": {
        "npm": list(set(frontend.get("dependencies_required", []))),
        "pip": list(set(backend.get("dependencies_required", []))),
        "env_vars": list(set(backend.get("environment_variables_required", [])))
    }
}

print(json.dumps(summary, indent=2))
```

---

## Notes for Adapting This Template

- The architect phase is ALWAYS sequential. The contract it produces is the single source of truth for all parallel agents.
- Frontend and backend agents must receive the exact same architect output. Copy it into both prompts verbatim.
- If a parallel agent deviates from the contract, flag it but do not re-run the agent unless the deviation is critical.
- The test agent can run in parallel with frontend and backend because it works from the contract, not from the implementation.
- For larger features, split frontend into multiple agents (e.g., one agent per major component).
