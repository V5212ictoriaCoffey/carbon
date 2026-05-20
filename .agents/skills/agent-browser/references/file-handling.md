# File Handling

This reference covers file upload, download, and filesystem interaction patterns for the agent-browser skill.

## File Uploads

### Single File Upload

Use the `uploadFile` command to interact with file input elements:

```typescript
import { uploadFile } from '../commands';

// Upload a single file to a file input element
await uploadFile(page, '#file-input', '/path/to/local/file.pdf');
```

### Multiple File Upload

```typescript
// Upload multiple files at once
await uploadFile(page, 'input[type="file"][multiple]', [
  '/path/to/file1.png',
  '/path/to/file2.png',
  '/path/to/file3.png',
]);
```

### Drag-and-Drop File Upload

Some interfaces require drag-and-drop instead of input interaction:

```typescript
import { dragAndDropFile } from '../commands';

// Simulate dragging a file onto a drop zone
await dragAndDropFile(page, '#drop-zone', '/path/to/file.csv');
```

## File Downloads

### Intercepting Downloads

Capture downloads before they are saved to disk:

```typescript
import { waitForDownload } from '../commands';

// Trigger a download and capture the result
const download = await waitForDownload(page, async () => {
  await page.click('#download-button');
});

console.log('Downloaded file name:', download.suggestedFilename());
console.log('Download URL:', download.url());
```

### Saving Downloaded Files

```typescript
import path from 'path';
import { waitForDownload } from '../commands';

const download = await waitForDownload(page, async () => {
  await page.click('#export-csv');
});

// Save to a specific path
const savePath = path.join('/tmp/downloads', download.suggestedFilename());
await download.saveAs(savePath);
```

### Download Timeout Configuration

```typescript
const download = await waitForDownload(
  page,
  async () => {
    await page.click('#generate-report');
  },
  { timeout: 60_000 } // 60 seconds for large files
);
```

## Reading Downloaded File Content

```typescript
import fs from 'fs/promises';
import path from 'path';

async function downloadAndRead(page: Page, selector: string): Promise<string> {
  const download = await waitForDownload(page, async () => {
    await page.click(selector);
  });

  const tmpPath = path.join('/tmp', download.suggestedFilename());
  await download.saveAs(tmpPath);

  const content = await fs.readFile(tmpPath, 'utf-8');
  await fs.unlink(tmpPath); // clean up temp file
  return content;
}
```

## File Input Visibility Workarounds

File inputs are often hidden. Use `setInputFiles` directly when the element is not interactable:

```typescript
// Bypass visibility restrictions on hidden file inputs
const fileInput = page.locator('input[type="file"]');
await fileInput.setInputFiles('/path/to/file.txt');
```

## Verifying Uploaded Files

After upload, verify the UI reflects the expected filename:

```typescript
await uploadFile(page, '#avatar-input', '/path/to/avatar.jpg');

// Check that the filename appears in the UI
const label = page.locator('.file-name-label');
await label.waitFor({ state: 'visible' });
const text = await label.textContent();
console.assert(text?.includes('avatar.jpg'), 'File name not shown in UI');
```

## Notes

- File paths must be absolute or resolved relative to the process working directory.
- Downloads are only intercepted when the browser context has `acceptDownloads: true` (set by default in this skill).
- Large file uploads may require increasing the default navigation timeout via `page.setDefaultTimeout()`.
- Drag-and-drop uploads rely on the `DataTransfer` API and may not work in all browser contexts.
