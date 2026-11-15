# Top-level constructs

A map of the SDK's public surface: the client you construct, the services on it, the request and response types they use, and the helpers, errors and platform clients around them. Paths are relative to `src/Anthropic/` unless noted. Targets `net8.0`, `net9.0` and `netstandard2.0`.

## The shape

Almost every call follows one path: build a `*Params` record, pass it to a method on a service reached through the client, and get back a typed model.

```
Platform clients   AnthropicBedrockClient, AnthropicVertexClient, AnthropicFoundryClient, ...   (subclass the client)
      │ inherits
Client             AnthropicClient : IAnthropicClient, ClientOptions, WithOptions(), WithRawResponse
      │ exposes
Services           Messages, Models, Files, Skills, UserProfiles, Organization, Beta.*
      │ take Params, return Models
Models             MessageCreateParams : ParamsBase, Message : JsonModel, ContentBlock (union), ApiEnum<TRaw,TEnum>, IPage<T>
      │ built on
Core               RetryingExecutor, Sse, ModelBase, HttpResponse<T>, Tracing, Exceptions, Credentials

Helpers (opt-in)   Aggregate(), Messages.Create<T>, BetaToolRunner, BetaMessageThread, BetaRefusalFallbackHandler, AsIChatClient()
```

```csharp
using Anthropic;
using Anthropic.Models.Messages;

// Reads ANTHROPIC_API_KEY / ANTHROPIC_AUTH_TOKEN / ANTHROPIC_BASE_URL
var client = new AnthropicClient();

Message response = await client.Messages.Create(new MessageCreateParams
{
    Model = Model.ClaudeOpus5_5,
    MaxTokens = 2048,
    Messages = [new UserMessageParam { Content = "Hello, Claude" }],
});

foreach (var block in response.Content)
    if (block.TryPickText(out var text)) Console.Write(text.Text);
```

## AnthropicClient

`AnthropicClient.cs`, `IAnthropicClient.cs`, `Core/ClientOptions.cs`

The single entry point. It implements `IAnthropicClient` (use the interface for DI and mocking) and is `IDisposable` because it owns an `HttpClient`.

| Member | What it does |
|---|---|
| `ApiKey`, `AuthToken` | Sent as `x-api-key` or `Authorization: Bearer`. Default from environment variables. |
| `BaseUrl` | The first-party API, or `ANTHROPIC_BASE_URL`. |
| `MaxRetries`, `Timeout` | Defaults are 2 retries and 10 minutes. Retries cover 408/409/429/5xx and I/O failures. |
| `Handlers` | `DelegatingHandler`s spliced into the HTTP pipeline. Middleware such as the refusal-fallback handler plugs in here. |
| `ResponseValidation` | When true, every response model is `Validate()`d eagerly. Off by default. |
| `OpenTelemetry` | Opt-in tracing spans. |
| `WithOptions(o => ...)` | A new client sharing the transport with modified options. Services have the same method. |
| `WithRawResponse` | A parallel client whose methods return `HttpResponse<T>`. |

`ClientOptions` is a record struct, so `WithOptions` works with `with` expressions:

```csharp
var patient = client.WithOptions(o => o with { Timeout = TimeSpan.FromMinutes(30), MaxRetries = 5 });
```

## Services

`Services/`, `Services/Beta/`, `Services/Organization/`

Each API resource is an interface/implementation pair (`IMessageService` / `MessageService`), with nested resources as properties. Every service has a `...WithRawResponse` twin.

```
client
├─ Messages        Create · CreateStreaming · CountTokens
│  └─ Batches      Create · Retrieve · List · Cancel · Results …
├─ Models          Retrieve · List
├─ Files
├─ Skills
├─ UserProfiles
├─ Organization    ApiKeys · Invites · Users · Workspaces · Analytics · Federation …
└─ Beta
   ├─ Messages     Create · CreateStreaming · CountTokens · ToolRunner · Thread · Batches
   ├─ Agents · Environments · Sessions · Deployments · DeploymentRuns   (Managed Agents)
   ├─ Vaults · MemoryStores · Dreams · Tunnels · Webhooks
   └─ Models · Files · Skills · UserProfiles · Organization
```

Methods are named `Create`, `Retrieve`, `Update`, `List`, `Delete`, with a `...Streaming` suffix for SSE endpoints (which return `IAsyncEnumerable<T>`). All take an optional `CancellationToken`.

## Params and models

`Models/`, `Core/ParamsBase.cs`, `Core/JsonModel.cs`, `Core/ModelBase.cs`

Namespaces mirror the API: `Anthropic.Models.Messages`, `Anthropic.Models.Files`, `Anthropic.Models.Beta.Messages`, `Anthropic.Models.Beta.Sessions`, and so on.

- **`ParamsBase`**: request records such as `MessageCreateParams`. Carries the body plus path, query and header parameters. Built with object initializers and `required init` properties.
- **`JsonModel : ModelBase`**: response and nested types such as `Message`, `Usage`, `TextBlock`. Immutable record classes backed by the raw JSON, exposed as `RawData`. `Validate()` checks required fields.

Because models keep the raw JSON, fields this SDK version doesn't know about survive a round trip. When relaying API output back into a request, pass the model through rather than rebuilding it from typed properties.

Build a request once, then vary it with `with`: `var next = parameters with { MaxTokens = 4096 };`. On spec-nullable properties (`CacheControl`, `Container`, `ContextManagement`), assigning `null` sends `"prop": null` rather than omitting the field.

## Unions and enums

`Models/Messages/ContentBlock.cs`, `Core/ApiEnum.cs`

C# has no native sum type, so polymorphic fields are generated as union classes. `ContentBlock` can hold a `TextBlock`, `ThinkingBlock`, `ToolUseBlock`, `ServerToolUseBlock`, `WebSearchToolResultBlock` and more. Each union offers four ways in:

```csharp
if (block.TryPickToolUse(out var toolUse)) Run(toolUse.Name, toolUse.Input);    // try-pattern
block.Switch(text => ..., thinking => ..., /* ... */);                         // exhaustive callback
string kind = block.Match(text => "text", thinking => "thinking", /* ... */);  // exhaustive projection
var texts = response.Content.Select(b => b.Value).OfType<TextBlock>();          // raw object
```

The same shape is used for stream events, deltas, tool definitions and Managed Agents events. Unions have implicit conversions from each variant.

Enum fields like `Message.StopReason` are `ApiEnum<string, StopReason>`, so values added to the API later don't break deserialization:

```csharp
if (response.StopReason?.Value() == StopReason.ToolUse) { ... }  // typed compare
string wire = response.StopReason!.Raw();                        // "tool_use"
```

`ToString()` on models and enums returns JSON (quotes included). Compare with `.Value()` or `.Raw()`.

## Streaming

`Core/Sse.cs`, `Helpers/SseAggregator.cs`, `Helpers/MessageContentAggregator.cs`, `SseAggregatorExtensions.cs`

`CreateStreaming` returns a lazy `IAsyncEnumerable<RawMessageStreamEvent>`. The request is sent when enumeration starts; enumerating twice sends two requests.

```csharp
await foreach (var ev in client.Messages.CreateStreaming(parameters))
    if (ev.TryPickContentBlockDelta(out var d) && d.Delta.TryPickText(out var t))
        Console.Write(t.Text);

// Whole stream → one Message
Message message = await client.Messages.CreateStreaming(parameters).Aggregate();

// Live events and the final Message
var aggregator = new MessageContentAggregator();
await foreach (var ev in client.Messages.CreateStreaming(parameters).CollectAsync(aggregator)) { }
Message final = aggregator.Message();
```

Beta mirrors: `BetaMessageContentAggregator`, and `BetaManagedAgentsEventAggregator` for session event streams.

## Raw responses and paging

`Core/HttpResponse.cs`, `Core/IPage.cs`, `IPageExtensions.cs`

`client.WithRawResponse` returns `HttpResponse<T>` with status, headers, `RequestID`, `WorkspaceID` and a lazy `Deserialize()`. Streaming calls return `StreamingHttpResponse<T>`.

List endpoints return a page type implementing `IPage<T>` (`Items`, `HasNext()`, `Next()`). `Paginate()` walks all pages:

```csharp
var page = await client.Models.List();
await foreach (var model in page.Paginate()) Console.WriteLine(model.ID);
```

## Helpers

`Helpers/`, `Helpers/Beta/`, `AnthropicClientExtensions.cs`, `src/Anthropic.Mcp/`

| Construct | Purpose |
|---|---|
| `Messages.Create<T>(...)` (stable) | Structured output. Infers a JSON schema from `T`, returns `StructuredMessage<T>`; call `.Content[0].Parsed()`. |
| `BetaToolRunner` | The agent loop. `client.Beta.Messages.ToolRunner(params, tools)` yields one `BetaMessage` per turn until the model stops. Mutate mid-run with `SetParams`, `PushMessages`, `AddTools`, `RemoveTools`, `CompactBeforeNextTurn`. |
| `BetaRunnableTool` | A `BetaTool` definition plus a `Run` delegate. Throw `BetaToolError` to return `is_error: true`. |
| `BetaAbstractMemoryTool` | Base class for the memory tool's storage backend. |
| `BetaMessageThread` | `client.Beta.Messages.Thread(params)`: `Send`, `SendStreaming`, `Fork`, typed `Thread<T>`. |
| `BetaRefusalFallbackHandler` | Client-side retry on another model when a request is refused. Added through `Handlers`. |
| `AsIChatClient(model)` (stable) | Adapts the client to `Microsoft.Extensions.AI.IChatClient`. Also `AsIHostedFileClient()` and `ToolUnion.AsAITool()`. |
| `BetaMcp` | In `Anthropic.Mcp`: turns an MCP client's tools into runnable tools for the tool runner. |

## Exceptions

`Exceptions/`, `Exceptions/AnthropicExceptionFactory.cs`

```
AnthropicException
├─ AnthropicServiceException            service reported an error · .ErrorType
│  ├─ AnthropicApiException             non-2xx · .StatusCode · .ResponseBody
│  │  ├─ AnthropicBadRequestException          400
│  │  ├─ AnthropicUnauthorizedException        401
│  │  ├─ AnthropicForbiddenException           403
│  │  ├─ AnthropicNotFoundException            404
│  │  ├─ AnthropicUnprocessableEntityException 422
│  │  ├─ AnthropicRateLimitException           429
│  │  ├─ Anthropic4xxException / Anthropic5xxException
│  │  └─ AnthropicUnexpectedStatusCodeException
│  └─ AnthropicSseException             error event inside a 200 stream
├─ AnthropicIOException                 transport failure · retryable
├─ AnthropicInvalidDataException        JSON doesn't fit the model · failed Validate()
└─ WorkloadIdentityException …
```

Outside the family by design: `ArgumentException`, `NotSupportedException` (operations a platform client can't offer), and `OperationCanceledException` (cancellation and timeouts).

## Credentials

`Credentials/`, `Core/ClientOptions.cs`

The no-arg constructor checks, first match wins:

1. `ANTHROPIC_API_KEY`
2. `ANTHROPIC_AUTH_TOKEN`
3. The active or `ANTHROPIC_PROFILE`-selected profile written by `ant auth login` (`CredentialsFileProvider`)
4. Workload Identity Federation environment variables (`WorkloadIdentityCredentials`)

To supply your own, set `ClientOptions.Credentials` to any `IAccessTokenProvider`.

## Platform packages

Each cloud client is a separate NuGet package and a subclass of `AnthropicClient`, so the surface is unchanged. Internally each inserts a `DelegatingHandler` that rewrites URLs, moves `model` into the path where needed, signs requests, and strips first-party API keys.

| Package | Client | Construct with |
|---|---|---|
| `Anthropic.Bedrock` | `AnthropicBedrockMantleClient` | `new(new MantleAwsClientOptions { ... })`. Preferred for new Bedrock code. |
| `Anthropic.Bedrock` | `AnthropicBedrockClient` | `AnthropicBedrockCredentialsHelper.FromEnv()`. Legacy InvokeModel path. |
| `Anthropic.Vertex` | `AnthropicVertexClient` | `DefaultAnthropicVertexCredentials.FromEnvAsync()` |
| `Anthropic.Foundry` | `AnthropicFoundryClient` | `new AnthropicFoundryApiKeyCredentials(key, resource)` |
| `Anthropic.Aws` | `AnthropicAwsClient` | `AwsClientOptions` |
| `Anthropic.GoogleCloud` | `AnthropicGoogleCloudClient` | `GoogleCloudClientOptions` |
| `Anthropic.Mcp` | `BetaMcp` (static) | Not a client; bridges MCP tools into the tool runner. |

## Stable vs beta

| | Stable | Beta |
|---|---|---|
| Service | `client.Messages` | `client.Beta.Messages` |
| Namespace | `Anthropic.Models.Messages` | `Anthropic.Models.Beta.Messages` |
| Types | `Message`, `UserMessageParam`, `Tool` | `BetaMessage`, `BetaUserMessageParam`, `BetaTool` |
| Aggregator | `MessageContentAggregator` | `BetaMessageContentAggregator` |

The two type families share nothing, so beta can change without touching stable code. The beta `MessageCreateParams` has the same name as the stable one; with both namespaces imported, alias one (`using Messages = Anthropic.Models.Messages;`).

`examples/` at the repo root has a runnable project for each construct above.
