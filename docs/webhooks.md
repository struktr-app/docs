# Webhooks

Webhooks allow you to receive real-time notifications when documents are processed asynchronously.

## Overview

When using async processing (`/parse/async` or batch processing), you can specify a webhook URL to receive notifications when processing is complete.

## Setting Up Webhooks

### In API Requests

Include the `webhook_url` parameter in your async requests:

```bash
curl -X POST https://api.struktr.app/v1/parse/async \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://example.com/document.pdf",
    "webhook_url": "https://your-server.com/webhooks/struktr"
  }'
```

### Using SDKs

**Python**
```python
job = client.parse_async(
    "document.pdf",
    webhook_url="https://your-server.com/webhooks/struktr"
)
```

**Node.js**
```javascript
const job = await client.parseAsync('document.pdf', {
  webhookUrl: 'https://your-server.com/webhooks/struktr',
});
```

**Go**
```go
job, err := client.ParseAsync(ctx, "document.pdf", &struktr.ParseAsyncOptions{
    WebhookURL: "https://your-server.com/webhooks/struktr",
})
```

## Webhook Events

| Event | Description |
|-------|-------------|
| `document.completed` | Document processing completed successfully |
| `document.failed` | Document processing failed |
| `batch.completed` | All documents in a batch have been processed |
| `batch.progress` | Progress update for batch processing |

## Webhook Payload

### document.completed

```json
{
  "event": "document.completed",
  "timestamp": "2024-01-15T10:30:02Z",
  "data": {
    "id": "doc_abc123",
    "status": "completed",
    "pages": 3,
    "created_at": "2024-01-15T10:30:00Z",
    "completed_at": "2024-01-15T10:30:02Z",
    "data": {
      "document_type": "invoice",
      "confidence": 0.95,
      "fields": {
        "vendor_name": {
          "value": "Acme Corporation",
          "confidence": 0.98
        },
        "total": {
          "value": 1547.50,
          "confidence": 0.96
        }
      },
      "tables": [...],
      "raw_text": "..."
    }
  }
}
```

### document.failed

```json
{
  "event": "document.failed",
  "timestamp": "2024-01-15T10:30:05Z",
  "data": {
    "id": "doc_xyz789",
    "status": "failed",
    "created_at": "2024-01-15T10:30:00Z",
    "error": {
      "code": "processing_failed",
      "message": "Unable to extract text from the document",
      "details": {
        "reason": "corrupt_file"
      }
    }
  }
}
```

### batch.completed

```json
{
  "event": "batch.completed",
  "timestamp": "2024-01-15T10:35:00Z",
  "data": {
    "batch_id": "batch_abc123",
    "status": "completed",
    "total": 5,
    "succeeded": 4,
    "failed": 1,
    "documents": [
      {"id": "doc_1", "status": "completed"},
      {"id": "doc_2", "status": "completed"},
      {"id": "doc_3", "status": "completed"},
      {"id": "doc_4", "status": "completed"},
      {"id": "doc_5", "status": "failed"}
    ]
  }
}
```

### batch.progress

```json
{
  "event": "batch.progress",
  "timestamp": "2024-01-15T10:32:00Z",
  "data": {
    "batch_id": "batch_abc123",
    "total": 5,
    "completed": 2,
    "pending": 3
  }
}
```

## Webhook Security

### Signature Verification

All webhook requests include a signature header for verification:

```
X-Struktr-Signature: sha256=abc123def456...
```

The signature is an HMAC-SHA256 hash of the request body using your webhook secret.

### Getting Your Webhook Secret

You can find your webhook secret in your dashboard at [struktr.app/dashboard/webhooks](https://struktr.app/dashboard/webhooks).

### Verification Examples

**Python**
```python
import hmac
import hashlib

def verify_webhook(payload: str, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)

# In your webhook handler
@app.route('/webhooks/struktr', methods=['POST'])
def webhook_handler():
    signature = request.headers.get('X-Struktr-Signature')
    payload = request.get_data(as_text=True)
    
    if not verify_webhook(payload, signature, WEBHOOK_SECRET):
        return 'Invalid signature', 401
    
    data = request.json
    # Process webhook...
    return 'OK', 200
```

**Node.js**
```javascript
const crypto = require('crypto');

function verifyWebhook(payload, signature, secret) {
  const expected = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(`sha256=${expected}`),
    Buffer.from(signature)
  );
}

// Express.js example
app.post('/webhooks/struktr', express.raw({ type: 'application/json' }), (req, res) => {
  const signature = req.headers['x-struktr-signature'];
  
  if (!verifyWebhook(req.body.toString(), signature, WEBHOOK_SECRET)) {
    return res.status(401).send('Invalid signature');
  }
  
  const data = JSON.parse(req.body);
  // Process webhook...
  res.status(200).send('OK');
});
```

**Go**
```go
func verifyWebhook(payload []byte, signature, secret string) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(payload)
    expected := "sha256=" + hex.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(expected), []byte(signature))
}
```

## Webhook Headers

Each webhook request includes these headers:

| Header | Description |
|--------|-------------|
| `Content-Type` | `application/json` |
| `X-Struktr-Signature` | HMAC-SHA256 signature |
| `X-Struktr-Event` | Event type |
| `X-Struktr-Delivery` | Unique delivery ID |
| `X-Struktr-Timestamp` | Request timestamp |

## Retry Policy

If your endpoint returns a non-2xx status code or times out, we will retry the webhook:

| Attempt | Delay |
|---------|-------|
| 1st retry | 1 minute |
| 2nd retry | 5 minutes |
| 3rd retry | 30 minutes |
| 4th retry | 2 hours |
| 5th retry | 12 hours |

After 5 failed attempts, the webhook will be marked as failed.

## Webhook Requirements

Your webhook endpoint must:

- Accept `POST` requests
- Return a `2xx` status code within 30 seconds
- Accept `application/json` content type
- Be publicly accessible via HTTPS

## Testing Webhooks

### Local Development

Use a tunneling service like [ngrok](https://ngrok.com) to test webhooks locally:

```bash
ngrok http 3000
# Use the generated URL as your webhook URL
```

### Webhook Tester

Use our webhook tester in the dashboard to send test events:

1. Go to [struktr.app/dashboard/webhooks](https://struktr.app/dashboard/webhooks)
2. Enter your webhook URL
3. Select an event type
4. Click "Send Test Event"

### CLI Tool

Test webhooks using our CLI:

```bash
struktr webhooks test https://your-server.com/webhooks/struktr --event document.completed
```

## Best Practices

1. **Respond quickly**: Return a 200 status immediately, then process asynchronously
2. **Verify signatures**: Always verify the webhook signature
3. **Handle duplicates**: Use the delivery ID to detect duplicate deliveries
4. **Use queues**: Queue webhook payloads for processing to handle high volumes
5. **Log everything**: Log webhook payloads for debugging

## Troubleshooting

### Common Issues

**Webhook not received**
- Verify your endpoint is publicly accessible
- Check firewall rules
- Ensure HTTPS is configured correctly

**Signature verification failing**
- Make sure you're using the raw request body
- Check your webhook secret is correct
- Verify the signature header name

**Timeouts**
- Return 200 immediately before processing
- Use background jobs for heavy processing

See [Error Handling](./errors.md) for more troubleshooting tips.
