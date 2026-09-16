# LlamaManager 0.2 Web UI and Security Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Redesign the LlamaManager Web UI with Auto / Light / Dark themes and harden browser-facing mutation flows with CSRF and safe Hugging Face token writes.

**Architecture:** Keep the current server-rendered FastAPI/Jinja application and domain boundaries. Add a small tokenized CSS visual system plus browser-local theme state; theme choice never writes machine config. Add a synchronizer-token CSRF layer to every browser mutation and preserve the Agent/domain API boundary.

**Tech Stack:** Python 3.11+, FastAPI/Starlette, Jinja2, vanilla JavaScript, CSS custom properties, pytest.

**Spec:** `docs/superpowers/specs/2026-09-15-llamamanager-mcp-theme-design.md`

## Global Constraints

- Web themes are exactly `Auto`, `Light`, and `Dark`.
- Theme choice is browser-local and must not rewrite shared or machine YAML.
- Existing routes and domain behavior remain backward-compatible unless explicitly changed below.
- Every browser-facing state-changing form must carry CSRF protection.
- Hugging Face token writes must reject empty/whitespace-only input.
- Raw Hugging Face tokens must never be rendered back into HTML.
- JavaScript is progressive enhancement; server-rendered forms remain usable without it.

---

### Task 1: Theme tokens and browser-local state

**Files:**
- Modify: `src/llamamanager/templates/base.html`
- Modify: `src/llamamanager/static/app.css`
- Modify: `src/llamamanager/static/theme.js`
- Test: `tests/test_web.py`

**Interfaces:**
- `data-theme="auto|light|dark"` on the root document element.
- `localStorage["llamamanager-theme"]` stores the explicit browser choice.
- `theme.js` applies stored choice and reacts to `prefers-color-scheme` only in `auto` mode.

- [ ] **Step 1: Write failing theme rendering tests**

```python
def test_base_template_renders_theme_controls(client):
    response = client.get("/")
    assert response.status_code == 200
    assert 'data-theme="auto"' in response.text
    assert 'data-theme-option="light"' in response.text
    assert 'data-theme-option="dark"' in response.text
    assert '/static/theme.js' in response.text
```

- [ ] **Step 2: Run RED**

```bash
pytest tests/test_web.py -q
```

Expected: theme controls and script assertions fail.

- [ ] **Step 3: Implement CSS custom-property palette and theme switcher**

Use semantic tokens only (`--bg`, `--surface`, `--text`, `--muted`, `--accent`, `--danger`, `--border`, `--shadow`) with explicit light/dark values. `auto` resolves via media query; no network calls or config writes.

- [ ] **Step 4: Run GREEN**

```bash
pytest tests/test_web.py -q
```

- [ ] **Step 5: Commit**

```bash
git add src/llamamanager/templates/base.html src/llamamanager/static/app.css src/llamamanager/static/theme.js tests/test_web.py
git commit -m "feat: add browser-local theme system"
```

### Task 2: CSRF infrastructure

**Files:**
- Modify: `src/llamamanager/web.py`
- Modify: `src/llamamanager/templates/base.html`
- Test: `tests/test_web_security.py`

**Interfaces:**
- `ensure_csrf_cookie(request, response) -> str`
- `validate_csrf(request, submitted_token) -> None`
- cookie `llamamanager_csrf` with `SameSite=Lax`, non-secret random value.

- [ ] **Step 1: Write failing CSRF cookie tests**

```python
def test_get_sets_csrf_cookie(client):
    response = client.get("/")
    assert response.cookies.get("llamamanager_csrf")
    assert "samesite=lax" in response.headers["set-cookie"].lower()
```

- [ ] **Step 2: Run RED**

```bash
pytest tests/test_web_security.py -q
```

- [ ] **Step 3: Implement cookie issuance and constant-time token comparison**

Use `secrets.token_urlsafe()` and `hmac.compare_digest()`; the token is anti-CSRF state, not an authentication bearer credential.

- [ ] **Step 4: Run GREEN**

```bash
pytest tests/test_web_security.py -q
```

- [ ] **Step 5: Commit**

### Task 3: Protect browser mutation routes

**Files:**
- Modify: `src/llamamanager/web.py`
- Modify: `src/llamamanager/templates/*.html`
- Test: `tests/test_web_security.py`

**Interfaces:** all POST form handlers call CSRF validation before performing mutation.

- [ ] **Step 1: Add parameterized failing test over every POST route**

Assert missing and mismatched tokens return `403` and do not call the underlying mutation.

- [ ] **Step 2: RED**

- [ ] **Step 3: Add hidden field**

```html
<input type="hidden" name="csrf_token" value="{{ csrf_token }}">
```

and validate before mutation.

- [ ] **Step 4: GREEN**

- [ ] **Step 5: Commit**

### Task 4: Safe Hugging Face token form

**Files:**
- Modify: `src/llamamanager/web.py`
- Modify: `src/llamamanager/templates/huggingface.html`
- Test: `tests/test_web_security.py`

**Interfaces:** token write endpoint accepts only non-empty trimmed token; HTML renders status/presence but never raw token.

- [ ] **Step 1: Test blank token rejected and secret not reflected**

- [ ] **Step 2: RED**

- [ ] **Step 3: Implement safe write/status UX**

- [ ] **Step 4: GREEN**

- [ ] **Step 5: Commit**

### Task 5: Web dashboard visual hierarchy

**Files:**
- Modify: templates + `app.css`
- Test: `tests/test_web.py`

**Interfaces:** stable route names, accessible labels, responsive grid/card/table layout.

- [ ] **Step 1: Add smoke assertions for header/nav/status cards and mobile viewport meta**

- [ ] **Step 2: RED**

- [ ] **Step 3: Implement redesigned shell without changing route behavior**

- [ ] **Step 4: GREEN**

- [ ] **Step 5: Commit**

### Task 6: Final security regression gate

**Files:** no new production files unless failures reveal defects.

- [ ] **Step 1: Run focused security tests**

```bash
pytest tests/test_web_security.py -q
```

- [ ] **Step 2: Run full suite**

```bash
pytest -q
```

- [ ] **Step 3: Compile and static smoke checks**

```bash
python -m compileall -q src
```

- [ ] **Step 4: Verify no secret values appear in generated HTML fixtures/logs**

- [ ] **Step 5: Commit only after clean verification**
