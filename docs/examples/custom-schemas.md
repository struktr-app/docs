# Custom Schemas Example

This example demonstrates how to define and use custom schemas to extract exactly the data you need from documents.

## Overview

Custom schemas allow you to:

- Define exactly which fields to extract
- Specify data types and formats
- Provide field descriptions for better accuracy
- Create nested and complex data structures
- Validate extracted data against your schema

## Basic Schema Definition

### Python

```python
from struktr import Struktr, Schema, Field

client = Struktr()

# Define a simple schema
schema = Schema({
    "company_name": Field(type="string"),
    "date": Field(type="string", format="date"),
    "total_amount": Field(type="number")
})

result = client.parse("document.pdf", schema=schema)

print(result.data)
# {
#   "company_name": "Acme Corporation",
#   "date": "2024-01-15",
#   "total_amount": 1250.50
# }
```

### Node.js

```javascript
import { Struktr } from '@struktr/sdk';

const client = new Struktr();

const schema = {
  type: 'object',
  properties: {
    companyName: { type: 'string' },
    date: { type: 'string', format: 'date' },
    totalAmount: { type: 'number' },
  },
};

const result = await client.parse('document.pdf', { schema });
console.log(result.data);
```

### API (cURL)

```bash
curl -X POST https://api.struktr.app/v1/parse \
  -H "Authorization: Bearer YOUR_API_KEY" \
  -F "file=@document.pdf" \
  -F 'options={
    "schema": {
      "type": "object",
      "properties": {
        "company_name": {"type": "string"},
        "date": {"type": "string", "format": "date"},
        "total_amount": {"type": "number"}
      }
    }
  }'
```

## Field Types

Struktr supports these field types:

| Type | Description | Example |
|------|-------------|---------|
| `string` | Text value | `"Acme Corp"` |
| `number` | Numeric value (float) | `1250.50` |
| `integer` | Whole number | `42` |
| `boolean` | True/false | `true` |
| `array` | List of values | `["item1", "item2"]` |
| `object` | Nested structure | `{"name": "John"}` |

### String Fields

```python
schema = Schema({
    "company_name": Field(type="string"),
    "address": Field(type="string"),
    "email": Field(type="string", format="email"),
    "phone": Field(type="string", format="phone"),
    "date": Field(type="string", format="date"),
    "time": Field(type="string", format="time")
})
```

### Number Fields

```python
schema = Schema({
    "quantity": Field(type="integer"),
    "price": Field(type="number"),
    "discount_percent": Field(type="number"),
    "total": Field(type="number")
})
```

### Boolean Fields

```python
schema = Schema({
    "is_paid": Field(type="boolean"),
    "has_warranty": Field(type="boolean"),
    "requires_signature": Field(type="boolean")
})
```

## Field Descriptions

Provide descriptions to improve extraction accuracy:

```python
schema = Schema({
    "vendor": Field(
        type="string",
        description="The name of the company or person selling goods/services"
    ),
    "invoice_number": Field(
        type="string",
        description="The unique identifier for this invoice, may start with INV, #, or similar"
    ),
    "due_date": Field(
        type="string",
        format="date",
        description="The date by which payment must be received"
    ),
    "total": Field(
        type="number",
        description="The final amount due after taxes and discounts"
    )
})
```

## Nested Objects

Create complex, nested data structures:

```python
schema = Schema({
    "vendor": Field(
        type="object",
        properties={
            "name": Field(type="string"),
            "address": Field(
                type="object",
                properties={
                    "street": Field(type="string"),
                    "city": Field(type="string"),
                    "state": Field(type="string"),
                    "zip": Field(type="string"),
                    "country": Field(type="string")
                }
            ),
            "contact": Field(
                type="object",
                properties={
                    "name": Field(type="string"),
                    "email": Field(type="string", format="email"),
                    "phone": Field(type="string")
                }
            )
        }
    ),
    "customer": Field(
        type="object",
        properties={
            "name": Field(type="string"),
            "address": Field(
                type="object",
                properties={
                    "street": Field(type="string"),
                    "city": Field(type="string"),
                    "state": Field(type="string"),
                    "zip": Field(type="string")
                }
            )
        }
    )
})

result = client.parse("invoice.pdf", schema=schema)

# Access nested data
print(result.data["vendor"]["name"])
print(result.data["vendor"]["address"]["city"])
print(result.data["customer"]["name"])
```

## Arrays

Extract lists of items:

```python
# Simple array
schema = Schema({
    "tags": Field(type="array", items={"type": "string"}),
    "page_numbers": Field(type="array", items={"type": "integer"})
})

# Array of objects
schema = Schema({
    "line_items": Field(
        type="array",
        items={
            "type": "object",
            "properties": {
                "description": Field(type="string"),
                "sku": Field(type="string"),
                "quantity": Field(type="integer"),
                "unit_price": Field(type="number"),
                "amount": Field(type="number")
            }
        }
    )
})

result = client.parse("invoice.pdf", schema=schema)

# Iterate over line items
for item in result.data["line_items"]:
    print(f"{item['description']}: {item['quantity']} x ${item['unit_price']}")
```

## Complex Schema Examples

### Invoice Schema

```python
invoice_schema = Schema({
    "vendor": Field(
        type="object",
        properties={
            "name": Field(type="string", description="Company name"),
            "address": Field(type="string"),
            "tax_id": Field(type="string", description="Tax ID or EIN"),
            "phone": Field(type="string"),
            "email": Field(type="string", format="email")
        }
    ),
    "customer": Field(
        type="object",
        properties={
            "name": Field(type="string"),
            "address": Field(type="string"),
            "account_number": Field(type="string")
        }
    ),
    "invoice_details": Field(
        type="object",
        properties={
            "number": Field(type="string"),
            "date": Field(type="string", format="date"),
            "due_date": Field(type="string", format="date"),
            "po_number": Field(type="string", description="Purchase order number"),
            "terms": Field(type="string", description="Payment terms like Net 30")
        }
    ),
    "line_items": Field(
        type="array",
        items={
            "type": "object",
            "properties": {
                "item_number": Field(type="string"),
                "description": Field(type="string"),
                "quantity": Field(type="number"),
                "unit": Field(type="string"),
                "unit_price": Field(type="number"),
                "discount": Field(type="number"),
                "tax_rate": Field(type="number"),
                "amount": Field(type="number")
            }
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
    "payment_info": Field(
        type="object",
        properties={
            "bank_name": Field(type="string"),
            "account_number": Field(type="string"),
            "routing_number": Field(type="string")
        }
    )
})
```

### Resume/CV Schema

```python
resume_schema = Schema({
    "personal_info": Field(
        type="object",
        properties={
            "name": Field(type="string"),
            "email": Field(type="string", format="email"),
            "phone": Field(type="string"),
            "address": Field(type="string"),
            "linkedin": Field(type="string", description="LinkedIn profile URL"),
            "website": Field(type="string", description="Personal website or portfolio")
        }
    ),
    "summary": Field(
        type="string",
        description="Professional summary or objective statement"
    ),
    "experience": Field(
        type="array",
        items={
            "type": "object",
            "properties": {
                "company": Field(type="string"),
                "title": Field(type="string"),
                "location": Field(type="string"),
                "start_date": Field(type="string"),
                "end_date": Field(type="string", description="End date or 'Present'"),
                "description": Field(type="string"),
                "achievements": Field(type="array", items={"type": "string"})
            }
        }
    ),
    "education": Field(
        type="array",
        items={
            "type": "object",
            "properties": {
                "institution": Field(type="string"),
                "degree": Field(type="string"),
                "field": Field(type="string", description="Field of study"),
                "graduation_date": Field(type="string"),
                "gpa": Field(type="number")
            }
        }
    ),
    "skills": Field(
        type="array",
        items={"type": "string"}
    ),
    "certifications": Field(
        type="array",
        items={
            "type": "object",
            "properties": {
                "name": Field(type="string"),
                "issuer": Field(type="string"),
                "date": Field(type="string"),
                "expires": Field(type="string")
            }
        }
    )
})
```

### Medical Record Schema

```python
medical_schema = Schema({
    "patient": Field(
        type="object",
        properties={
            "name": Field(type="string"),
            "dob": Field(type="string", format="date"),
            "gender": Field(type="string"),
            "mrn": Field(type="string", description="Medical record number"),
            "insurance_id": Field(type="string")
        }
    ),
    "visit_info": Field(
        type="object",
        properties={
            "date": Field(type="string", format="date"),
            "provider": Field(type="string"),
            "facility": Field(type="string"),
            "visit_type": Field(type="string")
        }
    ),
    "vitals": Field(
        type="object",
        properties={
            "blood_pressure": Field(type="string"),
            "heart_rate": Field(type="integer"),
            "temperature": Field(type="number"),
            "weight": Field(type="number"),
            "height": Field(type="number")
        }
    ),
    "diagnoses": Field(
        type="array",
        items={
            "type": "object",
            "properties": {
                "code": Field(type="string", description="ICD-10 code"),
                "description": Field(type="string")
            }
        }
    ),
    "medications": Field(
        type="array",
        items={
            "type": "object",
            "properties": {
                "name": Field(type="string"),
                "dosage": Field(type="string"),
                "frequency": Field(type="string"),
                "route": Field(type="string")
            }
        }
    ),
    "lab_results": Field(
        type="array",
        items={
            "type": "object",
            "properties": {
                "test_name": Field(type="string"),
                "value": Field(type="string"),
                "unit": Field(type="string"),
                "reference_range": Field(type="string"),
                "flag": Field(type="string", description="H for high, L for low, etc.")
            }
        }
    )
})
```

## Schema Validation

Validate your schema before using it:

```python
from struktr import Schema, Field, validate_schema

schema = Schema({
    "name": Field(type="string"),
    "age": Field(type="integer")
})

# Validate schema
errors = validate_schema(schema)

if errors:
    for error in errors:
        print(f"Schema error: {error}")
else:
    print("Schema is valid")
```

## Schema Templates

Create reusable schema components:

```python
# Define reusable field templates
address_fields = {
    "street": Field(type="string"),
    "city": Field(type="string"),
    "state": Field(type="string"),
    "zip": Field(type="string"),
    "country": Field(type="string")
}

contact_fields = {
    "name": Field(type="string"),
    "email": Field(type="string", format="email"),
    "phone": Field(type="string")
}

# Use in schemas
invoice_schema = Schema({
    "vendor": Field(
        type="object",
        properties={
            "name": Field(type="string"),
            "address": Field(type="object", properties=address_fields),
            "contact": Field(type="object", properties=contact_fields)
        }
    ),
    "customer": Field(
        type="object",
        properties={
            "name": Field(type="string"),
            "address": Field(type="object", properties=address_fields),
            "contact": Field(type="object", properties=contact_fields)
        }
    )
})
```

## Dynamic Schema Generation

Generate schemas programmatically:

```python
def create_form_schema(field_definitions: list) -> Schema:
    """Create a schema from a list of field definitions."""
    properties = {}
    
    for field_def in field_definitions:
        name = field_def["name"]
        field_type = field_def.get("type", "string")
        description = field_def.get("description", "")
        
        properties[name] = Field(
            type=field_type,
            description=description
        )
    
    return Schema(properties)

# Usage
field_defs = [
    {"name": "first_name", "type": "string", "description": "First name"},
    {"name": "last_name", "type": "string", "description": "Last name"},
    {"name": "email", "type": "string", "description": "Email address"},
    {"name": "age", "type": "integer", "description": "Age in years"}
]

schema = create_form_schema(field_defs)
result = client.parse("form.pdf", schema=schema)
```

## Handling Optional Fields

All fields are optional by default. If a field cannot be extracted, it will be `null` in the response:

```python
schema = Schema({
    "required_field": Field(type="string"),
    "optional_field": Field(type="string")
})

result = client.parse("document.pdf", schema=schema)

# Handle potentially missing fields
name = result.data.get("optional_field", "Default Value")

# Or check if field exists
if result.data.get("optional_field"):
    process_optional_field(result.data["optional_field"])
```

## Best Practices

1. **Be specific with descriptions**: The more context you provide, the better the extraction
2. **Use appropriate types**: Match field types to expected data (number vs string)
3. **Keep schemas focused**: Don't try to extract too many unrelated fields
4. **Use nested objects**: Group related fields together logically
5. **Test with sample documents**: Iterate on your schema based on results
6. **Handle missing data**: Always account for fields that may not be extracted

## Next Steps

- [Invoice Processing](./invoice-processing.md) - See schemas in action
- [Contract Analysis](./contract-analysis.md) - Complex schema examples
- [Batch Processing](./batch-processing.md) - Apply schemas to multiple documents
