# Test Development Guide

This document outlines the best practices for developing, running, troubleshooting, and fixing Playwright test cases in this project. Adhering to these guidelines ensures an efficient testing workflow and prevents wasting time on redundant test executions.

## 1. Test Code Organization and Development

### Extract Common Code into Helper Functions
- **Avoid duplication** by creating reusable helper functions for common actions.
- Place helper functions at the top of test files or in shared utility files.
- Example:
  ```typescript
  // GOOD - reusable helper function
  async function openSideMenu(page: Page) {
    await page.click('.menu-trigger-btn');
    await page.waitForSelector('.side-menu--open', { state: 'visible' });
    await page.waitForTimeout(400); // Wait for CSS transition
  }
  
  test('test 1', async ({ page }) => {
    await openSideMenu(page);
    // ... test logic
  });
  ```

### Mobile Testing Guidelines
- **Ensure elements are visible before clicking on mobile viewports.** Elements outside the viewport cannot be clicked by Playwright.
- Use `scrollIntoViewIfNeeded()` to bring elements into view before clicking:
  ```typescript
  // GOOD - ensure visibility before clicking
  const menuItem = page.locator('.side-menu-item:has-text("🗑️ Trash Bin")');
  await menuItem.scrollIntoViewIfNeeded();
  await menuItem.click();
  ```
- **DO NOT use JavaScript clicks to bypass visibility checks.** This defeats the purpose of testing real user interactions. Let the UI properly scroll elements into view or adjust the viewport size.

### Test Timeout Guidelines
- **DO NOT add timeouts** (e.g., `page.waitForTimeout()`) in test actions that don't require backend access.
- For UI interactions (clicks, form fills, navigation), rely on Playwright's built-in auto-waiting mechanisms.
- Timeouts should only be used for:
  - Waiting for backend API responses.
  - Waiting for async operations like sync, file uploads, etc.
  - Allowing UI animations to complete when necessary.

## 2. Test Execution Workflow

### Focus on One Failure at a Time
- **Never run the entire test suite repeatedly when there are known failures.**
- When there is a failed test case, don't run other cases until it is fixed.
- When the failed test case is fixed, we should run the suite to which the failed case belongs first, then run all test cases.

### Use the `-g` (grep) Option
- When a specific test fails, isolate it using the `-g` option with part of the test name.
- **Example:**
  ```bash
  npx playwright test tests/deletion-cases.spec.ts -g "can soft delete a notebook" --project="Mobile Chrome"
  ```
- This drastically reduces execution time and isolates the logs to the exact test you are debugging.

### Avoid `--workers=1`
- **Never use `--workers=1`.** We have configured `maxFailures: 1` in `playwright.config.ts`, so the test runner will automatically fail fast. Using `--workers=1` slows down test execution unnecessarily.

### Visual Debugging
- If you cannot deduce the issue from the logs, run the test in headed mode to visually inspect the browser.
- **Example:**
  ```bash
  npx playwright test -g "your test name" --headed
  ```

## 3. Troubleshooting Strategies

When a test case fails, follow these steps to diagnose and fix the issue. **Always fix the root cause in the source code if necessary, not just the test.**

### Capture Browser and Network Logs
**ALWAYS mind the console when test cases fail!** Playwright allows you to capture browser console logs and network requests, which are invaluable for debugging React/Ionic errors, API failures, or missing state. Many seemingly mysterious UI failures (like elements not appearing or timeouts) are caused by underlying JavaScript errors that are immediately visible in the console.

```typescript
const browserLogs = [];
page.on('console', msg => {
  browserLogs.push({ type: msg.type(), text: msg.text() });
});

page.on('request', req => console.log(`>> Request: ${req.method()} ${req.url()}`));
page.on('response', res => console.log(`<< Response: ${res.status()} ${res.url()}`));

// ... run your test actions ...
console.log('Browser console logs:', JSON.stringify(browserLogs, null, 2));
```

### Add Temporary Debug Logs
- **In test files:** Add `console.log` to capture values and state (e.g., `console.log('Current status:', await page.locator('.status-badge').textContent())`).
- **In source code:** Add temporary logs at key points (entry/exit of functions, catch blocks, state changes) to trace execution flow.
- *Remember to clean up debug logs after the test passes.*

### Inspect the DOM State and Structure
- **Do not blindly try different selectors.** When there is a failed action for an item, query the true structure of the page, inspect the elements, and *then* update the selector based on the reality of the DOM.
- **Use `page.evaluate()` to inspect DOM state:**
  ```typescript
  // Example: Dump the inner HTML of a problematic component
  const html = await page.locator('ion-action-sheet').evaluate(el => el.innerHTML);
  console.log('Action Sheet HTML:', html);
  
  // Or for shadow DOM:
  const shadowHtml = await page.locator('ion-action-sheet').evaluate(el => el.shadowRoot?.innerHTML);
  console.log('Action Sheet Shadow HTML:', shadowHtml);
  ```
- **Take screenshots on failure** to capture the exact visual state:
  ```typescript
  await page.screenshot({ path: 'test-failure.png' });
  ```

### Timeouts and Performance
- **Do not update the global timeout.** The global timeout (e.g., in `playwright.config.ts`) is set intentionally. Modifying it hides underlying performance issues or incorrect assertions.
- If you suspect that the timeout duration is too short for a specific test, verify this by printing timestamps before and after the slow operation, rather than blindly increasing the timeout.
  ```typescript
  console.log(`[${new Date().toISOString()}] Starting slow operation...`);
  // ... slow operation ...
  console.log(`[${new Date().toISOString()}] Finished slow operation.`);
  ```

## 4. Fixing Test Cases (Especially After UI Migrations)

### Update Selectors for Ionic
When migrating to Ionic, standard HTML tags (like `div`, `button`, `ul`, `li`) are often replaced by Ionic components (`ion-content`, `ion-button`, `ion-list`, `ion-item`).
- **Old:** `await page.click('.notebook-item .menu-btn');`
- **New:** `await page.locator('ion-item').locator('ion-button').click();`

### Handle Shadow DOM
Many Ionic components use Shadow DOM. Playwright penetrates Shadow DOM automatically in most cases, but if you are querying deeply nested parts, ensure your selectors are robust. Avoid relying on deep internal class names that Ionic might change.

### Manage Transitions and Animations
Ionic pages and modals use animations. Playwright's auto-waiting usually handles this, but if you encounter flakiness:
- Wait for the element to be explicitly visible: `await expect(page.locator('ion-modal')).toBeVisible();`
- In rare cases where auto-waiting fails during a transition, use a short, explicit timeout: `await page.waitForTimeout(500);` (Use sparingly).

### Local Storage and State Clearing
Ensure that tests are independent. Clear the state before each test to prevent data leakage.
```typescript
test.beforeEach(async ({ page }) => {
  await page.goto('/');
  await page.evaluate(() => localStorage.clear());
  await page.reload();
  // Wait for the Ionic router to initialize
  await page.waitForSelector('ion-router-outlet', { state: 'attached' });
});
```
