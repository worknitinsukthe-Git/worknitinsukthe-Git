# Laddu 👦🏽🟠 — The AI Chat Widget of Paddu's Sweets

### Complete end-to-end notes: what we built, every technology we used, and why

These notes explain **everything** about Laddu, the AI assistant widget on the Paddu's Sweets website —
from the button you click in the browser, all the way to the AI model and the database, and back.
Read them top to bottom once; later use them as a reference.

---

## Table of contents

1. [What we built](#1-what-we-built)
2. [The big picture (architecture)](#2-the-big-picture-architecture)
3. [Technologies & libraries — what they are and why we used them](#3-technologies--libraries)
   - [3.16 Tools used to build, run & test](#316-tools-used-to-build-run--test-the-project)
4. [Agentic AI concepts in plain English](#4-agentic-ai-concepts-in-plain-english)
5. [The life of one message (end-to-end walkthrough)](#5-the-life-of-one-message)
6. [Backend deep dive — `main.py` section 10](#6-backend-deep-dive--mainpy-section-10)
7. [Frontend deep dive — the widget in `index.html`](#7-frontend-deep-dive--the-widget-in-indexhtml)
8. [What Laddu stores in MongoDB](#8-what-laddu-stores-in-mongodb)
9. [Security & guardrails](#9-security--guardrails)
10. [Run, test & troubleshoot](#10-run-test--troubleshoot)
11. [Glossary](#11-glossary)
12. [Ideas for next steps](#12-ideas-for-next-steps)

---

## 1. What we built

**Laddu** is a chat assistant that lives in the bottom-right corner of the Paddu's Sweets website.
He is drawn as a cute traditional Indian boy (tilak, saffron kurta, holding a laddu).

He can:

| Ability | Example |
|---|---|
| Answer about sweets, prices, stock | "What's the price of kaju katli?" → **₹720 / 500g** |
| Clarify vague requests with **numbered choices** | "i want laddu" → lists all 6 ladoos as 1–6 |
| Add to the **same cart** the website uses | "5, 2 packs" → 2 × Motichoor Ladoo in your cart |
| Place a real order (only after you say **yes**) | Order `PS-1005` appears in *My Orders* and *Admin* |
| Check order status | "Where is my order?" |
| Arrange a callback | "I need 10 kg for a wedding, call me" |
| Remember the conversation | Survives page refresh and server restart |

### Chatbot vs. Agent — why Laddu is an *agent*

A plain **chatbot** only produces text from what it already "knows". It would *guess* prices (and be wrong).

An **agent** is an AI that can **decide to take actions** using tools, look at the results, and continue until the job is done.
Laddu decides by himself: *"To answer this I need to search the products"* → our code runs the search on the real database →
Laddu reads the result → answers with real prices. That **decide → act → observe → repeat** loop is what "agentic AI" means.

---

## 2. The big picture (architecture)

```
 ┌──────────────────────────── BROWSER (index.html) ─────────────────────────────┐
 │  Website pages (home, shop, cart, orders, admin)                              │
 │  Laddu widget: launcher 👦🏽 + popup + chat panel                              │
 │        │  fetch("/api/chat", {message: "5, 2 packs"})   + login token          │
 └────────┼──────────────────────────────────────────────────────────────────────┘
          │  HTTP (JSON)
          ▼
 ┌──────────────────────────── SERVER (main.py, FastAPI) ────────────────────────┐
 │  /api/chat route                                                              │
 │    1. load chat memory from MongoDB                                           │
 │    2. build messages = system prompt + recent history + new message           │
 │    3. run_agent()  ── the AGENT LOOP ──────────────┐                          │
 │         call_llm() ──────────────────────────────► │ ──► Sarvam AI (LLM)      │
 │         ◄── "please call add_to_cart(...)"         │                          │
 │         run_tool("add_to_cart") ──► MongoDB        │                          │
 │         call_llm() again with the tool result ───► │ ──► Sarvam AI            │
 │         ◄── final text reply                       │                          │
 │    4. save new messages to MongoDB                                            │
 │    5. return {reply, steps, cart_changed}                                     │
 └───────────────────────────────┬───────────────────────────────────────────────┘
                                 │
              ┌──────────────────┴──────────────────┐
              ▼                                     ▼
   ┌─────────────────────┐              ┌────────────────────────┐
   │  MongoDB Atlas      │              │  Sarvam AI API         │
   │  products, carts,   │              │  model: sarvam-105b    │
   │  orders, users,     │              │  (the "brain")         │
   │  conversations ...  │              └────────────────────────┘
   └─────────────────────┘
```

**Key idea:** the browser never talks to the AI directly. It only talks to **our server**.
The server holds the secret API key, decides what the AI may do, and talks to the database.

### Project files

| File | What's inside |
|---|---|
| `main.py` | The **entire backend**: website APIs (sections 1–9) + **Laddu's brain** (section 10) |
| `index.html` | The **entire frontend**: all pages, CSS, JavaScript + **Laddu widget** (JS section 7) |
| `requirements.txt` | The Python libraries to install |
| `.env` | Your secrets: `MONGODB_URI`, `ADMIN_PASSWORD`, `SARVAM_API_KEY`, `SARVAM_MODEL` (never share!) |
| `.env.example` | A template of `.env` without real secrets |
| `venv/` | Your Python virtual environment (the installed libraries live here) |

---

## 3. Technologies & libraries

### Summary table

| Layer | Technology | Version (your venv) | One-line purpose |
|---|---|---|---|
| Language | **Python** | 3.10.0 | Writes the whole backend |
| Web framework | **FastAPI** | 0.141.1 | Turns Python functions into web APIs (`/api/chat`) |
| (inside FastAPI) | **Starlette** | 1.7.0 | The low-level web toolkit FastAPI is built on |
| Data validation | **Pydantic** | 2.13.5 | Checks the shape of incoming JSON (`ChatIn`) |
| Web server | **Uvicorn** | 0.53.0 | Actually runs the app and listens on port 8000 |
| Database driver | **PyMongo** | 4.18.1 | Python ↔ MongoDB communication |
| (helper) | **dnspython** | 2.8.0 | Lets PyMongo resolve `mongodb+srv://` Atlas URLs |
| Database | **MongoDB Atlas** | cloud | Stores products, carts, orders, chats… |
| Config | **python-dotenv** | 1.2.3 | Loads secrets from `.env` |
| HTTP client | **httpx** | 0.28.1 | Our server calls the Sarvam AI API with it |
| AI model | **Sarvam AI** `sarvam-105b` | API | The LLM "brain" of Laddu |
| Frontend | **HTML5 + CSS3 + vanilla JavaScript** | browser | The widget UI and logic |
| Graphics | **Inline SVG** | browser | Laddu's mascot drawing |
| Fonts | **Google Fonts** (Playfair Display, Poppins) | CDN | The elegant look |

---

### 3.1 Python 3.10

- **What:** A popular, readable programming language.
- **Why here:** Almost all AI tutorials and tools use Python, and it is beginner-friendly.
- **Where:** `main.py`.

### 3.2 FastAPI

- **What:** A modern Python **web framework**. You write normal functions; FastAPI makes them reachable as URLs.
  ```python
  @app.post("/api/chat")
  def chat(body: ChatIn, user=Depends(current_user)):
      ...
  ```
  The decorator `@app.post("/api/chat")` means: *when the browser sends a POST request to `/api/chat`, run this function.*
- **Why here:**
  - Very little code for an API.
  - **Automatic docs** at `http://localhost:8000/docs` — you can test every endpoint from the browser (we used this in Checkpoint 3).
  - **Dependencies** (`Depends(current_user)`) — one line makes a route "login required".
  - Converts Python dicts to JSON automatically.
- **Where:** Every `@app.get / @app.post / @app.delete` in `main.py`. For Laddu: `GET/POST/DELETE /api/chat` and `GET /api/admin/chats`.
- **Note:** our route functions are normal `def` (not `async def`). FastAPI runs them in a thread pool, so a slow AI call
  does not freeze the whole server.

### 3.3 Starlette (installed automatically with FastAPI)

- **What:** The lightweight web toolkit FastAPI is built on (routing, requests, responses).
- **Why:** We don't use it directly; FastAPI uses it. We use its `FileResponse` (via `fastapi.responses`) to serve `index.html`.

### 3.4 Pydantic

- **What:** A library that **validates data** using Python classes.
  ```python
  class ChatIn(BaseModel):
      message: str
  ```
- **Why here:** If the browser sends `{"message": 123}` or forgets `message`, FastAPI + Pydantic reject it automatically
  with a clear error (HTTP 422). We don't write manual checks for the JSON shape.
- **Where:** `ChatIn` (Laddu), and `OrderIn`, `SignupIn`, `ProductIn`… for the website.

### 3.5 Uvicorn

- **What:** An **ASGI server** — the program that listens on a network port and hands each request to FastAPI.
- **Why here:** FastAPI is only the "application"; something must run it. Uvicorn is the standard choice.
- **Where:** Bottom of `main.py`: `uvicorn.run(app, host="127.0.0.1", port=8000)` — so `python main.py` starts everything.

### 3.6 PyMongo + `bson.ObjectId` + dnspython

- **What:** PyMongo is the official Python driver for MongoDB. `bson` (comes with PyMongo) provides `ObjectId`, MongoDB's
  unique ID type. dnspython lets PyMongo understand the `mongodb+srv://...` URL format Atlas gives you.
- **Why here:** Every piece of data (products, carts, chats) is read and written through it.
- **Where (Laddu):** `db.conversations.find_one(...)`, `$push` new messages, `db.products.find(...)` in tools, etc.
- **Special setting:** `MongoClient(..., tz_aware=True)` — dates come back labelled as UTC, so the browser shows them
  in your local time (IST) correctly.

### 3.7 MongoDB Atlas

- **What:** A **document database** in the cloud. Data is stored as JSON-like documents inside "collections"
  (like tables, but flexible).
- **Why here:**
  - A chat message is naturally a JSON document — perfect fit.
  - Conversations can hold messages of different shapes (user text, AI tool calls, tool results) without a fixed table schema.
  - Free cloud tier; nothing to install.
- **Where:** Collections used by Laddu: `conversations`, `products`, `carts`, `orders`, `contact_messages`,
  `site_content`, `shop_info`, `categories`, `users`, `sessions`, `meta` (see [section 8](#8-what-laddu-stores-in-mongodb)).

### 3.8 python-dotenv

- **What:** Reads a `.env` file and puts its values into environment variables.
- **Why here:** Secrets (MongoDB password, Sarvam API key) must **never** be written inside the code.
  They live in `.env`, which you never share or upload.
- **Where:** `load_dotenv(BASE_DIR / ".env")`, then `os.getenv("SARVAM_API_KEY")`.

### 3.9 httpx

- **What:** A modern Python library for making HTTP requests (like a browser, but from code).
- **Why here:** Our server must call Sarvam's API over the internet:
  ```python
  res = httpx.post(SARVAM_URL, json=payload, timeout=90,
                   headers={"Authorization": f"Bearer {SARVAM_API_KEY}"})
  ```
  We deliberately used **plain HTTP instead of an SDK** so you can *see* the real JSON going to and from the AI.
- **Where:** `call_llm()` in `main.py`.

### 3.10 Sarvam AI (`sarvam-105b`)

- **What:** An Indian AI company. `sarvam-105b` is their large language model (LLM), strong in English **and Indian
  languages** (Hindi, Hinglish, Telugu…).
- **API used:** `POST https://api.sarvam.ai/v1/chat/completions` — the industry-standard "chat completions" format
  (same shape as OpenAI's), with support for **tools / function calling**, which is what makes an agent possible.
- **Why here:** You already had a Sarvam API key, it understands Indian customers and languages, and it supports tool calling.
- **Settings we send** (in `call_llm`):

  | Field | Value | Meaning |
  |---|---|---|
  | `model` | `sarvam-105b` | Which AI model (can be changed with `SARVAM_MODEL` in `.env`) |
  | `messages` | list | The whole conversation (the AI has no memory of its own) |
  | `tools` | list of 8 | Descriptions of the functions Laddu may ask us to run |
  | `temperature` | `0.3` | Low = focused & consistent answers (high = creative/random) |
  | `max_tokens` | `4096` | Max length of the reply **including hidden "thinking"** |
  | `reasoning_effort` | `"low"` | This model can think before answering; "low" keeps it fast |

### 3.11 Python built-in modules (no install needed)

| Module | Used for in Laddu |
|---|---|
| `json` | Decode the AI's tool arguments (`json.loads`) and encode tool results (`json.dumps`) |
| `re` | Regular expressions: fuzzy product search (`loose_regex`), removing `<think>…</think>`, phone validation |
| `datetime` | Timestamps on messages; today's date inside Laddu's system prompt |
| `hashlib`, `hmac`, `secrets` | Website login: password hashing and secure session tokens (the widget relies on this login) |
| `os`, `sys`, `pathlib` | Reading `.env` values, terminal mode arguments (`sys.argv`), file paths |

### 3.12 Frontend: HTML5, CSS3, vanilla JavaScript

- **HTML** defines the widget's parts: launcher button, popup bubble, chat panel, input form.
- **CSS** makes it beautiful and responsive:
  - **CSS variables** (`--gold`, `--maroon`…) so the widget matches the website's dark-and-gold theme.
  - **Flexbox** for chat bubbles and the header layout.
  - **`@keyframes` animations**: `ld-bob` (Laddu gently floats), `ld-wiggle` (he waves when the popup appears),
    `ld-pop` (popup bounces in), `ld-open` (panel slides up), `ld-dots` (typing dots).
  - **Media query** `@media (max-width: 520px)` makes the chat full-screen on phones.
- **Vanilla JavaScript** (no framework) handles all the logic:
  - `fetch()` + `async/await` to call `/api/chat`.
  - `localStorage` keeps your **login token**; `sessionStorage` remembers "popup already shown" for this login.
  - **Event delegation** — one click listener on the message area handles all option buttons, even ones created later.
- **Why no React/Vue?** For one widget, plain JS is simpler, has no build step, and you can see exactly what happens.

### 3.13 Inline SVG (the mascot)

- **What:** SVG = *Scalable Vector Graphics*, pictures described with shapes (circles, paths) in code.
- **Why here:** Laddu is drawn with ~30 shapes directly in `index.html` (function `ladduSvg()`):
  - no image files to host, perfectly sharp at any size (launcher, header avatar, tiny message avatar),
  - tiny size, loads instantly.

### 3.14 Google Fonts

- **Playfair Display** (elegant serif) for "Laddu" in the header; **Poppins** for messages. Loaded from Google's CDN in `<head>`.

### 3.15 What we deliberately did NOT use (and why)

| Not used | Why not (for this project) |
|---|---|
| **LangChain / LlamaIndex / CrewAI** (agent frameworks) | They hide the agent loop inside the library. We wrote the loop **by hand** (≈25 lines in `run_agent`) so you truly understand agents. |
| **Sarvam / OpenAI SDKs** | Plain `httpx` shows you the real request/response JSON. |
| **React / Vue** | No build tools needed; the widget is ~250 lines of plain JS. |
| **Vector database / RAG** | Our knowledge (products, prices, orders) is structured data — a normal database query via tools is more accurate than similarity search. |
| **WebSockets / streaming** | Simpler request → response. (Streaming is a good next step — see [section 12](#12-ideas-for-next-steps).) |

### 3.16 Tools used to build, run & test the project

"Tools" in this project means two different things:

1. **Laddu's AI tools** — the 8 Python functions the AI can ask to run (`search_products`, `add_to_cart`,
   `place_order`…). Those are explained in [section 6.4](#64-the-8-tools).
2. **Developer tools** — the programs and services *we* used to build, run and test everything. They are listed below.

#### Tools you use every day to run the project

| Tool | What it is | How we used it |
|---|---|---|
| **Windows PowerShell / Terminal** | The command line | Run `python main.py`, `pip install`, and the terminal chat `python main.py laddu <email>` |
| **Python `venv`** | A private folder of libraries for this project (`venv\`) | `venv\Scripts\activate` → `(venv)` appears. Keeps this project's library versions separate from other projects |
| **pip** | Python's package installer | `pip install -r requirements.txt` installs FastAPI, PyMongo, httpx… |
| **`requirements.txt`** | The list of libraries the project needs | So anyone (or any server) can install the exact same things |
| **`.env` file** | A text file holding secrets | Keeps `MONGODB_URI`, `ADMIN_PASSWORD` and `SARVAM_API_KEY` out of the code |
| **Uvicorn** (through `python main.py`) | The web server | Serves the website + API at `http://localhost:8000` |
| **Web browser** (Chrome / Edge) | Where the website and widget run | Using the site, chatting with Laddu |
| **Browser DevTools** (F12) | The browser's built-in developer panel | **Console**: `localStorage.ps_token` to copy your login token. **Network** tab: watch each `/api/chat` request and response. **Phone icon** (device toolbar): test the mobile full-screen chat |
| **FastAPI Swagger UI** (`/docs`) | An interactive API page FastAPI generates automatically | Tested `POST /api/chat` with the token before the widget existed (Checkpoint 3) |
| **A code editor** (e.g. VS Code / Notepad++) | For reading and editing files | Open `main.py`, `index.html`, `.env`, `notes.md` |

#### Online services & dashboards

| Service | Used for |
|---|---|
| **MongoDB Atlas** (cloud.mongodb.com) | The cloud database. **Browse Collections** shows `conversations`, `orders`, `products`… live. **Network Access** fixes the `ServerSelectionTimeoutError` (allowed IP addresses) |
| **Sarvam AI dashboard** (dashboard.sarvam.ai) | Create or copy your `SARVAM_API_KEY` and check usage/credits |
| **Sarvam API docs** (docs.sarvam.ai) | Checked the chat-completions URL, auth header, model names, and the tool-calling format before writing `call_llm` |
| **Wikipedia / Wikimedia Commons** | Free photos of the sweets (the `image` URLs of products, e.g. the 5 new ladoos) |
| **Google Fonts** | Playfair Display + Poppins fonts |

#### Tools used while building & testing (by the AI coding assistant — not needed to run the app)

| Tool | Why it was used |
|---|---|
| **Claude Code** (AI coding assistant in the terminal) | Planned each checkpoint, wrote and edited `main.py` / `index.html`, ran the tests, wrote these notes |
| **Plan mode** | Each checkpoint was first written as a plan and approved by you before any code changed |
| **mongomock** | A fake in-memory MongoDB, so tests never touched your real database |
| **FastAPI `TestClient`** | Called the real API routes (`/api/chat`, `/api/orders`…) from a Python test script, no browser needed |
| **A fake Sarvam** (a small stand-in function) | Returned scripted AI replies and tool calls, so the agent loop was tested **without spending your API credits** |
| **`python -m py_compile main.py`** | Quick check that the Python file has no syntax errors |
| **Node.js `node --check`** | Quick check that the JavaScript inside `index.html` has no syntax errors |
| **Headless Microsoft Edge** | Took automatic screenshots of the website and widget (desktop + phone) to check the design (that's how the phone-layout bug was found) |
| **curl** | Fetched the Wikipedia image links for products and checked the images load |
| **PowerShell `Test-NetConnection`** | Checked your PC could reach the Atlas servers on port 27017 while debugging the MongoDB timeout |

#### Laddu's own debugging tools (built into the project)

| Tool | Command / place | Shows |
|---|---|---|
| **Terminal mode** | `python main.py laddu your@email.com` | Every AI call, `finish_reason`, token counts, and each 🔧 tool call with its result |
| `--no-memory` flag | add at the end | Laddu forgets each turn — shows why memory matters |
| `--raw` flag | add at the end | The raw JSON message Sarvam returns |
| **Tool tags in the widget** | under each reply | Which tools the agent used (🔍 searched sweets, 🛒 added to cart…) |
| **Admin → Laddu Chats** | `http://localhost:8000/#/admin` | Every customer conversation + tools used |
| **Admin → Orders** | `source: laddu` | Orders placed through the chat |

---

## 4. Agentic AI concepts in plain English

| Concept | Plain meaning | Where you see it in Laddu |
|---|---|---|
| **LLM** (Large Language Model) | A model trained on huge text that predicts the next words. Input: messages. Output: one new message. | `sarvam-105b` via `call_llm()` |
| **Token** | A piece of a word (~¾ of a word). AI usage and cost are counted in tokens. | Terminal shows `tokens: prompt=1455 completion=158` |
| **Messages & roles** | The conversation as a list. `system` = hidden rules, `user` = customer, `assistant` = AI, `tool` = result of a tool | Every call to Sarvam |
| **System prompt** | Hidden instructions that give the AI its personality and rules | `laddu_system_prompt(user)` |
| **Temperature** | Randomness. Low = predictable, high = creative | `0.3` |
| **Reasoning / thinking** | Some models "think" before answering. It uses tokens! | `reasoning_effort: "low"`, `<think>` removed by `THINK_RE` |
| **`max_tokens` & `finish_reason`** | Reply length limit. `finish_reason="length"` means it ran out — the answer may be empty | We raised 1024 → 4096 after an empty reply |
| **Tool / function calling** | We describe functions to the AI (name + description + JSON schema). The AI can reply *"call `search_products` with `{"query":"laddu"}`"* instead of text. **The AI never runs code — we do.** | `LADDU_TOOLS`, `tool_schemas()` |
| **Agent loop** | Repeat: ask AI → if it wants tools, run them and send results back → until it answers in text | `run_agent()` |
| **Memory** | The LLM forgets everything between calls. "Memory" = we store the conversation and send it every time | `conversations` collection |
| **Context window** | The max amount of text the model can read at once. We send only recent messages to stay fast/cheap | `trim_history()` — last 24 messages |
| **Guardrails** | Rules that keep the agent safe. **Prompt rules ask; code rules enforce** | `place_order` refuses without `confirmed=true` |
| **Hallucination** | When an AI confidently invents facts (e.g. a fake price) | Prevented by "ALWAYS use tools for facts" + real DB tools |

### The agent loop in one picture

```
          ┌────────────────────────────────────────────┐
          │  messages = [system, ...history, user msg] │
          └──────────────────────┬─────────────────────┘
                                 ▼
                     ┌──────────────────────┐
             ┌──────►│  call_llm(messages)  │  ← THINK (the AI decides)
             │       └──────────┬───────────┘
             │                  ▼
             │        does the reply contain tool_calls?
             │           │ yes                   │ no
             │           ▼                       ▼
             │   run each tool (our Python)   FINAL ANSWER → return to customer
             │   ← ACT: read/write MongoDB
             │           │
             │   append {"role":"tool", result} ← OBSERVE
             └───────────┘      (max 6 rounds = safety limit)
```

---

## 5. The life of one message

Let's follow a real example. Earlier, Laddu showed the numbered ladoo list. Now the customer taps option **5** and types
"2 packs" — or simply types **"5, 2 packs"**.

### Step 1 — Browser: the widget sends the message

`ladduSend("5, 2 packs")` in `index.html`:
- adds the customer bubble to the screen, shows the typing dots, disables the input,
- calls our API:

```http
POST /api/chat
Authorization: Bearer <your login token>
Content-Type: application/json

{"message": "5, 2 packs"}
```

### Step 2 — Server: who is this? (`current_user`)

FastAPI runs `current_user()` first: it looks up the token in the `sessions` collection → finds the user (e.g. Nitin).
No valid token → **401**, the AI is never called.

### Step 3 — Server: load memory and build the messages (`chat()` route)

```python
conv = active_conversation(user)               # this user's active chat from MongoDB
history = trim_history(conv["messages"])       # only the last ~24 messages
messages = [{"role": "system", "content": laddu_system_prompt(user)}] + llm_view(history)
messages.append({"role": "user", "content": "5, 2 packs"})
```

The history still contains the earlier `search_products` **tool result**, where option `"no": 5` is Motichoor Ladoo.
That is how the AI knows what "5" means.

### Step 4 — Agent loop, round 1: the AI decides to act

`run_agent()` → `call_llm()` sends everything (+ the 8 tool descriptions) to Sarvam. Sarvam replies **not with text**
but with a tool call:

```json
{
  "role": "assistant",
  "content": "",
  "tool_calls": [{
    "id": "call-7f3a...",
    "type": "function",
    "function": { "name": "add_to_cart", "arguments": "{\"product_name\": \"Motichoor Ladoo\", \"qty\": 2}" }
  }]
}
```

Note: `arguments` is a **JSON string**, so we decode it with `json.loads`.

### Step 5 — Our code runs the tool (`run_tool`)

```python
args, result = run_tool(user, "add_to_cart", '{"product_name": "Motichoor Ladoo", "qty": 2}')
```
→ `tool_add_to_cart(user, "Motichoor Ladoo", 2)` → finds the product, checks stock, calls `cart_add()`
(the **same** function the website's "Add to cart" button uses) → MongoDB `carts` updated.

The result is appended to the conversation:

```json
{"role": "tool", "tool_call_id": "call-7f3a...",
 "content": "{\"added\": \"2 x 500g Motichoor Ladoo\", \"cart\": {\"items\": [...], \"total\": 560}}"}
```

### Step 6 — Agent loop, round 2: the AI writes the answer

`call_llm()` again, now with the tool result included. Sarvam replies with plain text:

> Added **2 × 500g Motichoor Ladoo** to your cart! 🍬 Total: **₹560** …

No `tool_calls` → the loop ends.

### Step 7 — Save and respond

Only the **new** messages (user message, tool call, tool result, final answer) are pushed into the `conversations`
document with `$push $each`. The final answer also gets `at` (time) and `steps` (tools used). Response:

```json
{"reply": "Added **2 × 500g Motichoor Ladoo** to your cart! ...",
 "steps": ["add_to_cart"],
 "cart_changed": true}
```

### Step 8 — Browser: draw the reply

- `ladduFormat()` turns `**bold**` into bold and numbered lines into buttons (safely escaped).
- `steps` becomes the tag **🛒 added to cart** under the bubble.
- `cart_changed: true` → the widget calls `/api/cart` and updates the 🛒 count in the header. If you're on the Cart
  page, it re-renders live.

**Total:** 1 browser request, 2 AI calls, 1 tool run, a few database operations — usually a few seconds.

---

## 6. Backend deep dive — `main.py` section 10

### 6.1 Configuration constants

```python
SARVAM_API_KEY = os.getenv("SARVAM_API_KEY", "")
SARVAM_MODEL   = os.getenv("SARVAM_MODEL", "sarvam-105b")
SARVAM_URL     = "https://api.sarvam.ai/v1/chat/completions"
THINK_RE       = re.compile(r"<think>.*?</think>", re.S)   # hide the model's "thinking out loud"
MAX_AGENT_STEPS = 6      # max tool rounds per message (stops infinite loops)
SORRY = "Sorry, I got a little confused 😅 Could you please say that again?"
CHAT_HISTORY_LIMIT = 24  # how many past messages are sent to the AI
LLM_FIELDS = ("role", "content", "tool_calls", "tool_call_id")
```

### 6.2 `call_llm(messages, tools=None, debug=False)` — one call to the AI

1. If there's no API key → error *"Laddu is sleeping: SARVAM_API_KEY is missing in .env"* (HTTP 503).
2. Builds the payload (model, messages, temperature, max_tokens, reasoning_effort, tools).
3. `httpx.post(...)` with a 90-second timeout. Network error → 502; non-200 answer → 502 with Sarvam's message.
4. Takes `choices[0].message`, removes `<think>…</think>`, keeps only `role`, `content`, `tool_calls`.
5. **Empty reply protection:** if the reply has no text and no tool calls, it retries once; still empty → returns `SORRY`.
6. With `debug=True` (terminal mode) it prints message count, `finish_reason` and token usage.

### 6.3 `laddu_system_prompt(user)` — Laddu's personality and rules

Built fresh for every message, with the shop name from MongoDB, the customer's name/email, and today's date/time.
Every rule exists because of something we saw in testing:

| Rule | Why it was added |
|---|---|
| Cute polite shop boy, short replies, call customer by first name | Personality |
| ALWAYS use tools for facts; NEVER invent prices/stock | Prevent hallucination |
| NEVER say "let me check and get back" — check now | In Checkpoint 1 he promised to check and never did |
| General request ("laddu", "barfi") → search and show **every** variety | You wanted him to ask *which* laddu |
| **Numbered choices**, in tool order; a reply of "5" means option 5 of the latest list | You wanted to pick by number |
| Number + quantity → add to cart immediately | Fewer steps for the customer |
| 1 kg = 2 packs of 500g | Units are per pack |
| Buying flow: cart → name/phone/pickup-delivery → summary → **explicit yes** → `place_order(confirmed=true)` | Safe ordering |
| Pickup/Delivery as `1. Pickup` / `2. Delivery` on separate lines | So the widget shows them as buttons |
| More than stock → offer 1. all available, 2. callback, 3. other sweet | "100 packs" test conversation stalled |
| Bulk/wedding/complaints/"talk to a person" → `request_callback` | Hand-off to humans |
| Reply in the customer's language; only talk about the shop | Indian customers; stay on topic |

### 6.4 The 8 tools

Each tool is a normal Python function `tool_xxx(user, ...)`. The `user` is always passed **by our code** (from the
login), never by the AI — so Laddu can only touch **the logged-in customer's** cart and orders.

| Tool name (what the AI sees) | Python function | What it does | Reuses website code |
|---|---|---|---|
| `search_products` | `tool_search_products` | Fuzzy search; returns numbered (`no`) products with price, unit, stock | `find_products` |
| `get_product_details` | `tool_get_product_details` | One product incl. description | `find_one_product` |
| `get_shop_info` | `tool_get_shop_info` | Address, phone, hours, delivery, payment, about | `shop_info`, `site_content` |
| `view_cart` | `tool_view_cart` | Current cart and total | `cart_view` |
| `add_to_cart` | `tool_add_to_cart` | Adds packs, checks stock | `cart_add` (same as website button) |
| `place_order` | `tool_place_order` | Places order from cart **only if `confirmed=true`** | `order_from_cart` → `create_order` |
| `get_my_orders` | `tool_get_my_orders` | Last 5 orders + status | `orders` collection |
| `request_callback` | `tool_request_callback` | Saves a callback request for the shop team | `contact_messages` |

**How the AI learns about tools:** `LADDU_TOOLS` maps each name to *(function, description, parameters, required)*.
`tool_schemas()` converts it to the JSON format the AI understands:

```json
{"type": "function",
 "function": {
   "name": "add_to_cart",
   "description": "Add a product to the customer's cart (the same cart as the website).",
   "parameters": {"type": "object",
     "properties": {
       "product_name": {"type": "string", "description": "Exact product name from search_products"},
       "qty": {"type": "integer", "description": "Number of units (e.g. 500g packs or boxes). 1 kg = 2 packs of 500g."}},
     "required": ["product_name", "qty"]}}}
```

> 💡 The AI chooses tools **only from these names and descriptions**. Clear descriptions = a smarter agent.

`CART_TOOLS = {"add_to_cart", "place_order"}` — if one of these succeeds, the response says `cart_changed: true`.

### 6.5 Fuzzy search — `loose_regex(word)`

Customers type "mothichur", "laddu", "ladoos", "rawa". The database says "Motichoor Ladoo", "Rava Ladoo".
`loose_regex` builds a spelling-forgiving pattern:

1. lower-case, letters only; collapse doubled letters (`laddoo` → `lado`); drop a plural `s` (`ladoos` → `lado`)
2. every vowel group → `[aeiouy]+` (a, o, u, oo all match each other)
3. `h` → optional `h?` (mothi ≈ moti)
4. `v`/`w` → same (`rawa` ≈ `rava`)
5. each consonant may repeat (`d+`)

| Customer types | Finds |
|---|---|
| `laddu` / `ladoos` | all 6 ladoos |
| `mothichur` | Motichoor Ladoo |
| `rawa ladoo` | Rava Ladoo |
| `burfi` | Almond Barfi |
| `gulab jamoon` | Gulab Jamun |
| `kaj` | Kaju Katli |

`find_products` matches the **start of words in the name**, or **whole words in the description**
(this fixed a bug where "laddu" matched the word "loaded"). If no product matches *all* words, it accepts *any* word.
`find_one_product` tries the exact name first; if several match, it returns a helpful error listing them
so the AI asks the customer which one.

### 6.6 `run_tool(user, name, raw_args)` — running a tool safely

- Unknown tool name → `{"error": "Unknown tool '...'"}`
- Broken JSON arguments → `{"error": "Tool arguments were not valid JSON"}`
- Ignores arguments the AI made up (only keeps names defined in the schema)
- Any error inside the tool is **returned, not raised** → the AI reads the error and explains it to the customer
  (e.g. *"Only 40 × 500g of Kaju Katli left in stock"*) instead of the chat crashing.

### 6.7 `run_agent(user, messages, debug=False)` — the agent loop

```python
for _ in range(MAX_AGENT_STEPS):
    reply = call_llm(messages, tools=schemas)
    messages.append(reply)
    if not reply.get("tool_calls"):                      # plain text = done
        return {"reply": reply["content"], "steps": steps, "cart_changed": cart_changed}
    for call in reply["tool_calls"]:                     # the AI asked for tools
        args, result = run_tool(user, call["function"]["name"], call["function"].get("arguments"))
        messages.append({"role": "tool", "tool_call_id": call["id"], "content": json.dumps(result)})
```
If the AI keeps calling tools for 6 rounds, the loop stops and returns `SORRY` (protects your credits and time).

### 6.8 Chat API + memory

| Route | Login | What it does |
|---|---|---|
| `GET /api/chat` | user | Returns Laddu's texts (`name`, `tagline`, `popup`, `greeting`, `suggestions`, `avatar_emoji`) with `{name}` filled in + the **visible** messages of your active chat |
| `POST /api/chat` `{message}` | user | 1–1000 chars → runs the agent → saves new messages → `{reply, steps, cart_changed}` |
| `DELETE /api/chat` | user | "New chat": marks the active conversation `active: false` (kept for admin, not deleted) |
| `GET /api/admin/chats` | admin password | Latest 50 conversations for **Admin → Laddu Chats** |

Helper functions:

- `active_conversation(user)` — finds `{user_id, active: true}`.
- `trim_history(stored)` — keeps the last 24 messages **starting at a user message**, so a `tool` result is never
  sent without the assistant message that requested it (the API would reject that).
- `llm_view(stored)` — strips our extra fields (`at`, `steps`) before sending to the AI.
- `visible_messages(stored)` — what humans see: user messages + Laddu's final answers (hides tool traffic).
- `laddu_texts(user)` — reads `laddu_*` fields from `site_content` and replaces `{name}` with the first name.

If Sarvam fails in the middle, the route raises an error **before saving**, so the stored history stays clean.

### 6.9 Seed upgrades (`seed_database` + `meta` collection)

Your database already existed when we added Laddu, so new data is added with **versioned upgrades** that run **once**:

- **v2** — adds Besan, Boondi, Dry Fruit, Coconut, Rava Ladoo (only if missing).
- **v3** — adds Laddu's texts (`laddu_name`, `laddu_tagline`, `laddu_popup`, `laddu_greeting`, `laddu_suggestions`,
  `laddu_avatar_emoji`) to `site_content`, only the missing ones.

`meta` stores `{_id: "seed", version: 3}`. Because each upgrade runs once, products you delete in Admin don't come back.

### 6.10 Terminal mode — the learning playground

```
venv\Scripts\python main.py laddu your@email.com            # chat in the terminal as that user
venv\Scripts\python main.py laddu your@email.com --no-memory # history cleared every turn → watch him forget
venv\Scripts\python main.py laddu your@email.com --raw       # also print the raw JSON from Sarvam
```
Shows every step: `-> sending 6 messages … with 8 tools`, `<- reply | finish_reason=tool_calls`,
`🔧 search_products({"query": "laddu"}) -> {...}`. Terminal chats are **not saved** to MongoDB.

---

## 7. Frontend deep dive — the widget in `index.html`

### 7.1 HTML structure

```html
<div id="laddu" class="hidden">                       <!-- whole widget, shown after login -->
  <div class="ld-panel" id="ld-panel">                <!-- the chat window -->
    <div class="ld-head"> avatar · Laddu · tagline · ↻ new chat · ✕ close </div>
    <div class="ld-body" id="ld-body"> messages </div>
    <div class="ld-chips" id="ld-chips"> suggestion buttons </div>
    <form class="ld-input" id="ld-form"> text box + ➤ send </form>
  </div>
  <div class="ld-popup" id="ld-popup"> "Hi Nitin, I'm Laddu! 👋 ..." </div>
  <button class="ld-launcher" id="ld-launcher"> mascot + red dot </button>
</div>
```

### 7.2 The mascot — `ladduSvg()`

Drawn on a 100 × 100 canvas, from back to front:

| Part | Shape |
|---|---|
| Saffron kurta | a curved `path`, orange `#f28c28`, with a gold V-collar and two gold buttons |
| Neck, ears, face | rectangle + circles in warm brown skin tones |
| Hair + top tuft | dark paths over the head |
| Tilak | a red vertical line + yellow dot on the forehead |
| Eyes | dark ellipses with small white "shine" circles (cute look) |
| Cheeks | pink semi-transparent ellipses (blush) |
| Smile | a curved line (`Q` = quadratic curve) |
| Laddu in hands | orange circle with yellow dots (boondi), two small hands |

The same function draws the launcher, the header avatar and the small avatar next to each message.

### 7.3 CSS highlights

| Class / animation | Effect |
|---|---|
| `.ld-launcher` + `ld-bob` | Round gold-ringed button that gently floats up and down |
| `.ld-launcher.wave` + `ld-wiggle` | Laddu wiggles ("waves") when the popup appears |
| `.ld-popup` + `ld-pop` | Cream speech bubble bouncing in from the right |
| `.ld-panel` + `ld-open` | Chat window slides up with a fade |
| `.ld-msg.user` / `.ld-msg` | Customer bubbles right (gold tint), Laddu's left with mini avatar |
| `.ld-opt` | Tappable numbered option buttons (gold number circle) |
| `.ld-steps` | Small tool tags like "🔍 searched sweets" |
| `.ld-typing` + `ld-dots` | Three bouncing dots while Laddu thinks |
| `@media (max-width: 520px)` | Phones: panel becomes full-screen; launcher hides while chat is open |

### 7.4 JavaScript state and functions

```js
const LADDU = { info: null, messages: [], open: false, busy: false, popupTimer: null };
```
`info` = texts from the DB, `messages` = chat shown on screen, `busy` = waiting for a reply.

| Function | What it does |
|---|---|
| `initLaddu()` | Called after login (from `enterApp()`). `GET /api/chat` → fills name, tagline, popup text, avatars, history → shows the launcher → schedules the popup after **1.5 s** (once per login) |
| `ladduShowPopup()` | Shows the greeting bubble + wiggle; remembers in `sessionStorage` so it shows once |
| `ladduToggle(open)` | Opens/closes the chat panel; hides the popup and the red unread dot; focuses the input |
| `ladduReset()` | On logout: hides everything, clears messages and flags → next user starts fresh |
| `ladduFormat(text, interactive)` | **Safe mini-markdown**: escapes HTML first (`esc()`), then `**bold**`, `- ` bullets, blank-line gaps. Numbered lines (`1. ...`) become **option buttons** — but only in Laddu's **latest** message (a number always refers to the most recent list). It also splits `1. Pickup  2. Delivery` written on one line |
| `ladduBubble(m, interactive)` | HTML for one message: customer or Laddu (with avatar + tool tags) or error style |
| `ladduRender()` | Redraws the chat: greeting first, then history, typing dots when busy, suggestion chips while no customer message yet; disables input while busy; scrolls to bottom |
| `ladduSend(text)` | Shows your bubble → `POST /api/chat` → shows the reply; if `cart_changed` refreshes the cart count (and the Cart/My Orders page if open); errors appear as a soft red "Oops!" bubble |
| `ladduNewChat()` | Confirms, then `DELETE /api/chat` → empty chat with just the greeting |
| `LADDU_STEP_LABELS` | Friendly names for tool tags: `add_to_cart` → "🛒 added to cart", etc. |

**Event listeners:** launcher click, close ✕, new chat ↻, popup click (✕ only closes the popup), form submit (Enter),
option buttons and suggestion chips (via event delegation), and **Esc** to close.

### 7.5 How the widget plugs into the website

- `enterApp()` (after login) → calls `initLaddu()`.
- `logoutLocal()` → calls `ladduReset()`.
- The widget reuses the website's helpers: `api()` (adds the login token), `esc()` (safe HTML), `toast()`,
  `CART` + `updateCartCount()`, and `router()`.
- Laddu's cart and orders **are** the website's cart and orders — one system, two interfaces.

---

## 8. What Laddu stores in MongoDB

### `conversations` (Laddu's memory)

```json
{
  "_id": ObjectId("..."),
  "user_id": ObjectId("..."),
  "active": true,
  "created_at": "2026-09-23T10:15:00Z",
  "updated_at": "2026-09-23T10:17:42Z",
  "messages": [
    {"role": "user", "content": "i want laddu", "at": "..."},
    {"role": "assistant", "content": "", "tool_calls": [{"id": "call-1", "type": "function",
        "function": {"name": "search_products", "arguments": "{\"query\": \"laddu\"}"}}]},
    {"role": "tool", "tool_call_id": "call-1", "content": "{\"count\": 6, \"products\": [{\"no\": 1, ...}]}"},
    {"role": "assistant", "content": "1. **Besan Ladoo** – ₹320 / 500g ...", "at": "...", "steps": ["search_products"]}
  ]
}
```
The hidden `tool_calls` / `tool` messages are stored too — that's why "5" still works after a restart.

### Other collections Laddu uses

| Collection | Laddu reads / writes |
|---|---|
| `site_content` | reads `brand_name`, `about_text`, and `laddu_*` texts (editable in **Admin → Site content**) |
| `shop_info`, `categories` | read by `get_shop_info` |
| `products` | read by search tools; stock reduced when an order is placed |
| `carts` | written by `add_to_cart`, read by `view_cart` |
| `orders` | written by `place_order` with **`"source": "laddu"`**; read by `get_my_orders` |
| `contact_messages` | callback requests: message starts with `[Callback request via Laddu]`, `"source": "laddu"` |
| `users`, `sessions` | who is chatting (login) |
| `meta` | seed upgrade version |

---

## 9. Security & guardrails

| Protection | How |
|---|---|
| **API key stays secret** | Only the server knows `SARVAM_API_KEY` (in `.env`). The browser never talks to Sarvam. |
| **Login required** | Every `/api/chat` call needs a valid session token → `current_user()` |
| **Laddu acts only for you** | Tools receive `user` from the login, never from the AI → he can't see or change other customers' data |
| **No order without "yes"** | `tool_place_order` refuses unless `confirmed=true` — enforced **in code**, not just the prompt |
| **Prices & totals from the server** | `create_order` recalculates totals from the database; stock is reserved atomically; phone and address validated |
| **Safe tool execution** | Unknown tools/invalid JSON/made-up arguments rejected; errors returned to the AI, not crashes |
| **Loop limit** | Max 6 tool rounds per message |
| **Safe display (no XSS)** | All text is HTML-escaped with `esc()` before any formatting is applied |
| **Message size limit** | 1–1000 characters (checked in both browser and server) |
| **Stays on topic** | System prompt: only talk about the shop |
| **Admin transparency** | Every chat is readable in **Admin → Laddu Chats** |

**Known demo limitations** (fine for learning, fix before a real launch):
no rate limiting (a user could send many messages and spend credits), a simple shared admin password,
and prompt-only rules can still be bent by clever users (that's why the important rules are also in code).

---

## 10. Run, test & troubleshoot

### Run

```powershell
cd C:\Users\nitin\OneDrive\Desktop\Widget
venv\Scripts\activate          # (venv) appears in the prompt
python main.py                 # website + Laddu at http://localhost:8000
python main.py laddu your@email.com   # terminal playground
```
`.env` must contain `MONGODB_URI`, `ADMIN_PASSWORD`, `SARVAM_API_KEY` (and optionally `SARVAM_MODEL`).
New dependency list: `pip install -r requirements.txt`.

### Test the API directly (`/docs`)

1. Open the website, log in, press **F12 → Console**, type `localStorage.ps_token`, copy the value.
2. Open `http://localhost:8000/docs` → **POST /api/chat** → *Try it out*.
3. `authorization`: `Bearer <token>`, body `{"message": "i want laddu"}` → Execute.

### Problems we actually hit (and the fixes)

| Symptom | Cause | Fix |
|---|---|---|
| `ServerSelectionTimeoutError … No replica set members found` at startup | Temporary Atlas/network issue or your IP not allowed | Run again; check **Atlas → Network Access** (add your current IP, or 0.0.0.0/0 for a demo) |
| Laddu's reply was empty, `completion=1024` | The model's hidden thinking used all `max_tokens` | Raised to 4096 + automatic retry + `SORRY` fallback |
| "Let me check and get back to you" but never did | No tools yet (Checkpoint 1) | Tools + rule "check RIGHT NOW with a tool" |
| "i want laddu" didn't ask which one | Only one laddu in DB + no rule | Seed v2 (6 ladoos) + "show every variety" rule + numbered choices |
| Searching "laddu" returned Almond Barfi | Fuzzy pattern matched "loaded" in its description | Description must match **whole words** |
| "100 packs" conversation stalled on "okay" | Open question without choices | Stock rule with numbered options 1/2/3 |
| Chat not full-screen on a small screen | Breakpoint too small for some browsers | Media query changed to `max-width: 520px` |
| "Laddu is sleeping: SARVAM_API_KEY is missing" | Key not in `.env` | Add `SARVAM_API_KEY=...` and restart |
| Order times 5½ hours off | Dates without time zone | `MongoClient(..., tz_aware=True)` |

---

## 11. Glossary

| Term | Meaning |
|---|---|
| **API** | A way for programs to talk to each other (our browser ↔ our server, our server ↔ Sarvam) |
| **Endpoint / route** | One URL of an API, e.g. `POST /api/chat` |
| **HTTP methods** | `GET` = read, `POST` = create/send, `PATCH/PUT` = update, `DELETE` = remove |
| **JSON** | Text format for data: `{"message": "hi"}` |
| **Frontend / Backend** | What runs in the browser (`index.html`) / what runs on the server (`main.py`) |
| **Token (login)** | A random secret string proving you're logged in (`Authorization: Bearer ...`) |
| **Token (AI)** | A small piece of text the AI counts for length and cost |
| **LLM** | Large Language Model — the AI brain (`sarvam-105b`) |
| **Prompt / System prompt** | Text given to the AI / hidden instructions defining its behaviour |
| **Agent** | An AI that decides and takes actions through tools in a loop |
| **Tool / Function calling** | A function the AI can *ask* us to run, described by a JSON schema |
| **JSON Schema** | A description of what arguments a tool accepts (names, types, required) |
| **Context window** | How much text the AI can read at once |
| **Hallucination** | AI inventing facts |
| **Guardrail** | A safety rule (best enforced in code) |
| **ObjectId** | MongoDB's unique ID for a document |
| **Collection / Document** | MongoDB's "table" / "row" (but flexible JSON) |
| **Environment variable / `.env`** | Settings and secrets kept outside the code |
| **Virtual environment (venv)** | A private folder of Python libraries for this project |
| **XSS** | An attack that injects code into a web page — prevented by escaping text |

---

## 12. Ideas for next steps

1. **Streaming replies** — show Laddu's answer word by word (Server-Sent Events).
2. **Multilingual polish** — auto-detect Hindi/Telugu and add suggestion chips in those languages; Sarvam also has
   speech-to-text and text-to-speech APIs → **voice chat with Laddu**.
3. **Rich cards in chat** — product photos and an "Add to cart" button inside the chat bubble.
4. **Rate limiting** — e.g. max 20 messages per user per 10 minutes (protects your credits).
5. **Evaluation** — a list of test questions with expected tool calls, run automatically after every prompt change.
6. **Admin insights** — most asked products, chats that ended in orders, unanswered questions.
7. **Deployment** — host on a cloud server (Render / Railway / a VPS) with HTTPS and a real domain.

---

*Built step by step: Website → Checkpoint 1 (talk to the LLM) → Checkpoint 2 (tools + agent loop) →
Checkpoint 3 (chat API + memory in MongoDB) → Checkpoint 4 (the widget).* 🍬
