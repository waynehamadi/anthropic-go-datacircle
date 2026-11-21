# Datacircle tools for the Anthropic Go SDK

Find the best GTM APIs. We benchmark all the GTM APIs and we put the results on our website: https://datacircle.dev

This Go module gives the [Anthropic Go SDK](https://github.com/anthropics/anthropic-sdk-go)'s tool runner four tools on Datacircle's open API, https://api.datacircle.dev/v1. You need no key and no account.

| Tool | What the tool gives | Call |
| --- | --- | --- |
| `datacircle_list_apis` | Every GTM API on Datacircle, with the API's slug, number of plans, number of providers and best plan | `GET /apis` |
| `datacircle_best_plans` | One API's plans, sorted by score out of 100, best first. Each plan comes with a rank and a price per 1,000 calls | `GET /apis/{slug}/best` |
| `datacircle_search_plans` | Plans that match a provider or plan name, on all APIs or on one API. Sort by score, provider or price | `GET /plans` |
| `datacircle_provider` | The APIs a provider sells, and the provider's rank on each API | `GET /providers/{slug}` |

## Install

```bash
go get github.com/waynehamadi/anthropic-go-datacircle
```

It needs the Anthropic Go SDK v2.0.0 or later (`github.com/anthropics/anthropic-sdk-go/v2`).

## Use the tools

```go
import (
	"github.com/anthropics/anthropic-sdk-go/v2"
	datacircle "github.com/waynehamadi/anthropic-go-datacircle"
)

client := anthropic.NewClient()
runner := client.Beta.Messages.NewToolRunner(datacircle.NewTools(nil), anthropic.BetaToolRunnerParams{
	BetaMessageNewParams: anthropic.BetaMessageNewParams{
		Model:     anthropic.ModelClaudeOpus5_5,
		MaxTokens: 16000,
		Messages: []anthropic.BetaMessageParam{
			anthropic.NewBetaUserMessage(anthropic.NewBetaTextBlock("Who are the 3 best providers for the email-finder API?")),
		},
	},
	MaxIterations: 10,
})
message, err := runner.RunToCompletion(ctx)
if err != nil {
	log.Fatal(err)
}
for _, block := range message.Content {
	if block.Type == "text" {
		fmt.Println(block.Text)
	}
}
```

`NewToolRunnerStreaming` and a message thread's `NewToolRunner` call the same tools. The tools can sit beside your own tools made with the `toolrunner` package: `append(datacircle.NewTools(nil), myTool)`. Each tool also has its own constructor: `NewListAPIsTool`, `NewBestPlansTool`, `NewSearchPlansTool` and `NewProviderTool`. Give a model only the ones it needs.

Each tool is an `anthropic.BetaTool`, so you can also call it yourself:

```go
blocks, err := datacircle.NewBestPlansTool(nil).Execute(ctx, json.RawMessage(`{"slug":"email-finder","limit":3}`))
fmt.Println(blocks[0].OfText.Text)
```

## Managed Agents sessions

`datacircle.CustomTools()` gives the four tools as custom tools for the agent you create. Put each one in the agent's `Tools` as `anthropic.BetaAgentNewParamsToolUnion{OfCustom: &tool}`. In each session of that agent, the session's tool runner runs them:

```go
runner := client.Beta.Sessions.Events.NewToolRunner(ctx, sessionID, anthropic.SessionToolRunnerOptions{
	Tools: datacircle.NewTools(nil),
})
for runner.Next() {
	fmt.Println(runner.Current().Name)
}
```

## What the model reads

Each tool gives the model a JSON object as text. Each plan comes with its rank, provider, plan, score, price per 1,000 (`perThousand`, in dollars) and its page on datacircle.dev. That page lists every field. `datacircle_provider` starts with a sentence of its counts: the APIs where the provider has plans, those where it has a rank, and its best rank.

When the API gives no data, like for a slug it doesn't know, the tool result is an error (`is_error`) with the API's own words and what to call instead. When the API can't be reached, the error says so. The model reads the error and can call again, and the run goes on. A canceled context stops the call.

`datacircle.Config` sets the `HTTPClient` the tools use (a timeout, a proxy). `nil` uses `http.DefaultClient`.

## Network

Each tool call makes one HTTPS GET request to https://api.datacircle.dev/v1 and nothing else. The request holds only the tool's own input: no key, no config, nothing from your chats. The API takes up to 60 calls a minute from one IP address. The tool waits once, for a minute at most, when the API asks the tool to wait. The module depends only on the Anthropic Go SDK and `golang.org/x/text`, and stores nothing.

## Links

- Datacircle: https://datacircle.dev
- The API: https://datacircle.dev/api
- The API's spec: https://api.datacircle.dev/v1/openapi.json
- What each key means: https://datacircle.dev/fields

MIT license.
