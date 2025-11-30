# API Reference

Base URL: `https://api.struktr.app/v1`

All API requests require authentication via an API key passed in the `Authorization` header.

## Authentication

Include your API key in all requests:

```bash
curl -H "Authorization: Bearer YOUR_API_KEY" https://api.struktr.app/v1/parse
```

## Rate Limits

| Plan | Requests/Minute | Documents/Month |
|------|-----------------|-----------------|
| Free | 10 | 100 |
| Pro | 100 | 10,000 |
| Enterprise | Unlimited | Unlimited |

Rate limit headers are included in all responses:
- `X-RateLimit-Limit`: Maximum requests per minute
- `X-RateLimit-Remaining`: Remaining requests in the current window
- `X-RateLimit-Reset`: Unix timestamp when the limit resets

---

## Endpoints

### Parse Document

Convert a PDF document to structured JSON.

```
POST /parse
```

#### Request

**Headers**
| Header | Required | Description |
|--------|----------|-------------|
| `Authorization` | Yes | Bearer token with your API key |
| `Content-Type` | Yes | `multipart/form-data` or `application/json` |

**Body (multipart/form-data)**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `file` | File | Yes* | The PDF file to parse |
| `url` | String | Yes* | URL of the PDF to parse |
| `options` | JSON | No | Parsing options (see below) |

*Either `file` or `url` is required.

**Parsing Options**
```json
{
  "extract_tables": true,
  "extract_images": false,
  "ocr_enabled": true,
  "language": "en",
  "output_format": "structured",
  "schema": null
}
```

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `extract_tables` | Boolean | `true` | Extract tables from the document |
| `extract_images` | Boolean | `false` | Extract and include images as base64 |
| `ocr_enabled` | Boolean | `true` | Use OCR for scanned documents |
| `language` | String | `"en"` | Primary language of the document |
| `output_format` | String | `"structured"` | Output format: `structured`, `raw`, `markdown` |
| `schema` | Object | `null` | Custom schema for extraction |

#### Response

```json
{
  "id": "doc_abc123",
  "status": "completed",
  "created_at": "2024-01-15T10:30:00Z",
  "completed_at": "2024-01-15T10:30:02Z",
  "pages": 3,
  "data": {
    "document_type": "invoice",
    "confidence": 0.95,
    "fields": {
      "vendor_name": {
        "value": "Acme Corporation",
        "confidence": 0.98,
        "bounding_box": [100, 50, 300, 75]
      },
      "invoice_number": {
        "value": "INV-2024-0042",
        "confidence": 0.99,
        "bounding_box": [400, 50, 550, 75]
      },
      "date": {
        "value": "2024-01-10",
        "confidence": 0.97,
        "bounding_box": [400, 80, 550, 105]
      },
      "total": {
        "value": 1547.50,
        "confidence": 0.96,
        "bounding_box": [450, 500, 550, 525]
      }
    },
    "tables": [
      {
        "page": 1,
        "headers": ["Description", "Quantity", "Unit Price", "Amount"],
        "rows": [
          ["Widget A", "10", "$25.00", "$250.00"],
          ["Widget B", "5", "$50.00", "$250.00"],
          ["Service Fee", "1", "$1,047.50", "$1,047.50"]
        ]
      }
    ],
    "raw_text": "..."
  }
}
```

#### Example

```bash
curl -X POST https://api.struktr.app/v1/parse \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "file=@invoice.pdf" \
  -F 'options={"extract_tables": true}'
```

---

### Get Document

Retrieve a previously parsed document.

```
GET /documents/{document_id}
```

#### Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `document_id` | String | The unique document ID |

#### Response

Returns the same structure as the parse endpoint.

#### Example

```bash
curl https://api.struktr.app/v1/documents/doc_abc123 \
  -H "Authorization: Bearer YOUR_API_KEY"
```

---

### List Documents

List all parsed documents.

```
GET /documents
```

#### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `limit` | Integer | 20 | Number of documents to return (max 100) |
| `offset` | Integer | 0 | Pagination offset |
| `status` | String | - | Filter by status: `pending`, `completed`, `failed` |
| `from` | String | - | Filter documents created after this ISO 8601 date |
| `to` | String | - | Filter documents created before this ISO 8601 date |

#### Response

```json
{
  "data": [
    {
      "id": "doc_abc123",
      "status": "completed",
      "created_at": "2024-01-15T10:30:00Z",
      "pages": 3,
      "document_type": "invoice"
    }
  ],
  "pagination": {
    "total": 150,
    "limit": 20,
    "offset": 0,
    "has_more": true
  }
}
```

---

### Delete Document

Delete a parsed document and its data.

```
DELETE /documents/{document_id}
```

#### Response

```json
{
  "deleted": true,
  "id": "doc_abc123"
}
```

---

### Parse with Custom Schema

Extract data according to a custom schema.

```
POST /parse
```

Use the `schema` option to define exactly what fields to extract:

```json
{
  "schema": {
    "type": "object",
    "properties": {
      "company_name": {
        "type": "string",
        "description": "The name of the company issuing the document"
      },
      "total_amount": {
        "type": "number",
        "description": "The total monetary amount"
      },
      "line_items": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "description": { "type": "string" },
            "quantity": { "type": "integer" },
            "price": { "type": "number" }
          }
        }
      }
    }
  }
}
```

#### Example

```bash
curl -X POST https://api.struktr.app/v1/parse \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/invoice.pdf",
    "options": {
      "schema": {
        "type": "object",
        "properties": {
          "invoice_number": {"type": "string"},
          "total": {"type": "number"},
          "due_date": {"type": "string", "format": "date"}
        }
      }
    }
  }'
```

---

### Async Processing

For large documents, use async processing with webhooks.

```
POST /parse/async
```

#### Request

Same as `/parse`, with an additional `webhook_url` parameter:

```json
{
  "url": "https://example.com/large-document.pdf",
  "webhook_url": "https://your-server.com/webhooks/struktr",
  "options": {
    "extract_tables": true
  }
}
```

#### Response

```json
{
  "id": "doc_xyz789",
  "status": "pending",
  "created_at": "2024-01-15T10:30:00Z"
}
```

When processing is complete, a POST request will be sent to your webhook URL with the full document data.

---

### Batch Processing

Process multiple documents in a single request.

```
POST /parse/batch
```

#### Request

```json
{
  "documents": [
    {"url": "https://example.com/doc1.pdf"},
    {"url": "https://example.com/doc2.pdf"},
    {"url": "https://example.com/doc3.pdf"}
  ],
  "webhook_url": "https://your-server.com/webhooks/struktr",
  "options": {
    "extract_tables": true
  }
}
```

#### Response

```json
{
  "batch_id": "batch_abc123",
  "status": "processing",
  "total": 3,
  "documents": [
    {"id": "doc_1", "status": "pending"},
    {"id": "doc_2", "status": "pending"},
    {"id": "doc_3", "status": "pending"}
  ]
}
```

---

## Webhooks

Configure webhooks to receive notifications when documents are processed.

### Webhook Payload

```json
{
  "event": "document.completed",
  "timestamp": "2024-01-15T10:30:02Z",
  "data": {
    "id": "doc_abc123",
    "status": "completed",
    "data": { ... }
  }
}
```

### Webhook Events

| Event | Description |
|-------|-------------|
| `document.completed` | Document processing completed successfully |
| `document.failed` | Document processing failed |
| `batch.completed` | All documents in a batch have been processed |

### Webhook Security

Webhooks include a signature header for verification:

```
X-Struktr-Signature: sha256=abc123...
```

Verify the signature using your webhook secret:

```python
import hmac
import hashlib

def verify_webhook(payload, signature, secret):
    expected = hmac.new(
        secret.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)
```

---

## Error Responses

All errors follow a consistent format:

```json
{
  "error": {
    "code": "invalid_request",
    "message": "The file format is not supported",
    "details": {
      "supported_formats": ["pdf"]
    }
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `invalid_request` | 400 | The request was malformed |
| `authentication_failed` | 401 | Invalid or missing API key |
| `forbidden` | 403 | Not authorized for this resource |
| `not_found` | 404 | Resource not found |
| `rate_limit_exceeded` | 429 | Too many requests |
| `processing_failed` | 500 | Document processing failed |
| `service_unavailable` | 503 | Service temporarily unavailable |

See [Error Handling](./errors.md) for detailed troubleshooting.
