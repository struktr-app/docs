# Contract Analysis Example

This example demonstrates how to extract key information from legal contracts and agreements using Struktr.

## Overview

Contract analysis is essential for legal operations, compliance, and business intelligence. Struktr can automatically extract:

- Parties involved (names, roles, entities)
- Key dates (effective, termination, renewal)
- Monetary terms and payment conditions
- Obligations and responsibilities
- Termination clauses
- Governing law and jurisdiction

## Basic Contract Extraction

### Python

```python
from struktr import Struktr

client = Struktr()

# Parse a contract
result = client.parse("contract.pdf")

print(result.data)
# {
#   "document_type": "contract",
#   "contract_type": "service_agreement",
#   "parties": [
#     {
#       "name": "Acme Corporation",
#       "role": "service_provider",
#       "address": "123 Business St, New York, NY"
#     },
#     {
#       "name": "Widget Inc.",
#       "role": "client",
#       "address": "456 Commerce Ave, San Francisco, CA"
#     }
#   ],
#   "effective_date": "2024-01-01",
#   "termination_date": "2024-12-31",
#   "total_value": 120000.00,
#   "governing_law": "State of Delaware"
# }
```

### Node.js

```javascript
import { Struktr } from '@struktr/sdk';

const client = new Struktr();

const result = await client.parse('contract.pdf');
console.log(result.data);
```

## Custom Contract Schema

Define a comprehensive schema for detailed extraction:

```python
from struktr import Struktr, Schema, Field

client = Struktr()

contract_schema = Schema({
    "contract_type": Field(
        type="string",
        description="Type of contract (e.g., NDA, service agreement, employment)"
    ),
    "title": Field(type="string"),
    "parties": Field(
        type="array",
        items={
            "name": Field(type="string"),
            "type": Field(type="string", description="Individual, corporation, LLC, etc."),
            "role": Field(type="string", description="Role in the contract"),
            "address": Field(type="string"),
            "representative": Field(type="string", description="Signing representative")
        }
    ),
    "dates": Field(
        type="object",
        properties={
            "execution_date": Field(type="string", format="date"),
            "effective_date": Field(type="string", format="date"),
            "termination_date": Field(type="string", format="date"),
            "renewal_date": Field(type="string", format="date")
        }
    ),
    "financial_terms": Field(
        type="object",
        properties={
            "total_value": Field(type="number"),
            "currency": Field(type="string"),
            "payment_schedule": Field(type="string"),
            "payment_terms": Field(type="string", description="Net 30, Net 60, etc.")
        }
    ),
    "term_and_termination": Field(
        type="object",
        properties={
            "initial_term": Field(type="string"),
            "renewal_terms": Field(type="string"),
            "termination_for_cause": Field(type="string"),
            "termination_for_convenience": Field(type="string"),
            "notice_period": Field(type="string")
        }
    ),
    "key_clauses": Field(
        type="array",
        items={
            "clause_type": Field(type="string"),
            "summary": Field(type="string"),
            "page": Field(type="integer")
        }
    ),
    "confidentiality": Field(
        type="object",
        properties={
            "has_nda": Field(type="boolean"),
            "duration": Field(type="string"),
            "scope": Field(type="string")
        }
    ),
    "governing_law": Field(type="string"),
    "jurisdiction": Field(type="string"),
    "dispute_resolution": Field(type="string", description="Arbitration, litigation, etc.")
})

result = client.parse("contract.pdf", schema=contract_schema)
```

## Analyzing Different Contract Types

### Non-Disclosure Agreement (NDA)

```python
nda_schema = Schema({
    "agreement_type": Field(type="string"),
    "parties": Field(
        type="array",
        items={
            "name": Field(type="string"),
            "role": Field(type="string", description="Disclosing party or receiving party")
        }
    ),
    "effective_date": Field(type="string", format="date"),
    "confidential_information": Field(
        type="object",
        properties={
            "definition": Field(type="string"),
            "inclusions": Field(type="array", items={"type": "string"}),
            "exclusions": Field(type="array", items={"type": "string"})
        }
    ),
    "obligations": Field(type="array", items={"type": "string"}),
    "term": Field(type="string"),
    "survival_period": Field(type="string", description="How long obligations last after termination"),
    "governing_law": Field(type="string")
})

result = client.parse("nda.pdf", schema=nda_schema)
```

### Employment Agreement

```python
employment_schema = Schema({
    "employer": Field(
        type="object",
        properties={
            "name": Field(type="string"),
            "address": Field(type="string")
        }
    ),
    "employee": Field(
        type="object",
        properties={
            "name": Field(type="string"),
            "title": Field(type="string"),
            "department": Field(type="string")
        }
    ),
    "start_date": Field(type="string", format="date"),
    "employment_type": Field(type="string", description="Full-time, part-time, contractor"),
    "compensation": Field(
        type="object",
        properties={
            "base_salary": Field(type="number"),
            "currency": Field(type="string"),
            "pay_frequency": Field(type="string"),
            "bonus_structure": Field(type="string"),
            "equity": Field(type="string")
        }
    ),
    "benefits": Field(type="array", items={"type": "string"}),
    "work_location": Field(type="string"),
    "non_compete": Field(
        type="object",
        properties={
            "has_clause": Field(type="boolean"),
            "duration": Field(type="string"),
            "geographic_scope": Field(type="string")
        }
    ),
    "termination": Field(
        type="object",
        properties={
            "notice_period": Field(type="string"),
            "severance": Field(type="string")
        }
    )
})

result = client.parse("employment_agreement.pdf", schema=employment_schema)
```

### Service Level Agreement (SLA)

```python
sla_schema = Schema({
    "service_provider": Field(type="string"),
    "client": Field(type="string"),
    "services": Field(
        type="array",
        items={
            "name": Field(type="string"),
            "description": Field(type="string")
        }
    ),
    "service_levels": Field(
        type="array",
        items={
            "metric": Field(type="string"),
            "target": Field(type="string"),
            "measurement_period": Field(type="string")
        }
    ),
    "uptime_guarantee": Field(type="string"),
    "response_times": Field(
        type="array",
        items={
            "severity": Field(type="string"),
            "response_time": Field(type="string"),
            "resolution_time": Field(type="string")
        }
    ),
    "penalties": Field(
        type="array",
        items={
            "condition": Field(type="string"),
            "penalty": Field(type="string")
        }
    ),
    "exclusions": Field(type="array", items={"type": "string"})
})

result = client.parse("sla.pdf", schema=sla_schema)
```

## Contract Comparison

Compare two versions of a contract:

```python
from struktr import Struktr

client = Struktr()

# Parse both versions
original = client.parse("contract_v1.pdf")
amended = client.parse("contract_v2.pdf")

def compare_contracts(original, amended):
    """Compare two contract versions and identify changes."""
    changes = []
    
    # Compare dates
    for date_field in ["effective_date", "termination_date"]:
        orig_date = original.data.get(date_field)
        new_date = amended.data.get(date_field)
        if orig_date != new_date:
            changes.append({
                "field": date_field,
                "original": orig_date,
                "amended": new_date
            })
    
    # Compare financial terms
    orig_value = original.data.get("total_value")
    new_value = amended.data.get("total_value")
    if orig_value != new_value:
        changes.append({
            "field": "total_value",
            "original": orig_value,
            "amended": new_value,
            "change_percent": ((new_value - orig_value) / orig_value * 100) if orig_value else None
        })
    
    return changes

changes = compare_contracts(original, amended)
for change in changes:
    print(f"{change['field']}: {change['original']} → {change['amended']}")
```

## Key Date Extraction and Alerts

```python
from struktr import Struktr
from datetime import datetime, timedelta

client = Struktr()

def analyze_contract_dates(file_path):
    """Analyze contract and return important date alerts."""
    result = client.parse(file_path)
    
    today = datetime.now().date()
    alerts = []
    
    # Check termination date
    term_date_str = result.data.get("termination_date")
    if term_date_str:
        term_date = datetime.strptime(term_date_str, "%Y-%m-%d").date()
        days_until = (term_date - today).days
        
        if days_until < 0:
            alerts.append({
                "type": "expired",
                "message": f"Contract expired {abs(days_until)} days ago",
                "date": term_date_str,
                "severity": "critical"
            })
        elif days_until <= 30:
            alerts.append({
                "type": "expiring_soon",
                "message": f"Contract expires in {days_until} days",
                "date": term_date_str,
                "severity": "high"
            })
        elif days_until <= 90:
            alerts.append({
                "type": "renewal_reminder",
                "message": f"Contract expires in {days_until} days - consider renewal",
                "date": term_date_str,
                "severity": "medium"
            })
    
    # Check renewal date
    renewal_date_str = result.data.get("renewal_date")
    if renewal_date_str:
        renewal_date = datetime.strptime(renewal_date_str, "%Y-%m-%d").date()
        days_until = (renewal_date - today).days
        
        if days_until <= 30 and days_until > 0:
            alerts.append({
                "type": "renewal_deadline",
                "message": f"Renewal deadline in {days_until} days",
                "date": renewal_date_str,
                "severity": "high"
            })
    
    return {
        "contract_data": result.data,
        "alerts": alerts
    }

# Usage
analysis = analyze_contract_dates("contract.pdf")

print("Alerts:")
for alert in analysis["alerts"]:
    print(f"  [{alert['severity'].upper()}] {alert['message']}")
```

## Complete Contract Management System

```python
from struktr import Struktr, Schema, Field
from datetime import datetime, timedelta
import json

class ContractManager:
    def __init__(self):
        self.client = Struktr()
        self.schema = self._create_schema()
    
    def _create_schema(self):
        return Schema({
            "contract_type": Field(type="string"),
            "parties": Field(
                type="array",
                items={
                    "name": Field(type="string"),
                    "role": Field(type="string")
                }
            ),
            "effective_date": Field(type="string", format="date"),
            "termination_date": Field(type="string", format="date"),
            "total_value": Field(type="number"),
            "currency": Field(type="string"),
            "payment_terms": Field(type="string"),
            "governing_law": Field(type="string"),
            "auto_renewal": Field(type="boolean"),
            "notice_period_days": Field(type="integer"),
            "key_obligations": Field(type="array", items={"type": "string"})
        })
    
    def analyze_contract(self, file_path):
        """Analyze a contract and return structured insights."""
        result = self.client.parse(file_path, schema=self.schema)
        
        return {
            "id": result.id,
            "data": result.data,
            "risk_assessment": self._assess_risks(result.data),
            "important_dates": self._extract_important_dates(result.data),
            "summary": self._generate_summary(result.data)
        }
    
    def _assess_risks(self, data):
        """Assess contract risks based on extracted data."""
        risks = []
        
        # Check for missing governing law
        if not data.get("governing_law"):
            risks.append({
                "type": "missing_clause",
                "severity": "medium",
                "description": "No governing law specified"
            })
        
        # Check for auto-renewal without notice period
        if data.get("auto_renewal") and not data.get("notice_period_days"):
            risks.append({
                "type": "unfavorable_terms",
                "severity": "high",
                "description": "Auto-renewal without specified notice period"
            })
        
        # Check for high value contracts
        value = data.get("total_value", 0)
        if value > 100000:
            risks.append({
                "type": "high_value",
                "severity": "info",
                "description": f"High value contract: ${value:,.2f}"
            })
        
        return risks
    
    def _extract_important_dates(self, data):
        """Extract and categorize important dates."""
        dates = []
        
        if data.get("effective_date"):
            dates.append({
                "type": "effective_date",
                "date": data["effective_date"],
                "description": "Contract becomes effective"
            })
        
        if data.get("termination_date"):
            term_date = datetime.strptime(data["termination_date"], "%Y-%m-%d").date()
            dates.append({
                "type": "termination_date",
                "date": data["termination_date"],
                "description": "Contract terminates"
            })
            
            # Add reminder date if auto-renewal
            if data.get("auto_renewal") and data.get("notice_period_days"):
                notice_date = term_date - timedelta(days=data["notice_period_days"])
                dates.append({
                    "type": "notice_deadline",
                    "date": notice_date.isoformat(),
                    "description": f"Last day to give notice ({data['notice_period_days']} days before termination)"
                })
        
        return dates
    
    def _generate_summary(self, data):
        """Generate a human-readable contract summary."""
        parties = data.get("parties", [])
        party_names = [p.get("name", "Unknown") for p in parties]
        
        summary_parts = [
            f"This is a {data.get('contract_type', 'unknown type')} between {' and '.join(party_names)}."
        ]
        
        if data.get("effective_date"):
            summary_parts.append(f"The contract is effective from {data['effective_date']}.")
        
        if data.get("termination_date"):
            summary_parts.append(f"It terminates on {data['termination_date']}.")
        
        if data.get("total_value"):
            currency = data.get("currency", "USD")
            summary_parts.append(f"Total contract value: {currency} {data['total_value']:,.2f}.")
        
        if data.get("governing_law"):
            summary_parts.append(f"Governed by {data['governing_law']} law.")
        
        return " ".join(summary_parts)
    
    def bulk_analyze(self, file_paths):
        """Analyze multiple contracts and return a summary."""
        results = []
        
        batch = self.client.parse_batch(file_paths, schema=self.schema)
        batch_results = batch.wait()
        
        for result in batch_results:
            if result.status == "completed":
                analysis = {
                    "id": result.id,
                    "data": result.data,
                    "risk_assessment": self._assess_risks(result.data),
                    "important_dates": self._extract_important_dates(result.data)
                }
                results.append(analysis)
        
        return {
            "contracts": results,
            "summary": {
                "total_analyzed": len(results),
                "total_value": sum(c["data"].get("total_value", 0) for c in results),
                "high_risk_count": sum(
                    1 for c in results 
                    if any(r["severity"] == "high" for r in c["risk_assessment"])
                )
            }
        }


# Usage
manager = ContractManager()

# Analyze a single contract
analysis = manager.analyze_contract("service_agreement.pdf")

print("Contract Summary:")
print(analysis["summary"])

print("\nRisk Assessment:")
for risk in analysis["risk_assessment"]:
    print(f"  [{risk['severity'].upper()}] {risk['description']}")

print("\nImportant Dates:")
for date in analysis["important_dates"]:
    print(f"  {date['date']}: {date['description']}")

# Bulk analysis
contracts = ["contract1.pdf", "contract2.pdf", "contract3.pdf"]
bulk_results = manager.bulk_analyze(contracts)

print(f"\nBulk Analysis Summary:")
print(f"  Contracts analyzed: {bulk_results['summary']['total_analyzed']}")
print(f"  Total value: ${bulk_results['summary']['total_value']:,.2f}")
print(f"  High-risk contracts: {bulk_results['summary']['high_risk_count']}")
```

## Next Steps

- [Invoice Processing](./invoice-processing.md) - Extract data from invoices
- [Batch Processing](./batch-processing.md) - Process documents at scale
- [Custom Schemas](./custom-schemas.md) - Learn more about schema customization
