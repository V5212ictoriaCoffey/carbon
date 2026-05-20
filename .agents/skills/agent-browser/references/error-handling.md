# Error Handling in Agent Browser

This reference covers how to handle errors, timeouts, and unexpected states when using the agent browser skill.

## Common Error Types

### Navigation Errors

Errors that occur when navigating to a URL or waiting for page load:

```typescript
try {
  await browser.navigate('https://example.com');
} catch (error) {
  if (error.code === 'ERR_NAME_NOT_RESOLVED') {
    // DNS resolution failed — check the URL or network connectivity
    console.error('Could not resolve hostname:', error.message);
  } else if (error.code === 'ERR_CONNECTION_REFUSED') {
    // Server is not accepting connections
    console.error('Connection refused:', error.message);
  } else if (error.code === 'ERR_TIMED_OUT') {
    // Navigation took too long
    console.error('Navigation timed out:', error.message);
  } else {
    throw error; // Re-throw unknown errors
  }
}
```

### Element Not Found Errors

When a selector does not match any element within the timeout window:

```typescript
try {
  await browser.click('#submit-button');
} catch (error) {
  if (error.name === 'ElementNotFoundError') {
    // Element never appeared in the DOM
    // Take a snapshot to debug current page state
    const snapshot = await browser.snapshot();
    console.error('Element not found. Page state:', snapshot);
  }
}
```

### Timeout Errors

All async operations accept a `timeout` option (milliseconds). The default is `30000` (30 seconds).

```typescript
try {
  await browser.waitForSelector('.dynamic-content', { timeout: 5000 });
} catch (error) {
  if (error.name === 'TimeoutError') {
    console.error('Element did not appear within 5 seconds');
  }
}
```

### Network Errors During Interaction

Network failures that occur mid-interaction (e.g., form submission triggers a failed request):

```typescript
browser.on('requestfailed', (request) => {
  console.warn(`Request failed: ${request.url()} — ${request.failure()?.errorText}`);
});
```

## Retry Strategies

### Simple Retry Wrapper

```typescript
async function withRetry<T>(
  fn: () => Promise<T>,
  maxAttempts = 3,
  delayMs = 1000
): Promise<T> {
  let lastError: Error;
  for (let attempt = 1; attempt <= maxAttempts; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;
      if (attempt < maxAttempts) {
        console.warn(`Attempt ${attempt} failed, retrying in ${delayMs}ms...`);
        await new Promise((resolve) => setTimeout(resolve, delayMs));
        delayMs *= 2; // Exponential backoff
      }
    }
  }
  throw lastError!;
}

// Usage
const result = await withRetry(() => browser.click('#flaky-button'));
```

### Retry on Navigation

```typescript
async function navigateWithRetry(url: string, maxAttempts = 3): Promise<void> {
  await withRetry(async () => {
    await browser.navigate(url);
    // Verify page loaded correctly
    await browser.waitForSelector('body', { timeout: 10000 });
  }, maxAttempts);
}
```

## Graceful Degradation

### Fallback Selectors

When a primary selector may not exist, try alternatives:

```typescript
async function clickButton(browser: Browser): Promise<void> {
  const selectors = ['#submit', 'button[type="submit"]', '.btn-primary', 'button:last-child'];

  for (const selector of selectors) {
    const element = await browser.querySelector(selector);
    if (element) {
      await element.click();
      return;
    }
  }

  throw new Error('No submit button found with any known selector');
}
```

### Optional Element Interaction

```typescript
// Click an element only if it exists (e.g., a cookie banner)
async function dismissCookieBanner(browser: Browser): Promise<void> {
  const banner = await browser.querySelector('#cookie-accept', { timeout: 2000 }).catch(() => null);
  if (banner) {
    await banner.click();
  }
}
```

## Debugging Failed Steps

### Capture State on Failure

```typescript
async function runWithDiagnostics<T>(
  browser: Browser,
  fn: () => Promise<T>,
  label: string
): Promise<T> {
  try {
    return await fn();
  } catch (error) {
    console.error(`Step "${label}" failed:`, error);

    // Capture screenshot and DOM snapshot for debugging
    const [screenshot, snapshot] = await Promise.allSettled([
      browser.screenshot({ path: `debug-${label}-${Date.now()}.png` }),
      browser.snapshot(),
    ]);

    if (screenshot.status === 'fulfilled') {
      console.info('Screenshot saved:', screenshot.value);
    }
    if (snapshot.status === 'fulfilled') {
      console.info('DOM snapshot:', snapshot.value.substring(0, 500));
    }

    throw error;
  }
}
```

## Error Reference Table

| Error Name              | Cause                                         | Recommended Action                        |
|-------------------------|-----------------------------------------------|-------------------------------------------|
| `TimeoutError`          | Operation exceeded timeout                    | Increase timeout or check page state      |
| `ElementNotFoundError`  | Selector matched nothing                      | Verify selector, check page has loaded    |
| `NavigationError`       | Page failed to load                           | Check URL, network, and server status     |
| `FrameDetachedError`    | iframe was removed during interaction         | Re-acquire frame reference                |
| `TargetClosedError`     | Browser tab/page was closed unexpectedly      | Restart session                           |
| `ProtocolError`         | Low-level browser protocol failure            | Restart browser instance                  |
| `EvaluationError`       | JavaScript execution in page context failed   | Check injected script for syntax errors   |

## Best Practices

- Always set explicit timeouts rather than relying on defaults for critical paths.
- Use `try/catch` around navigation and element interactions in automated workflows.
- Capture a screenshot and snapshot whenever an unexpected error occurs.
- Prefer specific error types over catching all errors to avoid swallowing unexpected failures.
- Implement exponential backoff for retries to avoid overwhelming slow servers.
