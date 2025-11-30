# Python SDK

The official Python SDK for Struktr API.

## Installation

```bash
pip install struktr
```

**Requirements:** Python 3.8+

## Quick Start

```python
from struktr import Struktr

# Initialize the client
client = Struktr(api_key="your-api-key")

# Parse a PDF file
result = client.parse("document.pdf")

# Access the structured data
print(result.data)
```

## Configuration

### Client Options

```python
from struktr import Struktr

client = Struktr(
    api_key="your-api-key",
    base_url="https://api.struktr.app/v1",  # Optional: custom base URL
    timeout=30,  # Request timeout in seconds
    max_retries=3,  # Number of retry attempts
)
```

### Environment Variables

You can also configure the client using environment variables:

```bash
export STRUKTR_API_KEY="your-api-key"
```

```python
from struktr import Struktr

# API key is automatically read from environment
client = Struktr()
```

## Parsing Documents

### From File Path

```python
result = client.parse("invoice.pdf")
```

### From File Object

```python
with open("invoice.pdf", "rb") as f:
    result = client.parse(f)
```

### From URL

```python
result = client.parse_url("https://example.com/invoice.pdf")
```

### From Bytes

```python
pdf_bytes = download_pdf_somehow()
result = client.parse_bytes(pdf_bytes)
```

## Parsing Options

```python
from struktr import ParseOptions

options = ParseOptions(
    extract_tables=True,
    extract_images=False,
    ocr_enabled=True,
    language="en",
    output_format="structured"
)

result = client.parse("document.pdf", options=options)
```

### Available Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `extract_tables` | bool | `True` | Extract tables from the document |
| `extract_images` | bool | `False` | Include images as base64 |
| `ocr_enabled` | bool | `True` | Enable OCR for scanned documents |
| `language` | str | `"en"` | Document language |
| `output_format` | str | `"structured"` | Output format |

## Custom Schema Extraction

Define exactly what data to extract:

```python
from struktr import Schema, Field

schema = Schema({
    "invoice_number": Field(type="string"),
    "vendor": Field(type="string", description="Company name"),
    "total": Field(type="number"),
    "line_items": Field(
        type="array",
        items={
            "description": Field(type="string"),
            "quantity": Field(type="integer"),
            "amount": Field(type="number")
        }
    )
})

result = client.parse("invoice.pdf", schema=schema)

print(result.data["invoice_number"])  # "INV-2024-001"
print(result.data["total"])  # 1250.00
```

## Working with Results

### ParseResult Object

```python
result = client.parse("document.pdf")

# Document metadata
print(result.id)          # "doc_abc123"
print(result.status)      # "completed"
print(result.pages)       # 3
print(result.created_at)  # datetime object

# Extracted data
print(result.data)        # Full structured data dict
print(result.raw_text)    # Raw text content
print(result.tables)      # List of extracted tables
print(result.fields)      # Extracted fields with confidence scores
```

### Accessing Fields

```python
# Get a field value
vendor = result.get_field("vendor_name")
print(vendor.value)       # "Acme Corp"
print(vendor.confidence)  # 0.98

# Get with default
amount = result.get_field("amount", default=0)
```

### Working with Tables

```python
for table in result.tables:
    print(f"Table on page {table.page}")
    print(f"Headers: {table.headers}")
    
    for row in table.rows:
        print(row)
    
    # Convert to pandas DataFrame
    df = table.to_dataframe()
```

## Async Processing

For large documents, use async processing:

```python
# Start async processing
job = client.parse_async(
    "large-document.pdf",
    webhook_url="https://your-server.com/webhooks"
)

print(job.id)  # "doc_xyz789"
print(job.status)  # "pending"

# Poll for completion
result = job.wait(timeout=300)  # Wait up to 5 minutes

# Or check status manually
status = client.get_document(job.id)
```

## Batch Processing

Process multiple documents at once:

```python
# Submit batch
batch = client.parse_batch([
    "invoice1.pdf",
    "invoice2.pdf",
    "invoice3.pdf"
])

print(batch.id)  # "batch_abc123"
print(batch.total)  # 3

# Wait for all to complete
results = batch.wait()

for result in results:
    print(f"{result.id}: {result.data}")
```

## Document Management

### List Documents

```python
# List recent documents
documents = client.list_documents(limit=10)

for doc in documents:
    print(f"{doc.id}: {doc.status}")

# With filters
from datetime import datetime, timedelta

documents = client.list_documents(
    status="completed",
    from_date=datetime.now() - timedelta(days=7),
    limit=50
)
```

### Get Document

```python
doc = client.get_document("doc_abc123")
print(doc.data)
```

### Delete Document

```python
client.delete_document("doc_abc123")
```

## Error Handling

```python
from struktr import (
    Struktr,
    StruktrError,
    AuthenticationError,
    RateLimitError,
    ProcessingError
)

client = Struktr()

try:
    result = client.parse("document.pdf")
except AuthenticationError:
    print("Invalid API key")
except RateLimitError as e:
    print(f"Rate limited. Retry after {e.retry_after} seconds")
except ProcessingError as e:
    print(f"Processing failed: {e.message}")
except StruktrError as e:
    print(f"API error: {e.code} - {e.message}")
```

## Logging

Enable debug logging:

```python
import logging

logging.basicConfig(level=logging.DEBUG)

# Or just for Struktr
logging.getLogger("struktr").setLevel(logging.DEBUG)
```

## Examples

### Invoice Processing

```python
from struktr import Struktr, Schema, Field

client = Struktr()

# Define invoice schema
invoice_schema = Schema({
    "vendor": Field(type="string"),
    "invoice_number": Field(type="string"),
    "date": Field(type="string", format="date"),
    "due_date": Field(type="string", format="date"),
    "subtotal": Field(type="number"),
    "tax": Field(type="number"),
    "total": Field(type="number"),
    "line_items": Field(
        type="array",
        items={
            "description": Field(type="string"),
            "quantity": Field(type="integer"),
            "unit_price": Field(type="number"),
            "amount": Field(type="number")
        }
    )
})

result = client.parse("invoice.pdf", schema=invoice_schema)

# Process the extracted data
print(f"Invoice from: {result.data['vendor']}")
print(f"Total due: ${result.data['total']:.2f}")
print(f"Due date: {result.data['due_date']}")
```

### Contract Analysis

```python
contract_schema = Schema({
    "parties": Field(
        type="array",
        items={"name": Field(type="string"), "role": Field(type="string")}
    ),
    "effective_date": Field(type="string", format="date"),
    "termination_date": Field(type="string", format="date"),
    "governing_law": Field(type="string"),
    "key_terms": Field(type="array", items={"type": "string"})
})

result = client.parse("contract.pdf", schema=contract_schema)

for party in result.data["parties"]:
    print(f"{party['role']}: {party['name']}")
```

## API Reference

See the full [API Reference](../api-reference.md) for all available methods and options.
