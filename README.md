# Project: RAG-Context (Product Co-Pilot demo)

## Goal
Build a small working demo of our "Product Co-Pilot" using an open-source CRM as the sample code.
You ask a question (or paste a fake Jira ticket) and it answers in plain English, naming the exact files it used.
The demo will be presented to the team to show what can be built.

## Tools and choices
- **Demo project: Twenty CRM** (github.com/twentyhq/twenty). Real open-source CRM code. Fallback if too heavy: EspoCRM.
- **Scope: one module only** (e.g. company / person / opportunity). Matches the "pilot on one module" pitch.
- **Computer: Windows**, working in `C:\rag-demo` using VS Code and its built-in terminal.
- **Language: Python 3.12.10** (NOT 3.15, libraries don't support it yet), inside a private environment called `.venv`.
- **Chopper: a short Python script.** Cuts files into pieces tagged with file names.
- **Index and search: ChromaDB.** Free, finds the most relevant pieces.
- **Writer (AI): OpenRouter** (key already has $5 credit), using the `openai` Python library and model `openai/gpt-4o-mini`. Key is stored in the Windows variable `OPENROUTER_API_KEY` (never paste it in chat).
- **Chat screen: Streamlit.**
- **Method: RAG** (look up relevant material first, then answer), no AI training needed.

## Done so far
1. Read the proposal and agreed the plan and 8 phases. Works.
2. Phase 0 Setup: installed VS Code, created `C:\rag-demo`, installed Python 3.12.10 using winget or a direct download. Works.
3. Created the `.venv` environment with `py -3.12 -m venv .venv`. Works, `(.venv)` shows in the terminal.
4. Installed `openai`, `chromadb`, `streamlit` with `python -m pip install ...`. Works.
5. Ran `test_ai.py` and the AI replied "Hello!". Works. **Phase 0 is complete.**

**Phase plan:** 0 Setup (done), 1 Get source code, 2 Chop into chunks, 3 ChromaDB index, 4 Writer with cited answers, 5 Fake Jira ticket impact report, 6 Streamlit chat screen, 7 Presentation.

## Current code
**test_ai.py**
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
- **`pip is not recognized`:** use `python -m pip install ...` instead of `pip install ...`.
- **`No module named 'openai'`:** the install had failed. Fixed by reinstalling inside `.venv`.
- **NumPy "Unknown compiler" error:** Python 3.15 is too new, so pip tried to build libraries from scratch. Fixed by installing Python 3.12.10 and using `py -3.12 -m venv .venv`.
- **Python 3.12 not on python.org downloads page:** newer 3.12 releases have no Windows installer. Use 3.12.10 via `winget install -e --id Python.Python.3.12`.
- **Reminder:** every new terminal must show `(.venv)` at the start. If not, run `.\.venv\Scripts\Activate.ps1`.

## Stuck on right now
Nothing.

## Next step
**Phase 1:** download Twenty CRM from GitHub as a zip and extract only the one module we'll use into `C:\rag-demo\source`.
