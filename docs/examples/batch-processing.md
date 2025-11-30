# Batch Processing Example

This example demonstrates how to efficiently process multiple documents using Struktr's batch processing capabilities.

## Overview

Batch processing allows you to:

- Submit multiple documents in a single request
- Process documents in parallel for faster throughput
- Receive aggregated results and progress updates
- Handle large volumes efficiently with webhooks

## Basic Batch Processing

### Python

```python
from struktr import Struktr

client = Struktr()

# Submit multiple files for processing
batch = client.parse_batch([
    "invoice1.pdf",
    "invoice2.pdf",
    "invoice3.pdf",
    "invoice4.pdf",
    "invoice5.pdf"
])

print(f"Batch ID: {batch.id}")
print(f"Total documents: {batch.total}")

# Wait for all documents to complete
results = batch.wait(timeout=600)  # 10 minute timeout

# Process results
for result in results:
    if result.status == "completed":
        print(f"✓ {result.id}: {result.data.get('invoice_number')}")
    else:
        print(f"✗ {result.id}: Failed")
```

### Node.js

```javascript
import { Struktr } from '@struktr/sdk';

const client = new Struktr();

// Submit batch
const batch = await client.parseBatch([
  'invoice1.pdf',
  'invoice2.pdf',
  'invoice3.pdf',
  'invoice4.pdf',
  'invoice5.pdf',
]);

console.log(`Batch ID: ${batch.id}`);
console.log(`Total documents: ${batch.total}`);

// Wait for completion
const results = await batch.wait({ timeout: 600000 });

for (const result of results) {
  if (result.status === 'completed') {
    console.log(`✓ ${result.id}: ${result.data.invoiceNumber}`);
  } else {
    console.log(`✗ ${result.id}: Failed`);
  }
}
```

## Batch Processing with Options

Apply the same options to all documents in the batch:

```python
from struktr import Struktr, ParseOptions, Schema, Field

client = Struktr()

# Define a schema for all documents
invoice_schema = Schema({
    "vendor": Field(type="string"),
    "invoice_number": Field(type="string"),
    "total": Field(type="number")
})

options = ParseOptions(
    extract_tables=True,
    ocr_enabled=True,
    schema=invoice_schema
)

batch = client.parse_batch(
    ["invoice1.pdf", "invoice2.pdf", "invoice3.pdf"],
    options=options
)

results = batch.wait()
```

## Processing Files from a Directory

```python
from struktr import Struktr
import os
from pathlib import Path

client = Struktr()

def process_directory(directory: str, pattern: str = "*.pdf"):
    """Process all matching files in a directory."""
    path = Path(directory)
    files = list(path.glob(pattern))
    
    if not files:
        print(f"No files matching {pattern} found in {directory}")
        return []
    
    print(f"Found {len(files)} files to process")
    
    # Submit batch
    batch = client.parse_batch([str(f) for f in files])
    
    # Wait with progress updates
    while True:
        status = batch.status
        print(f"Progress: {status.completed}/{status.total}")
        
        if status.is_complete:
            break
        
        time.sleep(5)  # Check every 5 seconds
    
    return batch.results

# Usage
results = process_directory("./invoices/2024/january")
```

## Async Batch Processing with Webhooks

For large batches, use webhooks to receive results:

```python
from struktr import Struktr

client = Struktr()

# Submit batch with webhook
batch = client.parse_batch(
    ["doc1.pdf", "doc2.pdf", "doc3.pdf"],
    webhook_url="https://your-server.com/webhooks/struktr"
)

print(f"Batch submitted: {batch.id}")
# Don't wait - results will be sent to webhook
```

Webhook handler:

```python
from flask import Flask, request
import hmac
import hashlib

app = Flask(__name__)
WEBHOOK_SECRET = "your-webhook-secret"

def verify_signature(payload: str, signature: str) -> bool:
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)

@app.route('/webhooks/struktr', methods=['POST'])
def handle_webhook():
    signature = request.headers.get('X-Struktr-Signature')
    payload = request.get_data(as_text=True)
    
    if not verify_signature(payload, signature):
        return 'Invalid signature', 401
    
    data = request.json
    event = data['event']
    
    if event == 'document.completed':
        # Individual document completed
        doc = data['data']
        print(f"Document {doc['id']} completed")
        save_document_result(doc)
    
    elif event == 'batch.progress':
        # Batch progress update
        progress = data['data']
        print(f"Batch {progress['batch_id']}: {progress['completed']}/{progress['total']}")
    
    elif event == 'batch.completed':
        # Entire batch completed
        batch = data['data']
        print(f"Batch {batch['batch_id']} completed!")
        print(f"Succeeded: {batch['succeeded']}, Failed: {batch['failed']}")
    
    return 'OK', 200
```

## Monitoring Batch Progress

### Polling for Progress

```python
from struktr import Struktr
import time

client = Struktr()

batch = client.parse_batch(files)

while not batch.is_complete:
    status = batch.refresh()  # Get latest status
    
    print(f"Status: {status.completed}/{status.total} complete")
    print(f"  Pending: {status.pending}")
    print(f"  Processing: {status.processing}")
    print(f"  Completed: {status.completed}")
    print(f"  Failed: {status.failed}")
    
    # Calculate progress
    progress = (status.completed + status.failed) / status.total * 100
    print(f"  Progress: {progress:.1f}%")
    
    time.sleep(10)  # Wait 10 seconds before checking again

print("Batch complete!")
print(f"Final: {status.completed} succeeded, {status.failed} failed")
```

### Progress Callback

```python
from struktr import Struktr

client = Struktr()

def on_progress(status):
    """Called periodically with progress updates."""
    progress = (status.completed + status.failed) / status.total * 100
    print(f"Progress: {progress:.1f}% ({status.completed}/{status.total})")

batch = client.parse_batch(files)
results = batch.wait(on_progress=on_progress, poll_interval=5)
```

## Handling Batch Results

### Separating Successes and Failures

```python
from struktr import Struktr

client = Struktr()

batch = client.parse_batch(files)
results = batch.wait()

# Separate results
successes = [r for r in results if r.status == "completed"]
failures = [r for r in results if r.status == "failed"]

print(f"Succeeded: {len(successes)}")
print(f"Failed: {len(failures)}")

# Process successes
for result in successes:
    save_to_database(result.data)

# Handle failures
for result in failures:
    log_failure(result.id, result.error)
    # Optionally retry
    retry_later(result.id)
```

### Retrying Failed Documents

```python
from struktr import Struktr
import time

client = Struktr()

def process_with_retry(files: list, max_retries: int = 3):
    """Process files with automatic retry for failures."""
    all_results = []
    remaining_files = files.copy()
    
    for attempt in range(max_retries):
        if not remaining_files:
            break
        
        print(f"Attempt {attempt + 1}: Processing {len(remaining_files)} files")
        
        batch = client.parse_batch(remaining_files)
        results = batch.wait()
        
        # Separate results
        successes = [r for r in results if r.status == "completed"]
        failures = [r for r in results if r.status == "failed"]
        
        all_results.extend(successes)
        
        if failures:
            print(f"  {len(failures)} failures, will retry")
            # Get original file paths for failed documents
            remaining_files = [
                f for f, r in zip(remaining_files, results)
                if r.status == "failed"
            ]
            time.sleep(5)  # Wait before retry
        else:
            remaining_files = []
    
    if remaining_files:
        print(f"Final failures after {max_retries} attempts: {len(remaining_files)}")
    
    return all_results

# Usage
results = process_with_retry(files, max_retries=3)
```

## Optimizing Batch Performance

### Chunking Large Batches

```python
from struktr import Struktr
from concurrent.futures import ThreadPoolExecutor

client = Struktr()

def chunk_list(lst: list, chunk_size: int):
    """Split a list into chunks."""
    for i in range(0, len(lst), chunk_size):
        yield lst[i:i + chunk_size]

def process_large_batch(files: list, chunk_size: int = 100):
    """Process a large number of files in chunks."""
    all_results = []
    chunks = list(chunk_list(files, chunk_size))
    
    print(f"Processing {len(files)} files in {len(chunks)} chunks")
    
    for i, chunk in enumerate(chunks):
        print(f"Processing chunk {i + 1}/{len(chunks)}")
        
        batch = client.parse_batch(chunk)
        results = batch.wait()
        all_results.extend(results)
        
        print(f"  Completed: {len([r for r in results if r.status == 'completed'])}")
    
    return all_results

# Usage
all_files = get_all_invoice_files()  # Could be thousands
results = process_large_batch(all_files, chunk_size=100)
```

### Parallel Batch Processing

```python
from struktr import Struktr
from concurrent.futures import ThreadPoolExecutor, as_completed

def process_batch_parallel(files: list, num_workers: int = 4, chunk_size: int = 50):
    """Process batches in parallel using multiple workers."""
    chunks = list(chunk_list(files, chunk_size))
    all_results = []
    
    def process_chunk(chunk):
        client = Struktr()  # Create client per worker
        batch = client.parse_batch(chunk)
        return batch.wait()
    
    with ThreadPoolExecutor(max_workers=num_workers) as executor:
        futures = {executor.submit(process_chunk, chunk): i for i, chunk in enumerate(chunks)}
        
        for future in as_completed(futures):
            chunk_idx = futures[future]
            try:
                results = future.result()
                all_results.extend(results)
                print(f"Chunk {chunk_idx + 1} completed: {len(results)} documents")
            except Exception as e:
                print(f"Chunk {chunk_idx + 1} failed: {e}")
    
    return all_results
```

## Complete Batch Processing Pipeline

```python
from struktr import Struktr, Schema, Field, ParseOptions
from datetime import datetime
from pathlib import Path
import json
import csv

class BatchProcessor:
    def __init__(self, output_dir: str = "./output"):
        self.client = Struktr()
        self.output_dir = Path(output_dir)
        self.output_dir.mkdir(exist_ok=True)
        
        self.schema = Schema({
            "document_type": Field(type="string"),
            "vendor": Field(type="string"),
            "invoice_number": Field(type="string"),
            "date": Field(type="string", format="date"),
            "total": Field(type="number")
        })
    
    def process(self, files: list, chunk_size: int = 50):
        """Process files and save results."""
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        
        all_results = []
        all_errors = []
        chunks = list(self._chunk_list(files, chunk_size))
        
        for i, chunk in enumerate(chunks):
            print(f"Processing chunk {i + 1}/{len(chunks)} ({len(chunk)} files)")
            
            batch = self.client.parse_batch(
                chunk,
                options=ParseOptions(schema=self.schema)
            )
            
            results = batch.wait()
            
            for result in results:
                if result.status == "completed":
                    all_results.append({
                        "id": result.id,
                        "file": chunk[results.index(result)] if results.index(result) < len(chunk) else "unknown",
                        "data": result.data
                    })
                else:
                    all_errors.append({
                        "id": result.id,
                        "error": str(result.error)
                    })
        
        # Save results
        self._save_results(all_results, timestamp)
        self._save_errors(all_errors, timestamp)
        self._generate_report(all_results, all_errors, timestamp)
        
        return {
            "processed": len(all_results),
            "errors": len(all_errors),
            "output_dir": str(self.output_dir)
        }
    
    def _chunk_list(self, lst: list, size: int):
        for i in range(0, len(lst), size):
            yield lst[i:i + size]
    
    def _save_results(self, results: list, timestamp: str):
        # Save as JSON
        json_path = self.output_dir / f"results_{timestamp}.json"
        with open(json_path, "w") as f:
            json.dump(results, f, indent=2)
        
        # Save as CSV
        csv_path = self.output_dir / f"results_{timestamp}.csv"
        with open(csv_path, "w", newline="") as f:
            writer = csv.DictWriter(f, fieldnames=[
                "id", "vendor", "invoice_number", "date", "total"
            ])
            writer.writeheader()
            for r in results:
                writer.writerow({
                    "id": r["id"],
                    "vendor": r["data"].get("vendor", ""),
                    "invoice_number": r["data"].get("invoice_number", ""),
                    "date": r["data"].get("date", ""),
                    "total": r["data"].get("total", "")
                })
    
    def _save_errors(self, errors: list, timestamp: str):
        if errors:
            error_path = self.output_dir / f"errors_{timestamp}.json"
            with open(error_path, "w") as f:
                json.dump(errors, f, indent=2)
    
    def _generate_report(self, results: list, errors: list, timestamp: str):
        total_value = sum(r["data"].get("total", 0) for r in results)
        
        report = f"""
Batch Processing Report
=======================
Timestamp: {timestamp}

Summary
-------
Total files processed: {len(results) + len(errors)}
Successful: {len(results)}
Failed: {len(errors)}

Financial Summary
-----------------
Total value: ${total_value:,.2f}
Average per document: ${total_value / len(results) if results else 0:,.2f}

Output Files
------------
Results: results_{timestamp}.json
Results: results_{timestamp}.csv
Errors: errors_{timestamp}.json (if any)
"""
        
        report_path = self.output_dir / f"report_{timestamp}.txt"
        with open(report_path, "w") as f:
            f.write(report)
        
        print(report)


# Usage
processor = BatchProcessor(output_dir="./processed_invoices")

# Get all PDF files
invoice_files = list(Path("./invoices").glob("**/*.pdf"))
print(f"Found {len(invoice_files)} files")

# Process
summary = processor.process(invoice_files)
print(f"\nProcessed {summary['processed']} documents")
print(f"Errors: {summary['errors']}")
print(f"Output saved to: {summary['output_dir']}")
```

## Best Practices

1. **Use appropriate chunk sizes**: 50-100 documents per batch is usually optimal
2. **Implement retry logic**: Some failures may be transient
3. **Use webhooks for large batches**: Avoid long-running connections
4. **Monitor progress**: Provide feedback for long-running jobs
5. **Handle errors gracefully**: Log and retry failed documents
6. **Save intermediate results**: Don't lose work if processing is interrupted

## Next Steps

- [Invoice Processing](./invoice-processing.md) - Extract data from invoices
- [Contract Analysis](./contract-analysis.md) - Analyze legal documents
- [Custom Schemas](./custom-schemas.md) - Define custom extraction schemas
