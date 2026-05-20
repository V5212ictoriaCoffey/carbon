# Waiting Strategies

This reference covers strategies for waiting on page state, network activity, and element conditions in the agent-browser skill.

## Overview

Reliable browser automation requires proper synchronization between your code and the browser's state. Rushing interactions before the page is ready leads to flaky tests and missed elements.

## Built-in Wait Conditions

### `waitForSelector`

Waits until an element matching the selector appears in the DOM.

```typescript
import { Page } from 'playwright';

async function waitForElement(page: Page, selector: string, timeout = 30000) {
  const element = await page.waitForSelector(selector, {
    state: 'visible',
    timeout,
  });
  return element;
}
```

**State options:**
- `attached` — element exists in DOM (may be hidden)
- `detached` — element is removed from DOM
- `visible` — element is visible and not hidden by CSS
- `hidden` — element is hidden or removed

### `waitForURL`

Waits until the page navigates to a URL matching the pattern.

```typescript
async function waitForNavigation(page: Page, urlPattern: string | RegExp) {
  await page.waitForURL(urlPattern, { timeout: 30000 });
}
```

### `waitForLoadState`

Waits for the page to reach a specific load state.

```typescript
async function waitForPageLoad(page: Page) {
  // Options: 'load', 'domcontentloaded', 'networkidle'
  await page.waitForLoadState('networkidle', { timeout: 60000 });
}
```

> **Note:** `networkidle` waits until there are no network connections for at least 500ms. Avoid it on pages with long-polling or WebSocket connections.

### `waitForResponse`

Waits for a specific network response.

```typescript
async function waitForApiResponse(page: Page, urlPattern: string | RegExp) {
  const response = await page.waitForResponse(
    (resp) => {
      const url = resp.url();
      const matchesPattern =
        typeof urlPattern === 'string'
          ? url.includes(urlPattern)
          : urlPattern.test(url);
      return matchesPattern && resp.status() === 200;
    },
    { timeout: 30000 }
  );
  return response;
}
```

### `waitForRequest`

Waits for a specific outgoing network request.

```typescript
async function waitForApiRequest(page: Page, urlPattern: string | RegExp) {
  const request = await page.waitForRequest(urlPattern, { timeout: 30000 });
  return request;
}
```

## Custom Wait Utilities

### `waitForCondition`

Polls a condition function until it returns `true`.

```typescript
async function waitForCondition(
  condition: () => Promise<boolean> | boolean,
  options: { timeout?: number; interval?: number; message?: string } = {}
): Promise<void> {
  const { timeout = 30000, interval = 250, message = 'Condition not met' } = options;
  const deadline = Date.now() + timeout;

  while (Date.now() < deadline) {
    if (await condition()) return;
    await new Promise((resolve) => setTimeout(resolve, interval));
  }

  throw new Error(`Timeout: ${message} after ${timeout}ms`);
}
```

### `waitForElementCount`

Waits until a selector matches an expected number of elements.

```typescript
async function waitForElementCount(
  page: Page,
  selector: string,
  expectedCount: number,
  timeout = 10000
): Promise<void> {
  await waitForCondition(
    async () => {
      const elements = await page.$$(selector);
      return elements.length === expectedCount;
    },
    {
      timeout,
      message: `Expected ${expectedCount} elements matching "${selector}"`,
    }
  );
}
```

### `waitForTextContent`

Waits until an element contains specific text.

```typescript
async function waitForTextContent(
  page: Page,
  selector: string,
  expectedText: string,
  timeout = 10000
): Promise<void> {
  await page.waitForFunction(
    ({ sel, text }: { sel: string; text: string }) => {
      const el = document.querySelector(sel);
      return el?.textContent?.includes(text) ?? false;
    },
    { sel: selector, text: expectedText },
    { timeout }
  );
}
```

## Anti-Patterns to Avoid

### ❌ Hard-coded `setTimeout` delays

```typescript
// Bad: brittle, slow, and unreliable
await new Promise((resolve) => setTimeout(resolve, 3000));
await page.click('#submit');
```

### ✅ Wait for the actual condition

```typescript
// Good: waits exactly as long as needed
await page.waitForSelector('#submit', { state: 'visible' });
await page.click('#submit');
```

### ❌ Ignoring network activity after navigation

```typescript
// Bad: page may still be loading data
await page.goto('/dashboard');
await page.click('.data-table-row');
```

### ✅ Wait for data to load

```typescript
// Good: ensure data is rendered before interacting
await page.goto('/dashboard');
await page.waitForSelector('.data-table-row', { state: 'visible' });
await page.click('.data-table-row');
```

## Timeout Configuration

Set default timeouts globally to avoid repetition:

```typescript
import { BrowserContext } from 'playwright';

function configureTimeouts(context: BrowserContext) {
  // Default timeout for all actions and assertions
  context.setDefaultTimeout(30000);
  // Default timeout for page.goto and page.waitForNavigation
  context.setDefaultNavigationTimeout(60000);
}
```

## Related

- [Element Interaction](./element-interaction.md)
- [Network Interception](./network-interception.md)
- [Error Handling](./error-handling.md)
