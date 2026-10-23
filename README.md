# Project: RAG-Context (Product Co-Pilot demo)

## Goal
Build a small working demo of our "Product Co-Pilot" using an open-source CRM (Twenty CRM) as the sample code.
You ask a question (or paste a fake Jira ticket) and it answers in plain English, naming the exact files it used.
The demo will be presented to the team to show what can be built.

## Tools and choices
- **Demo project: Twenty CRM** (github.com/twentyhq/twenty), downloaded as `twenty.zip`. No Git needed.
- **Scope: Companies, People, Opportunities** (found by keyword in file paths, 167 files). Matches the "pilot on one module" pitch.
- **Computer: Windows**, working in `C:\rag-demo` using VS Code and its built-in terminal.
- **Language: Python 3.12.10** (NOT 3.15, libraries don't support it yet), inside a private environment called `.venv`.
- **Chopper: `chop.py`.** Cuts files into 40-line pieces (8 lines overlap), each tagged with file name and line numbers.
- **Index and search: ChromaDB.** Free, finds the most relevant pieces.
- **Writer (AI): OpenRouter** (key has $5 credit), using the `openai` library and model `openai/gpt-4o-mini`. Key is stored in the Windows variable `OPENROUTER_API_KEY` (never paste it in chat).
- **Chat screen: Streamlit.**
- **Method: RAG** (look up relevant material first, then answer), no AI training needed.

## Done so far
1. Read the proposal and agreed the plan and 8 phases. Works.
2. **Phase 0 Setup:** installed VS Code, Python 3.12.10, made `.venv`, installed `openai`, `chromadb`, `streamlit`. `test_ai.py` printed "Hello!". Works.
3. **Phase 1 Get source code:** downloaded `twenty.zip`, then `get_module.py` extracted **167 files** into the `source` folder (sub-folders `server` and `front`). Works.
4. **Phase 2 Chop:** `chop.py` is written but NOT run yet.

**Phase plan:** 0 Setup (done), 1 Source code (done), 2 Chop into chunks (script ready, not run), 3 ChromaDB index, 4 Writer with cited answers, 5 Fake Jira ticket impact report, 6 Streamlit chat screen, 7 Presentation.

## Current code
**get_module.py** (works, keeps files whose path contains company, people, opportunity, etc.)
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

# Show where the files came from (first 2 folder levels)
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

**chop.py** (written, not yet run)
```python
import json
from pathlib import Path

SOURCE_FOLDER = Path("source")
OUTPUT_FILE = "chunks.json"

LINES_PER_CHUNK = 40   # size of each piece
OVERLAP = 8            # lines repeated between pieces so nothing is cut mid-thought

chunks = []
files_read = 0

for path in sorted(SOURCE_FOLDER.rglob("*")):
    if not path.is_file():
        continue

    try:
        text = path.read_text(encoding="utf-8")
    except Exception:
        continue  # skip anything that can't be read as text

    lines = text.splitlines()
    if not lines:
        continue

    files_read += 1
    # Use forward slashes so file names look the same everywhere
    file_name = path.relative_to(SOURCE_FOLDER).as_posix()

    start = 0
    while start < len(lines):
        end = min(start + LINES_PER_CHUNK, len(lines))
        piece = "\n".join(lines[start:end])

        # Ignore pieces that are nearly empty
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

print(f"Read {files_read} files")
print(f"Created {len(chunks)} chunks")
print()
print("--- Sample chunk ---")
sample = chunks[len(chunks) // 2]
print(f"File: {sample['file']}  (lines {sample['start_line']} to {sample['end_line']})")
print(sample["text"][:400])
```

**test_ai.py** (works, from Phase 0)
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
- **Only 3 files extracted by folder name:** Twenty keeps Company/People code spread across many folders. Fixed by matching keywords in the file path instead.
- **2,351 files was too heavy:** caused by the huge `object-record` folder. Fixed by the keyword approach (167 files).
- **Every new terminal must show `(.venv)`.** If not, run `.\.venv\Scripts\Activate.ps1`.

## Stuck on right now
Nothing.

## Next step
Run `python chop.py` and check it prints `Created X chunks` (expected 700 to 2,500) and that `chunks.json` appears. Then start **Phase 3: build the ChromaDB index**.
