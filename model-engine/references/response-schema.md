# LLM response schema

Full response shape returned from a `runPixel` call that wraps an `LLM()` command. The top-level response is an envelope; the model output lives at `pixelReturn[0].output`.

## Example response

```json
{
    "insightID": "019db5ef-0bc6-77ad-83ec-ec413ed19271",
    "pixelReturn": [
        {
            "pixelId": "1",
            "pixelExpression": "LLM ( engine = \"...\" , command = [ \"<encode>Hello</encode>\" ] ... ) ;",
            "isMeta": false,
            "timeToRun": 5151,
            "output": {
                "numberOfTokensInResponse": 12,
                "numberOfTokensInPrompt": 47,
                "schemaVersion": 2,
                "messageType": "CHAT",
                "response": "Hello! How can I help you today?",
                "io": "OUTPUT",
                "parts": [
                    { "text": "Hello! How can I help you today?", "type": "TEXT" }
                ],
                "messageId": "019db5ef-46cf-70af-b356-5771ce903f26",
                "roomId": "019db5ef-0bc6-77ad-83ec-ec413ed19271"
            },
            "operationType": ["OPERATION"]
        }
    ]
}
```

## Envelope fields

- `insightID` — The insight ID used for the pixel execution.
- `pixelReturn[]` — array of results, one per pixel command in the call. For a single `LLM()` call, always index `[0]`.

## pixelReturn[0] fields

- `pixelId` — sequence ID of the command within the call.
- `pixelExpression` — the parsed pixel string SEMOSS actually executed. Useful for debugging encoding issues.
- `isMeta` — internal flag; ignore for model responses.
- `timeToRun` — execution time in milliseconds.
- `operationType` — categorization of the pixel; `["OPERATION"]` for LLM calls.

## pixelReturn[0].output fields — the model response

- `response` *(string)* — the model's text output. **Use this for simple text completions.**
- `parts[]` *(array)* — structured response parts, each `{ text, type }`. Use this instead of `response` when the output contains multiple content types (e.g. text + tool calls). `type` values include `"TEXT"`; other types appear for multimodal or agentic outputs.
- `messageType` — `"CHAT"` for conversational models.
- `io` — `"OUTPUT"` for model responses; `"INPUT"` on echoed prompts.
- `schemaVersion` — response schema version. Currently `2`.
- `numberOfTokensInPrompt`, `numberOfTokensInResponse` — token counts for billing and context tracking.
- `messageId` — unique ID for this specific message. Use to reference a turn in a conversation.
- `roomId` — conversation/session ID. Matches `insightID` for single-turn calls; persists across turns in multi-turn chat.

## Common access patterns

```typescript
// Simple text completion
const text = pixelReturn[0].output.response;

// Multi-part response (preferred for agentic flows)
const parts = pixelReturn[0].output.parts;
const textParts = parts.filter(p => p.type === "TEXT").map(p => p.text);

// Token accounting
const { numberOfTokensInPrompt, numberOfTokensInResponse } = pixelReturn[0].output;

// Conversation threading
const { messageId, roomId } = pixelReturn[0].output;
```