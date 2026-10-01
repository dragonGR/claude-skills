# Authorization matrix tests

Read this when writing or reviewing authorization tests for an HTTP or RPC API. What the policy should be is covered in security-engineering; this file is about proving the code enforces it.

## Rules for the harness

- Authenticate for real. Each actor gets a session or token minted through the same code path production uses (a login helper, or signing with the test key the app verifies against). No `dependency_overrides` on the current-user dependency, no "test mode" that skips middleware, no factory that makes every user staff.
- Test at the API. A UI that hides the admin button proves nothing; the attacker calls the endpoint directly.
- Every denial has a positive control in the same matrix: the identical request by an allowed actor succeeds. Without it, a 403 caused by a malformed request looks like a passing authorization test.
- After every denial, read the resource straight from the database (not through the API under test) and assert it is unchanged, and assert no external side effect was recorded.
- Assert the exact status. Pick the policy once (for example 404 across ownership or tenant boundaries so ids cannot be probed, 403 within a tenant for insufficient role, 401 for missing or expired credentials) and test it everywhere.

## Actors worth having

| Actor | Catches |
| --- | --- |
| Anonymous | Routes missing authentication entirely |
| Expired or revoked token of the owner | Token validation that checks the signature but not expiry or revocation |
| Owner | Positive control |
| Same tenant, not owner, same role | Missing object-level ownership check (IDOR) |
| Same tenant, lower role | Role checks missing on write or admin actions |
| Same tenant, admin | Positive control for admin-only actions |
| Other tenant's admin | Missing tenant scoping; the most expensive bug in multi-tenant systems |

## pytest template

```python
from dataclasses import dataclass

import pytest

READ, UPDATE, DELETE, PAY = "read", "update", "delete", "pay"


@dataclass(frozen=True)
class Action:
    method: str
    path: str
    body: dict | None
    success_status: int


ACTIONS = {
    READ: Action("GET", "/invoices/{id}", None, 200),
    UPDATE: Action("PATCH", "/invoices/{id}", {"memo": "changed by test"}, 200),
    DELETE: Action("DELETE", "/invoices/{id}", None, 204),
    PAY: Action("POST", "/invoices/{id}/pay", {}, 202),
}

ALLOWED = {
    "anonymous": set(),
    "owner_expired": set(),
    "owner": {READ, UPDATE, PAY},
    "tenant_member": {READ},
    "tenant_viewer": {READ},
    "tenant_admin": {READ, UPDATE, DELETE, PAY},
    "other_tenant_admin": set(),
}

UNAUTHENTICATED = {"anonymous", "owner_expired"}
CROSS_TENANT = {"other_tenant_admin"}


def denied_status(actor: str) -> int:
    if actor in UNAUTHENTICATED:
        return 401
    if actor in CROSS_TENANT:
        return 404
    return 403


@pytest.mark.parametrize("action", ACTIONS)
@pytest.mark.parametrize("actor", ALLOWED)
def test_invoice_access(actor, action, clients, invoice_of_owner, read_invoice_row, payment_fake):
    spec = ACTIONS[action]
    before = read_invoice_row(invoice_of_owner.id)

    response = clients[actor].request(
        spec.method, spec.path.format(id=invoice_of_owner.id), json=spec.body
    )

    if action in ALLOWED[actor]:
        assert response.status_code == spec.success_status
    else:
        assert response.status_code == denied_status(actor)
        assert read_invoice_row(invoice_of_owner.id) == before
        assert payment_fake.charges == []
```

`clients` is a fixture mapping actor names to HTTP clients carrying real credentials; `read_invoice_row` reads the row with a direct query. Because the allowed and denied cases share one table, every denied row has its positive control in the same parametrization.

## TypeScript template

```ts
import request from "supertest";
import { describe, expect, it } from "vitest";
import { app } from "../src/app";
import { credentialsFor, type Actor } from "./support/auth";
import { createInvoiceOwnedBy, readInvoiceRow } from "./support/fixtures";

type Case = { actor: Actor; method: "get" | "patch" | "delete"; status: number };

const cases: Case[] = [
  { actor: "owner", method: "get", status: 200 },
  { actor: "tenantMember", method: "get", status: 200 },
  { actor: "otherTenantAdmin", method: "get", status: 404 },
  { actor: "anonymous", method: "get", status: 401 },
  { actor: "owner", method: "patch", status: 200 },
  { actor: "tenantMember", method: "patch", status: 403 },
  { actor: "otherTenantAdmin", method: "patch", status: 404 },
  { actor: "tenantAdmin", method: "delete", status: 204 },
  { actor: "owner", method: "delete", status: 403 },
  { actor: "otherTenantAdmin", method: "delete", status: 404 },
];

describe("invoice access", () => {
  it.each(cases)("$actor $method -> $status", async ({ actor, method, status }) => {
    const invoice = await createInvoiceOwnedBy("owner");
    const before = await readInvoiceRow(invoice.id);

    const res = await request(app)
      [method](`/invoices/${invoice.id}`)
      .set(await credentialsFor(actor))
      .send(method === "patch" ? { memo: "changed by test" } : undefined);

    expect(res.status).toBe(status);
    if (status >= 400) expect(await readInvoiceRow(invoice.id)).toEqual(before);
  });
});
```

## Cases a plain matrix misses

- **List and search endpoints.** Create rows for two tenants, list as one, and assert none of the other tenant's ids appear. Also check counts and totals in the response, which leak even when rows are filtered.
- **Ids in the body.** `PATCH /invoices/{mine}` with `{"tenant_id": other}` or `{"owner_id": other}` must not move the row. `POST /transfers` with a `from_account` the caller does not own must be denied.
- **Nested routes.** `/tenants/{mine}/invoices/{theirs}`: the handler must check the invoice belongs to the tenant in the path and that the caller belongs to that tenant.
- **Bulk endpoints.** `ids=[mine, theirs]` must fail entirely or process only the caller's, as the API documents; never process both.
- **Indirect references.** Download links, export jobs, webhooks and signed URLs: fetch another user's export id, replay an expired signed URL.
- **State-dependent permissions.** Actions allowed in `draft` but not in `paid`; test each role in each state that matters.

## Catch routes nobody added to the matrix

New endpoints ship without authorization tests because nobody added them to the table. Enumerate the app's routes in a test and fail when one is neither in the matrix nor on an explicit public list. For FastAPI:

```python
from fastapi.routing import APIRoute

from app.main import app

PUBLIC_ROUTES = {("GET", "/health"), ("POST", "/auth/login")}


def test_every_route_is_covered_by_an_authz_matrix():
    routes = {
        (method, route.path)
        for route in app.routes
        if isinstance(route, APIRoute)
        for method in route.methods
    }
    covered = {(spec.method, spec.path) for spec in ALL_MATRIX_ACTIONS} | PUBLIC_ROUTES
    assert routes - covered == set()
```

`ALL_MATRIX_ACTIONS` is the union of the action tables from every resource's matrix. Other frameworks expose their route table too (Express router stacks, Django URL resolvers, Rails `routes`); the idea is the same.
