# Navigation Test Specification Guide

This document outlines the general specifications and best practices for writing navigation tests in front-end applications. When implementing navigation tests for a specific project, ensure the following criteria are met and documented.

## 1. List All Pages and Components

Every navigation test suite must begin by identifying the scope of navigation targets:

### Pages (Routes)
- **Primary Routes**: The main landing pages (e.g., Home, Dashboard, Index).
- **Secondary/Nested Routes**: Feature-specific pages accessible from primary routes (e.g., Item Detail, Settings, Profiles).
- **Overlay/Utility Routes**: Routes that might render as overlays or dedicated utility pages.
- **Fallback States**: 404 Not Found pages, unauthorized access redirects, or invalid parameterized routes.

### Components
- **Navigation Controls**: Menus, Sidebars, Tabs, Bottom Navigation, Header/Back buttons, and Breadcrumbs.
- **Overlays & Modals**: Action Sheets, Dialogs, Alerts, and Popovers that involve state or route changes.
- **Action Triggers**: Floating Action Buttons (FABs), primary call-to-action buttons, and inline links.

## 2. List All Navigation Cases

Document all possible navigation flows that a user can take within the application:

1. **Primary Navigation (Menu/Tabs)**
   - Opening and closing global navigation menus (e.g., Side Menu).
   - Navigating between main sections via menu items or tab clicks.
   
2. **Stack Navigation (Forward/Backward)**
   - Navigating from a list/parent page to a detail/child page.
   - Returning from a detail page to the previous page via UI back buttons, modal dismissals, or browser back history.

3. **In-Page & Overlay Navigation**
   - Triggering modals, alerts, or action sheets.
   - Dismissing overlays via cancel buttons, backdrop clicks, or keyboard events (e.g., Escape key).

4. **Deep Linking & Direct Access**
   - Accessing specific routes directly via URL.
   - Accessing parameterized routes (e.g., `/item/:id`) with valid IDs.
   - Accessing parameterized routes with invalid IDs to verify fallback behaviors.

## 3. Write Test Cases to Cover Navigation Cases

Test cases should be written to verify the correct rendering and state of the application for each navigation case outlined above.

### Verification Criteria
- **Visibility**: Assert that the target page or component is visibly attached to the DOM.
- **URL Correctness**: Assert that the browser URL path updates to the expected route.
- **Content Checks**: Verify that page-specific elements (e.g., page titles, headers, empty states) are displayed correctly.
- **Responsive Behavior**: Ensure navigation elements function correctly across different viewports (e.g., Desktop vs. Mobile constraints).

### Example Test Coverage Scenarios
- **Menu Tests**: Verify the menu opens, displays the correct items, navigates to the target page upon click, and closes automatically if required.
- **Flow Tests**: Verify a user can complete a full forward-and-back flow (e.g., click 'Add' -> open modal -> submit -> land on new detail page -> click back -> return to list).
- **Deep Link Tests**: Verify that loading a direct URL bypasses intermediate steps and correctly renders the target page or an error state for invalid URLs.

## 4. Verify Console Error Logs

A critical part of navigation testing is ensuring that transitions do not cause silent failures, rendering issues, or memory leaks.

- **Capture Logs**: Configure the test environment (e.g., Playwright, Cypress) to listen to browser console events.
- **Assertions**: Assert that **no console error logs** (e.g., `console.error`) are produced during any navigation transitions, component mounting, or unmounting phases.
- **Exceptions**: If specific expected errors are known (e.g., a deliberate network failure in a fallback test), they must be explicitly handled, ensuring all other unexpected errors fail the test.

## 5. Troubleshooting Common Issues

### Timeout Errors on Initial Mount (e.g., `ion-router-outlet`)

**Symptom**: Playwright tests time out waiting for the root application element or router outlet (e.g., `page.waitForSelector('ion-router-outlet', { state: 'attached' })`) to appear in the DOM.

**Likely Cause**: The frontend application failed to execute or crashed during the initial render cycle due to an uncaught JavaScript or React exception. This prevents the DOM from fully mounting, leaving Playwright waiting indefinitely.

**Debugging Steps**:

1. **Capture Silent Browser Errors**: Playwright does not surface browser console errors by default. Add event listeners in your `beforeEach` block to expose rendering crashes:
   ```typescript
   test.beforeEach(async ({ page }) => {
     page.on('pageerror', exception => {
       console.log(`[Uncaught Exception] ${exception}`);
     });
     page.on('console', msg => {
       if (msg.type() === 'error' || msg.type() === 'warning') {
         console.log(`[Browser ${msg.type()}] ${msg.text()}`);
       }
     });
     await page.goto('/');
   });
   ```

2. **Verify Bundle Execution**: Add a temporary `console.log('App Entry Point Reached');` in the main entry file (e.g., `main.tsx`) to verify the development server is successfully serving the JavaScript bundle.

3. **Check Context/Provider Hierarchy**: A common culprit in React apps is a missing context provider or undefined props. If a deeply nested component (like a hidden modal or side menu) crashes on mount due to missing context, it will take down the entire application render tree.

4. **Isolate the Failing Component**: Check the captured `pageerror` logs for the component stack trace. Fix the underlying application code (e.g., adding error boundaries, fixing null references) rather than altering test timeouts.
