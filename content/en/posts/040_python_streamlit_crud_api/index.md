+++
date = '2026-03-10T16:42:11+09:00'
draft = false
title = 'Build a Streamlit Demo App with a Mock API'
categories = ["streamlit"]
+++

"I need something I can demo by tomorrow." — Streamlit was built for exactly this moment.
This post walks through a working **approval workflow PoC** built in **2 files and roughly 100 lines of Python**, explaining the intent behind every design decision.



![Approve Audit App](approve_audit.png)

---

## 1. Project Structure

The entire project fits in two files.

```
project/
├── api_mock.py   # Business logic & in-memory data store
└── app.py        # Streamlit UI layer
```

Keeping the UI and logic separate means that when the time comes to go to production, you only need to swap out `api_mock.py` for a real HTTP client — the UI layer stays untouched.

---

## 2. api_mock.py — The In-Memory Pseudo API

Everything a real database or external API would handle is reproduced here using plain Python dicts and lists.

### Data Store

```python
# Module-level dicts act as the in-memory "database"
_RECORDS: Dict[int, Dict] = {
    1: {"id": 1, "title": "Record A", "status": "pending", "content": "Detail for Record A"},
    2: {"id": 2, "title": "Record B", "status": "pending", "content": "Detail for Record B"},
}
_AUDIT: List[Dict] = []  # Operation log
```

> **💡 Key insight:** Streamlit re-runs the entire script on every interaction, but module-level variables persist for the lifetime of the Python process. That makes this dict a cross-run state store — no database required for a PoC.

### API Function Reference

| Function | Arguments | Returns | Purpose |
|---|---|---|---|
| `fetch_records()` | `status="pending"` | `List[Dict]` | Fetch record list with optional status filter |
| `fetch_detail()` | `record_id` | `Dict` | Fetch a single record's full detail |
| `approve_record()` | `record_id, actor` | `bool` | Approve a record (with double-approval guard) |
| `fetch_audit()` | `limit=20` | `List[Dict]` | Fetch operation log, newest first |

### The Double-Approval Guard

```python
def approve_record(record_id: int, actor: str = "approver") -> bool:
    """Approve only if pending. Return True on success."""
    r = _RECORDS.get(record_id)
    if not r:
        return False
    if r["status"] != "pending":   # ← double-approval guard
        return False

    r["status"] = "approved"
    _AUDIT.append({
        "ts":        datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "actor":     actor,
        "action":    "approve",
        "record_id": record_id,
    })
    return True
```

The `status != "pending"` check enforces **idempotency**. Approving the same record twice will return `False` on the second attempt and leave no audit trace. That single line is what makes this logic safe enough to carry into production.

---

## 3. app.py — The Streamlit UI Layer

The UI follows a straightforward linear flow:

```
Sidebar config → Fetch record list → Show detail → Approve button → Audit log
```

### Sidebar: Mock Authentication

```python
# No real auth needed for a PoC — a selectbox handles role switching
actor    = st.sidebar.selectbox("Actor (mock)", ["approver", "requester", "admin"], index=0)
show_all = st.sidebar.checkbox("Show all records", value=False)
```

Each role maps to a realistic use case:

- **approver** — The only role that can click Approve. Wire to RBAC in production.
- **requester** — Read-only view of records they submitted.
- **admin** — Placeholder for future management operations like viewing all records.

### The Approve Button: Three Layers of Control

```python
can_approve = (actor == "approver")           # role check
is_pending  = (detail["status"] == "pending") # status check
disabled    = (not can_approve) or (not is_pending)  # either fails → button disabled

if st.button("Approve", disabled=disabled):
    ok = approve_record(detail["id"], actor=actor)
    if ok:
        st.success("Approved")
    else:
        st.warning("Not pending (already processed).")
    st.rerun()  # refresh UI immediately
```

> **✅ The classic defense:** The `disabled` flag provides a UI-level guard, while `approve_record()` enforces the same check at the logic level. Even if someone bypasses the UI, the backend holds the line — classic defense in depth.

### st.rerun() for Immediate Feedback

Streamlit normally re-runs the script after any widget interaction, but calling `st.rerun()` explicitly forces an immediate refresh right after the approval is processed. Without it, the user would see the record still marked as "pending" until the next natural re-run — a confusing experience that one line of code eliminates entirely.

### Audit Log Display

```python
audit = fetch_audit(limit=20)
if audit:
    st.dataframe(audit, width="stretch")  # renders a list of dicts as a table
else:
    st.caption("No audit events yet.")
```

`st.dataframe()` accepts a list of dicts and renders a fully interactive table — with sorting, scrolling, and fullscreen view — at zero implementation cost. An audit screen in one line.

---

## 4. What Makes This a Good PoC Design

| Principle | How it's applied |
|---|---|
| Swappable backend | The UI only calls API functions. Replace `api_mock.py` with an HTTP client and nothing else changes. |
| Defense in depth | UI `disabled` flag + logic-level `status` check provide two independent guards. |
| Audit trail | Every approval is logged with actor, timestamp (UTC), and record ID. |
| Immediate feedback | `st.rerun()` reflects the result of an action before the user has time to wonder. |

---

## 5. What to Build Next

### Short Term — Strengthen the PoC

Add a form to create and delete records, completing the CRUD loop. `st.form()` combined with `st.text_input()` gets this done in a handful of lines.

### Medium Term — Move to Production

Keep the function signatures in `api_mock.py` exactly as they are and replace the internals with real HTTP calls using `requests` or `httpx`. The UI layer requires zero changes.

### Long Term — Enterprise-Ready

Swap the mock `actor` selector for real authentication via Streamlit's built-in auth support or an external SSO provider, and you have an enterprise-grade approval workflow system.

> **💡 Note:** Streamlit can handle real production workloads, but if you need complex real-time updates or a highly customized frontend, consider migrating to FastAPI + React. The PoC is the right place to lock in requirements before making that call.

---

## Summary

- **Two-file separation** keeps logic and UI loosely coupled from day one
- Module-level `_RECORDS` dict serves as a cross-run in-memory database
- **Defense in depth** — UI `disabled` + logic `status` check together enforce idempotency
- `st.rerun()` delivers immediate UI feedback after an approval action
- `st.dataframe()` turns a list of dicts into a fully interactive audit table in one line
- The API function interface is designed to be swapped out, not thrown away