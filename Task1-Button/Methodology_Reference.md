# Automated Record Transfer System — Methodology Reference

**Document type:** Supplementary methodology note  
**Context:** Data pipeline supporting systematic literature review workflow  
**Platform:** Airtable (cloud-based relational database) with NocoDB REST API integration

---

## Abstract

This note documents the design and logic of an automated record transfer script developed to support a systematic literature review data pipeline. The script is deployed as a button action within a screening table and transfers qualified records — along with all associated metadata and PDF attachments — into a downstream extraction table. The workflow eliminates manual re-entry of bibliographic data between pipeline stages and maintains an auditable transfer log via a checkbox flag on the source record.

---

## 1. System Architecture

The pipeline consists of two database tables within a shared base environment:

- **Screening table** (*source*): Records enter this table following initial retrieval. Each record represents one article and carries bibliographic metadata, a PDF attachment, and screening decision fields. A button field on each record triggers the transfer script.
- **Extraction table** (*target*): Qualified records are transferred here for full-text review and data extraction.

A companion database (NocoDB) mirrors the base and provides a REST API used specifically for binary file transfer operations, which are not natively supported by the scripting environment's write API.

```
╔══════════════════════════════════════════════════════════════════════════════════════╗
║                           SHARED NocoDB BASE ENVIRONMENT                           ║
╚══════════════════════════════════════════════════════════════════════════════════════╝


┌──────────────────────────────┐
│        SCREENING TABLE       │
│           (SOURCE)           │
├──────────────────────────────┤
│ • Article metadata           │
│ • DOI information            │
│ • PDF attachment             │
│ • Transfer status checkbox   │
└──────────────────────────────┘
               │
               │ 1. User presses
               │    “Transfer” button
               ▼

        ┌──────────────────────────────┐
        │       TRANSFER SCRIPT        │
        │      (Automation Layer)      │
        ├──────────────────────────────┤
        │ • Reads source record        │
        │ • Validates field mappings   │
        │ • Creates target entry       │
        │ • Handles PDF upload         │
        │ • Updates transfer state     │
        └──────────────────────────────┘
                  │              │
                  │              │
                  │              │
                  │              │
                  │              ▼
                  │      ┌──────────────────────┐
                  │      │   NocoDB v3 REST API │
                  │      ├──────────────────────┤
                  │      │ Binary PDF Transfer  │
                  │      │ Attachment Handling  │
                  │      └──────────────────────┘
                  │                 │
                  │                 │
                  │                 ▼
                  │
                  │ 2. Direct field mapping
                  │    (metadata transfer)
                  ▼

┌──────────────────────────────┐
│       EXTRACTION TABLE       │
│           (TARGET)           │
├──────────────────────────────┤
│ • Newly created record       │
│ • Metadata replicated        │
│ • PDF attachment stored      │
│ • Ready for downstream use   │
└──────────────────────────────┘
               ▲
               │
               │ 3. Source record updated
               │    after successful transfer
               │
┌──────────────────────────────┐
│     TRANSFER FLAG UPDATE     │
├──────────────────────────────┤
│ ✓ Checkbox marked complete   │
└──────────────────────────────┘
```
---

## 2. Data Fields Transferred

The following bibliographic fields are transferred from source to target on each invocation. Field names are standardised across both tables.

| Field | Type | Transfer method |
|---|---|---|
| Article ID | Plain text | Direct write |
| Title | Plain text | Written at record creation |
| Journal Code | Single select | Option existence checked; added if absent |
| Publication Year | Numeric | Direct write |
| Volume | Plain text | Direct write |
| Issue | Plain text | Direct write |
| DOI | Plain text | Direct write |
| Open Access status | Single select | Option existence checked; skipped if absent |
| PDF attachment | Binary file | Via NocoDB REST API (base64 encoded) |

---

## 3. Script Logic

The script executes seven sequential steps. All steps are atomic — a failure at any step outputs a descriptive error message and halts execution without partial writes to subsequent steps.

### Step 1 — Source record resolution
The script reads the row context from the button click event to identify the triggering record. The record is loaded from a named view of the source table rather than the table directly, so that any active view filters are applied. If the record has been filtered out of the view (e.g. due to a prior screening decision), the script exits cleanly with an informative message.

### Step 2 — Field value extraction
Each field is read twice: once as a human-readable string (for logging) and once as a raw cell value (for writing). This dual-read pattern ensures that complex field types — particularly single-select objects and attachment arrays — are preserved in their native format when written to the target, rather than being cast to strings.

### Step 3 — Target record creation and plain-field population
A new record is created in the extraction table with the `Title` field as the initial value. Remaining plain-text fields are then written individually. The write helper skips null or empty values to avoid overwriting any pre-existing data in the target record.

### Step 4 — Single-select field: Journal Code
Single-select fields in this environment cannot receive values outside their defined option list. The script reads the current option list from the target field definition, performs a case-insensitive existence check, and — if the incoming value is not yet present — appends it to the option list before writing. This allows the Journal Code vocabulary to grow organically as new journals enter the pipeline without requiring manual option management.

### Step 5 — Single-select field: Open Access status
The same existence check is applied to the Open Access field. Unlike Journal Code, new values are **not** added automatically; instead, unrecognised values are skipped with a logged warning. This reflects the controlled-vocabulary nature of Open Access status: the set of valid values is fixed, and an unrecognised value most likely indicates a data quality issue in the source record rather than a legitimate new option.

### Step 6 — Transfer flag
A dedicated boolean (checkbox) field on the source record is set to true upon successful completion of the metadata transfer. This provides a persistent, queryable audit trail of which records have been transferred and prevents accidental duplicate transfers. The field is identified by its internal system ID rather than its display name, so the flag remains stable even if the field is renamed.

### Step 7 — PDF attachment transfer
Attachment files cannot be written directly to a record via the scripting environment's native API. The script resolves this constraint through the following procedure:

1. The attachment URL is read from the already-loaded source record object. A signed URL is preferred if present; the standard URL is used as a fallback.
2. The file bytes are fetched via an authenticated HTTP request.
3. The bytes are base64-encoded in 8 KB chunks to prevent memory exhaustion with large files.
4. The encoded payload is submitted to the companion REST API's file upload endpoint, targeting the specific record and field created in Step 3.

If no PDF is present on the source record, this step is skipped and the transfer is still considered complete.

---

## 4. Error Handling

The script uses a fail-fast pattern: each operation is wrapped in a `try/catch` block, and failures produce descriptive output before halting. The following conditions are explicitly handled:

| Condition | Behaviour |
|---|---|
| No row context (script run outside a button) | Exit with message |
| Source view not found | Exit with message |
| Record filtered out of source view | Exit with message |
| Target table not found | Exit with message |
| Record creation failure | Exit with message |
| Field update failure | Log error, continue to next field |
| Journal Code option add failure | Log error, continue |
| Transfer flag write failure | Log error, continue |
| No PDF on source record | Skip PDF step, complete transfer |
| Signed URL fetch failure (HTTP error) | Exit with status code |
| File byte fetch failure | Exit with status code |
| PDF upload failure | Exit with status code and response body |

---

## 5. Design Decisions & Limitations

**Why a companion REST API for PDF upload?**  
The scripting environment's native `table.updateRecordAsync()` method does not support writing binary attachment data from within a script. The REST API of a companion database — which mirrors the base — provides a file upload endpoint that accepts base64-encoded payloads, enabling this capability.

**Why individual field updates rather than a single batch write?**  
The `updateIfNotEmpty` helper skips null and empty values on a per-field basis. A batch write would require constructing an object with only non-empty fields, adding conditional logic for every field. The individual update pattern keeps the code uniform and ensures empty source fields never overwrite existing target data.

**Why is Open Access not auto-added to the option list?**  
Open Access status is a controlled vocabulary with a small, well-defined set of values. An unrecognised value arriving at the transfer step almost certainly reflects a data entry error upstream. Auto-adding it would silently introduce malformed options; skipping with a warning surfaces the issue for manual correction.

**Limitation — PDF URL expiry:**  
Attachment URLs issued by the platform are time-limited signed URLs. If there is significant latency between the record load (Step 1) and the PDF fetch (Step 7), the URL may expire. In practice, the intervening steps complete in under a few seconds, making expiry unlikely. If expiry becomes an issue, the URL should be re-fetched immediately before the byte-fetch request.

**Limitation — NocoDB record ID dependency:**  
The PDF upload endpoint requires the NocoDB internal integer row ID of the target record, which is returned by the record creation call in Step 3. This ID is only valid within the NocoDB environment and cannot be derived from the Airtable record ID. Passing an Airtable record ID to this endpoint will produce an HTTP 422 error.

---

## 6. Reproducibility Notes

To reproduce or adapt this workflow:

- The script requires button-context execution (`cursor.row`) and cannot be run as a standalone automation trigger.
- Both source and target tables must share the same field names for all transferred fields.
- The target table's `Journal_Code` field must be of type **Single select** to support programmatic option addition.
- The companion REST API must be version 3 or later; the upload endpoint path and request schema differ in earlier versions.
- No external libraries are used. The script relies entirely on the platform's native scripting API and the browser-standard `fetch()` function.
