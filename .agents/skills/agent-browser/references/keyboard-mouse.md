# Keyboard & Mouse Interaction Reference

This reference covers keyboard input simulation, mouse actions, drag-and-drop, and pointer events for the agent-browser skill.

## Keyboard Actions

### Typing Text

Use `type` to simulate keyboard input into a focused element:

```typescript
import { type Page } from 'playwright';

/**
 * Types text into the currently focused element or a target selector.
 * Simulates real keystrokes with configurable delay between keys.
 */
async function typeText(
  page: Page,
  selector: string,
  text: string,
  options?: { delay?: number; clearFirst?: boolean }
): Promise<void> {
  const { delay = 50, clearFirst = false } = options ?? {};

  await page.locator(selector).waitFor({ state: 'visible' });

  if (clearFirst) {
    await page.locator(selector).selectText();
    await page.keyboard.press('Backspace');
  }

  await page.locator(selector).type(text, { delay });
}
```

### Pressing Keys

```typescript
/**
 * Press a single key or key combination.
 * Supports modifiers: Control, Shift, Alt, Meta
 *
 * Examples:
 *   pressKey(page, 'Enter')
 *   pressKey(page, 'Control+A')
 *   pressKey(page, 'Shift+Tab')
 */
async function pressKey(page: Page, key: string): Promise<void> {
  await page.keyboard.press(key);
}

/**
 * Hold a modifier key while performing an action.
 */
async function withModifier(
  page: Page,
  modifier: 'Control' | 'Shift' | 'Alt' | 'Meta',
  action: () => Promise<void>
): Promise<void> {
  await page.keyboard.down(modifier);
  try {
    await action();
  } finally {
    await page.keyboard.up(modifier);
  }
}
```

## Mouse Actions

### Click Variants

```typescript
/**
 * Perform a standard click on a selector.
 * Waits for the element to be actionable before clicking.
 */
async function click(
  page: Page,
  selector: string,
  options?: { button?: 'left' | 'right' | 'middle'; clickCount?: number }
): Promise<void> {
  await page.locator(selector).click({
    button: options?.button ?? 'left',
    clickCount: options?.clickCount ?? 1,
  });
}

/**
 * Double-click an element.
 */
async function doubleClick(page: Page, selector: string): Promise<void> {
  await page.locator(selector).dblclick();
}

/**
 * Right-click to open context menu.
 */
async function rightClick(page: Page, selector: string): Promise<void> {
  await page.locator(selector).click({ button: 'right' });
}
```

### Hover

```typescript
/**
 * Hover over an element to trigger hover states or tooltips.
 * Optionally specify a position offset within the element.
 */
async function hover(
  page: Page,
  selector: string,
  options?: { position?: { x: number; y: number } }
): Promise<void> {
  await page.locator(selector).hover({ position: options?.position });
}
```

## Drag and Drop

```typescript
/**
 * Drag an element from source to target selector.
 * Uses mouse down, move, and up sequence for compatibility.
 */
async function dragAndDrop(
  page: Page,
  sourceSelector: string,
  targetSelector: string
): Promise<void> {
  await page.dragAndDrop(sourceSelector, targetSelector);
}

/**
 * Drag an element to specific page coordinates.
 * Useful when the drop target is a coordinate rather than an element.
 */
async function dragToCoordinates(
  page: Page,
  sourceSelector: string,
  targetX: number,
  targetY: number
): Promise<void> {
  const source = page.locator(sourceSelector);
  const sourceBounds = await source.boundingBox();

  if (!sourceBounds) {
    throw new Error(`Element not found or not visible: ${sourceSelector}`);
  }

  const startX = sourceBounds.x + sourceBounds.width / 2;
  const startY = sourceBounds.y + sourceBounds.height / 2;

  await page.mouse.move(startX, startY);
  await page.mouse.down();
  await page.mouse.move(targetX, targetY, { steps: 10 });
  await page.mouse.up();
}
```

## Scroll Actions

```typescript
/**
 * Scroll an element into view.
 */
async function scrollIntoView(page: Page, selector: string): Promise<void> {
  await page.locator(selector).scrollIntoViewIfNeeded();
}

/**
 * Scroll the page by a given pixel delta.
 */
async function scrollBy(
  page: Page,
  deltaX: number,
  deltaY: number
): Promise<void> {
  await page.mouse.wheel(deltaX, deltaY);
}
```

## Exported Utilities

```typescript
export {
  typeText,
  pressKey,
  withModifier,
  click,
  doubleClick,
  rightClick,
  hover,
  dragAndDrop,
  dragToCoordinates,
  scrollIntoView,
  scrollBy,
};
```

## Notes

- Always prefer `locator`-based APIs over `page.$` for better auto-waiting.
- Use `delay` in `type()` to avoid race conditions on slow inputs.
- `dragToCoordinates` uses `steps: 10` to simulate realistic mouse movement.
- Modifier keys must be released even if the action throws — use `try/finally`.
