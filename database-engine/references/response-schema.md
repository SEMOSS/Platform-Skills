# Database query response schema

Full response shape returned from a `runPixel` call that wraps a `SqlQuery()`, `SqlQueryBase64()`, or `Database()|Query()|Collect()` command. The top-level response is an envelope; the tabular result lives at `pixelReturn[0].output.data`.

## Example response

```json
{
    "insightID": "019dba7a-fe47-7832-a745-ffc5af0971d7",
    "pixelReturn": [
        {
            "pixelId": "0",
            "pixelExpression": "SqlQuery ( database = [ \"e188c7d8-076f-4847-967a-fff45f4ca355\" ] , query = [ \"<encode>SELECT CITY FROM SALES_DATA_SAMPLE WHERE SALES_DATA_SAMPLE.DEALSIZE = 'Small';</encode>\" ] , commit = [ true ] ) ;",
            "isMeta": false,
            "timeToRun": 30,
            "output": {
                "data": {
                    "values": [
                        ["NYC"],
                        ["Reims"],
                        ["Lille"],
                        ["San_Francisco"]
                    ],
                    "headers": ["CITY"],
                    "rawHeaders": ["CITY"]
                },
                "headerInfo": [
                    {
                        "dataType": "STRING",
                        "alias": "CITY",
                        "header": "CITY",
                        "type": "STRING",
                        "derived": false
                    }
                ],
                "sources": [
                    {
                        "name": "e188c7d8-076f-4847-967a-fff45f4ca355",
                        "type": "RAW_ENGINE_QUERY"
                    }
                ],
                "numCollected": 50,
                "taskId": "null"
            },
            "operationType": ["OPERATION"]
        }
    ]
}
```

## Envelope fields

- `insightID` — The insight ID used for the pixel execution.
- `pixelReturn[]` — array of results, one per pixel command in the call. For a single query pixel, always index `[0]`.

## pixelReturn[0] fields

- `pixelId` — sequence ID of the command within the call.
- `pixelExpression` — the parsed pixel string SEMOSS actually executed. Useful for debugging encoding issues.
- `isMeta` — internal flag; ignore for query responses.
- `timeToRun` — execution time in milliseconds.
- `operationType` — categorization of the pixel; `["OPERATION"]` for database queries.

## pixelReturn[0].output fields — the query response

- `data.values` *(array of arrays)* — rows returned by the query. Each row is a tuple whose cells align positionally with `data.headers`. **Use this as the primary payload.**
- `data.headers` *(string[])* — display column names. Aliased where the query aliased them.
- `data.rawHeaders` *(string[])* — raw underlying column names as reported by the engine (before any aliasing).
- `headerInfo[]` — per-column metadata, one entry per column, each `{ dataType, alias, header, type, derived }`. `dataType` / `type` values include `"STRING"`, `"NUMBER"`, `"DATE"`, etc. `derived` is `true` for columns produced by a SEMOSS transform rather than the underlying SQL.
- `sources[]` — `{ name, type }` identifying the engine(s) queried. `name` is the database engine ID; `type` is typically `"RAW_ENGINE_QUERY"`.
- `numCollected` *(number)* — number of rows actually returned, bounded by the `limit` argument.
- `taskId` *(string | "null")* — background-task ID when the query streamed; the literal string `"null"` for synchronous returns.

## Variant: `GetDatabaseTableStructure`

The envelope and `output.data.values` / `output.data.headers` shape is identical, but each row is a schema-metadata tuple rather than application data:

```
[logicalTable, logicalColumn, dataType, isVertex, physicalColumn, physicalTable]
```

See the `### Database structure` section of `SKILL.md` for how to interpret each column.

## Variant: modification queries (`commit=true` / `ExecQuery`)

INSERT / UPDATE / DELETE queries return the same envelope, but the `output` body typically carries a status / affected-row payload rather than a tabular `data.values`. Check `numCollected` and the top-level `errors` array from `runPixel` rather than assuming a rows-and-headers response.

## Common access patterns

```typescript
// Raw rows as array-of-arrays
const rows = pixelReturn[0].output.data.values;

// Map rows to objects keyed by header
const { headers, values } = pixelReturn[0].output.data;
const records = values.map(
  (row) => Object.fromEntries(headers.map((h, i) => [h, row[i]])),
);

// Inspect a column's type
const cityType = pixelReturn[0].output.headerInfo.find(
  (h) => h.header === "CITY",
)?.dataType;

// Row count (bounded by limit)
const { numCollected } = pixelReturn[0].output;
```
