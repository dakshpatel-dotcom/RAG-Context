# Project: RAG-Context (Product Co-Pilot demo)

## Goal
Build a small working demo of our "Product Co-Pilot" using an open-source CRM as the sample code.
You ask a question (or paste a fake Jira ticket) and it answers in plain English, naming the exact files it used.
The demo will be presented to the team to show what can be built.

## Tools and choices
- **Demo project: Twenty CRM** (github.com/twentyhq/twenty). Real, modern open-source CRM code. Fallback if too heavy: EspoCRM.
- **Scope: one module only** (e.g. Companies / People / Opportunities). Matches the "pilot on one module" pitch and keeps things small.
- **Language: Python.** Simple, and best supported for RAG tools.
- **Editor: VS Code.** Free and beginner friendly.
- **Chopper: a short Python script.** Cuts files into small pieces, each tagged with its file name.
- **Index and search: ChromaDB.** Free, stores the pieces and finds the most relevant ones.
- **Writer (AI): Claude API or a free local model.** NOT DECIDED YET (see "Stuck on right now").
- **Chat screen: Streamlit.** Turns the script into a simple web page.
- **Method: RAG** (look up relevant material first, then answer), so no AI training is needed.

## Done so far
1. Read the proposal document (problem, idea, RAG approach, pilot ask). Works: understood.
2. Agreed the plan: demo project, tools, and 8 phases (below). Works: plan is ready, no code has been run yet.

**Phase plan (we do them in order, one at a time):**
- Phase 0: Setup (Python, VS Code, project folder, "Hello World")
- Phase 1: Get the source code (download Twenty CRM, pick one module)
- Phase 2: Chop files into chunks ("created X chunks")
- Phase 3: Build the ChromaDB index (search test: "where is company created?")
- Phase 4: Add the Writer (cited answers in the terminal)
- Phase 5: Simulate a Jira ticket (impact report, the main demo moment)
- Phase 6: Simple chat screen with Streamlit
- Phase 7: Prepare the presentation (3 live demos, 5 slides, limitations slide)

## Current code
No code written yet. Code starts in Phase 0.

## Problems and fixes
None so far.

## Stuck on right now
Nothing is broken. Two answers are needed before Phase 0 can start:
1. Do you use **Windows or Mac**?
2. For the Writer: a small **paid API key** (a few dollars is enough for the whole demo) or a **free local model** (slower, less polished)?

## Next step
Answer the two questions above, then start **Phase 0: Setup**.

**How we work:** Claude writes every script in full. You copy it, run it, and report what happened. If stuck, say the phase number and paste the error or a screenshot. Don't skip phases.
