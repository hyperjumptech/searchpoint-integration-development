# Tools

All tools are called with JSON-RPC `tools/call`:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_file",
    "arguments": { "search": "q3 report" }
  }
}
```

`tools/list` returns the catalog with each tool's JSON Schema. Every tool rejects unknown parameters, so never send a key that is not documented here.

## General rules

- The owner is implicit: the server acts for the signed-in SearchPoint user. **Never send `owner_email`.**
- Start minimal. `{ "search": "q3" }` is a valid and often the best first call. Add filters only when the user asks for them.
- Do not guess filter values. Omit `source`, `kind`, `author`, or dates the user did not mention.
- Never send `type: ["*"]` or `source: "unknown"`. Omit the parameter to mean "all".
- IDs come from `search_file` (`value[]._source.id`), `list_favorites` (`fileId`), or `list_files_by_tag`. Pass them to `get_file`, the favorite tools, and the tag tools.
- All tools return JSON. Tool failures return a normal result with `isError: true` and a short explanation.

## Search

### `search_file`

Full-text search over the user's documents.

| Parameter | Type | Notes |
| --- | --- | --- |
| `search` | string, min 1 | **Required.** Full-text query. |
| `type` | string[] | File extensions without the dot, e.g. `["pdf"]` or `["docx","xlsx"]`. Omit for all types. |
| `kind` | `"video"`, `"document"`, `"image"` | Only if the user names a kind. |
| `author` | string | Name or email, e.g. `"satriyo"` or `"satriyo@hyperjump.tech"`. |
| `updated_after` | string | Lower bound on `updated_at`. ISO 8601 (`2023-01-01T00:00:00Z`) or `YYYY-MM-DD`. |
| `updated_before` | string | Upper bound on `updated_at`. Same formats. |
| `source` | enum | `"shared folder"`, `"apple notes"`, `"onedrive"`, `"onedrive-business"`, `"onedrive-personal"`, `"sharepoint"`, `"teams"`, `"unknown"`. Never use `"unknown"`. |
| `mode` | `"broad"` (default), `"exact"` | `broad` is typo-tolerant. `exact` for precise phrases, IDs, or codes. |
| `page` | integer ≥ 1 | 1-based. |
| `limit` | integer 1–50 | Default 10. |

Response: hits under `value[]._source`, pagination under `meta.pagination`. Use `_source.id` with `get_file`.

```json
{ "search": "invoice", "type": ["pdf"] }
```

```json
{ "search": "Q3 report", "type": ["pptx"], "source": "onedrive-business", "updated_after": "2026-07-01T00:00:00Z" }
```

```json
{ "search": "INV-2026-00123", "mode": "exact" }
```

Search strategy:

1. Start with just `search`.
2. Too many results? Add the filters the user mentioned.
3. Too few? Switch to `broad`, drop filters, or simplify the query.
4. Present the top hits (name, source, updated date), then offer `get_file` on the best candidate.

### `get_file`

Full metadata for one document. Returns a not-found error if the file does not exist or the user lacks access.

```json
{ "id": "<_source.id from search_file>" }
```

Typical flow: `search_file` → pick a hit → `get_file` with `_source.id`.

## Local sources

Desktop only. These manage which local folders are indexed.

### `list_local_sources`

No parameters. Returns `{"paths": ["/absolute/one", ...]}`.

### `add_local_source`

Registers a folder and starts indexing, the same as adding a folder in the app. The path must be **absolute**; `~` and relative paths are rejected. Returns `{"path": "/absolute/path"}`.

```json
{ "path": "/Users/alex/Documents/Contracts" }
```

### `remove_local_source`

Removes the folder and **deletes its indexed documents locally and on the backend**. Files on disk are untouched. Returns `{"path": "..."}`. Confirm with the user before calling.

```json
{ "path": "/Users/alex/Documents/Contracts" }
```

Call `list_local_sources` first to avoid duplicates and to get exact paths. Indexing after `add_local_source` is asynchronous, so results may not appear immediately.

## Favorites

Desktop only. IDs come from `search_file` or `get_file`.

### `add_favorite`

Idempotent: succeeds without writing if already a favorite. Returns `{"id": "<file_uuid>", "favorited": true}`.

```json
{ "id": "abc123" }
```

### `remove_favorite`

Idempotent: succeeds without writing if not a favorite. Returns `{"id": "<file_uuid>", "favorited": false}`.

```json
{ "id": "abc123" }
```

### `list_favorites`

Newest first. Optional `page` (≥ 1, default 1) and `limit` (≥ 1, default 10).

```json
{ "page": 2, "limit": 25 }
```

```json
{
  "items": [
    {
      "fileId": "",
      "fileName": "",
      "filePath": "",
      "fileUrl": "",
      "fileExtension": "",
      "fileSource": "",
      "favoritedAt": ""
    }
  ],
  "page": 1,
  "limit": 10,
  "total": 0
}
```

`total` is the full count, so use it to decide whether to fetch more pages.

## Tags

Desktop only. IDs come from `search_file` or `get_file`. Tag names are case-insensitive.

### `list_file_tags`

The tags on one file. Returns `{"id": "<file_uuid>", "tags": ["research", "q3"]}`. An empty array means the file has no tags.

```json
{ "id": "abc123" }
```

### `list_tags`

Every tag in the workspace with how many files carry it, sorted by name. Returns `{"items": [{"name": "research", "count": 42}]}`.

### `list_files_by_tag`

The files carrying one tag, most recently tagged first. This is how you answer "what files are tagged X". **Do not** pass the tag name to `search_file`; that matches file names and extracted keywords, not tags.

| Parameter | Type | Notes |
| --- | --- | --- |
| `tag` | string | **Required.** Case-insensitive. |
| `search` | string | Optional. Matches the file name and path only, case-insensitively, by substring. Splits on whitespace and every term must match. Omit it to list every file carrying the tag; `""` is rejected. |
| `page` | integer ≥ 1 | 1-based, default 1. |
| `limit` | integer 1–50 | Default 10. |

Returns the same shape as `search_file`, so `value[]._source.id` feeds straight into `get_file` and the id-taking tools.

```json
{ "tag": "research" }
```

```json
{ "tag": "OneAI", "search": "Feasibility", "limit": 10 }
```

A tag no file carries returns an empty `value` array and `total` 0 — not an error. Prefer `search` over paging: listing page 1 of a large tag and concluding a file is absent gives a wrong answer.

### `append_tag`

Adds tags, creating names that do not exist yet. `tags` is a non-empty array. Idempotent per tag; either every tag is written or none is. Returns `{"id": "<file_uuid>", "tags": ["research", "q3"]}`.

```json
{ "id": "abc123", "tags": ["research", "q3"] }
```

### `remove_tag`

Removes tags. `tags` is a non-empty array. Idempotent per tag. **Removing the last file carrying a tag deletes that tag name entirely.** Returns `{"id": "<file_uuid>", "tags": ["research", "q3"]}`.

```json
{ "id": "abc123", "tags": ["q3"] }
```

Call `list_file_tags` before `remove_tag` to see what is actually there, and `list_tags` when the user asks what tags exist.

## Document facts

Desktop only, and only when **Smart search** is on. SearchPoint reads the documents on the user's computer (agreements, invoices, quotations, proposals, and more) and records facts from them: document types, parties, people, dates, amounts, and choices, each with the line it was printed in.

When Smart search is off, no one is signed in, or a file has not been read yet, these tools return an error explaining why. `search_file` keeps working.

### `get_document_facts_guide`

No parameters. Read it first, once per session. Returns Markdown generated from what has been read so far: document types and counts, the facts each type holds, the words that search for them, example searches, SPARQL patterns to copy, and how far each fact can be trusted.

### `search_document_facts`

Find documents by what they say, not by file name or keyword. Use it for questions about a document's type, parties, signers, dates, and amounts, such as "agreement 2025", "invoice above 100 juta", "NDA with PT Contoh Abadi", or "quotation before March 2026".

| Parameter | Type | Notes |
| --- | --- | --- |
| `search` | string, min 1, max 500 | **Required.** Short search built from the words the guide lists: a document type, a fact name such as "due date", a name, a date, or an amount, joined by comparison words such as `above`, `before`, `since`, or `between`. English and Indonesian both work. |
| `page` | integer ≥ 1 | 1-based, default 1. |
| `limit` | integer 1–50 | Default 10. |

Every word must be understood. When one is not, nothing is returned and `not_understood` names the word and why, so search again without it. For file names, keywords, or cloud files, use `search_file`.

Returns:

```json
{
  "status": "ok",
  "value": [
    {
      "version": "1",
      "_source": {},
      "match": {
        "why": "one line",
        "reasons": [],
        "quote": "the printed line",
        "highlights": [],
        "draft": false,
        "entity_id": "..."
      }
    }
  ],
  "meta": { "pagination": {} },
  "understood": [],
  "not_understood": [],
  "sparql": "..."
}
```

`_source` has the same shape as `search_file`, so `_source.id` works with `get_file`, tags, and favorites. `draft` is true when the document still prints placeholders; drafts are listed last.

### `get_document_facts`

Every fact SearchPoint read about one thing: a document, an organization, a person, or a schema term such as `sp:Invoice`.

```json
{ "id": "spd:organization-contoh-abadi" }
```

Pass an `entity_id` from `search_document_facts` or `query_document_facts`, or a file id (`_source.id` from `search_file`, `list_favorites`, or `list_files_by_tag`) to get the document read from that file.

Each fact gives its property, its value, how it is known (printed in a document, inferred by a rule, or part of the schema), and its evidence: the quote it was printed in, the file it came from, the reader, and a confidence from 0 to 1. Quote the evidence when you tell the user a fact.

Returns `{status, value{id, name, types[], also_known_as[], facts[], mentioned_by[], mentioned_by_total, files[], merged_from[], merged_because[], problems[]}}`.

### `query_document_facts`

Read-only SPARQL over the document facts, for what `search_document_facts` cannot ask: totals, counts, grouping, every party of every agreement, or facts across many documents.

| Parameter | Type | Notes |
| --- | --- | --- |
| `sparql` | string, min 1, max 10000 | **Required.** The query. |
| `inferred` | boolean | `true` (default) includes facts the schema's rules infer, such as every NDA also being an Agreement. `false` gives only what was read from documents. |

Supports `SELECT` and `ASK` with `OPTIONAL`, `FILTER`, `GRAPH`, `GROUP BY`, `ORDER BY`, `LIMIT`, and `OFFSET`. `UNION`, `MINUS`, `NOT EXISTS`, `VALUES`, `BIND`, subqueries, and property paths are not supported. The prefixes `sp:`, `spd:`, `spx:`, `rdf:`, `rdfs:`, `owl:`, `xsd:`, `skos:`, `sh:`, `prov:`, and `schema:` are predeclared.

```json
{ "sparql": "SELECT ?invoice WHERE { ?invoice a sp:Invoice } LIMIT 5" }
```

Returns W3C SPARQL 1.1 Results JSON with at most 200 rows and a `truncated` flag when there were more. Each IRI value also carries `show` (its prefixed form) and `label`; each document IRI carries `file_id` for `get_file`.

## Common workflows

**Find and favorite a file**

1. `search_file` `{ "search": "annual report", "type": ["pdf"] }`
2. `add_favorite` `{ "id": "<_source.id of chosen hit>" }`

**Index a new folder, then search it**

1. `list_local_sources` `{}`
2. `add_local_source` `{ "path": "/Users/alex/Projects" }`
3. Wait for indexing, then `search_file` `{ "search": "readme" }`

**Review favorites, then get details**

1. `list_favorites` `{ "limit": 20 }`
2. `get_file` `{ "id": "<fileId>" }`

**Tag a file, then confirm**

1. `search_file` `{ "search": "invoice" }`
2. `append_tag` `{ "id": "<_source.id>", "tags": ["invoice", "q3"] }`
3. `list_file_tags` `{ "id": "<same id>" }`

**Answer "what files are tagged X"**

1. `list_tags` `{}` to confirm the exact name and count
2. `list_files_by_tag` `{ "tag": "<tag name>" }`
3. `get_file` per hit when full metadata is needed

**Answer "what do my contracts with PT Contoh Abadi say"**

1. `get_document_facts_guide` `{}` once per session
2. `search_document_facts` `{ "search": "agreement PT Contoh Abadi" }`
3. `get_document_facts` `{ "id": "<match.entity_id>" }`
