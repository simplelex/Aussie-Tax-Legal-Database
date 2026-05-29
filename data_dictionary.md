# Data Dictionary — ATO Legal Database

Data Target [ato.gov.au/law](https://www.ato.gov.au/law/view/), write UTF-8 CSV and NDJSON (JSON Lines).

---

## Common conventions

| Convention | Detail |
|---|---|
| **Delimiter** | ` \| ` (space-pipe-space) separates multiple values or sub-sections within a single cell |
| **Empty values** | Empty string `""`, never `NULL` or `N/A` |
| **Boolean columns** | Python `True` / `False` (written as-is by `csv.DictWriter`; JSON booleans `true`/`false` in NDJSON) |
| **Date strings** | Stored verbatim as scraped — no normalisation applied (see per-scraper notes) |
| **Multi-value columns** | Values joined with ` \| ` — e.g. `ITAA 1936 s 6 \| ITAA 1997 s 995-1` |
| **Encoding** | UTF-8, no BOM |
| **Dual output** | Every scraper writes a `.csv` and a parallel `.jsonl` (NDJSON). JSON keys match CSV column names exactly; one JSON object per line. The NDJSON is the recommended format for `datasets.load_dataset()`. |
| **Zip artefacts** | Each dataset produces four files: `<base>.csv`, `<base>.jsonl`, `<base>.zip` (CSV compressed), `<base>_jsonl.zip` (NDJSON compressed). Zips are refreshed only when new rows were written or the zip is older than its source. |

---

## 1. Edited Private Advice

**Output files:**
- `EV_Data/edited_private_advice_all.csv` — all years combined into one file
- `EV_Data/edited_private_advice_all.jsonl` — parallel NDJSON (one JSON object per row)
- `EV_Data/edited_private_advice_all.zip` — CSV compressed
- `EV_Data/edited_private_advice_all_jsonl.zip` — NDJSON compressed

**Source URL pattern:** `https://www.ato.gov.au/law/view/document?docid=EV/{Authorisation_Number}`

| Column | Type | Nullable | Description / Format |
|---|---|---|---|
| `Authorisation_Number` | string | No | Primary key. ATO-assigned identifier, e.g. `1-2ABCDEF`. Used to construct `Source_URL`. |
| `Date_of_Advice` | string | Yes | Date the private advice was issued. Scraped verbatim from the HTML label `Date of advice:`. Format is typically `D Month YYYY` (e.g. `1 July 2024`). Blank for older documents that omit this field. |
| `Subject` | string | Yes | Single-sentence topic description of the advice, e.g. `Capital gains tax — main residence exemption`. |
| `Ruling` | string | Yes | All question-and-answer pairs concatenated. Format: `Q1: [text] \| A1: [text] \| Q2: [text] \| A2: [text]`. |
| `Ruling_Period` | string | Yes | Income years to which the ruling applies, pipe-separated. Format: `Year ending 30 June 20XX \| Year ending 30 June 20XX`. |
| `Date_Scheme_Commenced` | string | Yes | Date the relevant arrangement or scheme commenced, if stated. Verbatim from HTML. Blank when not present. |
| `Relevant_Facts_and_Circumstances` | string | Yes | Full text of the facts and circumstances section, collapsed to single-space-separated prose. |
| `Relevant_Legislative_Provisions` | string | Yes | All act/section references concatenated with ` \| `, e.g. `Income Tax Assessment Act 1997 section 104-10 \| Income Tax Assessment Act 1997 section 116-20`. |
| `Reasons_for_Decision` | string | Yes | Summary paragraph plus all detailed reasoning sub-headings. Format: `Summary: [text] \| Detailed Reasoning - [Sub-heading]: [text]`. Sub-heading labels are preserved for parseability. |
| `Is_Archived` | boolean | No | `True` if the document carries an archival disclaimer in its HTML body; `False` otherwise. |
| `Source_URL` | string | No | Full URL of the scraped document, e.g. `https://www.ato.gov.au/law/view/document?docid=EV/1-2ABCDEF`. Stable resume key. |

---

## 2. ATO Interpretative Decisions

**Output files:**
- `EV_Data/ato_interpretative_decisions.csv`
- `EV_Data/ato_interpretative_decisions.jsonl`
- `EV_Data/ato_interpretative_decisions.zip`
- `EV_Data/ato_interpretative_decisions_jsonl.zip`

**Source URL pattern:** `https://www.ato.gov.au/law/view/document?docid=AID/{auth_num}`

| Column | Type | Nullable | Description / Format |
|---|---|---|---|
| `ATO_ID_Number` | string | No | Primary key. ATO ID label as it appears in the document, e.g. `ATO ID 2024/1`. |
| `Status` | string | Yes | Current standing, e.g. `Current`, `Authoritative`, `Withdrawn`. Blank when not present. |
| `Title` | string | Yes | Short descriptive title of the decision. |
| `Issue` | string | Yes | The question or issue being decided, as a prose block. |
| `Decision` | string | Yes | The ATO's ruling on the issue, as a prose block. |
| `Facts` | string | Yes | Factual background underpinning the decision. |
| `Reasons_for_Decision` | string | Yes | Full reasons text. Where sub-headings exist they are preserved with ` \| ` separating each section. |
| `Date_of_Decision` | string | Yes | Date the decision was made. Verbatim from HTML label `Date of decision:`. Typical format: `D Month YYYY` (e.g. `15 March 2023`). |
| `Year_of_Income` | string | Yes | Income year(s) to which the decision applies. Tries both `Year of income` and `Year(s) of income` labels. Typical format: `Year ending 30 June 20XX`. |
| `Legislative_References` | string | Yes | Act and section references, pipe-separated. |
| `Related_Public_Rulings_and_Determinations` | string | Yes | Related ruling/determination identifiers, pipe-separated, e.g. `TR 2023/4 \| TD 2019/7`. |
| `Related_ATO_Interpretative_Decisions` | string | Yes | Cross-referenced ATO ID numbers, pipe-separated. |
| `Subject_References` | string | Yes | Topical subject references, pipe-separated. |
| `Business_Line` | string | Yes | ATO business unit that produced the decision, e.g. `Public Groups and International`. |
| `Is_Archived` | boolean | No | `True` if the document carries an archival disclaimer; `False` otherwise. |
| `Source_URL` | string | No | Full document URL. Stable resume key. |

---

## 3. Public Rulings & Determinations

**Output files** (depends on `--folder` and `--type` flags):

| Flags | Output basename |
|---|---|
| *(default, both folders)* | `public_all` |
| `--folder rulings` | `public_rulings_all` |
| `--folder determinations` | `public_determinations_all` |
| `--type TR` (a ruling code) | `public_rulings_TR` |
| `--type TD` (a determination code) | `public_determinations_TD` |

Four artefacts per run are written to `EV_Data/`: `{basename}.csv`, `{basename}.jsonl`, `{basename}.zip`, and `{basename}_jsonl.zip`.

**Source URL pattern:** `https://www.ato.gov.au/law/view/document?docid={docid}`

**Known type codes:**

| Category | Codes |
|---|---|
| Rulings | `TR`, `GSTR`, `CR`, `PR`, `MT`, `SGR`, `FTR`, `SMSFR`, `ER`, `WTR`, `LCTR`, `MTR`, `TFNR`, `SFR`, `SCR` |
| Determinations | `TD`, `GSTD`, `LCTD`, `FTD`, `SGD`, `SMSFD`, `WTD`, `MTD`, `ED`, `TFND`, `SFD`, `SCD` |

| Column | Type | Nullable | Description / Format |
|---|---|---|---|
| `Document_Reference` | string | No | Primary identifier, e.g. `TR 2023/4`, `TD 2024/1`. Re-parsed from document HTML; falls back to the tree-walk value. |
| `Document_Type` | string | No | Top-level folder: `Ruling` or `Determination`. |
| `Document_Type_Code` | string | No | Short code derived from document title, e.g. `TR`, `GSTR`, `TD`. |
| `Title` | string | Yes | Full document title as it appears in the heading. |
| `Status` | string | Yes | Current standing, e.g. `Current`, `Draft`. Blank when not present. |
| `Date_of_Issue` | string | Yes | Date the ruling/determination was issued. Verbatim from HTML label `Date of issue`. Typical format: `D Month YYYY`. |
| `Date_of_Effect` | string | Yes | Date from which the ruling has effect. May be a prose sentence (e.g. `This Ruling applies from 1 July 2023`) rather than a bare date. |
| `Date_of_Withdrawal` | string | Yes | Date of withdrawal. Blank for current documents. |
| `What_This_Ruling_Is_About` | string | Yes | Introductory section describing the ruling's scope. |
| `Class_of_Entities_or_Scheme` | string | Yes | Description of the taxpayers or schemes to which the ruling applies. |
| `Ruling` | string | Yes | The ruling text (the operative paragraphs). |
| `Examples` | string | Yes | Any worked examples included in the ruling. |
| `Appendix_Explanation` | string | Yes | Content of the first (`Appendix — Explanation`) appendix. |
| `Other_Appendices` | string | Yes | Content of any additional appendices, pipe-separated by appendix heading. |
| `Previous_Rulings` | string | Yes | Identifiers of rulings/determinations this document replaces, pipe-separated. |
| `Related_Rulings` | string | Yes | Identifiers of related rulings/determinations, pipe-separated. |
| `Legislative_References` | string | Yes | Act and section references, pipe-separated. |
| `Case_References` | string | Yes | Case law citations, pipe-separated. |
| `Subject_References` | string | Yes | Topical subject references, pipe-separated. |
| `ATO_References` | string | Yes | ATO internal reference numbers, pipe-separated. |
| `Is_Withdrawn` | boolean | No | `True` if the document has been formally withdrawn; `False` otherwise. |
| `Source_URL` | string | No | Full document URL. Resume key. |

---

## 4. Practical Compliance Guidelines

**Output files:**
- `EV_Data/practical_compliance_guidelines.csv`
- `EV_Data/practical_compliance_guidelines.jsonl`
- `EV_Data/practical_compliance_guidelines.zip`
- `EV_Data/practical_compliance_guidelines_jsonl.zip`

**Source URL pattern:** `https://www.ato.gov.au/law/view/document?docid={docid}`

**Date format:** PCG dates are extracted by a regex that accepts either `D Month YYYY` (e.g. `1 July 2024`) or `YYYY-MM-DD` (e.g. `2024-07-01`). Values are stored verbatim in whichever format appears in the source document.

| Column | Type | Nullable | Description / Format |
|---|---|---|---|
| `PCG_Number` | string | No | Primary identifier, e.g. `PCG 2024/1`, `Draft PCG 2023/D1`. |
| `Document_Type` | string | No | `PCG` or `Draft PCG`. |
| `Title` | string | Yes | Full document title. |
| `Status` | string | Yes | e.g. `Current`, `Draft`. For draft PCGs this defaults to `Draft` when no explicit status field is present. |
| `Date_of_Issue` | string | Yes | Date issued. Format: `D Month YYYY` or `YYYY-MM-DD`. Blank when not in document. |
| `Date_of_Effect` | string | Yes | Date from which the PCG has effect. Same date formats as `Date_of_Issue`. |
| `Date_of_Withdrawal` | string | Yes | Date of withdrawal. Blank for current documents. |
| `Replaces` | string | Yes | Identifier(s) of prior PCG(s) this document supersedes, pipe-separated. |
| `Related_Rulings_and_Determinations` | string | Yes | Related ruling/determination identifiers, pipe-separated. |
| `Legislative_References` | string | Yes | Act and section references, pipe-separated. |
| `Summary` | string | Yes | Summary section text. |
| `Purpose_and_Scope` | string | Yes | Purpose and scope section text. |
| `Background` | string | Yes | Background section text. |
| `Compliance_Approach` | string | Yes | Compliance approach section text (often the main operative content). |
| `Examples` | string | Yes | Worked examples. |
| `Appendices` | string | Yes | Appendix content. |
| `Other_Sections` | string | Yes | Content from any sections not covered by the named columns above, concatenated with ` \| ` between sections. |
| `Compendium_Reference` | string | Yes | Compendium reference if present, e.g. `PCG 2024/1EC`. |
| `Is_Archived` | boolean | No | `True` if the document carries an archival disclaimer; `False` otherwise. |
| `Is_Draft` | boolean | No | `True` when `Document_Type` is `Draft PCG`; `False` otherwise. |
| `Source_URL` | string | No | Full document URL. Resume key. |
