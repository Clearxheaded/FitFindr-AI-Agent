# FitFindr 🛍️

A multi-tool AI agent that helps users find secondhand pieces and figure out how to wear them. Built with Python, Groq (llama-3.3-70b-versatile), and Gradio.

---

## Tool Inventory

### `search_listings(description: str, size: str | None, max_price: float | None) → list[dict]`
Searches the mock listings dataset for items matching the user's description, optional size, and optional price ceiling. Scores each listing by keyword overlap with the description and returns results sorted by relevance. Returns an empty list if nothing matches — never raises an exception.

### `suggest_outfit(new_item: dict, wardrobe: dict) → str`
Takes a listing dict and the user's wardrobe and calls the LLM to suggest 1-2 complete outfit combinations. If the wardrobe is empty, returns general styling advice for the item instead of wardrobe-specific suggestions.

### `create_fit_card(outfit: str, new_item: dict) → str`
Takes an outfit suggestion string and the selected listing and calls the LLM to generate a short, casual Instagram-style caption. Uses a higher temperature (1.2) to produce varied output across runs. Returns a descriptive error string if outfit is empty.

---

## Planning Loop

The agent runs a conditional sequence — not a fixed pipeline:

1. Parse the user's query with regex to extract a description, size, and max_price
2. Call `search_listings()` with the parsed parameters
   - If results are empty → set `session["error"]` and return early. `suggest_outfit` is never called with empty input.
   - If results found → store `results[0]` as `session["selected_item"]` and continue
3. Call `suggest_outfit()` with the selected item and wardrobe → store result in `session["outfit_suggestion"]`
4. Call `create_fit_card()` with the outfit suggestion and selected item → store result in `session["fit_card"]`
5. Return the completed session

The key branching point is step 2 — the agent behaves differently depending on whether search returns results.

---

## State Management

All state lives in a `session` dict initialized at the start of each interaction:

- `session["parsed"]` — extracted description, size, max_price from the query
- `session["search_results"]` — full list of matching listings
- `session["selected_item"]` — `results[0]`, passed into `suggest_outfit`
- `session["outfit_suggestion"]` — string returned by `suggest_outfit`, passed into `create_fit_card`
- `session["fit_card"]` — final caption string returned by `create_fit_card`
- `session["error"]` — set if any tool fails, signals early termination

No tool re-prompts the user or uses hardcoded values — every tool call receives its input directly from the session.

---

## Error Handling

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| `search_listings` | No results match the query | Returns `[]`, agent sets `session["error"]` = "No listings found — try a broader description, different size, or higher price limit." and stops |
| `suggest_outfit` | Wardrobe is empty | Returns general styling advice from the LLM instead of wardrobe-specific suggestions — never crashes |
| `create_fit_card` | Outfit string is empty | Returns `"Cannot create fit card: outfit description is missing."` — never raises an exception |

**Concrete example from testing:**
```bash
python -c "from tools import create_fit_card, search_listings; \
results = search_listings('vintage graphic tee', size=None, max_price=50); \
print(create_fit_card('', results[0]))"
# Output: Cannot create fit card: outfit description is missing.
```

---

## Spec Reflection

**One way planning.md helped:** Writing the planning loop as explicit conditional branches before touching `agent.py` made the implementation straightforward, the code matched the spec almost line for line. The early-return branch for empty search results was designed on paper first, which meant it worked correctly on the first run.

**One way implementation diverged:** The query parsing ended up simpler than expected. The spec left it open — regex, string splitting, or LLM parsing were all options. A simple regex approach (extracting `$XX` for price and size patterns like "M" or "XL") worked well enough without needing an LLM call, which kept the agent faster and cheaper to run.

---

## AI Usage

**Instance 1 — `search_listings` implementation:**
I gave Claude the Tool 1 spec from `planning.md` (inputs, return value, failure mode) and asked it to implement the function using `load_listings()` from the data loader. The generated code filtered by price and size correctly but used plain string concatenation that crashed on `None` fields. I fixed it by adding `or ""` fallbacks to every field access in the searchable string before running the tests.

**Instance 2 — `run_agent` planning loop:**
I gave Claude the Planning Loop and State Management sections from `planning.md` plus the architecture diagram and asked it to implement `run_agent()` in `agent.py`. The generated code matched the spec correctly: it branched on empty search results and stored values in the session dict at each step. I verified it by running both the happy path and no-results path from the CLI before connecting it to the Gradio UI.