# Example prompts

Written for Claude Code and Cursor with the server connected as in the
[README](README.md). You do not call the tools yourself — you describe what you want and the
assistant picks the tool.

One habit runs through all of these: **make the assistant look things up.** Left alone it
will sometimes answer a food question directly, because that is faster and feels helpful,
and that is the exact failure the server exists to fix.

## Log a meal

```
Log this and give me the totals: 220 g plain skyr, 60 g dry oats, one medium banana,
15 g peanut butter.

Search the catalog for each item and scale each one with the portion tool. Tell me the
gram weight you assumed for the banana. Do not quote a macro you have not looked up.
```

Expect two calls per food: a search that returns a `food_id` and macros per 100 g, then
`calculate_portion` on that id.

Two things to watch. "One medium banana" is not a gram weight, so the assistant has to pick
one and should say which — portion estimation is where nearly all of the error in a food log
lives. And dry oats and cooked oats are different rows, because cooked weight includes
absorbed water; say which one you weighed.

## Scale a portion

```
How many calories and how much protein are in 180 g of cooked chicken breast?
Search the catalog first, then scale it. Do not answer from memory.
```

Qualify the state of the food. "Chicken" is ambiguous in every direction that matters —
breast or thigh, raw or cooked, skin on or off — and each is a different row with
substantially different macros.

If the top hits are genuinely ambiguous, ask to see them rather than accepting a pick:

```
Show me the top five matches with their per-100 g calories before you choose one.
```

## Barcode

```
Look up barcode 5000112637922. If a 12-digit code misses, try it again with a leading
zero before telling me it is not there.
```

You type the digits. There is no scanner in a terminal, and the upside is that
identification cannot silently go wrong the way a misread scan can.

The lookup checks the catalog first and falls back to Open Food Facts, which covers a lot of
own-brand stock a US-centric catalog misses. Those fallback rows are crowd-contributed, so
sanity-check before logging:

```
Cross-check that row with the 4/4/9 rule before you log it — protein and carbs at about
4 kcal per gram, fat at about 9, against the stated calories. Flag a gap over 20%.
```

A gap of 40% or more usually means a per-serving figure sitting in a per-100 g field, a
misplaced decimal, or grams and milligrams confused.

UPC-A is 12 digits, EAN-13 is 13, and a UPC-A code is an EAN-13 with a leading zero, so
databases disagree about which form they store. A 14-digit code is usually an outer carton,
not the retail item.

## Photo

The image has to be **in** the tool call. Dragging a picture into the chat does not send it
to the server — MCP clients do not forward attachments, so the model sees your image and the
server never does. Anything it says about that attachment came from the model, not from a
lookup.

```
Read ./lunch.jpg, encode it as base64 and pass it to the photo tool as image_base64 with
content_type image/jpeg.

Then tell me what gram weight you assumed for each item, search each food in the catalog,
and scale it. I want the macros to come from lookups, not from the image.
```

Raw base64, no `data:` prefix. JPEG, PNG or WebP, up to 2 MB decoded. A URL is not
accepted. Phone photos are often 3-5 MB, so resize the long side to about 1,200 pixels
first.

Tell it what it cannot see, because hidden fat is invisible by design:

```
That is cooked in about 20 g of butter and the sauce is cream-based. The rice is a full
cup cooked. Redo the estimate with that.
```

## Total a recipe

```
Total this for four servings: 400 g dry red lentils, 800 g tinned chopped tomatoes,
200 g onion, 60 g olive oil, spices.

Search each ingredient first, then use the recipe tool. Give me the total and the
per-serving split.
```

Up to 40 ingredients as `{food_id, grams}` pairs plus a serving count. Spices are a rounding
error; the oil is not — 60 g of olive oil is over 500 kcal and it is the single most
commonly omitted item in any recipe calculation, because it went in the pan rather than the
bowl.

Weigh ingredients raw and divide by servings, **or** weigh the whole cooked dish and derive
a per-100 g cooked figure. Pick one and never mix them. Water evaporates during cooking;
calories do not.

## Daily targets

```
I am 34, male, 82 kg, 180 cm, lifting four times a week, desk job otherwise, trying to
lose fat. What should my daily calories and macros be?
```

One call to `calculate_macro_targets`. Treat the output as the start of a two-week
experiment, not a prescription: it is a formula scaled by an activity multiplier, and two
people with identical inputs genuinely differ by several hundred calories. Hold it for two
weeks, watch a **weekly average** weight rather than daily numbers, then adjust.

## Plan a day to a target

```
Build me a day at 2,150 kcal and at least 175 g protein from the foods in my profile.
Search every food and scale every portion with the tools. Show per-meal subtotals.
```

Then correct it, which is the part a tool-backed plan can do and a prompted one cannot:

```
Seven grams of protein short. Fix it without adding more than 60 kcal.
```

Order protein first, then fat to a floor, then carbohydrate to fill the remaining calories.
Protein is the binding constraint with the fewest substitutes; calories are easy to add at
the end.

**Watch the call budget when planning a week.** Seven distinct days of four meals is a
couple of hundred tool calls, which throttles against 20 a minute and would exhaust the
200-call trial budget. Build three or four day templates from about twenty foods and rotate
them instead — roughly 40 calls for the week.

## Standing instructions worth keeping in a file

The server stores nothing: there is no logging tool and no history is kept for you. Put your
profile and a running log in a file your client reads automatically — `CLAUDE.md` in the
working directory for Claude Code, project rules for Cursor.

```markdown
## Standing instructions
- Always call search_foods before quoting a macro. Never estimate a food from memory.
- If a lookup misses, say so and ask me for the label. Do not substitute a similar
  generic row.
- Cross-check every row with the 4/4/9 rule. Flag a gap over 20%.
- State the gram weight you assumed for anything I described without one.
- Append to the Log, never rewrite it. Mark estimated entries as estimated.

## Profile
- 34, male, 82 kg, 180 cm. Lifts 4x/week, desk job.
- Target 2,150 kcal, 175 g protein.
- Allergy: shellfish. Dislikes: cottage cheese.

## Core foods (per 100 g)
| Food | kcal | P | F | C | food_id |

## Log
```

Resolve your twenty or so regular foods once and record them here with their `food_id`.
After that most log entries cost no tool calls at all, because the values are already in the
file.

These are instructions in a prompt, not enforcement. Compliance is high and it is not total,
which is fine for a preference and **not good enough for an allergy** — read the ingredient
list on anything unfamiliar yourself, every time, whatever the file says.

## Check the plumbing

```
Which tools did you call for that, and with what arguments?
```

Worth asking every few days. If the answer is vague, nothing was looked up and you are back
to the model guessing, which is the one thing this server is for.

```
Read nutrition://limits and tell me where I am against my plan.
```
