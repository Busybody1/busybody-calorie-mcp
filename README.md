# BusyBody MCP — nutrition tools for Claude Code and Cursor

A hosted [Model Context Protocol](https://modelcontextprotocol.io) server that gives an
AI assistant a real food database. The assistant searches the catalog, scales a portion,
totals a recipe, reads a barcode or estimates a photo, and writes its answer from what the
tools returned instead of from what the model remembers.

- Product page: <https://calorieapi.io/mcp>
- Pricing and the 7-day trial: <https://calorieapi.io/mcp/pricing>
- Server URL: `https://calorieapiadmin.com/mcp`

There is no code to install. The server is hosted; this repository holds the connect
configuration and example prompts.

## Connect

You need an API key from the dashboard after starting a plan. The full key is shown once,
when you create it. It is not emailed.

**Claude Code**

```bash
claude mcp add --transport http calorie-api \
  https://calorieapiadmin.com/mcp \
  --header "X-API-Key: YOUR_KEY"
```

**Cursor** — add to your MCP config (see [`examples/cursor-mcp.json`](examples/cursor-mcp.json))

```json
{
  "mcpServers": {
    "calorie-api": {
      "url": "https://calorieapiadmin.com/mcp",
      "headers": { "X-API-Key": "YOUR_KEY" }
    }
  }
}
```

Any client that can set a request header works with the same URL and header name.

> **Claude.ai, Claude Desktop and Claude mobile cannot connect yet.** Those apps add remote
> servers through a connector flow built around OAuth sign-in, and OAuth is not enabled on
> this server. Use Claude Code or Cursor with the header above. The product page will say so
> when browser sign-in becomes available.

Keep the key out of anything you publish. Do not commit it, paste it into an issue, or put
it in a screenshot.

## Tools

| Tool | Returns | Arguments |
|---|---|---|
| `search_foods` | Matching foods with calories and macros per 100 g, plus a `food_id` | `query`, `limit` (1-25), `verified_only` |
| `get_food_nutrition` | Full nutrient list for one food | `food_id` |
| `suggest_foods` | Short autocomplete suggestions | `query` |
| `lookup_barcode` | UPC or EAN digits to macros per 100 g | `upc` (8-14 digits) |
| `calculate_portion` | One food scaled to a gram weight | `food_id`, `grams` |
| `calculate_recipe` | Totals and per-serving macros, up to 40 ingredients | `ingredients[{food_id, grams}]`, `servings` |
| `calculate_macro_targets` | Daily calorie and macro targets | `age`, `gender`, `weight_kg`, `height_cm`, `activity`, `goal` |
| `analyze_food_photo` | Estimated foods in an image | `image_base64`, `content_type` |

Two prompts, `log_a_meal` and `plan_my_macros`, and one resource, `nutrition://limits`,
which reports the limits on your own plan.

**`food_id` comes from `search_foods` and nowhere else.** The portion, recipe and nutrient
tools all require one. An assistant that invents an id gets an error, which is intended —
the alternative would be scaling a row that does not exist. If you see that error, ask the
assistant to search first.

## Example prompts

Full set in [`PROMPTS.md`](PROMPTS.md). The short version:

**Log a meal**

```
Log this and give me the totals: 220 g plain skyr, 60 g dry oats, one medium banana.
Search the catalog for each item, tell me the gram weight you assumed for the banana,
and scale each one with the portion tool.
```

**Scale a portion**

```
How many calories and how much protein are in 180 g of cooked chicken breast?
Search the catalog first, then scale it — do not answer from memory.
```

**Barcode**

```
Look up barcode 5000112637922. If a 12-digit code misses, try it again with a
leading zero before telling me it is not there.
```

**Photo** — the image must be *inside the tool call*, as `image_base64`. Dropping a picture
into the chat does not send it to the server; MCP clients do not forward attachments, so the
model sees your image and the server never does.

```
Read ./lunch.jpg, encode it as base64 and pass it to the photo tool as image_base64 with
content_type image/jpeg. Then search each food it identifies and scale it to the weights
you estimated.
```

Raw base64, no `data:` prefix. `image/jpeg`, `image/png` or `image/webp`. Up to 2 MB
decoded. A URL is not accepted.

## Limits

Per person, not per service.

| | Trial | Paid |
|---|---|---|
| Calls per minute | 5 | 20 |
| Call budget | 200 total, never reset | 10,000 successful per month |
| Foods per search | 10 | 25 |
| Photo estimates | 10 total | 150 per month, 20 per day |

The trial is 7 days and requires a card. Current pricing is on
<https://calorieapi.io/mcp/pricing>.

## Not included

**REST plans do not include this server.** A Free, Basic, Core, Plus, Enterprise or Custom
plan does not grant MCP access. It is a separate subscription because it is a separate
product.

**This server does not include the REST API.** On this plan, REST search, foods, calc and
vision return 403. Your account, billing and authentication routes work normally, so you can
still manage your own subscription and keys.

**Personal use only.** A commercial usage header on an MCP key is rejected. An application,
site or service that puts this data in front of other people needs a REST plan and a
commercial licence — see <https://calorieapi.io/pricing>.

**Not medical advice.** The macro-target tool is a formula, not a measurement. If you have a
diagnosed condition, a therapeutic diet, an eating-disorder history, or you are pregnant or
breastfeeding, a qualified professional needs to review any plan an assistant produces.

## What has not been measured

No accuracy benchmark has been published for the photo tool, no multi-day comparison of
assistant totals against weighed food, and no catalog hit-rate figure. Any number attached to
this product that does not come with a method is not ours. The tool behaviour, argument
schemas and limits documented here are checkable against the server in a few minutes.

## Licence

MIT. See [LICENSE](LICENSE).
