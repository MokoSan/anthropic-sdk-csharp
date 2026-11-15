# Performance review

A static read of the SDK's hot paths at commit `2a7781f8`. **None of this is measured.** No .NET SDK was available when it was written, so allocation counts and memory figures are estimates from the code. Paths are relative to `src/Anthropic/`.

Most of the code involved (models, union converters, `JsonModelConverter`, `*Params.BodyContent()`) is generated. Per the repo's `CLAUDE.md`, fixes there belong in the generator.

| # | Area | Severity | Fix effort |
|---|---|---|---|
| 1 | Streaming: each SSE event is parsed into several copies | High | Medium (generator) |
| 2 | Aggregators keep every event until the stream ends | High for long outputs | Medium |
| 3 | Request bodies go through a UTF-16 string, rebuilt per retry | Medium, large requests | Low (generator) |
| 4 | Tool runner re-serializes the whole history every turn | Medium, long loops | Medium |
| 5 | Constant discriminators re-serialized in constructors and `Validate()` | Low | Low (generator) |
| 6 | `JsonDictionary.GetHashCode()` always returns 0 | Low, callers only | Low |

## 1. Streaming: every event is parsed into several copies

For one `content_block_delta` event (a few tokens of text):

1. **`Core/Sse.cs:26`**: `SseParser.Create(stream)` decodes the event's data from UTF-8 into a `string`. `Core/Sse.cs:54` then calls `JsonSerializer.Deserialize<T>(item.Data)`, which transcodes it back to UTF-8 to parse.
2. **`Models/Messages/RawMessageStreamEvent.cs:421`**: the union converter parses the whole event into a `JsonElement` (one `JsonDocument` plus a copy of the bytes) to read `"type"` (line 425), then deserializes the concrete event type from that element, and keeps the element as the union's raw value.
3. **`Core/JsonModelConverter.cs:18`**: `Deserialize<Dictionary<string, JsonElement>>` clones every property into its own `JsonDocument` with its own byte copy. `FromRawUnchecked` (line 25) then copies the dictionary into a `FrozenDictionary`, and the `JsonDictionary` constructor (`Core/JsonDictionary.cs:52`) allocates a `ConcurrentDictionary` cache sized to the property count.
4. Reading `.Delta` goes through the delta union converter and `JsonModelConverter` again, one level down.

Roughly: ~7 `JsonDocument`s, 2 `Dictionary`s, 2 `FrozenDictionary`s and 2 `ConcurrentDictionary`s per event, with the delta text copied about four times. Responses stream hundreds to tens of thousands of these.

**Suggested fixes**

- Parse SSE data from bytes: `SseParser.Create(stream, (eventType, data) => JsonSerializer.Deserialize<T>(data, options))` hands the item parser a `ReadOnlySpan<byte>` and skips the string.
- In `JsonModelConverter.Read`, use one `JsonDocument.ParseValue(ref reader)` and fill the dictionary from `RootElement.EnumerateObject()`. The property elements then share one document and one buffer instead of one each.
- In the union converters, call the variant's `FromRawUnchecked` directly from the already-parsed element instead of re-entering `JsonSerializer.Deserialize<TVariant>(element)`.
- Allocate `JsonDictionary._deserializedData` on first property access (`Lazy`/`Interlocked.CompareExchange`), so models that are never read don't pay for it.

## 2. Aggregators keep every event until the stream ends

`Helpers/SseAggregator.cs:84-90`: `Add` appends every parsed event to `_messages[FilterResult]` and only folds them in `GetResult` when `Message()` is called. Each retained event carries all the allocations from #1, so peak memory grows with output length. For a 32K-token response with thinking, a rough estimate is tens of MB and possibly over 100 MB. `.Aggregate()` and the tool runner's streaming mode both go through this path.

`Core/StreamedToolInput.cs:30` concatenates partial tool-input JSON at the end, which is fine on its own, but the partial events it concatenates are also retained.

**Suggested fix:** fold each event as it arrives. Keep a `StringBuilder` per content-block index (text, thinking, signature, partial JSON) plus the start/delta/stop metadata, and build the `Message` from those.

Constraint: `SseAggregator<TMessage, TResult>` is public and `GetResult(IReadOnlyDictionary<FilterResult, IList<TMessage>>)` is `protected abstract`. Changing that shape breaks subclasses, so the incremental path should live inside the sealed `MessageContentAggregator` and `BetaMessageContentAggregator` (override `Add`, or add a new virtual hook with a default that keeps today's behavior).

## 3. Request bodies go through a UTF-16 string

`Models/Messages/MessageCreateParams.cs:611` (same pattern in 104 generated `*Params` files):

```csharp
return new StringContent(
    JsonSerializer.Serialize(this.RawBodyData, ModelBase.SerializerOptions),
    Encoding.UTF8, "application/json");
```

`Serialize` produces a UTF-16 string (2 bytes per char), then `StringContent` encodes it back to UTF-8. For a 1M-token request (~4 MB of JSON) that's ~8 MB of string plus ~4 MB of bytes, both on the large object heap. `BodyContent()` is called inside the per-attempt path (`AnthropicClient.cs:717`, `Core/RetryingExecutor.cs:156`), so each retry repeats it.

**Suggested fix:** `new ByteArrayContent(JsonSerializer.SerializeToUtf8Bytes(RawBodyData, options))` with `Content-Type: application/json`. `ExecuteOnce` already strips the `charset` suffix, so the wire request is unchanged. Params are immutable, so the bytes can be computed once and reused across retries (a new `HttpContent` per attempt is still needed, since content can't be resent after disposal).

## 4. Tool runner re-serializes the whole history every turn

`Helpers/Beta/BetaToolRunner.cs`: each turn appends the assistant message and tool results, then `BuildRequestParams(messages)` (called at line 442) does `_currentParams with { Messages = messages, Tools = ... }`. `PushMessages` (line 174) does the same with `[.. p.Messages, .. messages]`.

The `with` setter goes through `JsonDictionary.Set` (`Core/JsonDictionary.cs:92`), which calls `JsonSerializer.SerializeToElement` on the whole message list immediately. #3 then serializes it again to send. Over N turns, the total serialization work is O(N²) in conversation size.

**Suggested fix:** make `Set` store the value and serialize lazily, when `RawData` is read or the body is built. A cheaper partial fix: reuse the previously serialized elements for messages that haven't changed, since `RawData` on existing message models is already JSON.

## 5. Constant discriminators re-serialized

Generated constructors and `Validate()` call `JsonSerializer.SerializeToElement("text")` and similar (`Models/Messages/TextBlock.cs:66,74,87`; 4,282 call sites under `Models/`). Each call allocates a `JsonDocument`. It matters when building many blocks or when `ResponseValidation` is on.

**Suggested fix:** a `static readonly JsonElement` per discriminator in the generated class, or a shared cache keyed by string.

## 6. `GetHashCode()` always returns 0

`Core/JsonDictionary.cs:197` returns `0`, while `Equals` uses `JsonElement.DeepEquals`. This is legal but means any model used as a `HashSet` or `Dictionary` key puts every entry in one bucket, so lookups become linear scans with deep comparisons. Nothing inside the SDK does this; it only affects callers.

**Suggested fix:** hash the key set plus a few cheap values (type discriminator, `id` when present), or document that models shouldn't be used as keys.

## Checked and fine

- **Tracing** (`Core/Tracing.cs:205`): `IsEnabled` checks for listeners before building any span data, so it costs almost nothing when OpenTelemetry isn't configured.
- **Delta merging** in `MessageContentAggregator.MergeBlock`: one `string.Join` per block, linear in output size.
- **Non-streaming responses** (`Core/HttpResponse.cs`): deserialized straight from the response stream with `DeserializeAsync`.

## Suggested next steps

1. Install the .NET 9 SDK and add a BenchmarkDotNet project that replays a recorded SSE stream through a fake `HttpMessageHandler` into `CreateStreaming(...).Aggregate()`, reporting allocated bytes and Gen0/LOH collections. That gives a baseline for #1 and #2.
2. Prototype #1 (byte-based SSE parsing plus single-document `JsonModelConverter`) and compare.
3. #3 is small and low-risk and could go first as a generator change.
