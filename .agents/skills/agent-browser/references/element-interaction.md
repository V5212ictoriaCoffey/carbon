# Element Interaction Reference

This document covers how to interact with DOM elements using the agent-browser skill, including clicking, typing, selecting, and asserting element states.

## Finding Elements

### By CSS Selector
```typescript
const element = await browser.findElement('button.submit-btn');
const elements = await browser.findElements('.list-item');
```

### By Text Content
```typescript
const button = await browser.findElementByText('Submit');
const link = await browser.findElementByText('Learn more', { exact: false });
```

### By Role (Accessibility)
```typescript
const dialog = await browser.findElementByRole('dialog');
const nav = await browser.findElementByRole('navigation', { name: 'Main' });
```

### By Test ID
```typescript
const form = await browser.findElementByTestId('login-form');
```

---

## Clicking Elements

### Basic Click
```typescript
await browser.click('button#submit');
```

### Click with Options
```typescript
await browser.click('a.nav-link', {
  button: 'left',       // 'left' | 'right' | 'middle'
  clickCount: 2,        // double-click
  delay: 50,            // ms between mousedown and mouseup
  modifiers: ['Shift'], // 'Alt' | 'Control' | 'Meta' | 'Shift'
});
```

### Force Click (bypasses actionability checks)
```typescript
await browser.click('button.hidden-btn', { force: true });
```

### Hover
```typescript
await browser.hover('.dropdown-trigger');
```

---

## Typing and Input

### Fill Input Field
```typescript
// Clears existing value and types new one
await browser.fill('input[name="email"]', 'user@example.com');
```

### Type (simulates keystroke-by-keystroke)
```typescript
await browser.type('input#search', 'carbon framework', { delay: 30 });
```

### Press Key
```typescript
await browser.press('input#search', 'Enter');
await browser.press('body', 'Escape');
await browser.press('textarea', 'Control+A'); // Select all
```

### Clear Input
```typescript
await browser.clear('input[name="username"]');
```

### Upload File
```typescript
await browser.uploadFile('input[type="file"]', '/path/to/file.pdf');
await browser.uploadFile('input[type="file"]', ['/path/file1.png', '/path/file2.png']);
```

---

## Select and Checkbox

### Select Dropdown Option
```typescript
await browser.selectOption('select#country', 'US');
await browser.selectOption('select#tags', ['typescript', 'nodejs']); // multi-select
await browser.selectOption('select#size', { label: 'Large' });
```

### Check / Uncheck Checkbox
```typescript
await browser.check('input[type="checkbox"]#agree');
await browser.uncheck('input[type="checkbox"]#newsletter');
```

### Set Checkbox State
```typescript
await browser.setChecked('input#terms', true);
```

---

## Drag and Drop

```typescript
await browser.dragAndDrop('.draggable-item', '.drop-zone');

// With precise coordinates
await browser.dragAndDrop('.card', '.column', {
  sourcePosition: { x: 10, y: 10 },
  targetPosition: { x: 50, y: 100 },
});
```

---

## Scrolling

```typescript
// Scroll element into view
await browser.scrollIntoView('.footer-section');

// Scroll by pixel amount
await browser.scroll('main', { deltaY: 500 });

// Scroll to bottom of page
await browser.scroll('body', { deltaY: 99999 });
```

---

## Reading Element State

### Get Text Content
```typescript
const text = await browser.getText('h1.page-title');
const allTexts = await browser.getAllTexts('.list-item');
```

### Get Attribute / Property
```typescript
const href = await browser.getAttribute('a.link', 'href');
const value = await browser.getProperty('input#email', 'value');
```

### Check Visibility / Enabled State
```typescript
const isVisible = await browser.isVisible('.modal');
const isEnabled = await browser.isEnabled('button#submit');
const isChecked = await browser.isChecked('input#agree');
```

### Get Bounding Box
```typescript
const box = await browser.getBoundingBox('.hero-image');
console.log(box); // { x, y, width, height }
```

---

## Waiting for Elements

```typescript
// Wait until element appears
await browser.waitForElement('.toast-notification', { timeout: 5000 });

// Wait until element is hidden
await browser.waitForElement('.loading-spinner', {
  state: 'hidden',
  timeout: 10000,
});

// Wait for text to appear
await browser.waitForText('.status-badge', 'Success');
```

---

## Assertions

```typescript
await browser.assertText('h1', 'Welcome to Carbon');
await browser.assertVisible('.success-banner');
await browser.assertHidden('.error-message');
await browser.assertAttributeEquals('input#email', 'type', 'email');
await browser.assertCount('.result-item', 5);
```

---

## Notes

- All selectors support CSS, XPath (prefix with `xpath=`), and text (prefix with `text=`).
- By default, interaction methods wait for the element to be actionable (visible, stable, not obscured).
- Use `{ timeout }` option on any method to override the default 30s timeout.
- Snapshot refs can be combined with element interactions — see [snapshot-refs.md](./snapshot-refs.md).
