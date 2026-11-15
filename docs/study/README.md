# SDK study notes

Notes from reading through the Anthropic C# SDK at commit `2a7781f8` (release 13.8.0). Nothing in `src/` is changed on this branch.

| File | What it is |
|---|---|
| [architecture.md](architecture.md) | The top-level constructs: client, services, params and models, unions, streaming, helpers, errors, credentials, platform packages, stable vs beta. |
| [sdk-map.html](sdk-map.html) | The same material as a standalone page with a layer diagram and highlighted code. Open it in a browser. |
| [performance-review.md](performance-review.md) | A static review of the hot paths (streaming, model deserialization, request serialization, tool runner), with ranked findings and suggested fixes. |

## Caveats

- The performance findings come from reading code, not from profiling. No .NET SDK was installed when they were written, so allocation counts and memory figures are estimates.
- Most of `src/Anthropic/Models/` and the core JSON infrastructure is generated from the OpenAPI spec. Per the repo's `CLAUDE.md`, fixes there need to land in the generator too, not only in the generated files.
