# Node.js SDK

The official Node.js SDK for Struktr API.

## Installation

```bash
npm install @struktr/sdk
```

Or with yarn:

```bash
yarn add @struktr/sdk
```

**Requirements:** Node.js 16+

## Quick Start

```javascript
import { Struktr } from '@struktr/sdk';

// Initialize the client
const client = new Struktr({ apiKey: 'your-api-key' });

// Parse a PDF file
const result = await client.parse('document.pdf');

// Access the structured data
console.log(result.data);
```

## Configuration

### Client Options

```javascript
import { Struktr } from '@struktr/sdk';

const client = new Struktr({
  apiKey: 'your-api-key',
  baseUrl: 'https://api.struktr.app/v1', // Optional: custom base URL
  timeout: 30000, // Request timeout in milliseconds
  maxRetries: 3, // Number of retry attempts
});
```

### Environment Variables

You can also configure the client using environment variables:

```bash
export STRUKTR_API_KEY="your-api-key"
```

```javascript
import { Struktr } from '@struktr/sdk';

// API key is automatically read from environment
const client = new Struktr();
```

## Parsing Documents

### From File Path

```javascript
const result = await client.parse('invoice.pdf');
```

### From Buffer

```javascript
import { readFileSync } from 'fs';

const buffer = readFileSync('invoice.pdf');
const result = await client.parseBuffer(buffer);
```

### From URL

```javascript
const result = await client.parseUrl('https://example.com/invoice.pdf');
```

### From Stream

```javascript
import { createReadStream } from 'fs';

const stream = createReadStream('invoice.pdf');
const result = await client.parseStream(stream);
```

## Parsing Options

```javascript
const result = await client.parse('document.pdf', {
  extractTables: true,
  extractImages: false,
  ocrEnabled: true,
  language: 'en',
  outputFormat: 'structured',
});
```

### Available Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `extractTables` | boolean | `true` | Extract tables from the document |
| `extractImages` | boolean | `false` | Include images as base64 |
| `ocrEnabled` | boolean | `true` | Enable OCR for scanned documents |
| `language` | string | `'en'` | Document language |
| `outputFormat` | string | `'structured'` | Output format |

## Custom Schema Extraction

Define exactly what data to extract:

```javascript
const schema = {
  type: 'object',
  properties: {
    invoiceNumber: { type: 'string' },
    vendor: { type: 'string', description: 'Company name' },
    total: { type: 'number' },
    lineItems: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          description: { type: 'string' },
          quantity: { type: 'integer' },
          amount: { type: 'number' },
        },
      },
    },
  },
};

const result = await client.parse('invoice.pdf', { schema });

console.log(result.data.invoiceNumber); // "INV-2024-001"
console.log(result.data.total); // 1250.00
```

## Working with Results

### ParseResult Object

```javascript
const result = await client.parse('document.pdf');

// Document metadata
console.log(result.id);        // "doc_abc123"
console.log(result.status);    // "completed"
console.log(result.pages);     // 3
console.log(result.createdAt); // Date object

// Extracted data
console.log(result.data);      // Full structured data object
console.log(result.rawText);   // Raw text content
console.log(result.tables);    // Array of extracted tables
console.log(result.fields);    // Extracted fields with confidence scores
```

### Accessing Fields

```javascript
// Get a field value
const vendor = result.getField('vendorName');
console.log(vendor.value);      // "Acme Corp"
console.log(vendor.confidence); // 0.98

// Get with default
const amount = result.getField('amount', { default: 0 });
```

### Working with Tables

```javascript
for (const table of result.tables) {
  console.log(`Table on page ${table.page}`);
  console.log(`Headers: ${table.headers.join(', ')}`);

  for (const row of table.rows) {
    console.log(row);
  }

  // Convert to array of objects
  const data = table.toObjects();
}
```

## Async Processing

For large documents, use async processing:

```javascript
// Start async processing
const job = await client.parseAsync('large-document.pdf', {
  webhookUrl: 'https://your-server.com/webhooks',
});

console.log(job.id);     // "doc_xyz789"
console.log(job.status); // "pending"

// Poll for completion
const result = await job.wait({ timeout: 300000 }); // Wait up to 5 minutes

// Or check status manually
const status = await client.getDocument(job.id);
```

## Batch Processing

Process multiple documents at once:

```javascript
// Submit batch
const batch = await client.parseBatch([
  'invoice1.pdf',
  'invoice2.pdf',
  'invoice3.pdf',
]);

console.log(batch.id);    // "batch_abc123"
console.log(batch.total); // 3

// Wait for all to complete
const results = await batch.wait();

for (const result of results) {
  console.log(`${result.id}: ${JSON.stringify(result.data)}`);
}
```

## Document Management

### List Documents

```javascript
// List recent documents
const documents = await client.listDocuments({ limit: 10 });

for (const doc of documents.data) {
  console.log(`${doc.id}: ${doc.status}`);
}

// With filters
const sevenDaysAgo = new Date();
sevenDaysAgo.setDate(sevenDaysAgo.getDate() - 7);

const filtered = await client.listDocuments({
  status: 'completed',
  from: sevenDaysAgo,
  limit: 50,
});
```

### Get Document

```javascript
const doc = await client.getDocument('doc_abc123');
console.log(doc.data);
```

### Delete Document

```javascript
await client.deleteDocument('doc_abc123');
```

## Error Handling

```javascript
import {
  Struktr,
  StruktrError,
  AuthenticationError,
  RateLimitError,
  ProcessingError,
} from '@struktr/sdk';

const client = new Struktr();

try {
  const result = await client.parse('document.pdf');
} catch (error) {
  if (error instanceof AuthenticationError) {
    console.error('Invalid API key');
  } else if (error instanceof RateLimitError) {
    console.error(`Rate limited. Retry after ${error.retryAfter} seconds`);
  } else if (error instanceof ProcessingError) {
    console.error(`Processing failed: ${error.message}`);
  } else if (error instanceof StruktrError) {
    console.error(`API error: ${error.code} - ${error.message}`);
  } else {
    throw error;
  }
}
```

## TypeScript Support

The SDK is written in TypeScript and includes full type definitions:

```typescript
import { Struktr, ParseResult, ParseOptions, Schema } from '@struktr/sdk';

const client = new Struktr({ apiKey: 'your-api-key' });

const options: ParseOptions = {
  extractTables: true,
  ocrEnabled: true,
};

const result: ParseResult = await client.parse('document.pdf', options);

// Type-safe access to fields
interface InvoiceData {
  invoiceNumber: string;
  vendor: string;
  total: number;
  lineItems: Array<{
    description: string;
    quantity: number;
    amount: number;
  }>;
}

const invoiceResult = await client.parse<InvoiceData>('invoice.pdf', {
  schema: invoiceSchema,
});

// invoiceResult.data is typed as InvoiceData
console.log(invoiceResult.data.invoiceNumber);
```

## Examples

### Invoice Processing

```javascript
import { Struktr } from '@struktr/sdk';

const client = new Struktr();

// Define invoice schema
const invoiceSchema = {
  type: 'object',
  properties: {
    vendor: { type: 'string' },
    invoiceNumber: { type: 'string' },
    date: { type: 'string', format: 'date' },
    dueDate: { type: 'string', format: 'date' },
    subtotal: { type: 'number' },
    tax: { type: 'number' },
    total: { type: 'number' },
    lineItems: {
      type: 'array',
      items: {
        type: 'object',
        properties: {
          description: { type: 'string' },
          quantity: { type: 'integer' },
          unitPrice: { type: 'number' },
          amount: { type: 'number' },
        },
      },
    },
  },
};

const result = await client.parse('invoice.pdf', { schema: invoiceSchema });

// Process the extracted data
console.log(`Invoice from: ${result.data.vendor}`);
console.log(`Total due: $${result.data.total.toFixed(2)}`);
console.log(`Due date: ${result.data.dueDate}`);
```

### Express.js Webhook Handler

```javascript
import express from 'express';
import crypto from 'crypto';

const app = express();
app.use(express.json());

const WEBHOOK_SECRET = process.env.STRUKTR_WEBHOOK_SECRET;

function verifyWebhook(payload, signature) {
  const expected = crypto
    .createHmac('sha256', WEBHOOK_SECRET)
    .update(JSON.stringify(payload))
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(`sha256=${expected}`),
    Buffer.from(signature)
  );
}

app.post('/webhooks/struktr', (req, res) => {
  const signature = req.headers['x-struktr-signature'];

  if (!verifyWebhook(req.body, signature)) {
    return res.status(401).send('Invalid signature');
  }

  const { event, data } = req.body;

  switch (event) {
    case 'document.completed':
      console.log(`Document ${data.id} completed:`, data.data);
      // Process the completed document
      break;
    case 'document.failed':
      console.error(`Document ${data.id} failed:`, data.error);
      break;
  }

  res.status(200).send('OK');
});

app.listen(3000);
```

## API Reference

See the full [API Reference](../api-reference.md) for all available methods and options.
