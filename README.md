# Medallion Pipeline for Unstructured Documents: Extraction, Data Quality, and Vector Search

A proof of concept in Microsoft Fabric that turns PDF documents into searchable chunks, with data quality checks and file-to-chunk lineage at every step.

> **Scope:** proof of concept on a small set of public, open-access medical case reports. It demonstrates the pattern. It is not a production system.

## Architecture

```
PDF files (Files/bronze)             immutable raw input
        |
   01 Bronze manifest                sha256 per file: identity and lineage (Delta)
        |
   02 Silver pages                   one row per page: native text, OCR fallback, quality flags (Delta)
        |
   03 Silver chunks                  ~200-word overlapping chunks per page (Delta)
        |
   04 Embeddings + Eventhouse        vectors, cosine-similarity search (KQL)
```

## What it demonstrates

- **Bronze is immutable.** Files are never edited. A manifest records each file's SHA-256, path, size, load time, and batch.
- **Hash-based lineage.** `doc_id` equals the file hash, so any page or chunk traces back to its source file and load.
- **Routing, not guessing.** Pages with a text layer use native extraction. Near-empty pages go to OCR when configured, and OCR confidence is recorded.
- **Data quality as data.** Problems are written to `dq_doc_pages` (`EMPTY_PAGE`, `LOW_OCR_CONFIDENCE`, `OCR_ERROR`, `DUPLICATE_FILE`) and not hidden.
- **Re-runnable notebooks.** Manifest and extraction skip what is already loaded. Chunks are rebuilt deterministically.
- **Search by meaning.** The query "high blood sugar emergency" returns chunks about hyperglycemia, which keyword search would miss.

## Run order

| Step | Notebook | Output |
|---|---|---|
| 1 | `notebooks/01_bronze_manifest.ipynb` | `bronze_doc_manifest` |
| 2 | `notebooks/02_silver_extract_pages.ipynb` | `silver_doc_pages`, `dq_doc_pages` |
| 3 | `notebooks/03_silver_chunking.ipynb` | `silver_doc_chunks` |
| 4 | `eventhouse/setup.kql` (command 1), then `notebooks/04_embeddings_and_vector_search.ipynb`, then `setup.kql` (commands 2 to 3) | `DocChunksVec` |

## Setup

1. Create a Fabric lakehouse and attach it to each notebook as the default lakehouse.
2. Put your own PDFs in `Files/bronze/unstructureddata`. **No documents are included in this repository.**
3. Optional OCR: create an Azure Document Intelligence resource and set `DOCINTEL_ENDPOINT` and `DOCINTEL_KEY` as environment variables or from a secret store. Without them, OCR is skipped and near-empty pages are flagged.
4. Create an Eventhouse and KQL database, then edit `KUSTO_URI` and `KUSTO_DB` in notebook 04.
5. Provide an embedding API key at run time when prompted. Never commit keys.

## Results

On 7 sample articles (163 chunks, 1536-dimension vectors), retrieval checks matched the expected content for questions worded differently from the text. Abbreviations (for example `T1`/`T2` instead of "type 1/2") needed to be known by the tests.

## Limitations

- Files are loaded by hand. There is no connector, scheduler, or incremental source.
- No Gold layer, access control per document, or answer generation.
- Small test set: 7 documents and 7 retrieval checks.
- Embeddings use a public API for practice. A company deployment needs an approved, in-boundary service.
- Extraction noise (headers, footers, publisher statements) is not cleaned yet.

## What I would add for production

- Source connectors (SharePoint, document management systems) with incremental loads and watermarks
- Scheduled orchestration with retries and alerting
- Secrets in a key vault and managed identities
- Text cleaning rules and document type handling in Silver
- Per-document permissions carried into search
- A larger evaluation set and monitoring of quality and cost
- Audit and validation documentation for regulated use

## Security

No keys, endpoints, workspace identifiers, or documents are stored in this repository. See `.gitignore`.
