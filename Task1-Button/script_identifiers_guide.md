# Script Identifiers Reference Guide

This document lists every identifier in the transfer button script that must be updated when deploying to a new base or set of tables.

---

## 1. Airtable Table & View Names

These are the plain display names of your tables and views in Airtable. Change the string values to match whatever they are called in your new base.

| Constant | Current Value | What It Identifies |
|---|---|---|
| `SOURCE_TABLE_NAME` | `"RA Filter"` | The Airtable table the button lives on |
| `SOURCE_VIEW_NAME` | `"RA Filter"` | The view used to load the source record |
| `TARGET_TABLE_NAME` | `"Stream 1"` | The Airtable table records are copied into |

**How to update:** Simply replace the string with the display name of your table or view exactly as it appears in Airtable (case-sensitive).

```js
const SOURCE_TABLE_NAME = "Your New Source Table";
const SOURCE_VIEW_NAME  = "Your New View Name";
const TARGET_TABLE_NAME = "Your New Target Table";
```

---

## 2. Airtable Field ID (Checkbox)

| Constant | Current Value | What It Identifies |
|---|---|---|
| `TRANSFER_CHECKBOX_FIELD_ID` | `"cnxpjkjloygs0b9"` | Internal field ID of the checkbox that gets ticked on transfer |

This is the **internal field ID** (not the column name) of the checkbox column on the source table.

**How to find it:**
1. In Airtable, click the column header of your checkbox field.
2. Select **Customize field**.
3. Click **Copy field ID** at the bottom of the panel.
4. Paste the copied ID into the constant.

```js
const TRANSFER_CHECKBOX_FIELD_ID = "your_new_field_id_here";
```

---

## 3. NocoDB IDs & Credentials

These are used by the script to fetch signed URLs and upload PDF attachments via the NocoDB API.

| Constant | Current Value | What It Identifies |
|---|---|---|
| `NOCO_BASE_URL` | `"https://app.nocodb.com"` | NocoDB instance URL |
| `NOCO_API_TOKEN` | `"NznZCY..."` | API authentication token |
| `NOCO_BASE_ID` | `"peupdms4mcx2q90"` | NocoDB base (database) ID |
| `RA_TABLE_ID` | `"mshgg62sdfnldh4"` | NocoDB table ID for the source table (RA Filter) |
| `STREAM1_TABLE_ID` | `"m7wpv6vrusxu803"` | NocoDB table ID for the target table (Stream 1) |
| `PDF_FIELD_ID` | `"cpkdbatjdangzen"` | NocoDB field ID for the PDF attachment column in the target table |

### Generating a NocoDB API Token

1. Log into your NocoDB instance.
2. Click your **profile avatar** in the top-right corner.
3. Select **Team & Settings** (or **Account** depending on your version).
4. Go to the **API Tokens** tab.
5. Click **+ Add new token**, give it a name (e.g. `airtable-transfer`), and click **Save**.
6. Copy the token immediately — it will not be shown again.
7. Paste it into the script:

```js
const NOCO_API_TOKEN = "your_token_here";
```

> **Security note:** Treat this token like a password. Do not commit it to version control or share it publicly. Anyone with this token can read and write your NocoDB data.

### Finding Your Base ID and Table IDs

1. In NocoDB, open the target base.
2. Click on any table, then open the **three-dot menu (⋯)** next to the table name in the sidebar.
3. Select **API Snippet**.
4. The URL in the snippet will look like:
   ```
   https://app.nocodb.com/api/v3/data/peupdms4mcx2q90/m7wpv6vrusxu803/records
   ```
   - The first ID segment is your **Base ID** (`NOCO_BASE_ID`)
   - The second ID segment is your **Table ID** (`RA_TABLE_ID` or `STREAM1_TABLE_ID`)
5. Repeat for both the source and target tables to get both table IDs.

### Finding the PDF Field ID

1. In NocoDB, open the **target table** (Stream 1 or equivalent).
2. Click the **three-dot menu (⋯)** on the PDF attachment column header.
3. Select **Copy field ID** (or check the **Fields** settings panel — the ID is displayed there).
4. Paste it into the script:

```js
const PDF_FIELD_ID = "your_pdf_field_id_here";
```

---

## 4. Field Names

These must exactly match the column names in **both** the source and target tables. They are used in `getCellValue()`, `updateRecordAsync()`, and `createRecordAsync()` calls throughout the script.

| Constant | Current Value |
|---|---|
| `FIELD_ARTICLE_ID` | `"Article_ID"` |
| `FIELD_JOURNAL_CODE` | `"Journal_Code"` |
| `FIELD_PUBLICATION_YEAR` | `"Publication Year"` |
| `FIELD_VOLUME` | `"Volume"` |
| `FIELD_ISSUE` | `"Issue"` |
| `FIELD_TITLE` | `"Title"` |
| `FIELD_DOI` | `"DOI"` |
| `FIELD_PDF` | `"PDF"` |
| `FIELD_OA` | `"Open Access"` |

**How to update:** Replace each string with the exact column name as it appears in Airtable (case-sensitive, including spaces).

```js
const FIELD_TITLE = "Your Column Name Here";
```

> **Note:** The `FIELDS` array at the bottom of the constants block is built automatically from these constants — you only need to update the individual `FIELD_*` values; the array will update itself.

---

## Quick Checklist

When deploying to a new base, work through these in order:

- [ ] Update `SOURCE_TABLE_NAME`, `SOURCE_VIEW_NAME`, `TARGET_TABLE_NAME`
- [ ] Find and update `TRANSFER_CHECKBOX_FIELD_ID` from the source table
- [ ] Generate a new `NOCO_API_TOKEN`
- [ ] Update `NOCO_BASE_ID` from the API snippet
- [ ] Update `RA_TABLE_ID` (source table NocoDB ID)
- [ ] Update `STREAM1_TABLE_ID` (target table NocoDB ID)
- [ ] Update `PDF_FIELD_ID` (PDF column in the target table)
- [ ] Verify all `FIELD_*` names match column names in both tables
