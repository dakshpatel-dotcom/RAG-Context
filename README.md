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
- **Writer (AI): OpenRouter** (key has $5 credit), using the `openai` library and model `openai/gpt-4o-mini`. Key is stored in the Windows variable `OPENROUTER_API_KEY` (never paste it in chat).
- **Chat screen: Streamlit** (Phase 6).
- **Method: RAG** (look up relevant material first, then answer), no AI training needed.

## Done so far
1. Read the proposal and agreed the plan and 8 phases. Works.
2. **Phase 0 Setup:** VS Code, Python 3.12.10, `.venv`, installed `openai`, `chromadb`, `streamlit`. `test_ai.py` printed "Hello!". Works.
3. **Phase 1 Source code:** downloaded `twenty.zip`; `get_module.py` extracted 167 files into the `source` folder (`server` and `front` sub-folders). Works.
4. **Phase 2 Chop:** `chop.py` read 157 files and created **421 chunks** in `chunks.json`. Works.
5. **Phase 3 Index:** `build_index.py` stored 421 chunks in ChromaDB. Test searches returned the right files (e.g. `create-company.service.ts` for "where is a company created?"). Works.
6. **Phase 4 Writer:** `ask.py` is written (searches, then the AI answers with sources) but NOT run yet.

**Phase plan:** 0 Setup (done), 1 Source code (done), 2 Chop (done), 3 Index (done), 4 Writer with cited answers (script ready, not run), 5 Fake Jira ticket impact report, 6 Streamlit chat screen, 7 Presentation (3 live demos, 5 slides, limitations slide).

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

**ask.py** (Phase 4, written but not yet run)
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

## Stuck on right now
Nothing is broken. `ask.py` has not been run yet.

## Next step
Run `python ask.py` (with `(.venv)` showing) and try 2 or 3 questions, for example "Where is a company created and what happens when it is created?". Paste the answers back to check they match the listed source files. Then start **Phase 5: fake Jira ticket impact report**.

**How we work:** the assistant writes every script in full, one small step at a time, and gives complete files rather than edits. The user runs it and pastes back the output or error. Do not skip phases.
