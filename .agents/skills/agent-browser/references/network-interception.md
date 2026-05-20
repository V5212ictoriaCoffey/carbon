# Network Interception

This reference covers how to intercept, mock, and monitor network requests during browser automation sessions.

## Overview

Network interception allows you to:
- Mock API responses without a real backend
- Block specific requests (ads, trackers, etc.)
- Modify request/response headers
- Simulate network conditions (slow connections, errors)
- Record and replay network traffic

## Basic Request Interception

### Intercepting All Requests

```typescript
import { BrowserSession } from '../session';

const session = new BrowserSession();
await session.start();

// Enable request interception
await session.page.setRequestInterception(true);

session.page.on('request', (request) => {
  // Allow all requests by default
  request.continue();
});
```

### Blocking Specific Requests

```typescript
session.page.on('request', (request) => {
  const url = request.url();
  const blockedDomains = ['ads.example.com', 'tracker.analytics.io'];

  if (blockedDomains.some((domain) => url.includes(domain))) {
    request.abort();
  } else {
    request.continue();
  }
});
```

## Mocking API Responses

### Simple Response Mock

```typescript
session.page.on('request', (request) => {
  if (request.url().includes('/api/users') && request.method() === 'GET') {
    request.respond({
      status: 200,
      contentType: 'application/json',
      body: JSON.stringify([
        { id: 1, name: 'Alice', email: 'alice@example.com' },
        { id: 2, name: 'Bob', email: 'bob@example.com' },
      ]),
    });
  } else {
    request.continue();
  }
});
```

### Mocking Error Responses

```typescript
session.page.on('request', (request) => {
  if (request.url().includes('/api/restricted')) {
    request.respond({
      status: 403,
      contentType: 'application/json',
      body: JSON.stringify({ error: 'Forbidden', message: 'Access denied' }),
    });
  } else {
    request.continue();
  }
});
```

## Modifying Requests

### Adding Authentication Headers

```typescript
session.page.on('request', (request) => {
  const headers = {
    ...request.headers(),
    Authorization: `Bearer ${process.env.API_TOKEN}`,
    'X-Custom-Header': 'agent-browser/1.0',
  };

  request.continue({ headers });
});
```

### Redirecting Requests

```typescript
session.page.on('request', (request) => {
  const url = request.url();

  // Redirect staging API calls to local mock server
  if (url.includes('staging-api.example.com')) {
    const newUrl = url.replace('staging-api.example.com', 'localhost:3001');
    request.continue({ url: newUrl });
  } else {
    request.continue();
  }
});
```

## Monitoring Network Activity

### Capturing Response Data

```typescript
const networkLog: Array<{ url: string; status: number; duration: number }> = [];

session.page.on('response', async (response) => {
  const request = response.request();
  const timing = response.timing();

  networkLog.push({
    url: request.url(),
    status: response.status(),
    duration: timing ? timing.receiveHeadersEnd - timing.sendStart : -1,
  });
});

// After navigation, inspect captured requests
await session.page.goto('https://example.com');
console.log('Network requests:', networkLog.length);
```

### Waiting for Specific Requests

```typescript
// Wait for a specific API call to complete
const [response] = await Promise.all([
  session.page.waitForResponse((res) =>
    res.url().includes('/api/data') && res.status() === 200
  ),
  session.page.click('#load-data-button'),
]);

const data = await response.json();
console.log('API response:', data);
```

## Simulating Network Conditions

```typescript
// Simulate slow 3G connection
const client = await session.page.target().createCDPSession();
await client.send('Network.emulateNetworkConditions', {
  offline: false,
  downloadThroughput: (750 * 1024) / 8, // 750 kbps
  uploadThroughput: (250 * 1024) / 8,   // 250 kbps
  latency: 100,                          // 100ms RTT
});

// Simulate offline mode
await client.send('Network.emulateNetworkConditions', {
  offline: true,
  downloadThroughput: 0,
  uploadThroughput: 0,
  latency: 0,
});
```

## Best Practices

- Always call `request.continue()`, `request.abort()`, or `request.respond()` — leaving a request unhandled will cause a timeout
- Use `request.resourceType()` to filter by type (`document`, `stylesheet`, `image`, `script`, `xhr`, `fetch`)
- Disable interception when not needed to avoid performance overhead
- Use `page.waitForResponse()` instead of arbitrary timeouts when waiting for API calls
- Log intercepted requests during debugging to understand traffic patterns

## Related

- [Session Management](./session-management.md)
- [Authentication](./authentication.md)
- [Proxy Support](./proxy-support.md)
