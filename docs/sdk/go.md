# Go SDK

The official Go SDK for Struktr API.

## Installation

```bash
go get github.com/struktr-app/struktr-go
```

**Requirements:** Go 1.19+

## Quick Start

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/struktr-app/struktr-go"
)

func main() {
    // Initialize the client
    client := struktr.NewClient("your-api-key")

    // Parse a PDF file
    result, err := client.Parse(context.Background(), "document.pdf")
    if err != nil {
        log.Fatal(err)
    }

    // Access the structured data
    fmt.Printf("%+v\n", result.Data)
}
```

## Configuration

### Client Options

```go
import "github.com/struktr-app/struktr-go"

client := struktr.NewClient(
    "your-api-key",
    struktr.WithBaseURL("https://api.struktr.app/v1"),
    struktr.WithTimeout(30*time.Second),
    struktr.WithMaxRetries(3),
)
```

### Environment Variables

You can also configure the client using environment variables:

```bash
export STRUKTR_API_KEY="your-api-key"
```

```go
import "github.com/struktr-app/struktr-go"

// API key is automatically read from environment
client := struktr.NewClientFromEnv()
```

## Parsing Documents

### From File Path

```go
result, err := client.Parse(ctx, "invoice.pdf")
```

### From Reader

```go
file, err := os.Open("invoice.pdf")
if err != nil {
    log.Fatal(err)
}
defer file.Close()

result, err := client.ParseReader(ctx, file, "invoice.pdf")
```

### From URL

```go
result, err := client.ParseURL(ctx, "https://example.com/invoice.pdf")
```

### From Bytes

```go
pdfBytes := downloadPDF()
result, err := client.ParseBytes(ctx, pdfBytes, "document.pdf")
```

## Parsing Options

```go
options := &struktr.ParseOptions{
    ExtractTables:  true,
    ExtractImages:  false,
    OCREnabled:     true,
    Language:       "en",
    OutputFormat:   struktr.OutputStructured,
}

result, err := client.Parse(ctx, "document.pdf", options)
```

### Available Options

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `ExtractTables` | bool | `true` | Extract tables from the document |
| `ExtractImages` | bool | `false` | Include images as base64 |
| `OCREnabled` | bool | `true` | Enable OCR for scanned documents |
| `Language` | string | `"en"` | Document language |
| `OutputFormat` | OutputFormat | `OutputStructured` | Output format |

## Custom Schema Extraction

Define exactly what data to extract:

```go
schema := &struktr.Schema{
    Type: "object",
    Properties: map[string]*struktr.SchemaProperty{
        "invoice_number": {Type: "string"},
        "vendor":         {Type: "string", Description: "Company name"},
        "total":          {Type: "number"},
        "line_items": {
            Type: "array",
            Items: &struktr.SchemaProperty{
                Type: "object",
                Properties: map[string]*struktr.SchemaProperty{
                    "description": {Type: "string"},
                    "quantity":    {Type: "integer"},
                    "amount":      {Type: "number"},
                },
            },
        },
    },
}

result, err := client.Parse(ctx, "invoice.pdf", &struktr.ParseOptions{
    Schema: schema,
})

// Access typed data
invoiceNumber := result.Data["invoice_number"].(string)
total := result.Data["total"].(float64)
```

## Working with Results

### ParseResult Struct

```go
result, err := client.Parse(ctx, "document.pdf")
if err != nil {
    log.Fatal(err)
}

// Document metadata
fmt.Println(result.ID)        // "doc_abc123"
fmt.Println(result.Status)    // "completed"
fmt.Println(result.Pages)     // 3
fmt.Println(result.CreatedAt) // time.Time

// Extracted data
fmt.Printf("%+v\n", result.Data)   // map[string]interface{}
fmt.Println(result.RawText)        // Raw text content
fmt.Printf("%+v\n", result.Tables) // []Table
fmt.Printf("%+v\n", result.Fields) // map[string]Field
```

### Accessing Fields

```go
// Get a field value
vendor, ok := result.GetField("vendor_name")
if ok {
    fmt.Println(vendor.Value)      // "Acme Corp"
    fmt.Println(vendor.Confidence) // 0.98
}

// Get with type assertion
if vendorStr, ok := result.GetString("vendor_name"); ok {
    fmt.Println(vendorStr)
}

if total, ok := result.GetFloat("total"); ok {
    fmt.Printf("Total: $%.2f\n", total)
}
```

### Working with Tables

```go
for _, table := range result.Tables {
    fmt.Printf("Table on page %d\n", table.Page)
    fmt.Printf("Headers: %v\n", table.Headers)

    for _, row := range table.Rows {
        fmt.Println(row)
    }

    // Convert to slice of maps
    data := table.ToMaps()
    for _, row := range data {
        fmt.Printf("%+v\n", row)
    }
}
```

## Async Processing

For large documents, use async processing:

```go
// Start async processing
job, err := client.ParseAsync(ctx, "large-document.pdf", &struktr.ParseAsyncOptions{
    WebhookURL: "https://your-server.com/webhooks",
})
if err != nil {
    log.Fatal(err)
}

fmt.Println(job.ID)     // "doc_xyz789"
fmt.Println(job.Status) // "pending"

// Poll for completion
result, err := job.Wait(ctx, 5*time.Minute)
if err != nil {
    log.Fatal(err)
}

// Or check status manually
doc, err := client.GetDocument(ctx, job.ID)
```

## Batch Processing

Process multiple documents at once:

```go
// Submit batch
batch, err := client.ParseBatch(ctx, []string{
    "invoice1.pdf",
    "invoice2.pdf",
    "invoice3.pdf",
})
if err != nil {
    log.Fatal(err)
}

fmt.Println(batch.ID)    // "batch_abc123"
fmt.Println(batch.Total) // 3

// Wait for all to complete
results, err := batch.Wait(ctx)
if err != nil {
    log.Fatal(err)
}

for _, result := range results {
    fmt.Printf("%s: %+v\n", result.ID, result.Data)
}
```

## Document Management

### List Documents

```go
// List recent documents
documents, err := client.ListDocuments(ctx, &struktr.ListOptions{
    Limit: 10,
})

for _, doc := range documents.Data {
    fmt.Printf("%s: %s\n", doc.ID, doc.Status)
}

// With filters
sevenDaysAgo := time.Now().AddDate(0, 0, -7)
filtered, err := client.ListDocuments(ctx, &struktr.ListOptions{
    Status: struktr.StatusCompleted,
    From:   sevenDaysAgo,
    Limit:  50,
})
```

### Get Document

```go
doc, err := client.GetDocument(ctx, "doc_abc123")
if err != nil {
    log.Fatal(err)
}
fmt.Printf("%+v\n", doc.Data)
```

### Delete Document

```go
err := client.DeleteDocument(ctx, "doc_abc123")
```

## Error Handling

```go
import (
    "errors"
    "github.com/struktr-app/struktr-go"
)

result, err := client.Parse(ctx, "document.pdf")
if err != nil {
    var authErr *struktr.AuthenticationError
    var rateErr *struktr.RateLimitError
    var procErr *struktr.ProcessingError
    var apiErr *struktr.APIError

    switch {
    case errors.As(err, &authErr):
        log.Println("Invalid API key")
    case errors.As(err, &rateErr):
        log.Printf("Rate limited. Retry after %v\n", rateErr.RetryAfter)
    case errors.As(err, &procErr):
        log.Printf("Processing failed: %s\n", procErr.Message)
    case errors.As(err, &apiErr):
        log.Printf("API error: %s - %s\n", apiErr.Code, apiErr.Message)
    default:
        log.Fatal(err)
    }
}
```

## Context Support

All methods accept a context for cancellation and timeouts:

```go
// With timeout
ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
defer cancel()

result, err := client.Parse(ctx, "document.pdf")

// With cancellation
ctx, cancel := context.WithCancel(context.Background())

go func() {
    time.Sleep(10 * time.Second)
    cancel() // Cancel after 10 seconds
}()

result, err := client.Parse(ctx, "large-document.pdf")
if errors.Is(err, context.Canceled) {
    log.Println("Request was cancelled")
}
```

## Examples

### Invoice Processing

```go
package main

import (
    "context"
    "fmt"
    "log"

    "github.com/struktr-app/struktr-go"
)

func main() {
    client := struktr.NewClientFromEnv()
    ctx := context.Background()

    // Define invoice schema
    schema := &struktr.Schema{
        Type: "object",
        Properties: map[string]*struktr.SchemaProperty{
            "vendor":         {Type: "string"},
            "invoice_number": {Type: "string"},
            "date":           {Type: "string", Format: "date"},
            "due_date":       {Type: "string", Format: "date"},
            "subtotal":       {Type: "number"},
            "tax":            {Type: "number"},
            "total":          {Type: "number"},
            "line_items": {
                Type: "array",
                Items: &struktr.SchemaProperty{
                    Type: "object",
                    Properties: map[string]*struktr.SchemaProperty{
                        "description": {Type: "string"},
                        "quantity":    {Type: "integer"},
                        "unit_price":  {Type: "number"},
                        "amount":      {Type: "number"},
                    },
                },
            },
        },
    }

    result, err := client.Parse(ctx, "invoice.pdf", &struktr.ParseOptions{
        Schema: schema,
    })
    if err != nil {
        log.Fatal(err)
    }

    // Process the extracted data
    vendor, _ := result.GetString("vendor")
    total, _ := result.GetFloat("total")
    dueDate, _ := result.GetString("due_date")

    fmt.Printf("Invoice from: %s\n", vendor)
    fmt.Printf("Total due: $%.2f\n", total)
    fmt.Printf("Due date: %s\n", dueDate)
}
```

### HTTP Webhook Handler

```go
package main

import (
    "crypto/hmac"
    "crypto/sha256"
    "encoding/hex"
    "encoding/json"
    "fmt"
    "net/http"
    "os"
)

type WebhookPayload struct {
    Event     string                 `json:"event"`
    Timestamp string                 `json:"timestamp"`
    Data      map[string]interface{} `json:"data"`
}

func verifyWebhook(payload []byte, signature string, secret string) bool {
    mac := hmac.New(sha256.New, []byte(secret))
    mac.Write(payload)
    expected := "sha256=" + hex.EncodeToString(mac.Sum(nil))
    return hmac.Equal([]byte(expected), []byte(signature))
}

func webhookHandler(w http.ResponseWriter, r *http.Request) {
    signature := r.Header.Get("X-Struktr-Signature")
    
    var payload WebhookPayload
    if err := json.NewDecoder(r.Body).Decode(&payload); err != nil {
        http.Error(w, "Invalid payload", http.StatusBadRequest)
        return
    }

    payloadBytes, _ := json.Marshal(payload)
    if !verifyWebhook(payloadBytes, signature, os.Getenv("STRUKTR_WEBHOOK_SECRET")) {
        http.Error(w, "Invalid signature", http.StatusUnauthorized)
        return
    }

    switch payload.Event {
    case "document.completed":
        fmt.Printf("Document completed: %+v\n", payload.Data)
    case "document.failed":
        fmt.Printf("Document failed: %+v\n", payload.Data)
    }

    w.WriteHeader(http.StatusOK)
}

func main() {
    http.HandleFunc("/webhooks/struktr", webhookHandler)
    http.ListenAndServe(":8080", nil)
}
```

## API Reference

See the full [API Reference](../api-reference.md) for all available methods and options.
