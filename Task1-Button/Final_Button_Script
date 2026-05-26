// ======================================================
// CONSTANTS 
// ======================================================

const SOURCE_TABLE_NAME          = "RA Filter";
const SOURCE_VIEW_NAME           = "RA Filter";
const TARGET_TABLE_NAME          = "Stream 1";
const TRANSFER_CHECKBOX_FIELD_ID = "cshkkkxvhh87qs1";

const NOCO_BASE_URL    = "https://app.nocodb.com";
const NOCO_API_TOKEN   = "NznZCYDSMkYbTvo7qxrXINAqPkHvXgU67PVMV0G9";
const NOCO_BASE_ID     = "p5vl8aexmitl9iw";
const RA_TABLE_ID      = "mb4sus1c91hbkp0"; 
const STREAM1_TABLE_ID = "myyjd4jln70o7yd";
const PDF_FIELD_ID     = "cqm1d2ui749olys";

const FIELD_ARTICLE_ID       = "Article_ID";
const FIELD_JOURNAL_CODE     = "Journal_Code";
const FIELD_PUBLICATION_YEAR = "Publication Year";
const FIELD_VOLUME           = "Volume";
const FIELD_ISSUE            = "Issue";
const FIELD_TITLE            = "Title";
const FIELD_DOI              = "DOI";
const FIELD_PDF              = "PDF";
const FIELD_OA               = "Open Access";

const FIELDS = [
  FIELD_ARTICLE_ID, FIELD_JOURNAL_CODE, FIELD_PUBLICATION_YEAR,
  FIELD_VOLUME, FIELD_ISSUE, FIELD_TITLE, FIELD_DOI, FIELD_PDF, FIELD_OA,
];

// ======================================================
// HELPERS 
// ======================================================

function toPrintableString(record, fieldName) {
  if (record.getCellValueAsString) {
    const s = record.getCellValueAsString(fieldName);
    return (s && String(s).trim().length > 0) ? String(s) : "";
  }
  const v = record.getCellValue(fieldName);
  if (v === null || v === undefined) return "";
  if (typeof v === "string") return v.trim().length ? v : "";
  if (typeof v === "number" || typeof v === "boolean") return String(v);
  try {
    const s = JSON.stringify(v);
    return s && s !== "{}" && s !== "[]" ? s : "";
  } catch { return ""; }
}

function attachmentCount(record, fieldName) {
  const v = record.getCellValue(fieldName);
  if (!v) return 0;
  if (Array.isArray(v)) return v.length;
  if (typeof v === "object") return 1;
  return 0;
}

function isEmptyCell(v) {
  if (v === null || v === undefined) return true;
  if (typeof v === "string") return v.trim().length === 0;
  if (Array.isArray(v)) return v.length === 0;
  return false;
}

async function updateIfNotEmpty(table, recordId, fieldMap) {
  for (const [fieldName, value] of Object.entries(fieldMap)) {
    if (isEmptyCell(value)) continue;
    try {
      await table.updateRecordAsync(recordId, { [fieldName]: value });
    } catch (err) {
      output.text(`❌ Failed — ${fieldName}: ${err}`);
    }
  }
}

async function fetchRecordById(table, id) {
  const q = await table.selectRecordsAsync();
  return q.records.find(r => r.id === id) || null;
}

// ======================================================
// STEP 1 — LOAD SOURCE RECORD
// ======================================================

script.step({
  title: "Transfer to Stream 1",
  description: "Loading source record and transferring all fields",
  color: "blue",
  icon: "checkCircle",
});

const rowCtx = cursor?.row;
if (!rowCtx) { output.text("❌ No row context — run from a Button on a record."); return; }

const sourceTable = base.getTable(SOURCE_TABLE_NAME);
const sourceView  = sourceTable.views?.find(v => v.name === SOURCE_VIEW_NAME);
if (!sourceView) { output.text(`❌ View not found: "${SOURCE_VIEW_NAME}"`); return; }

const record = await sourceView.selectRecordAsync(rowCtx.id, { fields: FIELDS });
if (!record) { output.text("❌ Could not load record — it may be filtered out of the view."); return; }

// ======================================================
// STEP 2 — READ FIELD VALUES
// ======================================================

const articleId       = toPrintableString(record, FIELD_ARTICLE_ID);
const journalCode     = toPrintableString(record, FIELD_JOURNAL_CODE);
const publicationYear = toPrintableString(record, FIELD_PUBLICATION_YEAR);
const volume          = toPrintableString(record, FIELD_VOLUME);
const issue           = toPrintableString(record, FIELD_ISSUE);
const title           = toPrintableString(record, FIELD_TITLE);
const doi             = toPrintableString(record, FIELD_DOI);
const oa              = toPrintableString(record, FIELD_OA);

const rawJournalCode     = record.getCellValue(FIELD_JOURNAL_CODE);
const rawPublicationYear = record.getCellValue(FIELD_PUBLICATION_YEAR);
const rawVolume          = record.getCellValue(FIELD_VOLUME);
const rawIssue           = record.getCellValue(FIELD_ISSUE);
const rawTitle           = record.getCellValue(FIELD_TITLE);
const rawDoi             = record.getCellValue(FIELD_DOI);
const rawPdf             = record.getCellValue(FIELD_PDF);
const rawOA              = record.getCellValue(FIELD_OA);
const rawArticleId       = record.getCellValue(FIELD_ARTICLE_ID);
const pdfCount           = attachmentCount(record, FIELD_PDF);

output.text(`📄 ${articleId} — ${title}`);
output.text(`   Journal: ${journalCode || "—"}  |  Year: ${publicationYear || "—"}  |  Vol: ${volume || "—"}  |  Issue: ${issue || "—"}`);
output.text(`   DOI: ${doi || "—"}  |  OA: ${oa || "—"}  |  PDF: ${pdfCount > 0 ? "✓" : "none"}`);

// ======================================================
// STEP 3 — CREATE TARGET RECORD + UPDATE PLAIN FIELDS
// ======================================================

script.step({
  title: "Creating Stream 1 record",
  description: "Writing all metadata fields",
  color: "purple",
  icon: "sync",
});

output.text(`• Article ID:   ${articleId || "—"}`);
output.text(`• Title:        ${title || "—"}`);
output.text(`• Journal:      ${journalCode || "—"}`);
output.text(`• Year:         ${publicationYear || "—"}`);
output.text(`• Volume:       ${volume || "—"}`);
output.text(`• Issue:        ${issue || "—"}`);
output.text(`• DOI:          ${doi || "—"}`);
output.text(`• Open Access:  ${oa || "—"}`);
output.text(`• PDF:          ${pdfCount > 0 ? "✓" : "none"}`);

const targetTable = base.getTable(TARGET_TABLE_NAME);
if (!targetTable) { output.text(`❌ Target table not found: "${TARGET_TABLE_NAME}"`); return; }

let createResult;
try {
  createResult = await targetTable.createRecordAsync({ [FIELD_TITLE]: rawTitle ?? "" });
} catch (err) {
  output.text(`❌ Could not create record in ${TARGET_TABLE_NAME}: ${err}`);
  return;
}

await updateIfNotEmpty(targetTable, createResult, {
  [FIELD_ARTICLE_ID]:       rawArticleId,
  [FIELD_PUBLICATION_YEAR]: rawPublicationYear,
  [FIELD_VOLUME]:           rawVolume,
  [FIELD_ISSUE]:            rawIssue,
  [FIELD_TITLE]:            rawTitle,
  [FIELD_DOI]:              rawDoi,
});

// ======================================================
// STEP 4 — SINGLE SELECT: Journal_Code
// ======================================================

const desiredJournal = String(rawJournalCode ?? "").trim();

if (desiredJournal) {
  const journalField   = targetTable.getField(FIELD_JOURNAL_CODE);
  const currentChoices = journalField?.options?.choices || [];
  const exists         = currentChoices.some(c => c.title.trim().toLowerCase() === desiredJournal.toLowerCase());

  if (!exists) {
    try {
      await journalField.updateOptionsAsync({
        choices: [
          ...currentChoices.map(c => ({ id: c.id, title: c.title, color: c.color ?? "#cfdffe" })),
          { title: desiredJournal, color: "#6c757d" }
        ]
      });
    } catch (err) {
      output.text(`❌ Could not add Journal_Code choice "${desiredJournal}": ${err}`);
    }
  }

  const afterChoices = targetTable.getField(FIELD_JOURNAL_CODE)?.options?.choices || [];
  const nowExists    = afterChoices.some(c => c.title.trim().toLowerCase() === desiredJournal.toLowerCase());

  if (nowExists) {
    try {
      await targetTable.updateRecordAsync(createResult, { [FIELD_JOURNAL_CODE]: desiredJournal });
    } catch (err) {
      output.text(`❌ Could not set Journal_Code: ${err}`);
    }
  }
}

// ======================================================
// STEP 5 — SINGLE SELECT: Open Access
// ======================================================

const desiredOA = String(rawOA ?? "").trim();

if (desiredOA) {
  const oaField   = targetTable.getField(FIELD_OA);
  const oaChoices = oaField?.options?.choices || [];
  const oaExists  = oaChoices.some(c => c.title.trim().toLowerCase() === desiredOA.toLowerCase());

  if (oaExists) {
    try {
      await targetTable.updateRecordAsync(createResult, { [FIELD_OA]: desiredOA });
    } catch (err) {
      output.text(`❌ Could not set Open Access: ${err}`);
    }
  } else {
    output.text(`⚠️ Open Access value "${desiredOA}" not found in target — skipping.`);
  }
}

// ======================================================
// STEP 6 — FLAG SOURCE CHECKBOX
// ======================================================

try {
  await sourceTable.updateRecordAsync(rowCtx.id, { [TRANSFER_CHECKBOX_FIELD_ID]: true });
} catch (err) {
  output.text(`❌ Could not flag source checkbox: ${err}`);
}

// ======================================================
// STEP 7 — PDF TRANSFER
// ======================================================

script.step({
  title: "PDF Transfer",
  description: "Fetching and uploading PDF attachment",
  color: "green",
  icon: "upload",
});

if (!rawPdf || !Array.isArray(rawPdf) || rawPdf.length === 0) {
  output.text("⚠️ No PDF in source record — skipping.");
  output.text(`✅ Transfer complete: ${articleId}`);
  return;
}

const sourceFile = rawPdf[0];
if (!sourceFile.url) {
  output.text("❌ Source file has no URL.");
  return;
}

// Fetch fresh signed URL from v3 API
const freshUrl = sourceFile.signedUrl ?? sourceFile.url;
if (!freshUrl) { output.text("❌ No URL on source file."); return; }

// Fetch file bytes and base64 encode
let base64Data;
try {
  const fileResponse = await fetch(freshUrl, { credentials: "include" });
  if (!fileResponse.ok) { output.text(`❌ Could not fetch file bytes: HTTP ${fileResponse.status}`); return; }

  const uint8Array = new Uint8Array(await fileResponse.arrayBuffer());
  let binary = "";
  for (let i = 0; i < uint8Array.length; i += 8192) {
    binary += String.fromCharCode(...uint8Array.subarray(i, i + 8192));
  }
  base64Data = btoa(binary);
} catch (err) {
  output.text(`❌ Failed to read file bytes: ${err}`);
  return;
}

// Upload base64 directly to Stream 1 cell via v3 API
try {
  const uploadResponse = await fetch(
    `${NOCO_BASE_URL}/api/v3/data/${NOCO_BASE_ID}/${STREAM1_TABLE_ID}/records/${createResult}/fields/${PDF_FIELD_ID}/upload`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json", "xc-token": NOCO_API_TOKEN },
      body: JSON.stringify({ contentType: "application/pdf", file: base64Data, filename: sourceFile.title }),
    }
  );
  if (!uploadResponse.ok) {
    const errText = await uploadResponse.text();
    output.text(`❌ PDF upload failed (HTTP ${uploadResponse.status}): ${errText}`);
    return;
  }
} catch (err) {
  output.text(`❌ PDF upload request failed: ${err}`);
  return;
}

output.text(`✅ Transfer complete: ${articleId} — "${title}"`);
