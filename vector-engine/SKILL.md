---
name: vector-engine
description: Use when writing code in an app that does semantic search, RAG, or ingests documents into a vector database on the platform — running nearest-neighbor queries, listing/adding/removing documents, or feeding retrieved chunks into an LLM. Covers VectorDatabaseQuery(), ListDocumentsInVectorDatabase(), CreateEmbeddingsFromDocuments(), CreateEmbeddingsFromVectorCSVFile(), and RemoveDocumentFromVectorDatabase() pixel commands via @semoss/sdk's runPixel, plus listing engines with MyEngines(engineTypes=["VECTOR"]). Do not use for raw SQL/graph queries (see database-engine) or direct LLM calls without retrieval (see model-engine).
---

# Vector Engine

Query and manage a vector database on the platform using `runPixel` from `@semoss/sdk` with the `VectorDatabaseQuery()` pixel command as the primary entry point. Results are typically passed into an `LLM()` call to implement RAG — see the [RAG pattern](#rag-pattern) section below.

## Usage

```typescript
import { runPixel } from "@semoss/sdk";

const VECTOR_ID = "1222b449-1bc6-4358-9398-1ed828e4f26a";
const query = "Time sheet";

const { errors, pixelReturn } = await runPixel(
  `VectorDatabaseQuery(engine="${VECTOR_ID}", command="<encode>${query}</encode>", limit=5, filters=[], metaFilters=[]);`,
);

if (errors.length) throw new Error(errors[0]);

const hits = pixelReturn[0].output as Array<{
  Score: number;
  idx: number;
  Source: string;
  Modality: string;
  Divider: string;
  Part: string;
  Tokens: number;
  Content: string;
  Weighted_RRF_Score: number;
  BM25_Score: number;
}>;
```

Each hit carries both the text chunk (`Content`) and the source document (`Source`), plus hybrid-search scoring fields. Lower `Score` is closer; `Weighted_RRF_Score` and `BM25_Score` are the underlying component scores from the hybrid rank fusion.

The variations below show only the pixel string — the one that goes inside the `runPixel` template literal. The surrounding `runPixel(...)` call, the `errors` check, and the response parsing are the same as above.

### Filtering by source document

`filters` is an array of `Filter()` expressions matched against the chunk's `Source`. Use this to restrict retrieval to a named subset of documents.

```
VectorDatabaseQuery(engine="${VECTOR_ID}", command="<encode>${query}</encode>", limit=5, filters=[Filter(Source == ["doc1.pdf", "doc2.pdf"])]);
```

### Filtering by metadata

`metaFilters` is an array of `Filter()` expressions matched against arbitrary metadata keys attached to chunks at embed time.

```
VectorDatabaseQuery(engine="${VECTOR_ID}", command="<encode>${query}</encode>", limit=5, metaFilters=[Filter(department == "finance")]);
```

Both `filters` and `metaFilters` can be combined in the same call.

### Listing documents in the vector index

Returns one entry per unique `Source` currently indexed.

```
ListDocumentsInVectorDatabase(engine="${VECTOR_ID}");
```

Response shape (at `pixelReturn[0].output`):

```typescript
Array<{ fileName: string; fileSize: number; lastModified: string }>;
```

### Adding documents from the current insight

Use when the files were uploaded into the current insight/room workspace. `CreateEmbeddingsFromDocuments` handles the chunking, embedding, and indexing.

```
CreateEmbeddingsFromDocuments(engine="${VECTOR_ID}", filePaths=["fileName1.pdf", "fileName2.pdf"]);
```

### Adding documents from app or user space

When the files live outside the current insight — in a project/app folder or the user's personal space — pass `space`. Use `space="app_id"` where `app_id` is the project/app UUID, or `space="user"` for the user space.

```
CreateEmbeddingsFromDocuments(engine="${VECTOR_ID}", filePaths=["docs/file1.pdf"], space="app_id");
```

### Adding pre-chunked CSV data

Use `CreateEmbeddingsFromVectorCSVFile` when you have already chunked the source material and want to skip the platform's default splitter. Supported inputs: `.csv` files, or `.zip` archives containing CSV files.

The CSV must have exactly these headers, case-sensitive:

```
Source,Modality,Divider,Part,Tokens,Content
doc1.pdf,text,1,0,120,"First chunk of text"
```

Quotes on values are optional unless a value contains commas, newlines, or quotes.

```
CreateEmbeddingsFromVectorCSVFile(engine="${VECTOR_ID}", filePaths=["fileName1.csv", "fileName2.csv"]);
```

Use the same `space` argument as `CreateEmbeddingsFromDocuments` when the CSVs live in app or user space:

```
CreateEmbeddingsFromVectorCSVFile(engine="${VECTOR_ID}", filePaths=["vector_data/chunks.csv"], space="app_id");
```

### Removing documents from the vector index

Pass `fileNames` — the source identifiers as listed by `ListDocumentsInVectorDatabase`, not file paths.

```
RemoveDocumentFromVectorDatabase(engine="${VECTOR_ID}", fileNames=["fileName1.pdf", "fileName2.pdf"]);
```

## Response shape

Unlike `SqlQuery` (which returns a tabular `{ data: { values, headers } }` object), vector pixels return their payload directly at `pixelReturn[0].output`:

- `VectorDatabaseQuery` → array of hit objects with `Score`, `Source`, `Content`, etc.
- `ListDocumentsInVectorDatabase` → array of `{ fileName, fileSize, lastModified }`.
- `CreateEmbeddingsFromDocuments` / `CreateEmbeddingsFromVectorCSVFile` → string status (`"Successfully embedded all files"`) plus a structured `additionalOutput[]` with per-file `{ fileName, status, insertedRecords, failedRecords, totalRecords }`.
- `RemoveDocumentFromVectorDatabase` → status payload confirming the removal.

For the full response schema of each pixel, see `references/response-schema.md`.

## RAG pattern

The typical flow is a two-step chain: retrieve chunks with `VectorDatabaseQuery`, then feed them into an `LLM()` call as grounding context. See the `model-engine` skill for the LLM half.

```typescript
import { runPixel } from "@semoss/sdk";

const VECTOR_ID = "1222b449-1bc6-4358-9398-1ed828e4f26a";
const MODEL_ID = "6dd0bbfd-cd3b-4f2c-b13a-fe4545872e3d";
const question = "Time sheet";

const { pixelReturn: retrievalReturn } = await runPixel(
  `VectorDatabaseQuery(engine="${VECTOR_ID}", command="<encode>${question}</encode>", limit=3);`,
);

const hits = retrievalReturn[0].output as Array<{ Source: string; Content: string }>;
const context = hits
  .map((h) => `* Document Name: ${h.Source}, ${h.Content}`)
  .join("\n");

const prompt = `Answer the question using only the context below.\n\nQuestion: ${question}\n\nContext:\n\`\`\`\n${context}\n\`\`\``;

const { pixelReturn: answerReturn } = await runPixel(
  `LLM(engine="${MODEL_ID}", command=["<encode>${prompt}</encode>"], paramValues=[{"temperature":0.1}]);`,
);

const answer = answerReturn[0].output.response;
```

Keep the prompt explicit about citing `Source` and `Part`/`Divider` so the model's reasoning trace references the retrieved chunks.

## Listing available vector engines

Before querying, you often need to let the user pick a vector database — or find one programmatically. Use `MyEngines` with `engineTypes=["VECTOR"]`.

```typescript
import { runPixel } from "@semoss/sdk";

const { errors, pixelReturn } = await runPixel(
  `MyEngines(engineTypes=["VECTOR"], limit=[50], offset=[0]);`
);

if (errors.length) throw new Error(errors[0]);

const vectorEngines = pixelReturn[0].output as Array<{
  engine_id: string;
  engine_name: string;
  engine_display_name: string;
  engine_subtype: string;   // e.g. "FAISS", "CHROMA", "WEAVIATE"
  engine_cost: string;
  engine_favorite: 0 | 1;
}>;
```

### Filtering and paging

`MyEngines` accepts several optional arguments. All are arrays, even when passing a single value:

- `filterWord=["policy"]` — substring match against engine name.
- `limit=[50]`, `offset=[0]` — paging. Omit both to return all results.
- `onlyFavorites=[true]` — restrict to the user's favorited engines.
- `sort={"ENGINENAME": "ASC"}` — sort by `ENGINENAME` or `DATECREATED`, direction `ASC` or `DESC`.

```
MyEngines(engineTypes=["VECTOR"], filterWord=["policy"], sort={"ENGINENAME": "ASC"}, limit=[20], offset=[0]);
```

### Response field conventions

Use `engine_*` fields (`engine_id`, `engine_name`, `engine_display_name`, `engine_subtype`, etc.). The response also contains `app_*` and `database_*` fields with the same values — these are legacy aliases and should not be used in new code.

Common pattern — render a picker and use the selected `engine_id` as `VECTOR_ID` in the `VectorDatabaseQuery()` call above:

```typescript
const [engines, setEngines] = useState<VectorEngine[]>([]);
const [selectedId, setSelectedId] = useState<string>("");

useEffect(() => {
  runPixel(`MyEngines(engineTypes=["VECTOR"], limit=[50], offset=[0]);`)
    .then(({ pixelReturn }) => setEngines(pixelReturn[0].output));
}, []);
```
