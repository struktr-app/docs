# Struktr Documentation

<p align="center">
  <strong>Transform PDFs into structured, machine-readable data</strong>
</p>

<p align="center">
  <a href="#getting-started">Getting Started</a> •
  <a href="./docs/api-reference.md">API Reference</a> •
  <a href="./docs/sdk/">SDKs</a> •
  <a href="./docs/examples/">Examples</a>
</p>

---

## What is Struktr?

Struktr is a powerful API and SDK platform that converts unstructured PDF documents into clean, structured JSON data. Whether you're processing invoices, contracts, reports, or any other document type, Struktr extracts the information you need in a format that's easy to work with.

### Key Features

- 🔄 **PDF to JSON Conversion** - Transform any PDF into structured, queryable JSON data
- 📊 **Table Extraction** - Accurately extract tables with row/column relationships preserved
- 📝 **Form Field Recognition** - Automatically identify and extract form fields and their values
- 🏷️ **Named Entity Recognition** - Identify dates, amounts, names, addresses, and more
- 🔗 **Document Linking** - Cross-reference related documents and maintain relationships
- ⚡ **High Performance** - Process documents in seconds with our optimized pipeline
- 🔒 **Enterprise Security** - SOC 2 compliant with end-to-end encryption

## Getting Started

### 1. Get Your API Key

Sign up at [struktr.app](https://struktr.app) to get your API key. Your key will be displayed in your dashboard immediately after registration.

### 2. Install the SDK

Choose your preferred language:

**Python**
```bash
pip install struktr
```

**Node.js**
```bash
npm install @struktr/sdk
```

**Go**
```bash
go get github.com/struktr-app/struktr-go
```

### 3. Convert Your First PDF

**Python**
```python
from struktr import Struktr

client = Struktr(api_key="your-api-key")

# Convert a PDF to structured JSON
result = client.parse("invoice.pdf")

# Access extracted data
print(result.data)
# {
#   "document_type": "invoice",
#   "vendor": "Acme Corp",
#   "invoice_number": "INV-2024-001",
#   "total": 1250.00,
#   "line_items": [...]
# }
```

**Node.js**
```javascript
import { Struktr } from '@struktr/sdk';

const client = new Struktr({ apiKey: 'your-api-key' });

// Convert a PDF to structured JSON
const result = await client.parse('invoice.pdf');

console.log(result.data);
// {
//   document_type: 'invoice',
//   vendor: 'Acme Corp',
//   invoice_number: 'INV-2024-001',
//   total: 1250.00,
//   line_items: [...]
// }
```

## Use Cases

### Invoice Processing
Extract vendor information, line items, totals, and payment details from invoices automatically.

### Contract Analysis
Pull key terms, parties, dates, and clauses from legal documents.

### Report Digitization
Convert financial reports, research papers, and business documents into queryable data.

### Form Processing
Extract filled form data including checkboxes, text fields, and signatures.

## Documentation

| Resource | Description |
|----------|-------------|
| [API Reference](./docs/api-reference.md) | Complete API documentation with all endpoints |
| [Python SDK](./docs/sdk/python.md) | Python SDK installation and usage guide |
| [Node.js SDK](./docs/sdk/nodejs.md) | Node.js SDK installation and usage guide |
| [Go SDK](./docs/sdk/go.md) | Go SDK installation and usage guide |
| [Examples](./docs/examples/) | Real-world examples and use cases |
| [Webhooks](./docs/webhooks.md) | Configure webhooks for async processing |
| [Error Handling](./docs/errors.md) | Error codes and troubleshooting guide |

## Support

- 📧 Email: support@struktr.app
- 💬 Discord: [Join our community](https://discord.gg/struktr)
- 📖 Status: [status.struktr.app](https://status.struktr.app)

## License

See [LICENSE](./LICENSE) for details.