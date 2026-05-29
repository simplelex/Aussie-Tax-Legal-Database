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
| `Document_Type` | string | No | `Final PCG` (published guideline), `Draft PCG` (formal draft), `Compendium` (EC / Early Commentary consultation draft), or `Unknown` (unusual docid format). |
| `Title` | string | Yes | Full document title. |
| `Status` | string | Yes | `Current` (default for all published PCGs), `Draft` (draft and EC consultation docs), or `Withdrawn`. `Current` is inferred when no withdrawal or draft indicators are found in the document text. |
| `Date_of_Issue` | string | Yes | Date issued, extracted from the "Commissioner of Taxation [date]" sign-off paragraph at the end of each PCG. Format: `D Month YYYY`. Blank for EC (Early Commentary) consultation drafts (which have no Commissioner sign-off) and a small number of older PCGs without this footer. |
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
| `Is_Draft` | boolean | No | `True` for `Draft PCG` and `Compendium` (EC / Early Commentary) doc types, and for any `PCG_Number` ending in `EC`; `False` for finalized PCGs. |
| `Source_URL` | string | No | Full document URL. Resume key. |

---

## 5. Taxpayer Alerts

**Output files:**
- `EV_Data/taxpayer_alerts_all.csv`
- `EV_Data/taxpayer_alerts_all.jsonl`
- `EV_Data/taxpayer_alerts_all.zip`
- `EV_Data/taxpayer_alerts_all_jsonl.zip`

**Source URL pattern:** `https://www.ato.gov.au/law/view/document?docid=TPA/TA{year}{num}/NAT/ATO/00001`

| Column | Type | Nullable | Description / Format |
|---|---|---|---|
| `Alert_Number` | string | No | Primary identifier, e.g. `TA 2026/1`. Parsed from H2 on the document page. |
| `Title` | string | Yes | Full descriptive title of the alert, e.g. `Contrived property development arrangements…`. |
| `Date_of_Issue` | string | Yes | Date the alert was issued. Verbatim from page metadata. Typical format: `D Month YYYY`. Blank when not present. |
| `Status` | string | Yes | `Current` or `Withdrawn`. Derived from page header text. |
| `Overview` | string | Yes | Paragraphs under the "Overview" heading, joined with ` \| `. |
| `Description` | string | Yes | Paragraphs under the "Description" heading, joined with ` \| `. |
| `Example` | string | Yes | Paragraphs under the "Example" heading, joined with ` \| `. Blank when no example is included. |
| `Our_Concerns` | string | Yes | Paragraphs under the "Our concerns" heading. |
| `What_ATO_Is_Doing` | string | Yes | Paragraphs under the "What we are doing" heading. |
| `What_You_Should_Do` | string | Yes | Paragraphs under the "What you should do" heading. |
| `Related_Documents` | string | Yes | Pipe-separated document and legislative references from inline links in the article body (e.g. `PS LA 2008/15 \| ITAA 1997 Div 70`). Nav links and self-references excluded. |
| `Is_Withdrawn` | boolean | No | `True` if the alert appears in the "Withdrawn" sub-folder of the tree; `False` otherwise. |
| `Source_URL` | string | No | Full document URL. Resume key. |

---

## 6. Decision Impact Statements

**Output files:**
- `EV_Data/decision_impact_statements_all.csv`
- `EV_Data/decision_impact_statements_all.jsonl`
- `EV_Data/decision_impact_statements_all.zip`
- `EV_Data/decision_impact_statements_all_jsonl.zip`

**Source URL pattern:** `https://www.ato.gov.au/law/view/document?docid=LIT/ICD/{court_ref}/00001`

| Column | Type | Nullable | Description / Format |
|---|---|---|---|
| `Case_Name` | string | No | Full case citation, e.g. `Commissioner of Taxation v Bendel [2025] FCAFC 15`. Parsed from H2. |
| `Venue_Reference_No` | string | Yes | Court file number, e.g. `VID 903 of 2023`. Parsed from `Venue Reference No:` label. |
| `Venue` | string | Yes | Court or tribunal name, e.g. `Full Federal Court`. Parsed from `Venue:` label. |
| `Judgment_Date` | string | Yes | Date of the judgment. Format: `D Month YYYY`. |
| `Date_Published` | string | Yes | Date the DIS was published by the ATO. Parsed from the page title parenthetical `(Published DD Month YYYY)`. |
| `Document_Type` | string | No | `Decision Impact Statement` or `Interim Decision Impact Statement`. |
| `Summary_of_Decision` | string | Yes | Paragraphs under the "Summary of decision" heading, joined with ` \| `. |
| `Overview_of_Facts` | string | Yes | Paragraphs under the "Overview of facts" heading, joined with ` \| `. |
| `Issues_Decided` | string | Yes | All issue sub-sections concatenated: `Issue 1 – [topic]: [text] \| Issue 2 – [topic]: [text]`. |
| `ATO_View_of_Decision` | string | Yes | The ATO's response — whether it accepts, appeals, or will update guidance. This is the critical field. |
| `Administrative_Treatment` | string | Yes | Paragraphs under the "Administrative treatment" heading. |
| `Related_Documents` | string | Yes | Pipe-separated document and legislative references from inline links (e.g. `TD 2022/11 \| PCG 2022/2`). |
| `Is_Interim` | boolean | No | `True` if H1 is "Interim Decision Impact Statement"; `False` otherwise. |
| `Source_URL` | string | No | Full document URL. Resume key. |

---

## 7. Law Administration Practice Statements

**Output files:**
- `EV_Data/law_admin_practice_statements_all.csv`
- `EV_Data/law_admin_practice_statements_all.jsonl`
- `EV_Data/law_admin_practice_statements_all.zip`
- `EV_Data/law_admin_practice_statements_all_jsonl.zip`

**Source URL pattern:** `https://www.ato.gov.au/law/view/document?docid=PSR/PS{year}{num}/NAT/ATO/00001` (regular); `DPS/DPSLA{year}{num}/NAT/ATO/00001` (draft)

| Column | Type | Nullable | Description / Format |
|---|---|---|---|
| `Statement_Reference` | string | No | Primary identifier, e.g. `PS LA 2026/1`. Parsed from H2. |
| `Title` | string | Yes | Subject line of the statement, e.g. `Self-managed superannuation funds — education directions…`. |
| `Date_of_Issue` | string | Yes | Date issued. Format: `D Month YYYY`. Parsed from `Date of Issue:` metadata block. |
| `Date_of_Effect` | string | Yes | Date from which the practice statement has effect. Same format as `Date_of_Issue`. |
| `Document_Type` | string | No | `Law Administration Practice Statement`, `Draft Law Administration Practice Statement`, or `Law Administration Practice Statement (GA)`. |
| `Has_Compendium` | boolean | No | `True` if the page references a compendium (e.g. `PS LA 2026/1EC`); `False` otherwise. |
| `Body` | string | Yes | All numbered sections concatenated: `1. [Heading]: [text] \| 2. [Heading]: [text] \| … \| Appendix: [text]`. |
| `Related_Documents` | string | Yes | Pipe-separated document and legislative references from inline links. |
| `Legislative_References` | string | Yes | Legislative references from the `Legislative References:` metadata block at end of page, pipe-separated. |
| `Is_Draft` | boolean | No | `True` when `Document_Type` starts with `Draft`; `False` otherwise. |
| `Is_Withdrawn` | boolean | No | `True` if page header indicates the statement has been withdrawn; `False` otherwise. |
| `Source_URL` | string | No | Full document URL. Resume key. |
