# Project: RAG-Context (Product Co-Pilot demo)

## Goal
Build a small working demo of our "Product Co-Pilot" using an open-source CRM (Twenty CRM) as the sample code.
You ask a question (or paste a fake Jira ticket) and it answers in plain English, naming the exact files it used.
The demo will be presented to the team to show what can be built. The user is a total beginner (non-techie), so every step must be small, with full scripts and plain language.

## Tools and choices
- **Demo project: Twenty CRM** (github.com/twentyhq/twenty), downloaded as `twenty.zip`. No Git needed.
- **Scope: Companies, People, Opportunities** (found by keyword in file paths). Matches the "pilot on one module" pitch.
- **Computer: Windows**, working in `C:\rag-demo` using VS Code and its built-in terminal (PowerShell).
- **Language: Python 3.12.10** (NOT 3.15, libraries don't support it yet), inside a private environment called `.venv`.
- **Chopper: `chop.py`.** 40-line pieces, 8 lines overlap, each tagged with file name and line numbers. Skips fake demo data (paths containing "seed" or "prefill").
- **Index and search: ChromaDB**, saved in the `chroma_db` folder, collection name `twenty_code`. Uses its built-in free search model (all-MiniLM-L6-v2).
- **Writer (AI): OpenRouter** (key has $5 credit), using the `openai` library. `ask.py` and the Ask tab of `app.py` use the cheap model `openai/gpt-4o-mini`; `impact.py` and the ticket tab of `app.py` use the stronger `openai/gpt-4o` (a few cents per report). The key is stored in the Windows variable `OPENROUTER_API_KEY`. It is read ONLY by name from that variable. NEVER type or paste the key into a script or into chat.
- **Chat screen: Streamlit** (`app.py`, Phase 6, done). Two tabs: "Ask a question" and "Jira ticket impact report".
- **Method: RAG** (look up relevant material first, then answer), no AI training needed.

## Done so far
1. Read the proposal and agreed the plan and 8 phases. Works.
2. **Phase 0 Setup:** VS Code, Python 3.12.10, `.venv`, installed `openai`, `chromadb`, `streamlit`. `test_ai.py` printed "Hello!". Works.
3. **Phase 1 Source code:** downloaded `twenty.zip`; `get_module.py` extracted 167 files into the `source` folder (`server` and `front` sub-folders). Works.
4. **Phase 2 Chop:** `chop.py` read 157 files and created **421 chunks** in `chunks.json`. Works.
5. **Phase 3 Index:** `build_index.py` stored 421 chunks in ChromaDB. Test searches returned the right files (e.g. `create-company.service.ts` for "where is a company created?"). Works.
6. **Phase 4 Writer:** `ask.py` run and tested with 3 questions. Works.
   - "Where is a company created and what happens when it is created?" returned a plain-English answer citing `create-company.service.ts`.
   - "How are people linked to companies?" returned a grounded but partial answer (email domain is used to find the company, in `create-company-and-contact.service.ts`).
   - "How does billing and payment work?" (out of scope on purpose) correctly replied "I couldn't find this in the material I was given." instead of making something up.
7. **Phase 5 Fake Jira ticket impact report:** `ticket.txt` (fake ticket "Add Industry field to Companies") plus `impact.py`, which writes `impact_report.md`. Works well enough for the demo (see "What we learned" below).
8. **Phase 6 Streamlit chat screen:** `app.py` gives a web page (run with `streamlit run app.py`, opens at `http://localhost:8501`) with two tabs.
   - **Ask a question:** text box plus an Ask button. Shows the answer and the "Sources the AI read" list. Confirmed working with a screenshot (question "What is people?").
   - **Jira ticket impact report:** text box prefilled from `ticket.txt` (can be edited or replaced with another ticket) plus a "Create impact report" button. Shows the 6-section report, a "Download report (.md)" button, and a "Code the AI was given" box. It also saves `impact_report.md`.
   - Results stay on screen when you switch tabs (kept in Streamlit session state).
   - The user marked Phase 6 as done.

**Phase plan:** 0 Setup (done), 1 Source code (done), 2 Chop (done), 3 Index (done), 4 Writer with cited answers (done), 5 Fake Jira ticket impact report (done), 6 Streamlit chat screen (done), 7 Presentation (3 live demos, 5 slides, limitations slide) (**parked for now, skipped by choice, not started**).

## What we learned (use for the limitations slide)
- The "Sources the AI read" list printed under each answer is the trustworthy part. Line numbers the AI writes inside its own answer are its guesses, because it only sees 40-line pieces.
- Search matches on words, not meaning. It can miss related files that use different words. In the Industry ticket, our own keyword search found files that already carry industry data from an outside enrichment service (`to-workspace-company-enrichment.util.ts`, `sanitize-workspace-company-enrichment.util.ts` and the person versions), but they never reached the AI, so the report called enrichment "unclear".
- Impact report v1 (one search, cheap model) was weak. v2 (5 search angles) found the field-definition file. v3 (6 angles, `gpt-4o`, stricter rules) names `compute-company-standard-flat-field-metadata.util.ts` as where the field is defined and finds `people-data-labs-company-data.type.ts` (already has an industry value), and it admits what it did not see (no backend filtering found).
- Remaining weaknesses of the v3 report: lists `create-company.service.ts` first (weak pick), ignores the company page layout file, suggests an onboarding import file (stretch), and says the view-fields file must "support filtering" (it only controls columns).
- Suggested limitations wording: "Search can miss related files that use different words, so the report is a draft for BA review, not a final answer." This matches the proposal: AI drafts, BA reviews and approves.
- Our `source` folder only holds files extracted by keyword, so some definitions may simply not be in it.
- Out-of-scope questions get an honest "I couldn't find this", which is good to show live.
- **Short, vague questions give thin answers.** In the Streamlit screen, "What is people?" matched the emailing files (`add-people-to-message-list...`) and the answer only described people as contacts added to message lists. Specific questions work much better, so use these for live demos: "Where is a company created and what happens when it is created?", "How are people linked to companies?", and the out-of-scope "How does billing and payment work?".

## Current code
**get_module.py** (Phase 1)
```python
import shutil
import zipfile
from pathlib import Path

ZIP_FILE = "twenty.zip"
OUTPUT_FOLDER = Path("source")

# Keep any file whose path contains one of these words
KEYWORDS = ["company", "companies", "person", "people", "opportunit"]

# Only look inside these two parts of the project
ROOTS = {
    "server": "packages/twenty-server/src/",
    "front": "packages/twenty-front/src/",
}

WANTED_TYPES = {".ts", ".tsx", ".json", ".md", ".sql", ".graphql"}

# Skip noise: tests, translations, auto-generated files
SKIP_WORDS = [".test.", ".spec.", ".stories.", "/locales/", "/generated",
              "__tests__", "__mocks__", "node_modules"]

MAX_FILES = 700  # safety limit

if OUTPUT_FOLDER.exists():
    shutil.rmtree(OUTPUT_FOLDER)

zf = zipfile.ZipFile(ZIP_FILE)
names = [n for n in zf.namelist() if not n.endswith("/")]

picked = []
for name in names:
    lower = name.lower()
    if Path(name).suffix not in WANTED_TYPES:
        continue
    if any(s in lower for s in SKIP_WORDS):
        continue
    for part, root in ROOTS.items():
        if root in name:
            rest = name.split(root, 1)[1]
            if any(k in rest.lower() for k in KEYWORDS):
                picked.append((part, rest, name))
            break

where = {}
for part, rest, _ in picked:
    top = "/".join(rest.split("/")[:2])
    key = f"{part}/{top}"
    where[key] = where.get(key, 0) + 1

print(f"Files matched: {len(picked)}")
print("--- Biggest sources (top 20) ---")
for key, n in sorted(where.items(), key=lambda x: -x[1])[:20]:
    print(f"  {key}: {n}")
print()

if len(picked) > MAX_FILES:
    print(f"Too many files (limit is {MAX_FILES}). Nothing extracted.")
    print("Paste this output to Claude and we will trim it.")
else:
    for part, rest, name in picked:
        target = OUTPUT_FOLDER / part / rest
        target.parent.mkdir(parents=True, exist_ok=True)
        target.write_bytes(zf.read(name))
    print(f"Done. Extracted {len(picked)} files into the '{OUTPUT_FOLDER}' folder.")
```

**chop.py** (Phase 2)
```python
import json
from pathlib import Path

SOURCE_FOLDER = Path("source")
OUTPUT_FILE = "chunks.json"

LINES_PER_CHUNK = 40   # size of each piece
OVERLAP = 8            # lines repeated between pieces so nothing is cut mid-thought

# Skip files whose path contains any of these words (fake demo data)
SKIP_PATH_WORDS = ["seed", "prefill"]

chunks = []
files_read = 0
files_skipped = 0

for path in sorted(SOURCE_FOLDER.rglob("*")):
    if not path.is_file():
        continue

    path_text = path.as_posix().lower()
    if any(word in path_text for word in SKIP_PATH_WORDS):
        files_skipped += 1
        continue

    try:
        text = path.read_text(encoding="utf-8")
    except Exception:
        continue  # skip anything that can't be read as text

    lines = text.splitlines()
    if not lines:
        continue

    files_read += 1
    file_name = path.relative_to(SOURCE_FOLDER).as_posix()

    start = 0
    while start < len(lines):
        end = min(start + LINES_PER_CHUNK, len(lines))
        piece = "\n".join(lines[start:end])

        if len(piece.strip()) > 50:
            chunks.append({
                "id": f"chunk-{len(chunks)}",
                "file": file_name,
                "start_line": start + 1,
                "end_line": end,
                "text": piece,
            })

        if end == len(lines):
            break
        start = end - OVERLAP

with open(OUTPUT_FILE, "w", encoding="utf-8") as f:
    json.dump(chunks, f, indent=2)

print(f"Read {files_read} files (skipped {files_skipped} demo-data files)")
print(f"Created {len(chunks)} chunks")
print()
print("--- Sample chunk ---")
sample = chunks[len(chunks) // 2]
print(f"File: {sample['file']}  (lines {sample['start_line']} to {sample['end_line']})")
print(sample["text"][:400])
```

**build_index.py** (Phase 3)
```python
import json
import chromadb

CHUNKS_FILE = "chunks.json"
DB_FOLDER = "chroma_db"
COLLECTION = "twenty_code"

with open(CHUNKS_FILE, encoding="utf-8") as f:
    chunks = json.load(f)
print(f"Loaded {len(chunks)} chunks")

client = chromadb.PersistentClient(path=DB_FOLDER)

# Start fresh each time so we never get duplicates
try:
    client.delete_collection(COLLECTION)
except Exception:
    pass
collection = client.create_collection(COLLECTION)

print("Building the index (the first run downloads a small search model, please wait)...")
BATCH = 100
for i in range(0, len(chunks), BATCH):
    batch = chunks[i:i + BATCH]
    collection.add(
        ids=[c["id"] for c in batch],
        documents=[f"File: {c['file']}\n{c['text']}" for c in batch],
        metadatas=[{
            "file": c["file"],
            "start_line": c["start_line"],
            "end_line": c["end_line"],
        } for c in batch],
    )
    print(f"  stored {min(i + BATCH, len(chunks))} / {len(chunks)}")

print(f"\nIndex ready: {collection.count()} chunks saved in '{DB_FOLDER}'")

# Quick search test
tests = [
    "where is a company created?",
    "how are people linked to companies?",
]
for q in tests:
    print(f"\n=== Question: {q}")
    result = collection.query(query_texts=[q], n_results=5)
    for meta in result["metadatas"][0]:
        print(f"  {meta['file']}  (lines {meta['start_line']} to {meta['end_line']})")
```

**ask.py** (Phase 4, works)
```python
import os
import chromadb
from openai import OpenAI

DB_FOLDER = "chroma_db"
COLLECTION = "twenty_code"
MODEL = "openai/gpt-4o-mini"
HOW_MANY_CHUNKS = 6

ai = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.environ["OPENROUTER_API_KEY"],
)
db = chromadb.PersistentClient(path=DB_FOLDER)
collection = db.get_collection(COLLECTION)

SYSTEM_PROMPT = """You are a Product Co-Pilot for a CRM product.
You help business analysts and new team members understand how the product works.

Rules:
- Use ONLY the code excerpts provided below. Do not guess.
- Explain in plain English for a non-technical reader. Avoid jargon, and explain any technical word you must use.
- Mention the file names (and line numbers) that support your answer.
- If the excerpts do not contain the answer, say clearly: "I couldn't find this in the material I was given."
- Keep the answer short and well organised.
"""


def answer(question):
    # 1. The librarian finds the most relevant chunks
    found = collection.query(query_texts=[question], n_results=HOW_MANY_CHUNKS)
    docs = found["documents"][0]
    metas = found["metadatas"][0]

    # 2. Put the chunks together as "open book" material
    material = ""
    for doc, meta in zip(docs, metas):
        material += (
            f"\n--- {meta['file']} (lines {meta['start_line']} to {meta['end_line']}) ---\n"
            f"{doc}\n"
        )

    # 3. The writer reads the material and answers
    reply = ai.chat.completions.create(
        model=MODEL,
        messages=[
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": f"Code excerpts:\n{material}\n\nQuestion: {question}"},
        ],
    )
    text = reply.choices[0].message.content

    # 4. Print the answer and the sources the AI was given
    print("\n" + "=" * 60)
    print(text)
    print("\nSources the AI read:")
    seen = set()
    for meta in metas:
        line = f"  - {meta['file']} (lines {meta['start_line']} to {meta['end_line']})"
        if line not in seen:
            seen.add(line)
            print(line)
    print("=" * 60 + "\n")


print("Product Co-Pilot ready. Type your question, or type 'exit' to stop.\n")
while True:
    q = input("Your question: ").strip()
    if q.lower() in ("exit", "quit", ""):
        break
    answer(q)
```

**ticket.txt** (Phase 5 fake Jira ticket)
```
Title: Add "Industry" field to Companies

Client requirement: Our sales team wants to record which industry each company belongs to (for example Banking, Retail, Healthcare). The industry should be visible on the company list and on the company detail page, and users should be able to filter the company list by industry.
```

**impact.py** (Phase 5, works, current version v3)
```python
import os
import chromadb
from openai import OpenAI

DB_FOLDER = "chroma_db"
COLLECTION = "twenty_code"
MODEL = "openai/gpt-4o"
CHUNKS_PER_SEARCH = 5
MAX_CHUNKS = 20
TICKET_FILE = "ticket.txt"
REPORT_FILE = "impact_report.md"

ai = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.environ["OPENROUTER_API_KEY"],
)
db = chromadb.PersistentClient(path=DB_FOLDER)
collection = db.get_collection(COLLECTION)

SYSTEM_PROMPT = """You are a Product Co-Pilot for a CRM product.
A new client requirement (a Jira ticket) has arrived. Write an impact report for a Business Analyst who is not a programmer.

Rules:
- Use ONLY the code excerpts provided. Do not guess.
- Write in plain English. Explain any technical word you must use.
- Only name files and line ranges that appear in the excerpt headers. Never invent a file name.
- Go through the excerpts one by one before writing. Read each file name and its content, and be accurate about what it is about (a file about people is not about companies).
- Any file that defines the fields of the thing named in the ticket (where its data columns are declared) MUST appear in section 2.
- Look carefully for things the product ALREADY does that relate to the ticket, such as data that already exists or arrives from an outside service. Report these in section 3.
- For every file you list, add one plain sentence saying what that file does and why it matters to this ticket.
- If something the ticket needs is not covered by the excerpts, say so plainly in section 4, for example "I did not see where this is defined in the material I was given."

Write the report with exactly these sections:
## 1. What the ticket asks for
(2 to 3 sentences)
## 2. Areas likely affected
(a short list; for each item give the file, line range, and one sentence on why)
## 3. Existing parts that could be reused
(a short list; for each item give the file and one sentence on what already exists)
## 4. Risks, dependencies and gaps
(a short list, including anything the excerpts did not cover)
## 5. Questions for the team
(2 to 4 questions the BA should clarify before work starts)
## 6. Confidence
(High, Medium or Low, with one sentence of reason)
"""

with open(TICKET_FILE, encoding="utf-8") as f:
    ticket = f.read()

title = ticket.splitlines()[0].replace("Title:", "").strip()

# Search from several angles, then merge the results
searches = [
    ticket,
    f"{title} where the field is defined (data model, database, metadata)",
    f"{title} list screen columns and filters",
    f"{title} detail page layout",
    f"{title} existing data or feature that already covers this",
    f"{title} data that already exists or is imported from an outside service",
]

print("Reading the ticket and searching the code from 6 angles...")
picked = {}
for query in searches:
    found = collection.query(query_texts=[query], n_results=CHUNKS_PER_SEARCH)
    for doc, meta in zip(found["documents"][0], found["metadatas"][0]):
        key = (meta["file"], meta["start_line"], meta["end_line"])
        if key not in picked:
            picked[key] = (doc, meta)

items = list(picked.values())[:MAX_CHUNKS]
print(f"Collected {len(items)} different pieces of code.")

material = ""
for doc, meta in items:
    material += (
        f"\n--- {meta['file']} (lines {meta['start_line']} to {meta['end_line']}) ---\n"
        f"{doc}\n"
    )

print("Writing the impact report (this can take up to a minute)...")
reply = ai.chat.completions.create(
    model=MODEL,
    messages=[
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": f"Code excerpts:\n{material}\n\nJira ticket:\n{ticket}"},
    ],
)
report = reply.choices[0].message.content

with open(REPORT_FILE, "w", encoding="utf-8") as f:
    f.write(report)

print("\n" + "=" * 60)
print(report)
print("\nCode the AI was given:")
for doc, meta in items:
    print(f"  - {meta['file']} (lines {meta['start_line']} to {meta['end_line']})")
print("=" * 60)
print(f"\nReport also saved to {REPORT_FILE}")
```

**app.py** (Phase 6, works; run with `streamlit run app.py`, stop with Ctrl+C in the terminal)
```python
import os

import chromadb
import streamlit as st
from openai import OpenAI

# ---------- Settings ----------
DB_FOLDER = "chroma_db"
COLLECTION = "twenty_code"
ASK_MODEL = "openai/gpt-4o-mini"      # cheap model for quick questions
IMPACT_MODEL = "openai/gpt-4o"        # stronger model for impact reports
ASK_CHUNKS = 6
CHUNKS_PER_SEARCH = 5
MAX_CHUNKS = 20
TICKET_FILE = "ticket.txt"
REPORT_FILE = "impact_report.md"

ASK_PROMPT = """You are a Product Co-Pilot for a CRM product.
You help business analysts and new team members understand how the product works.

Rules:
- Use ONLY the code excerpts provided below. Do not guess.
- Explain in plain English for a non-technical reader. Avoid jargon, and explain any technical word you must use.
- Mention the file names (and line numbers) that support your answer.
- If the excerpts do not contain the answer, say clearly: "I couldn't find this in the material I was given."
- Keep the answer short and well organised.
"""

IMPACT_PROMPT = """You are a Product Co-Pilot for a CRM product.
A new client requirement (a Jira ticket) has arrived. Write an impact report for a Business Analyst who is not a programmer.

Rules:
- Use ONLY the code excerpts provided. Do not guess.
- Write in plain English. Explain any technical word you must use.
- Only name files and line ranges that appear in the excerpt headers. Never invent a file name.
- Go through the excerpts one by one before writing. Read each file name and its content, and be accurate about what it is about (a file about people is not about companies).
- Any file that defines the fields of the thing named in the ticket (where its data columns are declared) MUST appear in section 2.
- Look carefully for things the product ALREADY does that relate to the ticket, such as data that already exists or arrives from an outside service. Report these in section 3.
- For every file you list, add one plain sentence saying what that file does and why it matters to this ticket.
- If something the ticket needs is not covered by the excerpts, say so plainly in section 4, for example "I did not see where this is defined in the material I was given."

Write the report with exactly these sections:
## 1. What the ticket asks for
(2 to 3 sentences)
## 2. Areas likely affected
(a short list; for each item give the file, line range, and one sentence on why)
## 3. Existing parts that could be reused
(a short list; for each item give the file and one sentence on what already exists)
## 4. Risks, dependencies and gaps
(a short list, including anything the excerpts did not cover)
## 5. Questions for the team
(2 to 4 questions the BA should clarify before work starts)
## 6. Confidence
(High, Medium or Low, with one sentence of reason)
"""

# ---------- Page setup ----------
st.set_page_config(page_title="Product Co-Pilot", page_icon="🧭", layout="wide")
st.title("🧭 Product Co-Pilot")
st.caption("Demo on the Twenty CRM code (Companies, People, Opportunities). "
           "AI drafts, a person reviews.")

# The key is read ONLY by the name of the Windows variable. Never type the key here.
api_key = os.environ.get("OPENROUTER_API_KEY")
if not api_key:
    st.error("The OPENROUTER_API_KEY variable was not found. "
             "Close this terminal, open a NEW one, activate .venv and run the app again.")
    st.stop()


@st.cache_resource
def load_tools():
    ai = OpenAI(base_url="https://openrouter.ai/api/v1", api_key=api_key)
    db = chromadb.PersistentClient(path=DB_FOLDER)
    collection = db.get_collection(COLLECTION)
    return ai, collection


try:
    ai, collection = load_tools()
except Exception as e:
    st.error(f"Could not open the search index in '{DB_FOLDER}'. Details: {e}")
    st.stop()


def build_material(items):
    material = ""
    for doc, meta in items:
        material += (
            f"\n--- {meta['file']} (lines {meta['start_line']} to {meta['end_line']}) ---\n"
            f"{doc}\n"
        )
    return material


def source_lines(items):
    seen = []
    for _, meta in items:
        line = f"{meta['file']} (lines {meta['start_line']} to {meta['end_line']})"
        if line not in seen:
            seen.append(line)
    return seen


def answer_question(question):
    found = collection.query(query_texts=[question], n_results=ASK_CHUNKS)
    items = list(zip(found["documents"][0], found["metadatas"][0]))
    reply = ai.chat.completions.create(
        model=ASK_MODEL,
        messages=[
            {"role": "system", "content": ASK_PROMPT},
            {"role": "user",
             "content": f"Code excerpts:\n{build_material(items)}\n\nQuestion: {question}"},
        ],
    )
    return reply.choices[0].message.content, source_lines(items)


def make_impact_report(ticket):
    title = ticket.strip().splitlines()[0].replace("Title:", "").strip()
    searches = [
        ticket,
        f"{title} where the field is defined (data model, database, metadata)",
        f"{title} list screen columns and filters",
        f"{title} detail page layout",
        f"{title} existing data or feature that already covers this",
        f"{title} data that already exists or is imported from an outside service",
    ]
    picked = {}
    for query in searches:
        found = collection.query(query_texts=[query], n_results=CHUNKS_PER_SEARCH)
        for doc, meta in zip(found["documents"][0], found["metadatas"][0]):
            key = (meta["file"], meta["start_line"], meta["end_line"])
            if key not in picked:
                picked[key] = (doc, meta)
    items = list(picked.values())[:MAX_CHUNKS]

    reply = ai.chat.completions.create(
        model=IMPACT_MODEL,
        messages=[
            {"role": "system", "content": IMPACT_PROMPT},
            {"role": "user",
             "content": f"Code excerpts:\n{build_material(items)}\n\nJira ticket:\n{ticket}"},
        ],
    )
    report = reply.choices[0].message.content
    with open(REPORT_FILE, "w", encoding="utf-8") as f:
        f.write(report)
    return report, source_lines(items)


def default_ticket():
    try:
        with open(TICKET_FILE, encoding="utf-8") as f:
            return f.read()
    except Exception:
        return "Title: \n\nClient requirement: "


# ---------- Two tabs ----------
tab_ask, tab_ticket = st.tabs(["💬 Ask a question", "🎫 Jira ticket impact report"])

# ----- Tab 1: questions -----
with tab_ask:
    st.subheader("Ask how the product works")
    with st.form("ask_form"):
        question = st.text_input(
            "Your question",
            placeholder="For example: Where is a company created and what happens when it is created?",
        )
        ask_clicked = st.form_submit_button("Ask")

    if ask_clicked and question.strip():
        with st.spinner("Searching the code and writing the answer..."):
            try:
                text, sources = answer_question(question.strip())
                st.session_state["answer"] = (question.strip(), text, sources)
            except Exception as e:
                st.session_state["answer"] = None
                st.error(f"Something went wrong: {e}")

    saved = st.session_state.get("answer")
    if saved:
        q, text, sources = saved
        st.markdown(f"**Question:** {q}")
        st.markdown(text)
        st.markdown("**Sources the AI read:**")
        for line in sources:
            st.markdown(f"- `{line}`")
        st.caption("The source list above is the trustworthy part. "
                   "Line numbers written inside the answer are the AI's own guesses.")

# ----- Tab 2: Jira ticket -----
with tab_ticket:
    st.subheader("Paste a Jira ticket and get an impact report")
    ticket_text = st.text_area("Ticket text (this one is the fake demo ticket)",
                               value=default_ticket(), height=180)
    report_clicked = st.button("Create impact report")

    if report_clicked and ticket_text.strip():
        with st.spinner("Searching the code from 6 angles and writing the report "
                        "(this can take up to a minute)..."):
            try:
                report, sources = make_impact_report(ticket_text.strip())
                st.session_state["report"] = (report, sources)
            except Exception as e:
                st.session_state["report"] = None
                st.error(f"Something went wrong: {e}")

    saved_report = st.session_state.get("report")
    if saved_report:
        report, sources = saved_report
        st.markdown(report)
        st.download_button("Download report (.md)", report, file_name="impact_report.md")
        with st.expander("Code the AI was given"):
            for line in sources:
                st.markdown(f"- `{line}`")
        st.caption("This is a draft for BA review, not a final answer. "
                   "Search can miss related files that use different words.")
```

**test_ai.py** (Phase 0, works)
```python
import os
from openai import OpenAI

client = OpenAI(
    base_url="https://openrouter.ai/api/v1",
    api_key=os.environ["OPENROUTER_API_KEY"],
)

reply = client.chat.completions.create(
    model="openai/gpt-4o-mini",
    messages=[{"role": "user", "content": "Say hello in one short sentence."}],
)

print(reply.choices[0].message.content)
```

## How to start the demo (quick reminder)
1. Open VS Code, open the folder `C:\rag-demo`, open the terminal.
2. Check the line starts with `(.venv)`. If not, run `.\.venv\Scripts\Activate.ps1`.
3. Check the key: `if ($env:OPENROUTER_API_KEY) { "Key found" } else { "Key missing" }`
4. Run `streamlit run app.py`. The browser opens at `http://localhost:8501`. If Streamlit asks for an email on the first run, just press Enter.
5. To stop the app: click the terminal and press Ctrl+C.

## Problems and fixes
- **`pip is not recognized`:** use `python -m pip install ...`.
- **`No module named 'openai'`:** the install had failed. Reinstall inside `.venv`.
- **NumPy "Unknown compiler" error:** Python 3.15 is too new. Install 3.12.10 (`winget install -e --id Python.Python.3.12`) and use `py -3.12 -m venv .venv`.
- **Python 3.12 missing on python.org:** newer 3.12 releases have no Windows installer, so use winget or 3.12.10.
- **Only 3 files extracted by folder name / 2,351 files was too heavy:** fixed by matching keywords in the file path (167 files).
- **Chunks full of fake demo data:** fixed by skipping paths containing "seed" or "prefill".
- **`IndentationError` after hand-editing a script:** Python needs exact spaces at the start of lines. Replace the whole file with the full version instead of editing by hand.
- **Every new terminal must show `(.venv)`.** If not, run `.\.venv\Scripts\Activate.ps1`.
- **A new chat did not know our phases:** chats do not share memory. Start a new chat by attaching this file (see the first message you send).
- **`KeyError: 'sk-or-v1-...'` when running `ask.py`:** the API key itself had been typed inside `os.environ[...]` on line 12. That line must read exactly `api_key=os.environ["OPENROUTER_API_KEY"],` (the NAME of the Windows variable, never the key). Python prints the key in this error, which leaked it into chat. Fix: open the file, Ctrl+G to line 12, retype the line, then Ctrl+S.
- **Edits seem to have no effect:** the file was not saved. A white dot on the file's tab means unsaved; press Ctrl+S until it turns into an X.
- **Before pasting terminal output into chat:** scan it for anything starting with `sk-` and replace it with XXXX.
- **Setting the key:** run `setx OPENROUTER_API_KEY "your-key"` once, then close the terminal and open a NEW one (Windows only reads new variables in new terminals). Check with: `if ($env:OPENROUTER_API_KEY) { "Key found" } else { "Key missing" }`.
- **Streamlit page says "OPENROUTER_API_KEY variable was not found":** the terminal is old. Close it, open a NEW terminal, activate `.venv`, and run `streamlit run app.py` again.
- **Streamlit answer is thin or off-topic for a short question (e.g. "What is people?"):** not a bug. Search matches words, so a vague question pulls in the wrong files. Ask a specific question instead.

## Still to do
- **Replace the OpenRouter key.** The old key (starting `sk-or-v1-f909...`) was pasted into chat several times. Delete it at openrouter.ai/keys, create a new one, run `setx OPENROUTER_API_KEY "new-key"`, and open a fresh terminal. Never put the key in a file or in chat. (Not confirmed done yet.)
- **Phase 7 Presentation (parked, skipped for now):** 3 live demos, 5 slides, limitations slide. Use the "What we learned" section above for the limitations slide and the specific demo questions listed there.

## Stuck on right now
Nothing is broken.

## Next step
Phase 6 is done and Phase 7 is parked by choice. When ready, either replace the OpenRouter key first, or start **Phase 7: presentation** (3 live demos, 5 slides, limitations slide). Do not skip the key replacement before presenting.

**How we work:** the assistant writes every script in full, one small step at a time, and gives complete files rather than edits. The user runs it and pastes back the output or error (after scanning for `sk-`). Do not skip phases.
