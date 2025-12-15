# Bug Report: Deno `node:fs` `createWriteStream` Polyfill Fails to Write Data on Premature Process Exit with `pdfkit`

## Summary

When `pdfkit` is used as a `Readable` stream and piped to Deno's `node:fs` `createWriteStream` polyfill, it fails to produce a complete PDF. This occurs because `pdfkit` itself (in versions 0.13.0 and 0.17.2, and likely others) has a bug where its `Readable` stream does not emit the `'end'` event after `doc.end()` when form fields are present in the document.

While this `pdfkit` bug affects both Node.js and Deno, native Node.js `fs.createWriteStream` will still write the partial data that `pdfkit` managed to push to the stream before the process exits, resulting in a truncated but non-empty PDF file. In contrast, Deno's `node:fs` `createWriteStream` polyfill produces a **0-byte file**, indicating it fails to flush *any* data to disk if the piped `Readable` stream does not properly signal its end, and the process exits.

This difference in behavior on process exit for an unfinalized `WritableStream` indicates an incompatibility or bug in Deno's `node:fs` `createWriteStream` polyfill, leading to more severe data loss than in native Node.js.

## Environment Details

*   **Deno Version:** <Please provide your Deno version, e.g., `deno --version` output>
*   **Node.js Version:** <Please provide your Node.js version, e.g., `node --version` output>
*   **`pdfkit` Versions Tested:** `0.13.0` and `0.17.2` (behavior is consistent across these versions for this bug).

## Steps to Reproduce

### 1. Node.js `pdfkit` Minimal Failing Case (`test_pdfkit_node.js`)

This demonstrates that `pdfkit` fails to emit `'end'` in Node.js, and produces a truncated PDF.

**File:** `test_pdfkit_node.js` (create this file in a Node.js project directory, e.g., `/Users/jpravetz/dev/epdoc/bikelog`)

```javascript
// test_pdfkit_node.js
// Node.js test case for pdfkit@0.17.2 stream finalization issue.
// This test directly uses pdfkit@0.17.2 from node_modules and native Node.js fs.
// Expected outcome: The script should complete successfully (exit code 0),
//                   and the generated PDF file should be incomplete (truncated, missing %%EOF).

const PDFDocument = require('pdfkit'); // Ensure pdfkit@0.17.2 is installed
const fs = require('fs');
const path = require('path');

console.log('--- Node.js Minimal Stream Test Start ---');

const outputPath = path.join(__dirname, 'tmp', 'test_pdfkit_node_output.pdf');
const outputFile = fs.createWriteStream(outputPath);

// --- Stream Event Listeners for Debugging ---
const doc = new PDFDocument({
  size: [918, 1188],
  compress: false,
});

doc.on('data', (chunk) => {
  console.log(`[Readable-Doc] Event: data (chunk size: ${chunk.length})`);
});
doc.on('end', () => {
  console.log('[Readable-Doc] Event: end'); // This event is NOT emitted by pdfkit
});
doc.on('error', (err) => {
  console.error('[Readable-Doc] Event: error', err);
  process.exit(1);
});

outputFile.on('pipe', () => {
  console.log('[Writable-Stream] Event: pipe');
});
outputFile.on('finish', () => {
  console.log('[Writable-Stream] Event: finish'); // This event is NOT emitted by WritableStream
});
outputFile.on('close', () => {
  console.log('[Writable-Stream] Event: close'); // This event is NOT emitted by WritableStream
  console.log('--- Node.js Minimal Stream Test Finished ---');
});
outputFile.on('error', (err) => {
  console.error('[Writable-Stream] Event: error', err);
  process.exit(1);
});

// --- Pipe the streams ---
doc.pipe(outputFile);

// --- PDF Content with Form Fields (problematic configuration) ---
doc.font('Helvetica');
doc.initForm();
doc.text('Page 1 Content');

const fieldDay = doc.formField('day');
const fieldSummary = doc.formField('summary');
const fieldGrid = doc.formField('grid');

const buttonOpts = {
  label: "Test Button",
  Parent: fieldGrid
};
doc.formPushButton('testButton', 100, 100, 80, 20, buttonOpts);

// Simulate the finalization calls found in the working Node.js application
doc.flushPages();
doc.end();

console.log('--- doc.end() called ---');

// Node.js will implicitly keep the process alive long enough to write the partial data.
```

**`package.json` for Node.js Project (install `pdfkit`):**

```json
{
  "name": "pdfkit-test",
  "version": "1.0.0",
  "main": "test_pdfkit_node.js",
  "dependencies": {
    "pdfkit": "^0.17.2"
  }
}
```

**Execute Node.js Test:**

```bash
# In the Node.js project directory
npm install
mkdir -p tmp
node test_pdfkit_node.js
```

**Expected Node.js Output:**

The script will print the `data` events and `--- doc.end() called ---`, and then exit cleanly. It will **not** print the `finish` or `close` events for the `WritableStream`, nor the `end` event for the `Readable` stream.

**Verify Node.js Output File:**

```bash
tail ./tmp/test_pdfkit_node_output.pdf
# Expected: A truncated PDF, ending abruptly (e.g., with 'endobj'), but containing some data.
#           File size should be > 0 bytes.
```

### 2. Deno `pdfkit` Minimal Failing Case (`test_pdfkit_deno.ts`)

This demonstrates the same `pdfkit` bug in Deno, but Deno produces a 0-byte file.

**File:** `test_pdfkit_deno.ts` (create this file in your Deno project directory, e.g., `/Users/jpravetz/dev/@jpravetz/bikelog`)

```typescript
// test_pdfkit_deno.ts
// Deno test case for pdfkit@0.17.2 stream finalization issue.
// This test uses pdfkit@0.17.2 and Deno's node:fs for stream handling.
// Expected outcome: The script should complete without hanging (exit code 0),
//                   but the generated PDF file will be EMPTY (0 bytes).

import * as fs from 'node:fs';
import PDFDocument from 'npm:pdfkit@^0.17.2'; // Using pdfkit@0.17.2
import * as path from 'node:path';

console.log('--- Deno Test (pdfkit@0.17.2) Start ---');

const doc = new PDFDocument({
  size: [918, 1188],
  compress: false,
});

// --- Stream Event Listeners for Debugging ---
doc.on('data', (chunk) => {
  console.log(`[Readable-Doc] Event: data (chunk size: ${chunk.length})`);
});
doc.on('end', () => {
  console.log('[Readable-Doc] Event: end'); // This event is NOT emitted by pdfkit
});
doc.on('error', (err) => {
  console.error('[Readable-Doc] Event: error', err);
  Deno.exit(1);
});

const outputPath = path.join(Deno.cwd(), 'tmp', 'test_pdfkit_deno_output.pdf');
const stream = fs.createWriteStream(outputPath);

stream.on('pipe', () => {
  console.log('[Writable-Stream] Event: pipe');
});
stream.on('finish', () => {
  console.log('[Writable-Stream] Event: finish'); // This event is NOT emitted by WritableStream
});
stream.on('close', () => {
  console.log('[Writable-Stream] Event: close'); // This event is NOT emitted by WritableStream
  console.log('--- Deno Test (pdfkit@0.17.2) Finished ---');
});
stream.on('error', (err) => {
  console.error('[Writable-Stream] Event: error', err);
  Deno.exit(1);
});


// --- Pipe the streams ---
doc.pipe(stream);

// --- PDF Content with Form Fields (problematic configuration) ---
doc.font('Helvetica');
doc.initForm();
doc.text('Page 1 Content');

const fieldDay = doc.formField('day');
const fieldSummary = doc.formField('summary');
const fieldGrid = doc.formField('grid');

const buttonOpts = {
  label: "Test Button",
  Parent: fieldGrid
};
doc.formPushButton('testButton', 100, 100, 80, 20, buttonOpts);

// Simulate the finalization calls found in the working Node.js application
doc.flushPages();
doc.end();

console.log('--- doc.end() called ---');

// Deno will implicitly keep the process alive until stream events finish, but
// in this case, they don't finish due to the pdfkit bug. The process exits early.
```

**`deno.json` for Deno Project (ensure `pdfkit` is installed):**

```json
{
  "imports": {
    "pdfkit": "npm:pdfkit@^0.17.2",
    "node:stream": "npm:@types/node/stream@latest",
    "node:fs": "npm:@types/node/fs@latest",
    "node:path": "npm:@types/node/path@latest"
  }
}
```
*(Note: `npm:@types/node/stream@latest`, `npm:@types/node/fs@latest`, `npm:@types/node/path@latest` are placeholders. You should check your `deno.json` for the exact corresponding Deno standard library imports for Node compatibility or use the npm specifiers if they are explicitly allowed and work. For `pdfkit`, `npm:pdfkit@^0.17.2` is correct.)*

**Execute Deno Test:**

```bash
# In the Deno project directory
deno cache test_pdfkit_deno.ts
mkdir -p tmp
deno run -A test_pdfkit_deno.ts
```

**Expected Deno Output:**

The script will print the `data` events and `--- doc.end() called ---`, and then exit cleanly. It will **not** print the `finish` or `close` events for the `WritableStream`, nor the `end` event for the `Readable` stream.

**Verify Deno Output File:**

```bash
ls -l ./tmp/test_pdfkit_deno_output.pdf
# Expected: File size should be 0 bytes.
```

### 3. Generic Stream Comparison

These tests demonstrate that Deno's basic `node:stream` and `node:fs` polyfills work correctly for simple streams.

**File:** `test_stream_node.js` (from previous steps)

**File:** `test_stream_deno.ts` (from previous steps)

---

## Explanation

The core issue is a bug in `pdfkit` (versions 0.13.0 and 0.17.2) where its `Readable` stream fails to emit the `'end'` event after `doc.end()` is called, specifically when form fields are present in the document. This prevents the piped `WritableStream` from receiving the signal to fully flush and close.

However, the key difference between Node.js and Deno lies in how their `fs.createWriteStream` implementations handle such an unfinalized stream upon process exit:

*   **Native Node.js `fs.createWriteStream`:** On process exit, Node.js implicitly attempts to flush any buffered data of unfinalized `WritableStream`s to disk. This results in a **truncated, but non-empty**, PDF file, containing all data that `pdfkit` pushed before the script's main thread completed.
*   **Deno `node:fs` `createWriteStream` Polyfill:** On process exit, Deno's polyfill *does not* appear to flush buffered data if the piped `Readable` stream has not properly signaled its end. This results in a **0-byte file**, indicating complete data loss.

This difference in behavior is a bug in Deno's `node:fs` `createWriteStream` polyfill, making it less robust than its native Node.js counterpart when dealing with misbehaving `Readable` streams. Deno should, at minimum, attempt to flush any pending data to the file before process exit, even if the `Readable` stream has not properly called `end()`.
