# FitFindr — planning.md

> Complete this document before writing any implementation code.
> Your spec and agent diagram are what you'll use to direct AI tools (Claude, Copilot, etc.) to generate your implementation — the more specific they are, the more useful the generated code will be.
> Your planning.md will be reviewed as part of your submission.
> Update it before starting any stretch features.

---

## Tools

List every tool your agent will use. For each tool, fill in all four fields.
You must have at least 3 tools. The three required tools are listed — add any additional tools below them.

### Tool 1: search_listings

**What it does:**
It searches the listings dataset and returns items that match the user's request.

**Input parameters:**
- `description` (str): the type of item the user wants, e.g. "vintage graphic tee"
- `size` (str or None): clothing size to filter by, e.g. "M", None if user doesn't specify
- `max_price` (float or None): maximum price the user will pay, None if no limit

**What it returns:**
A list of listing dicts. Each dict has fields like title, price, platform, condition, size, style_tags, etc. Returns an empty list [] if nothing matches.

**What happens if it fails or returns nothing:**
 If the list is empty, the agent should tell the user something like "No listings found for that description, try a broader search term or higher price" and stop without calling the next tools.

---

### Tool 2: suggest_outfit

**What it does:**
Takes a specific listing and the user's wardrobe, asks the LLM to suggest a complete outfit combination.

**Input parameters:**
- `new_item` (dict): the listing selected from search results
- `wardrobe` (dict): the user's current wardrobe with an items list

**What it returns:**
A string, the LLM's outfit suggestion, e.g. "Pair this with your wide-leg jeans and platform Docs for a 90s grunge look."

**What happens if it fails or returns nothing:**
If wardrobe['items'] is empty, still return general styling advice for the item rather than crashing, e.g. "This tee pairs well with wide-leg jeans and chunky sneakers."

---

### Tool 3: create_fit_card

**What it does:**
Takes the outfit suggestion and the selected item, asks the LLM to generate a short, shareable caption, the kind of thing someone would post on Instagram.

**Input parameters:**
- `outfit` (str): the outfit suggestion string returned by suggest_outfit
- `new_item` (dict): the listing dict from search results

**What it returns:**
A string: a casual, social-media style caption, e.g. "thrifted this faded band tee off depop for $22 and honestly it was made for my wide-legs". Should vary each time for different inputs.

**What happens if it fails or returns nothing:**
 If outfit is an empty string, return a descriptive error message string like "Cannot create fit card: outfit description is missing.", do not crash.

---

### Additional Tools (if any)

<!-- Copy the block above for any tools beyond the required three -->

---

## Planning Loop

**How does your agent decide which tool to call next?**
1. Call search_listings(description, size, max_price)
   - If results == [] :
       → set session["error"] = "No listings found - try a broader search or higher price"
       → stop, return session
   - If results is not empty:
       → set session["selected_item"] = results[0]
       → continue to step 2

2. Call suggest_outfit(session["selected_item"], wardrobe)
   - Store the returned string as session["outfit_suggestion"]
   - Continue to step 3

3. Call create_fit_card(session["outfit_suggestion"], session["selected_item"])
   - Store the returned string as session["fit_card"]
   - Return session

---

## State Management

**How does information from one tool get passed to the next?**
All state is stored in a `session` dict that persists across tool calls within one interaction. Keys:
- `session["selected_item"]` — set after search_listings succeeds, passed into suggest_outfit
- `session["outfit_suggestion"]` — set after suggest_outfit returns, passed into create_fit_card
- `session["fit_card"]` — set after create_fit_card returns, shown to the user
- `session["error"]` — set if any tool fails, signals the agent to stop

---

## Error Handling

For each tool, describe the specific failure mode you're handling and what the agent does in response.

| Tool | Failure mode | Agent response |
|------|-------------|----------------|
| search_listings | No results match the query | Tell the user "No listings found - try a broader search term or higher price" and stop |
| suggest_outfit | Wardrobe is empty | Return general styling advice for the item instead of wardrobe-specific suggestions |
| create_fit_card | Outfit input is missing or incomplete | Return "Cannot create fit card: outfit description is missing." as a string, do not crash |

---

## Architecture

User query
    │
    ▼
Planning Loop
    │
    ├─► search_listings(description, size, max_price)
    │       │
    │       ├── results == [] ──► session["error"] = "No listings found..." ──► STOP
    │       │
    │       └── results not empty
    │               │
    │               ▼
    │       session["selected_item"] = results[0]
    │               │
    ├─► suggest_outfit(session["selected_item"], wardrobe)
    │               │
    │               ▼
    │       session["outfit_suggestion"] = "..."
    │               │
    └─► create_fit_card(session["outfit_suggestion"], session["selected_item"])
                    │
                    ▼
            session["fit_card"] = "..."
                    │
                    ▼
            Return session to user

---

## AI Tool Plan

**Milestone 3 — Individual tool implementations:**
- Tool 1: I'll give Claude the Tool 1 spec from planning.md (inputs, return value, failure mode)
  and ask it to implement search_listings() using load_listings() from data_loader.py.
  I'll verify it filters by all three parameters and returns [] when nothing matches.

- Tool 2: I'll give Claude the Tool 2 spec and ask it to implement suggest_outfit() using
  the Groq API with llama-3.3-70b-versatile. I'll verify it handles empty wardrobe without crashing.

- Tool 3: I'll give Claude the Tool 3 spec and ask it to implement create_fit_card() using
  the Groq API. I'll verify outputs vary across runs and it returns an error string for empty input.

**Milestone 4 — Planning loop and state management:**
- I'll give Claude the Planning Loop, State Management sections, and Architecture diagram
  and ask it to implement run_agent() in agent.py. I'll verify it branches correctly
  when search_listings returns [] and doesn't call suggest_outfit in that case.

---

## A Complete Interaction (Step by Step)

**Example query:** "I'm looking for a vintage graphic tee under $30, size M."

**Step 1:** Call search_listings("vintage graphic tee", size="M", max_price=30.0)
— returns a list of matching listings, e.g. [{"title": "Faded Band Tee", "price": 22, ...}]
— set session["selected_item"] = results[0]

**Step 2:** Call suggest_outfit(session["selected_item"], wardrobe)
— returns "Pair this with your wide-leg jeans and chunky sneakers for a 90s look."
— set session["outfit_suggestion"] = that string

**Step 3:** Call create_fit_card(session["outfit_suggestion"], session["selected_item"])
— returns "thrifted this faded band tee off depop for $22 and it was made for my wide-legs 🖤"
— set session["fit_card"] = that string

**Final output:** User sees the fit card caption and outfit suggestion in the UI.