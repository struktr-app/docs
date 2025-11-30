# Invoice Processing Example

This example demonstrates how to extract structured data from invoices using Struktr.

## Overview

Invoice processing is one of the most common document automation tasks. Struktr can automatically extract:

- Vendor information
- Invoice numbers and dates
- Line items with descriptions, quantities, and amounts
- Subtotals, taxes, and totals
- Payment terms and due dates

## Basic Invoice Extraction

### Python

```python
from struktr import Struktr

client = Struktr()

# Parse an invoice
result = client.parse("invoice.pdf")

# The API automatically detects invoice structure
print(result.data)
# {
#   "document_type": "invoice",
#   "vendor_name": "Acme Corporation",
#   "vendor_address": "123 Business St, New York, NY 10001",
#   "invoice_number": "INV-2024-0042",
#   "invoice_date": "2024-01-10",
#   "due_date": "2024-02-10",
#   "subtotal": 1250.00,
#   "tax": 112.50,
#   "total": 1362.50,
#   "line_items": [
#     {
#       "description": "Consulting Services",
#       "quantity": 10,
#       "unit_price": 125.00,
#       "amount": 1250.00
#     }
#   ]
# }
```

### Node.js

```javascript
import { Struktr } from '@struktr/sdk';

const client = new Struktr();

const result = await client.parse('invoice.pdf');

console.log(result.data);
// Same structure as Python example
```

## Custom Invoice Schema

For more control over extraction, define a custom schema:

```python
from struktr import Struktr, Schema, Field

client = Struktr()

# Define exactly what fields to extract
invoice_schema = Schema({
    "vendor": Field(
        type="object",
        properties={
            "name": Field(type="string"),
            "address": Field(type="string"),
            "phone": Field(type="string"),
            "email": Field(type="string"),
            "tax_id": Field(type="string", description="Tax ID or EIN")
        }
    ),
    "invoice_details": Field(
        type="object",
        properties={
            "number": Field(type="string"),
            "date": Field(type="string", format="date"),
            "due_date": Field(type="string", format="date"),
            "po_number": Field(type="string", description="Purchase order number")
        }
    ),
    "customer": Field(
        type="object",
        properties={
            "name": Field(type="string"),
            "address": Field(type="string")
        }
    ),
    "line_items": Field(
        type="array",
        items={
            "description": Field(type="string"),
            "quantity": Field(type="number"),
            "unit": Field(type="string", description="Unit of measure"),
            "unit_price": Field(type="number"),
            "amount": Field(type="number"),
            "tax_rate": Field(type="number")
        }
    ),
    "totals": Field(
        type="object",
        properties={
            "subtotal": Field(type="number"),
            "discount": Field(type="number"),
            "tax": Field(type="number"),
            "shipping": Field(type="number"),
            "total": Field(type="number")
        }
    ),
    "payment_terms": Field(type="string"),
    "notes": Field(type="string")
})

result = client.parse("invoice.pdf", schema=invoice_schema)
```

## Batch Invoice Processing

Process multiple invoices efficiently:

```python
from struktr import Struktr
import os

client = Struktr()

# Get all PDF files in a directory
invoice_dir = "./invoices"
invoice_files = [
    os.path.join(invoice_dir, f) 
    for f in os.listdir(invoice_dir) 
    if f.endswith(".pdf")
]

# Process in batch
batch = client.parse_batch(invoice_files)

# Wait for all to complete
results = batch.wait()

# Process results
for result in results:
    if result.status == "completed":
        print(f"Invoice {result.data['invoice_number']}: ${result.data['total']}")
    else:
        print(f"Failed to process: {result.id}")
```

## Async Processing with Webhooks

For large volumes, use async processing:

```python
from struktr import Struktr

client = Struktr()

# Start async processing
job = client.parse_async(
    "large-invoice.pdf",
    webhook_url="https://your-server.com/webhooks/struktr"
)

print(f"Processing started: {job.id}")
```

Webhook handler:

```python
from flask import Flask, request
import hmac
import hashlib

app = Flask(__name__)
WEBHOOK_SECRET = "your-webhook-secret"

@app.route('/webhooks/struktr', methods=['POST'])
def handle_webhook():
    # Verify signature
    signature = request.headers.get('X-Struktr-Signature')
    payload = request.get_data(as_text=True)
    
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()
    
    if not hmac.compare_digest(f"sha256={expected}", signature):
        return 'Invalid signature', 401
    
    data = request.json
    
    if data['event'] == 'document.completed':
        invoice = data['data']['data']
        
        # Save to database
        save_invoice({
            'invoice_number': invoice['invoice_number'],
            'vendor': invoice['vendor_name'],
            'total': invoice['total'],
            'due_date': invoice['due_date']
        })
        
        print(f"Invoice processed: {invoice['invoice_number']}")
    
    return 'OK', 200
```

## Working with Tables

Invoices often contain tables of line items:

```python
result = client.parse("invoice.pdf", options={"extract_tables": True})

# Access extracted tables
for table in result.tables:
    print(f"Table on page {table.page}")
    print(f"Headers: {table.headers}")
    
    for row in table.rows:
        print(row)
    
    # Convert to pandas DataFrame
    df = table.to_dataframe()
    print(df)
```

## Confidence Scores

Access confidence scores for extracted fields:

```python
result = client.parse("invoice.pdf")

# Check confidence for specific fields
vendor_field = result.get_field("vendor_name")
print(f"Vendor: {vendor_field.value}")
print(f"Confidence: {vendor_field.confidence}")  # 0.0 to 1.0

# Filter fields by confidence threshold
high_confidence_fields = {
    key: field
    for key, field in result.fields.items()
    if field.confidence >= 0.9
}
```

## Complete Example: Invoice Processing Pipeline

```python
from struktr import Struktr, Schema, Field
import pandas as pd
from datetime import datetime
import json

class InvoiceProcessor:
    def __init__(self):
        self.client = Struktr()
        self.schema = self._create_schema()
    
    def _create_schema(self):
        return Schema({
            "vendor_name": Field(type="string"),
            "invoice_number": Field(type="string"),
            "invoice_date": Field(type="string", format="date"),
            "due_date": Field(type="string", format="date"),
            "subtotal": Field(type="number"),
            "tax": Field(type="number"),
            "total": Field(type="number"),
            "line_items": Field(
                type="array",
                items={
                    "description": Field(type="string"),
                    "quantity": Field(type="number"),
                    "unit_price": Field(type="number"),
                    "amount": Field(type="number")
                }
            )
        })
    
    def process_invoice(self, file_path):
        """Process a single invoice and return structured data."""
        result = self.client.parse(file_path, schema=self.schema)
        
        if result.status != "completed":
            raise Exception(f"Failed to process invoice: {result.status}")
        
        return {
            "id": result.id,
            "file": file_path,
            "data": result.data,
            "confidence": result.data.get("confidence", 0),
            "processed_at": datetime.now().isoformat()
        }
    
    def process_batch(self, file_paths, min_confidence=0.8):
        """Process multiple invoices and filter by confidence."""
        results = []
        errors = []
        
        batch = self.client.parse_batch(file_paths, schema=self.schema)
        batch_results = batch.wait()
        
        for result in batch_results:
            if result.status == "completed":
                invoice = {
                    "id": result.id,
                    "data": result.data,
                    "confidence": result.data.get("confidence", 0)
                }
                
                if invoice["confidence"] >= min_confidence:
                    results.append(invoice)
                else:
                    errors.append({
                        "id": result.id,
                        "reason": "low_confidence",
                        "confidence": invoice["confidence"]
                    })
            else:
                errors.append({
                    "id": result.id,
                    "reason": "processing_failed"
                })
        
        return results, errors
    
    def to_dataframe(self, invoices):
        """Convert processed invoices to a pandas DataFrame."""
        rows = []
        
        for invoice in invoices:
            data = invoice["data"]
            rows.append({
                "invoice_id": invoice["id"],
                "vendor": data.get("vendor_name"),
                "invoice_number": data.get("invoice_number"),
                "invoice_date": data.get("invoice_date"),
                "due_date": data.get("due_date"),
                "subtotal": data.get("subtotal"),
                "tax": data.get("tax"),
                "total": data.get("total"),
                "line_item_count": len(data.get("line_items", []))
            })
        
        return pd.DataFrame(rows)
    
    def export_to_json(self, invoices, output_path):
        """Export processed invoices to JSON."""
        with open(output_path, "w") as f:
            json.dump(invoices, f, indent=2)


# Usage
processor = InvoiceProcessor()

# Process a single invoice
invoice = processor.process_invoice("invoice.pdf")
print(f"Processed: {invoice['data']['invoice_number']}")
print(f"Total: ${invoice['data']['total']:.2f}")

# Process multiple invoices
invoice_files = ["invoice1.pdf", "invoice2.pdf", "invoice3.pdf"]
results, errors = processor.process_batch(invoice_files)

print(f"Successfully processed: {len(results)}")
print(f"Errors: {len(errors)}")

# Convert to DataFrame for analysis
df = processor.to_dataframe(results)
print(df)

# Export to JSON
processor.export_to_json(results, "processed_invoices.json")
```

## Next Steps

- [Contract Analysis](./contract-analysis.md) - Extract data from legal documents
- [Custom Schemas](./custom-schemas.md) - Learn more about schema customization
- [Batch Processing](./batch-processing.md) - Process documents at scale
