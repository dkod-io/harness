# Evaluation Guide: Skeptical Testing via Chrome DevTools

Deep reference for the Evaluator agent on testing patterns, scoring calibration, and
chrome-devtools usage.

## Philosophy

From Anthropic's harness research:

> "Tuning a standalone evaluator to be skeptical turns out to be far more tractable than
> making a generator critical of its own work."

> "Agents tend to respond by confidently praising the work — even when, to a human observer,
> the quality is obviously mediocre."

You exist because generators cannot honestly evaluate their own output. Your job is to be
the honest signal. Default to FAIL. Require proof of PASS.

## Chrome DevTools MCP Patterns

### Basic Page Testing — Verify Loading Completes

**DO NOT** `wait_for(selector: "body")` — `body` always exists instantly. You must confirm
the page finishes loading. A page stuck on a spinner is BROKEN, not loaded.

```
1. navigate_page(url: "http://localhost:<port>")
2. take_screenshot()                              → evidence: initial state (may show loading)
3. evaluate_script(expression: `
     (() => {
       const isActiveLoadingClass = (cls) =>
         /(^|[\s-])(spinner|loading|skeleton)([\s-]|$)/i.test(cls) &&
         !/(complete|done|finished|hidden|loaded)/i.test(cls);
       const indicators = [
         ...document.querySelectorAll('[aria-busy="true"]'),
         ...[...document.querySelectorAll('[class]')].filter(el =>
           isActiveLoadingClass(el.className)
         ),
         ...[...document.querySelectorAll('*')].filter(el =>
           el.children.length === 0 &&
           /^(loading|please wait)/i.test(el.textContent.trim())
         )
       ];
       return {
         isLoading: indicators.length > 0,
         indicators: indicators.map(el => ({
           tag: el.tagName,
           class: el.className,
           text: el.textContent.trim().slice(0, 50)
         }))
       };
     })()
   `)                                             → detect if page is still loading
4. // If isLoading is true: wait 10 seconds, then re-run step 3
   // If still loading after 10s → FAIL (score 3/10 max)
5. take_screenshot()                              → evidence: page in final state (must show data)
6. list_console_messages()                        → evidence: no fetch errors
```

This pattern is generic — it probes the DOM dynamically for common loading indicators
instead of relying on project-specific selectors. The evaluator adapts to whatever loading
patterns the generators implemented.

**If step 4 times out** (still loading after 10s):
- The page has a broken data flow → FAIL (score 3/10 max)
- Check console for: failed fetch calls, CORS errors, 404/500 API responses
- Check network requests for: hanging requests, wrong URLs, missing auth headers
- Include both screenshots (initial + still-loading) as evidence

### Form Interaction

```
1. navigate_page(url: "http://localhost:5173/login")
2. take_screenshot()                              → evidence: form renders
3. fill(selector: "#email", value: "test@test.com")
4. fill(selector: "#password", value: "password123")
5. click(selector: "button[type=submit]")
6. wait_for(selector: ".dashboard", timeout: 5000)
7. take_screenshot()                              → evidence: login succeeds
8. list_console_messages()                        → evidence: no errors
```

### Form Validation (Error Path)

```
1. navigate_page(url: "http://localhost:5173/signup")
2. click(selector: "button[type=submit]")         → submit empty form
3. wait_for(selector: ".error-message", timeout: 3000)
4. take_screenshot()                              → evidence: validation shown
5. evaluate_script(expression: "document.querySelectorAll('.error-message').length")
                                                  → evidence: error count
```

### API Testing via Script Evaluation

```
evaluate_script(expression: `
  fetch('/api/tasks', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ title: 'Test Task', priority: 'high' })
  }).then(r => ({ status: r.status, ok: r.ok }))
`)
→ evidence: { status: 201, ok: true }
```

### Responsive Testing

```
1. resize_page(width: 375, height: 812)           → mobile viewport
2. take_screenshot()                              → evidence: mobile layout
3. evaluate_script(expression: "window.getComputedStyle(document.querySelector('nav')).display")
                                                  → evidence: mobile nav behavior
4. resize_page(width: 1440, height: 900)           → desktop viewport
5. take_screenshot()                              → evidence: desktop layout
```

### Navigation Testing

```
1. navigate_page(url: "http://localhost:5173")
2. click(selector: "a[href='/tasks']")
3. wait_for(selector: ".task-list", timeout: 5000)
4. take_screenshot()                              → evidence: navigation works
5. evaluate_script(expression: "window.location.pathname")
                                                  → evidence: "/tasks"
```

### Performance Testing

```
lighthouse_audit(url: "http://localhost:5173", categories: ["performance"])
→ evidence: performance score, FCP, LCP, CLS metrics
```

### Console Error Detection

```
list_console_messages()
→ filter for type: "error"
→ evidence: list of JS errors with stack traces
```

### CRUD End-to-End Flow

```
1. navigate_page(url: "http://localhost:5173/tasks")
2. take_screenshot()                              → evidence: initial state

# CREATE
3. click(selector: ".create-task-btn")
4. fill(selector: "#task-title", value: "E2E Test Task")
5. fill(selector: "#task-description", value: "Created by evaluator")
6. click(selector: "button[type=submit]")
7. wait_for(selector: ".task-card", timeout: 5000)
8. take_screenshot()                              → evidence: task created

# READ
9. evaluate_script(expression: "document.querySelector('.task-card .title').textContent")
                                                  → evidence: "E2E Test Task"

# UPDATE
10. click(selector: ".task-card .edit-btn")
11. fill(selector: "#task-title", value: "Updated Task")
12. click(selector: "button[type=submit]")
13. wait_for(text: "Updated Task", timeout: 5000)
14. take_screenshot()                             → evidence: task updated

# DELETE
15. click(selector: ".task-card .delete-btn")
16. wait_for(selector: ".confirm-dialog", timeout: 3000)
17. click(selector: ".confirm-delete-btn")
18. take_screenshot()                             → evidence: task deleted
```

## Scoring Calibration

### Example: "Login page with email/password"

| What you see | Score | Reasoning |
|-------------|-------|-----------|
| No login page exists | 1 | Not implemented |
| Page exists but form doesn't render | 2 | Attempted but broken |
| Form renders, submit does nothing | 3 | Visual only, no functionality |
| Form submits but always returns 500 | 4 | Backend connected but broken |
| Form works for valid credentials only | 5 | Happy path works, no error handling |
| Form works + shows generic error on failure | 6 | Basic error handling |
| Form works + specific errors + loading state | 7 | **Pass threshold** |
| Above + remember me + redirect to dashboard | 8 | Good UX touches |
| Above + rate limiting + CSRF protection | 9 | Security conscious |
| Above + accessibility + keyboard navigation | 10 | Production-quality |

### Example: "Task list with filtering"

| What you see | Score | Reasoning |
|-------------|-------|-----------|
| No task list page | 1 | Not implemented |
| Page exists but data doesn't load | 3 | Shell without content |
| List loads but no filtering UI | 5 | Core works, feature missing |
| Filtering UI exists but doesn't work | 5 | Decoration without function |
| One filter works (e.g., status) | 6 | Partial implementation |
| All specified filters work | 7 | **Pass threshold** |
| Filters + URL state preservation | 8 | Good engineering |
| Above + keyboard shortcuts + clear all | 9 | Polished |

### Example: "Page with async data loading"

**This is the most commonly missed failure.** A page that renders a spinner forever looks
"partially working" but is completely broken from the user's perspective.

| What you see | Score | Reasoning |
|-------------|-------|-----------|
| Page shows spinner indefinitely ("Loading...") | 3 | **Broken data flow** — API call fails or returns wrong format. User sees nothing useful. |
| Page shows spinner then error message | 4 | At least the error is handled, but feature is non-functional |
| Page shows spinner, then data after 5-10 seconds | 5 | Works but unacceptably slow load time |
| Page shows spinner, then data after 2-5 seconds | 7 | Works, acceptable load time |
| Page shows skeleton, then data under 1 second | 9 | Good UX with loading state |
| Page shows data instantly (SSR or cached) | 10 | Optimal |

**KEY: A spinner that never resolves is NOT a 5 or 6. It is a 3.** The user gets zero value.
The feature is broken. Don't be generous because "the UI shell looks nice."

## Fallback: Testing Without Chrome DevTools

If chrome-devtools MCP is unavailable, use Bash-based testing:

### API Testing
```bash
# Health check
curl -s http://localhost:8000/health | jq .

# CRUD
curl -s -X POST http://localhost:8000/api/tasks \
  -H 'Content-Type: application/json' \
  -d '{"title":"Test","priority":"high"}' | jq .

curl -s http://localhost:8000/api/tasks | jq '.[] | .title'

# Error handling
curl -s -X POST http://localhost:8000/api/tasks \
  -H 'Content-Type: application/json' \
  -d '{}' | jq .status
```

### Frontend Smoke Testing
```bash
# Check if dev server responds
curl -s http://localhost:5173 | head -20

# Check for key HTML elements
curl -s http://localhost:5173 | grep -c '<div id="root">'

# Check that static assets load
curl -s -o /dev/null -w "%{http_code}" http://localhost:5173/assets/index.js
```

### Test Suite Execution
```bash
# Run existing tests
npm test 2>&1
npx vitest run 2>&1
pytest -v 2>&1
```

Note in the eval report that live UI testing was not performed and the evaluation is limited
to API testing, static analysis, and test suite execution.

## Evidence Collection

Every score MUST have evidence. Evidence types:

| Evidence Type | When to Use |
|--------------|-------------|
| Screenshot | Visual assertions (UI renders, layout correct) |
| Console output | Error detection, warning detection |
| HTTP response | API functionality (status code, body) |
| DOM evaluation | Element existence, text content, computed styles |
| Lighthouse score | Performance metrics |
| Test output | Automated test results |
| Bash output | Build success, server startup, file existence |

**Never score without evidence.** "I believe this works based on reading the code" is NOT
evidence. Run it. Test it. Screenshot it.

## Feedback Quality

When a criterion fails, your feedback must be actionable:

### Good Feedback
```json
{
  "criterion": "Task creation shows validation errors for empty title",
  "score": 4,
  "evidence": "Submitted empty form — page shows spinner indefinitely. Console: Unhandled rejection: AxiosError at TaskForm.tsx:34. Server returns 400 but client doesn't handle the error response.",
  "fix_hint": "In src/components/TaskForm.tsx, the onSubmit handler (line 30-40) catches the error but doesn't update state. Add: setError(err.response.data.message) in the catch block, and render {error && <Alert>{error}</Alert>} below the form."
}
```

### Bad Feedback
```json
{
  "criterion": "Task creation shows validation errors",
  "score": 4,
  "evidence": "Doesn't work",
  "fix_hint": "Fix the form"
}
```

The generator will receive your fix_hint verbatim. Make it precise enough that a single
targeted edit can fix it.
