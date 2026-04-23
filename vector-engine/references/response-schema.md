# Vector engine response schemas

Full response shapes returned from `runPixel` calls that wrap the vector pixels: `VectorDatabaseQuery`, `ListDocumentsInVectorDatabase`, `CreateEmbeddingsFromDocuments`, `CreateEmbeddingsFromVectorCSVFile`, and `RemoveDocumentFromVectorDatabase`. Each call has a common envelope and a pixel-specific `output`.

Note: unlike `SqlQuery`, whose output wraps rows in `{ data: { values, headers } }`, vector pixels put the payload directly at `pixelReturn[0].output` — usually a raw array or string.

## Envelope fields

Same envelope as every other pixel call:

- `insightID` — insight ID used for the pixel execution.
- `pixelReturn[]` — array of results, one per pixel command in the call. For a single vector pixel, always index `[0]`.

Each `pixelReturn[i]` carries `pixelId`, `pixelExpression` (the parsed pixel SEMOSS actually executed — useful for debugging encoding), `isMeta`, `timeToRun` (ms), and `operationType`. For successful embedding calls, `operationType` is `["SUCCESS"]`; for queries and listings it is `["OPERATION"]`.

## `VectorDatabaseQuery` output

`pixelReturn[0].output` is an array of hit objects ranked by hybrid similarity.

### Example response

```json
{
    "insightID": "019dbb4c-db55-7ea8-bc46-41c95ec49287",
    "pixelReturn": [
        {
            "pixelId": "3",
            "pixelExpression": "VectorDatabaseQuery ( engine = \"1222b449-1bc6-4358-9398-1ed828e4f26a\" , command = '<encode>Time sheet</encode>' , limit = 3 ) ;",
            "isMeta": false,
            "timeToRun": 1027,
            "output": [
                {
                    "Score": 0.6172683238983154,
                    "idx": 2,
                    "Source": "timesheet_guide.pdf",
                    "Modality": "text",
                    "Divider": "1",
                    "Part": "2",
                    "Tokens": 132,
                    "Content": "advise that you will use the manual timesheet form until the WBS code is made available...",
                    "Weighted_RRF_Score": 0.004918032786885246,
                    "BM25_Score": 0.05683182179927826
                }
            ],
            "operationType": ["OPERATION"]
        }
    ]
}
```

### Hit fields

- `Score` *(number)* — combined hybrid-search score. **Lower is closer.**
- `idx` *(number)* — internal chunk index within the vector store.
- `Source` *(string)* — source document identifier (e.g. `"timesheet_guide.pdf"`). Pass this back in `filters` to scope future queries.
- `Modality` *(string)* — chunk modality: `"text"`, etc.
- `Divider` *(string)* — page / section boundary the chunk came from. Often maps 1:1 to page number for PDFs.
- `Part` *(string)* — chunk ordinal within the `Divider` (`"0"`, `"1"`, ...).
- `Tokens` *(number)* — token count of the chunk's `Content`.
- `Content` *(string)* — the chunk text to feed into an LLM prompt.
- `Weighted_RRF_Score` *(number)* — reciprocal-rank-fusion component of the hybrid score.
- `BM25_Score` *(number)* — lexical BM25 component of the hybrid score.

## `ListDocumentsInVectorDatabase` output

`pixelReturn[0].output` is an array with one entry per unique `Source` currently indexed.

```json
{
    "insightID": "019dbb4c-db55-7ea8-bc46-41c95ec49287",
    "pixelReturn": [
        {
            "pixelId": "7",
            "pixelExpression": "ListDocumentsInVectorDatabase ( engine = \"1222b449-1bc6-4358-9398-1ed828e4f26a\" ) ;",
            "isMeta": false,
            "timeToRun": 19,
            "output": [
                {
                    "fileName": "timesheet_guide.pdf",
                    "fileSize": 198.26953125,
                    "lastModified": "2026-03-27 12:22:27"
                }
            ],
            "operationType": ["OPERATION"]
        }
    ]
}
```

- `fileName` *(string)* — source identifier. Use this as the `Source` in `filters` and as `fileNames` when removing.
- `fileSize` *(number)* — size in kilobytes.
- `lastModified` *(string)* — ISO-like timestamp (`"YYYY-MM-DD HH:mm:ss"`).

## `CreateEmbeddingsFromDocuments` / `CreateEmbeddingsFromVectorCSVFile` output

`pixelReturn[0].output` is a status string. Per-file detail lives in `pixelReturn[0].additionalOutput[0].output`.

```json
{
    "insightID": "019dbb4c-db55-7ea8-bc46-41c95ec49287",
    "pixelReturn": [
        {
            "pixelId": "8",
            "pixelExpression": "CreateEmbeddingsFromDocuments ( engine = \"1222b449-1bc6-4358-9398-1ed828e4f26a\" , filePaths = [ \"/130_WEILER1103_OF306.pdf\" ] ) ;",
            "isMeta": false,
            "timeToRun": 3338,
            "output": "Successfully embedded all files",
            "operationType": ["SUCCESS"],
            "additionalOutput": [
                {
                    "output": [
                        {
                            "fileName": "130_WEILER1103_OF306.csv",
                            "status": "SUCCESS",
                            "insertedRecords": 1,
                            "failedRecords": 0,
                            "totalRecords": 1
                        }
                    ],
                    "operationType": ["OPERATION"]
                }
            ]
        }
    ]
}
```

- `output` *(string)* — human-readable summary (`"Successfully embedded all files"` on full success).
- `operationType` — `["SUCCESS"]` for the top-level operation status.
- `additionalOutput[0].output[]` — per-file record: `{ fileName, status, insertedRecords, failedRecords, totalRecords }`. For mixed-result batches, inspect this array to find partial failures. `fileName` here is the normalized CSV name the platform produced, not necessarily the original upload name.

## `RemoveDocumentFromVectorDatabase` output

Same envelope shape as the embedding calls — a status string at `output` plus a per-file breakdown in `additionalOutput`. Check `operationType` for `"SUCCESS"` and inspect `additionalOutput` if you need per-file confirmation.

## Common access patterns

```typescript
// Retrieve and use content chunks
const hits = pixelReturn[0].output as Array<{ Source: string; Content: string; Score: number }>;
const contextBlock = hits
  .map((h) => `* Document Name: ${h.Source}\n${h.Content}`)
  .join("\n\n");

// Group hits by source document
const bySource = hits.reduce<Record<string, typeof hits>>((acc, h) => {
  (acc[h.Source] ??= []).push(h);
  return acc;
}, {});

// Check embedding success
const success = pixelReturn[0].operationType.includes("SUCCESS");
const perFile = pixelReturn[0].additionalOutput?.[0]?.output ?? [];
const failed = perFile.filter((f) => f.status !== "SUCCESS");
```
