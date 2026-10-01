# Playwright end-to-end tests

Read this when writing, reviewing or de-flaking Playwright tests.

Keep E2E for a few journeys where breakage is expensive: sign-up and login, checkout and payment, the core create-and-share flow. Everything that can be tested below the browser is faster and more reliable there.

## Locators

Use locators that follow what the user perceives, in this order: `getByRole` with the accessible name, `getByLabel` for form fields, `getByText` for non-interactive content, `getByTestId` for elements with no role or stable text (a price cell, a chart container).

```ts
// Before: breaks when a wrapper div is added or the class is renamed
await page.locator("#root > div > div:nth-child(3) > button.btn-primary").click();

// After
await page.getByRole("button", { name: "Place order" }).click();
await expect(page.getByTestId("order-total")).toHaveText(expectedTotalText);
```

A role locator that cannot find the button is often an accessibility bug (an icon button with no name, a clickable `div`). Fix the markup rather than falling back to CSS.

## Waiting

Every Playwright action waits for its element to be actionable, and every `await expect(locator)...` matcher retries until its timeout. Use them and nothing else for synchronization.

```ts
// Before: fixed sleep, then a one-shot check that never retries
await page.waitForTimeout(2000);
expect(await page.locator(".toast").isVisible()).toBe(true);

// After
await expect(page.getByRole("status")).toHaveText("Payment received");
```

- `expect(await locator.isVisible()).toBe(true)`, `expect(await locator.textContent()).toBe(...)` and `expect(await locator.count()).toBe(n)` read once and do not retry. Use `toBeVisible`, `toHaveText`, `toHaveCount`.
- `waitUntil: "networkidle"` is marked discouraged in the Playwright docs; assert on the UI state that means "ready".
- For state that is not in the DOM (a row in the database, a webhook received), poll it: `await expect.poll(() => fetchOrderStatus(orderId)).toBe("paid")`.
- Negative assertions pass the instant they are evaluated if the page has not rendered yet. `await expect(adminLink).not.toBeVisible()` right after `goto` proves nothing. First assert something positive that shows the page finished loading, then the negative.

## Isolation and data

Each test gets a fresh browser context, but not fresh server data. Tests fail in the suite and pass alone (or the reverse) when they share server state.

- Each test creates what it needs through the API or a seeding helper, with unique values, instead of relying on an earlier test's order, user or cart. Module-level `let orderId` written by one test and read by another breaks as soon as tests run in parallel or in a different order; with `fullyParallel: true`, tests in the same file already run in different workers.
- Log in once per role in a setup project and save `storageState` to a file under the test output directory that is git-ignored. Those files contain live session cookies; never commit them.
- Credentials for test accounts come from environment variables or the CI secret store, for test accounts in a test environment. A base URL or credential that defaults to production is a defect.

## Authorization in E2E

A test that checks the UI hides the admin menu for a member proves only the UI. Check the API with the member's session as well:

```ts
test.use({ storageState: memberStorageStatePath });

test("member cannot read admin billing", async ({ page }) => {
  await page.goto("/settings");
  await expect(page.getByRole("heading", { name: "Settings" })).toBeVisible();
  await expect(page.getByRole("link", { name: "Billing" })).toHaveCount(0);

  const response = await page.request.get("/api/admin/billing");
  expect(response.status()).toBe(403);
});
```

`page.request` uses the page's cookies, so the API call carries the member's session. Pair it with the same test for an admin session to prove the 403 comes from the role check. The full matrix belongs in API tests (`authz-matrix.md`); E2E only confirms the wiring.

## Time

Control the browser clock with `page.clock` (see `determinism.md`) instead of waiting for real timeouts. A session-expiry warning after 15 minutes should take milliseconds to test.

## Flaky tests and retries

- Reproduce first: `npx playwright test path/to/spec.ts --repeat-each=50 --workers=4`. Most flakes reproduce under repetition with parallel workers.
- The trace (`trace: "on-first-retry"` in config, then `npx playwright show-trace`) shows the DOM and network at each step of the failing attempt.
- Retries turn a failure into a pass labeled "flaky". If nobody reads that label, retries delete the signal. A checkout test that sometimes shows the coupon applied twice is reporting a real double-submit bug, not noise.
- On critical paths, run with `--fail-on-flaky-tests`, or set `test.describe.configure({ retries: 0 })` on money and auth flows, so a flaky result fails the build.
- Set `forbidOnly: !!process.env.CI` so a committed `test.only` cannot silently skip the rest of the suite.

## Review checklist for a spec file

- Are locators role, label, text or test id based, with no structural CSS or XPath?
- Is there no `waitForTimeout`, no `networkidle`, and no `expect(await ...)` on async UI state?
- Does every negative assertion follow a positive one that proves the page loaded?
- Does each test create its own data, with nothing carried between tests in module variables?
- Are authorization tests backed by API calls with the lower-privilege session?
- Are there no credentials, cookies or production URLs in the spec, config or committed files?
- Are retries off or flaky-failing for money and auth flows?
