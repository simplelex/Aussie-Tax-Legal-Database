# Australian Taxation Office Legal Database — Machine-Readable

**60,000+ ATO rulings, determinations, and private advice. Structured CSV. RAG-ready on day one.**

The ATO Legal Database spans 15 years of Australian tax law — private advice, interpretative decisions, public rulings, and compliance guidelines. It's all public. It's all messy HTML. Getting it into a format an LLM can actually use takes weeks of data engineering.

This dataset skips that. Every document is one CSV row. Every logical section is its own column. Source URLs included.

---

> **This repository contains the full structured dataset and data dictionary.**
> Full column reference is in [`data_dictionary.md`](data_dictionary.md).
> If this dataset saves you time, consider supporting the project:
>
> **[☕ Buy Me a Coffee](https://buymeacoffee.com/simplelex)**

---

## Table of contents

- [Direct download links](#direct-download-links)
- [What's in the dataset](#whats-in-the-dataset)
- [Why this is RAG-ready out of the box](#why-this-is-rag-ready-out-of-the-box)
- [Schema highlights](#schema-highlights)
- [Quickstart](#quickstart)
  - [RAG chunking with LangChain](#rag-chunking-with-langchain)
  - [Filter query example](#filter-query-example-pinecone--weaviate--chroma)
  - [Recommended metadata fields](#recommended-metadata-fields-to-index-in-your-vector-store)
- [Recommended chunking strategy](#recommended-chunking-strategy)
- [Who this is for](#who-this-is-for)
- [Support the project](#support-the-project)
- [Data source](#data-source)

---

## Direct download links

- [edited_private_advice_all.zip](../../releases/download/latest-data/edited_private_advice_all.zip)
- [ato_interpretative_decisions.zip](../../releases/download/latest-data/ato_interpretative_decisions.zip)
- [public_all.zip](../../releases/download/latest-data/public_all.zip)
- [practical_compliance_guidelines.zip](../../releases/download/latest-data/practical_compliance_guidelines.zip)

---

## What's in the dataset

| Dataset | Document Type | Est. Documents | Coverage |
|---|---|---|---|
| Edited Private Advice | Private rulings issued to individual taxpayers | ~50,000 | 2011–present |
| ATO Interpretative Decisions | Binding ATO ID decisions | ~10,000 | All years |
| Public Rulings & Determinations | TR, TD, GSTR, CR, GSTD + 22 other type codes | ~2,000 | All years |
| Practical Compliance Guidelines | PCG, Draft PCG | ~200 | All years |

**Total size:** ~400–500 MB uncompressed across all four datasets.
**Token estimate:** ~80–120 million tokens (at ~1,500–2,000 tokens/document average).

Each zip contains a single UTF-8 CSV file. Files are updated automatically on weekdays when new ATO documents are detected.

---

## Why this is RAG-ready out of the box

**One row = one complete document.** No reassembly, no PDF parsing, no HTML cleanup.

**Section-level columns.** Each logical section (Facts, Ruling, Reasons, Legislative References) is its own column — map directly to RAG chunks without regex.

**Pipe-delimited sub-sections.** Multi-part fields use ` | ` as a consistent separator (e.g. `Q1: [text] | A1: [text]`). Fine-grained chunking is trivial.

**Source URLs included.** Every row links back to the live ATO page. Citations are built in — no hallucination risk on provenance.

**Status flags.** `Is_Archived`, `Is_Withdrawn`, `Is_Draft` let you filter to current authoritative documents in one line. Critical for legal RAG accuracy.

**Structured metadata.** Date fields, type codes, legislative references, and subject references are all separate columns — use them as vector database metadata filters.

---

## Schema highlights

Full column reference with types, nullability, and format notes: [`data_dictionary.md`](data_dictionary.md)

**Edited Private Advice** — 11 columns
`Authorisation_Number` · `Date_of_Advice` · `Subject` · `Ruling` · `Ruling_Period` · `Date_Scheme_Commenced` · `Relevant_Facts_and_Circumstances` · `Relevant_Legislative_Provisions` · `Reasons_for_Decision` · `Is_Archived` · `Source_URL`

**ATO Interpretative Decisions** — 16 columns
`ATO_ID_Number` · `Status` · `Title` · `Issue` · `Decision` · `Facts` · `Reasons_for_Decision` · `Date_of_Decision` · `Year_of_Income` · `Legislative_References` · `Related_Public_Rulings_and_Determinations` · `Related_ATO_Interpretative_Decisions` · `Subject_References` · `Business_Line` · `Is_Archived` · `Source_URL`

**Public Rulings & Determinations** — 22 columns
`Document_Reference` · `Document_Type` · `Document_Type_Code` · `Title` · `Status` · `Date_of_Issue` · `Date_of_Effect` · `Date_of_Withdrawal` · `What_This_Ruling_Is_About` · `Class_of_Entities_or_Scheme` · `Ruling` · `Examples` · `Appendix_Explanation` · `Other_Appendices` · `Previous_Rulings` · `Related_Rulings` · `Legislative_References` · `Case_References` · `Subject_References` · `ATO_References` · `Is_Withdrawn` · `Source_URL`

**Practical Compliance Guidelines** — 21 columns
`PCG_Number` · `Document_Type` · `Title` · `Status` · `Date_of_Issue` · `Date_of_Effect` · `Date_of_Withdrawal` · `Replaces` · `Related_Rulings_and_Determinations` · `Legislative_References` · `Summary` · `Purpose_and_Scope` · `Background` · `Compliance_Approach` · `Examples` · `Appendices` · `Other_Sections` · `Compendium_Reference` · `Is_Archived` · `Is_Draft` · `Source_URL`

---

## Quickstart

```python
import pandas as pd

# Load a dataset
df = pd.read_csv("edited_private_advice_all.csv")

# Filter to current documents only
active = df[df["Is_Archived"] == False]

# Filter by topic
cgt = active[active["Subject"].str.contains("CGT|capital gains", case=False, na=False)]

print(f"{len(active):,} active documents, {len(cgt):,} CGT-related")
```

### RAG chunking with LangChain

```python
from langchain.schema import Document

def row_to_chunks(row):
    chunks = []
    metadata = {
        "source": row["Source_URL"],
        "authorisation_number": row["Authorisation_Number"],
        "date": row["Date_of_Advice"],
        "subject": row["Subject"],
        "is_archived": row["Is_Archived"],
    }
    # Ruling is the primary retrieval chunk
    if row["Ruling"]:
        chunks.append(Document(page_content=row["Ruling"], metadata={**metadata, "section": "ruling"}))
    # Reasons are the secondary chunk
    if row["Reasons_for_Decision"]:
        chunks.append(Document(page_content=row["Reasons_for_Decision"], metadata={**metadata, "section": "reasons"}))
    return chunks

active_docs = []
for _, row in active.iterrows():
    active_docs.extend(row_to_chunks(row))
```

### Filter query example (Pinecone / Weaviate / Chroma)

Find current Tax Rulings about CGT main residence exemption issued after 2015:

```
filter: Document_Type_Code = "TR"
    AND Is_Withdrawn = False
    AND Date_of_Issue >= "2015"
semantic search on: Ruling + What_This_Ruling_Is_About
```

### Recommended metadata fields to index in your vector store

`Document_Type_Code` · `Status` · `Date_of_Issue` · `Year_of_Income` · `Business_Line` · `Is_Archived` · `Is_Withdrawn`

---

## Recommended chunking strategy

| Column(s) | Chunk type | Suggested use |
|---|---|---|
| `Subject` / `Title` | Document title | Metadata / retrieval filter |
| `Ruling` | Operative text | Primary retrieval chunk |
| `Reasons_for_Decision` | Reasoning chain | Secondary retrieval chunk |
| `Relevant_Facts_and_Circumstances` / `Facts` | Context | Supporting chunk |
| `Legislative_References` | Structured references | Metadata filter |
| `Source_URL` | Citation | Always attach to answer |
| `Is_Archived` / `Is_Withdrawn` | Freshness flag | Pre-filter before search |

Sub-sections within a field are delimited by ` | ` — split on this to create paragraph-level chunks without regex.

---

## Who this is for

| User | Use case |
|---|---|
| LegalTech / TaxTech startups | Building Australian tax AI without months of data engineering |
| Big 4 / mid-tier accounting firms | Feeding an internal AI the full history of ATO rulings |
| Law schools & tax researchers | The first structured, citable corpus of ATO private and public rulings |
| LLM fine-tuning teams | ~100M tokens of expert Australian tax reasoning, already cleaned |
| Government / policy consultants | Bulk analysis of ATO trends across 15 years by topic, legislation, or date |

---

## Support the project

This dataset is free. If it saves you time, a coffee is appreciated:

**[☕ Buy Me a Coffee](https://buymeacoffee.com/simplelex)**

---

## Data source

All documents are sourced from the [ATO Legal Database](https://www.ato.gov.au/law/view/) (ato.gov.au). Data is publicly available under ATO copyright. This dataset is a structured, machine-readable representation of that public information.
