# Error Handling

This guide covers common errors and how to handle them when using the Struktr API.

## Error Response Format

All API errors follow a consistent format:

```json
{
  "error": {
    "code": "error_code",
    "message": "Human-readable error message",
    "details": {
      "additional": "context"
    }
  }
}
```

## HTTP Status Codes

| Status | Meaning |
|--------|---------|
| 200 | Success |
| 400 | Bad Request - Invalid parameters |
| 401 | Unauthorized - Invalid or missing API key |
| 403 | Forbidden - Not authorized for this resource |
| 404 | Not Found - Resource doesn't exist |
| 429 | Too Many Requests - Rate limit exceeded |
| 500 | Internal Server Error - Something went wrong |
| 503 | Service Unavailable - Temporarily unavailable |

## Error Codes

### Authentication Errors

#### `authentication_failed`

**HTTP Status:** 401

The API key is invalid or missing.

```json
{
  "error": {
    "code": "authentication_failed",
    "message": "Invalid API key provided"
  }
}
```

**Solutions:**
- Verify your API key is correct
- Check that the `Authorization` header is formatted correctly: `Bearer YOUR_API_KEY`
- Ensure your API key is active in your dashboard

### Request Errors

#### `invalid_request`

**HTTP Status:** 400

The request was malformed or missing required parameters.

```json
{
  "error": {
    "code": "invalid_request",
    "message": "Missing required parameter: file or url",
    "details": {
      "required": ["file", "url"],
      "provided": []
    }
  }
}
```

**Solutions:**
- Check that all required parameters are included
- Verify the request body is valid JSON
- Ensure file uploads are properly formatted

#### `invalid_file_format`

**HTTP Status:** 400

The uploaded file is not a supported format.

```json
{
  "error": {
    "code": "invalid_file_format",
    "message": "The file format is not supported",
    "details": {
      "provided_format": "docx",
      "supported_formats": ["pdf"]
    }
  }
}
```

**Solutions:**
- Convert your document to PDF before uploading
- Use a PDF generation tool like wkhtmltopdf or Puppeteer

#### `file_too_large`

**HTTP Status:** 400

The uploaded file exceeds the maximum size limit.

```json
{
  "error": {
    "code": "file_too_large",
    "message": "File size exceeds the maximum limit",
    "details": {
      "max_size_mb": 50,
      "provided_size_mb": 75
    }
  }
}
```

**Solutions:**
- Compress the PDF file
- Split the document into smaller parts
- Contact support for larger file limits

#### `invalid_url`

**HTTP Status:** 400

The provided URL is invalid or inaccessible.

```json
{
  "error": {
    "code": "invalid_url",
    "message": "Unable to access the provided URL",
    "details": {
      "url": "https://example.com/document.pdf",
      "reason": "connection_timeout"
    }
  }
}
```

**Solutions:**
- Verify the URL is publicly accessible
- Check that the URL returns a PDF file
- Ensure there are no authentication requirements

### Authorization Errors

#### `forbidden`

**HTTP Status:** 403

You don't have permission to access this resource.

```json
{
  "error": {
    "code": "forbidden",
    "message": "You do not have permission to access this document",
    "details": {
      "resource_id": "doc_xyz789"
    }
  }
}
```

**Solutions:**
- Verify you're using the correct API key
- Check that the document belongs to your account

### Rate Limiting

#### `rate_limit_exceeded`

**HTTP Status:** 429

You've exceeded the rate limit for your plan.

```json
{
  "error": {
    "code": "rate_limit_exceeded",
    "message": "Rate limit exceeded. Please retry after 60 seconds",
    "details": {
      "limit": 100,
      "remaining": 0,
      "reset_at": "2024-01-15T10:31:00Z"
    }
  }
}
```

**Solutions:**
- Wait for the rate limit to reset
- Implement exponential backoff in your code
- Upgrade to a higher plan for increased limits

**Rate Limit Headers:**
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1705318260
```

### Resource Errors

#### `not_found`

**HTTP Status:** 404

The requested resource doesn't exist.

```json
{
  "error": {
    "code": "not_found",
    "message": "Document not found",
    "details": {
      "resource_type": "document",
      "resource_id": "doc_abc123"
    }
  }
}
```

**Solutions:**
- Verify the document ID is correct
- Check if the document was deleted

### Processing Errors

#### `processing_failed`

**HTTP Status:** 500

Document processing failed.

```json
{
  "error": {
    "code": "processing_failed",
    "message": "Failed to process the document",
    "details": {
      "reason": "corrupt_file",
      "document_id": "doc_abc123"
    }
  }
}
```

**Possible reasons:**
- `corrupt_file` - The PDF file is corrupted
- `password_protected` - The PDF is password protected
- `no_text_content` - No extractable content found
- `unsupported_encoding` - Unsupported text encoding
- `internal_error` - Internal processing error

**Solutions:**
- Verify the PDF file is valid and not corrupted
- Remove password protection from the PDF
- Enable OCR for scanned documents
- Contact support for persistent errors

#### `timeout`

**HTTP Status:** 500

Document processing timed out.

```json
{
  "error": {
    "code": "timeout",
    "message": "Document processing timed out",
    "details": {
      "timeout_seconds": 60,
      "document_id": "doc_abc123"
    }
  }
}
```

**Solutions:**
- Use async processing for large documents
- Split the document into smaller parts

### Service Errors

#### `service_unavailable`

**HTTP Status:** 503

The service is temporarily unavailable.

```json
{
  "error": {
    "code": "service_unavailable",
    "message": "Service is temporarily unavailable. Please try again later.",
    "details": {
      "retry_after": 300
    }
  }
}
```

**Solutions:**
- Wait and retry the request
- Check [status.struktr.app](https://status.struktr.app) for service status

## Handling Errors in SDKs

### Python

```python
from struktr import (
    Struktr,
    StruktrError,
    AuthenticationError,
    RateLimitError,
    ProcessingError,
    InvalidRequestError
)

client = Struktr()

try:
    result = client.parse("document.pdf")
except AuthenticationError:
    # Invalid API key
    print("Please check your API key")
except RateLimitError as e:
    # Rate limited - wait and retry
    print(f"Rate limited. Retry after {e.retry_after} seconds")
    time.sleep(e.retry_after)
    # Retry the request
except InvalidRequestError as e:
    # Bad request parameters
    print(f"Invalid request: {e.message}")
    print(f"Details: {e.details}")
except ProcessingError as e:
    # Document processing failed
    print(f"Processing failed: {e.message}")
    print(f"Reason: {e.details.get('reason')}")
except StruktrError as e:
    # Other API errors
    print(f"API error: {e.code} - {e.message}")
```

### Node.js

```javascript
import {
  Struktr,
  StruktrError,
  AuthenticationError,
  RateLimitError,
  ProcessingError,
  InvalidRequestError,
} from '@struktr/sdk';

const client = new Struktr();

try {
  const result = await client.parse('document.pdf');
} catch (error) {
  if (error instanceof AuthenticationError) {
    console.error('Please check your API key');
  } else if (error instanceof RateLimitError) {
    console.error(`Rate limited. Retry after ${error.retryAfter} seconds`);
    await sleep(error.retryAfter * 1000);
    // Retry the request
  } else if (error instanceof InvalidRequestError) {
    console.error(`Invalid request: ${error.message}`);
    console.error('Details:', error.details);
  } else if (error instanceof ProcessingError) {
    console.error(`Processing failed: ${error.message}`);
    console.error('Reason:', error.details?.reason);
  } else if (error instanceof StruktrError) {
    console.error(`API error: ${error.code} - ${error.message}`);
  } else {
    throw error;
  }
}
```

### Go

```go
import (
    "errors"
    "time"
    "github.com/struktr-app/struktr-go"
)

result, err := client.Parse(ctx, "document.pdf")
if err != nil {
    var authErr *struktr.AuthenticationError
    var rateErr *struktr.RateLimitError
    var procErr *struktr.ProcessingError
    var reqErr *struktr.InvalidRequestError
    var apiErr *struktr.APIError

    switch {
    case errors.As(err, &authErr):
        log.Println("Please check your API key")
    case errors.As(err, &rateErr):
        log.Printf("Rate limited. Retry after %v\n", rateErr.RetryAfter)
        time.Sleep(rateErr.RetryAfter)
        // Retry the request
    case errors.As(err, &reqErr):
        log.Printf("Invalid request: %s\n", reqErr.Message)
        log.Printf("Details: %+v\n", reqErr.Details)
    case errors.As(err, &procErr):
        log.Printf("Processing failed: %s\n", procErr.Message)
    case errors.As(err, &apiErr):
        log.Printf("API error: %s - %s\n", apiErr.Code, apiErr.Message)
    default:
        log.Fatal(err)
    }
}
```

## Retry Strategies

### Exponential Backoff

For transient errors, implement exponential backoff:

```python
import time
import random

def parse_with_retry(client, file_path, max_retries=5):
    for attempt in range(max_retries):
        try:
            return client.parse(file_path)
        except RateLimitError as e:
            if attempt == max_retries - 1:
                raise
            wait_time = e.retry_after or (2 ** attempt + random.random())
            time.sleep(wait_time)
        except StruktrError as e:
            if e.code in ['service_unavailable', 'timeout']:
                if attempt == max_retries - 1:
                    raise
                wait_time = 2 ** attempt + random.random()
                time.sleep(wait_time)
            else:
                raise
```

### Which Errors to Retry

| Error Code | Retry? |
|------------|--------|
| `rate_limit_exceeded` | Yes, after waiting |
| `service_unavailable` | Yes, with backoff |
| `timeout` | Yes, consider async |
| `processing_failed` | Maybe, depends on reason |
| `authentication_failed` | No |
| `invalid_request` | No |
| `forbidden` | No |
| `not_found` | No |

## Debugging Tips

1. **Enable debug logging** to see request/response details
2. **Check response headers** for rate limit and request ID info
3. **Use the request ID** when contacting support
4. **Validate your input** before making API calls
5. **Test with sample documents** before processing production data

## Getting Help

If you're experiencing persistent errors:

1. Check [status.struktr.app](https://status.struktr.app) for service issues
2. Search our [community forum](https://community.struktr.app)
3. Contact support at support@struktr.app with:
   - Request ID (from response headers)
   - Error code and message
   - Document details (without sensitive data)
