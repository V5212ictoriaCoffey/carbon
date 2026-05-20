# Accessibility Testing with agent-browser

This reference covers accessibility auditing, ARIA inspection, and assistive technology simulation using the agent-browser skill.

## Overview

The agent-browser skill provides built-in support for accessibility testing through axe-core integration and native browser accessibility APIs. Use these capabilities to audit pages for WCAG compliance, inspect ARIA attributes, and simulate screen reader interactions.

## Running an Accessibility Audit

### Full Page Audit

```typescript
import { createBrowserAgent } from '../agent-browser';

const agent = await createBrowserAgent();
await agent.navigate('https://example.com');

const results = await agent.accessibility.audit();

console.log(`Violations: ${results.violations.length}`);
console.log(`Passes: ${results.passes.length}`);
console.log(`Incomplete: ${results.incomplete.length}`);
```

### Scoped Audit

Limit the audit to a specific element or subtree:

```typescript
const results = await agent.accessibility.audit({
  selector: '#main-content',
  runOnly: {
    type: 'tag',
    values: ['wcag2a', 'wcag2aa'],
  },
});
```

### Audit Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `selector` | `string` | `'body'` | CSS selector to scope the audit |
| `runOnly` | `RunOnly` | all rules | Restrict which axe rules run |
| `rules` | `Record<string, RuleObject>` | `{}` | Override individual rule configuration |
| `resultTypes` | `string[]` | all | Filter result types to return |
| `reporter` | `'v1' \| 'v2' \| 'no-passes'` | `'v2'` | Result reporter format |

## Inspecting the Accessibility Tree

### Get Full Tree

```typescript
const tree = await agent.accessibility.snapshot();
console.log(JSON.stringify(tree, null, 2));
```

### Get Node Properties

```typescript
const node = await agent.accessibility.getNode('#submit-button');

console.log(node.role);        // e.g. 'button'
console.log(node.name);        // accessible name
console.log(node.description); // accessible description
console.log(node.state);       // e.g. { disabled: false, focused: true }
```

### Traverse the Tree

```typescript
const tree = await agent.accessibility.snapshot({ interestingOnly: false });

function walk(node: AccessibilityNode, depth = 0) {
  const indent = '  '.repeat(depth);
  console.log(`${indent}[${node.role}] ${node.name ?? ''}`);
  for (const child of node.children ?? []) {
    walk(child, depth + 1);
  }
}

walk(tree);
```

## ARIA Attribute Inspection

```typescript
// Check a specific ARIA attribute
const expanded = await agent.accessibility.getAttribute('#dropdown', 'aria-expanded');
console.log(expanded); // 'true' | 'false' | null

// Get all ARIA attributes on an element
const attrs = await agent.accessibility.getAttributes('#modal');
// Returns: { 'aria-modal': 'true', 'aria-labelledby': 'modal-title', ... }
```

## Focus Management

### Check Focus Order

```typescript
const focusOrder = await agent.accessibility.getFocusOrder();
// Returns an array of selectors in tab order
console.log(focusOrder);
// ['#skip-link', 'nav a:first-child', 'nav a:nth-child(2)', '#search-input', ...]
```

### Assert Focus Trap

Verify that focus is correctly trapped within a modal or dialog:

```typescript
await agent.click('#open-modal');

const trapped = await agent.accessibility.isFocusTrapped('#modal');
consert.ok(trapped, 'Focus should be trapped inside the modal');
```

## Color Contrast

```typescript
const contrastResults = await agent.accessibility.checkContrast({
  selector: 'p, h1, h2, h3, a',
  level: 'AA', // 'A' | 'AA' | 'AAA'
});

for (const failure of contrastResults.failures) {
  console.warn(
    `Low contrast on ${failure.selector}: ` +
    `ratio ${failure.ratio.toFixed(2)} (required ${failure.required})`
  );
}
```

## Screen Reader Simulation

Simulate how a screen reader would announce an element:

```typescript
const announcement = await agent.accessibility.getAnnouncement('#alert-banner');
console.log(announcement);
// e.g. 'Alert: Your session will expire in 5 minutes. role=alert'
```

## Asserting Violations

Throw on any accessibility violations during a test:

```typescript
const results = await agent.accessibility.audit();

if (results.violations.length > 0) {
  const summary = results.violations
    .map(v => `[${v.impact}] ${v.id}: ${v.description} (${v.nodes.length} nodes)`)
    .join('\n');
  throw new Error(`Accessibility violations found:\n${summary}`);
}
```

## Integration with Snapshots

Accessibility snapshots can be stored alongside visual snapshots for regression testing. See [snapshot-refs.md](./snapshot-refs.md) for details on managing snapshot references.

```typescript
await agent.accessibility.assertSnapshot('homepage-a11y', {
  selector: 'main',
  updateOnMismatch: process.env.UPDATE_SNAPSHOTS === 'true',
});
```

## Error Handling

Accessibility methods throw `AccessibilityError` on unexpected failures. See [error-handling.md](./error-handling.md) for the full error hierarchy.

```typescript
try {
  await agent.accessibility.audit();
} catch (err) {
  if (err instanceof AccessibilityError) {
    console.error('Audit engine failed:', err.message);
  }
}
```

## Related References

- [element-interaction.md](./element-interaction.md) — Interacting with DOM elements
- [snapshot-refs.md](./snapshot-refs.md) — Managing visual and structural snapshots
- [error-handling.md](./error-handling.md) — Error types and recovery strategies
