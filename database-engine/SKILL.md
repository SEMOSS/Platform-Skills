---
name: database-engine
description: Use when writing code in an app that queries a relational or graph database on the platform, running SELECTs, inserts, updates, deletes, or fetching schema/table structure. Covers the SqlQuery(), SqlQueryBase64(), Database()|Query()|Collect()/ExecQuery(), and GetDatabaseTableStructure() pixel commands via @semoss/sdk's runPixel, plus listing databases with MyEngines(engineTypes=["DATABASE"]). Do not use for LLM calls (see model-engine) or vector database queries.
---

# Database Engine

Query a database on the platform using `runPixel` from `@semoss/sdk` with the `SqlQuery()` pixel command. `SqlQuery` auto-detects the SQL type and routes SELECTs through a `limit`-style path and inserts/updates/deletes through a `commit`-style path.

## Usage

```typescript
import { runPixel } from "@semoss/sdk";

const DATABASE_ID = "e188c7d8-076f-4847-967a-fff45f4ca355";
const sql = "SELECT CITY FROM SALES_DATA_SAMPLE WHERE SALES_DATA_SAMPLE.DEALSIZE = 'Small'";

const { errors, pixelReturn } = await runPixel(
  `SqlQuery(database="${DATABASE_ID}", query="<encode>${sql}</encode>", limit=500);`,
);

if (errors.length) throw new Error(errors[0]);

const { headers, values } = pixelReturn[0].output.data;
// headers: ["CITY"]
// values:  [["NYC"], ["Reims"], ["Lille"], ...]
```

The variations below show only the pixel string — the one that goes inside the `runPixel` template literal. The surrounding `runPixel(...)` call, the `errors` check, and the response parsing are the same as above.

### Insert / update / delete with SqlQuery

Pass `commit=true` instead of `limit`. `SqlQuery` auto-detects the SQL type, so the same pixel handles any modification statement.

```
SqlQuery(database="${DATABASE_ID}", query="<encode>UPDATE table_name SET column1 = value1 WHERE condition</encode>", commit=true);
```

### Base64-encoded queries

`SqlQueryBase64` has the same wrapper behavior as `SqlQuery`; only the query input format changes (base64-encoded UTF-8 SQL string). Useful when a SQL string contains characters that are awkward to escape in a pixel literal.

```
SqlQueryBase64(database="${DATABASE_ID}", query="U0VMRUNUICogRlJPTSB0YWJsZV9uYW1lOw==", limit=500);
```

`U0VMRUNUICogRlJPTSB0YWJsZV9uYW1lOw==` decodes to `SELECT * FROM table_name;`.

### Pipe syntax (Database | Query | Collect / ExecQuery)

Equivalent to `SqlQuery`, but splits the call into explicit stages: select the database, run a query, then collect results or execute. Use this when you want to chain the query into another pixel, or when you prefer the stage to be explicit.

```
Database(database="${DATABASE_ID}") | Query("<encode>SELECT * FROM table_name</encode>") | Collect(500);
```

```
Database(database="${DATABASE_ID}") | Query("<encode>INSERT INTO table_name (col) VALUES ('x')</encode>") | ExecQuery();
```

`Collect(n)` terminates a SELECT (equivalent to `SqlQuery(..., limit=n)`). `ExecQuery()` terminates a modification (equivalent to `SqlQuery(..., commit=true)`).

### Database structure

Fetch logical + physical metadata for every table/column (or vertex/property, for graph databases).

```
GetDatabaseTableStructure(database="${DATABASE_ID}");
```

Each row in `output.data.values` is a 6-tuple:

1. Logical table name (RDBMS) or vertex name (graph)
2. Logical column name (RDBMS) or property name (graph)
3. Data type of the column/property
4. Whether this row represents a graph vertex itself rather than a property on it (only meaningful for RDF/graph databases)
5. Physical column/property name as stored in the database
6. Physical table/vertex name as stored in the database

## Response shape

`pixelReturn[0].output` contains:
- `data.values` — 2D array of rows; each row is a tuple whose cells align with `data.headers`
- `data.headers` — display column names (aliased where the query aliased them)
- `data.rawHeaders` — raw underlying column names
- `headerInfo[]` — per-column metadata `{ dataType, alias, header, type, derived }`
- `sources[]` — engines that served the query: `{ name, type }`
- `numCollected` — number of rows actually returned (bounded by `limit`)

For the full response schema, see `references/response-schema.md`.

## Listing available databases

Before running a query, you often need to let the user pick a database — or find one programmatically. Use the `MyEngines` pixel with `engineTypes=["DATABASE"]` to list databases the current user has access to.

```typescript
import { runPixel } from "@semoss/sdk";

const { errors, pixelReturn } = await runPixel(
  `MyEngines(engineTypes=["DATABASE"], limit=[50], offset=[0]);`
);

if (errors.length) throw new Error(errors[0]);

const databases = pixelReturn[0].output as Array<{
  engine_id: string;
  engine_name: string;
  engine_display_name: string;
  engine_subtype: string;   // e.g. "H2_DB", "POSTGRES", "MYSQL", "TINKER"
  engine_cost: string;
  engine_favorite: 0 | 1;
}>;
```

### Filtering and paging

`MyEngines` accepts several optional arguments. All are arrays, even when passing a single value:

- `filterWord=["sales"]` — substring match against engine name.
- `limit=[50]`, `offset=[0]` — paging. Omit both to return all results.
- `onlyFavorites=[true]` — restrict to the user's favorited engines.
- `sort={"ENGINENAME": "ASC"}` — sort by `ENGINENAME` or `DATECREATED`, direction `ASC` or `DESC`.

```
MyEngines(engineTypes=["DATABASE"], filterWord=["sales"], sort={"ENGINENAME": "ASC"}, limit=[20], offset=[0]);
```

### Response field conventions

Use `engine_*` fields (`engine_id`, `engine_name`, `engine_display_name`, `engine_subtype`, etc.). The response also contains `app_*` and `database_*` fields with the same values — these are legacy aliases and should not be used in new code.

Common pattern — render a picker and use the selected `engine_id` as `DATABASE_ID` in the `SqlQuery()` call above:

```typescript
const [databases, setDatabases] = useState<Database[]>([]);
const [selectedId, setSelectedId] = useState<string>("");

useEffect(() => {
  runPixel(`MyEngines(engineTypes=["DATABASE"], limit=[50], offset=[0]);`)
    .then(({ pixelReturn }) => setDatabases(pixelReturn[0].output));
}, []);
```
