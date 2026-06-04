# Discharge Summary Agent — Part 1

## Overview

An agentic AI system that reads raw patient source-note PDFs and produces a structured, clinically safe discharge summary draft for clinician review.

---

## How to Run

### 1. Open in Google Colab
Upload `discharge_summary_agent.ipynb` to [colab.research.google.com](https://colab.research.google.com).

### 2. Set your API key
In **Step 2**, set:
```python
OPENAI_API_KEY = "sk-your-key-here"
```
Or set it as a Colab secret / environment variable.

### 3. Upload patient PDFs
Create a folder `/content/patient_1/` and upload all PDFs for that patient (admission note, progress notes, lab results, medication records — any combination).

### 4. Run all cells
Run top to bottom. The agent will:
- Extract text from all PDFs
- Plan and execute steps dynamically
- Save outputs to `/content/outputs/`

### 5. Multi-patient batch
Uncomment and set folders in the last cell:
```python
results = run_batch(["/content/patient_1", "/content/patient_2"])
```

---

## Agent Loop Design

The agent uses a **plan → execute → update → repeat** loop:

```
load_all_pdfs()
    ↓
for step in range(MAX_STEPS):
    action, reasoning = plan_next_action(memory)   ← LLM decides
    result = execute_tool(action, memory, corpus)
    update memory
    emit trace step
    if action == "DONE": break
```

**What makes it a real agent:** The `plan_next_action()` function asks the LLM to look at the current memory state and decide what to do next. The LLM can:
- Prioritize missing safety-critical fields
- Decide when conflicts or medication changes need escalation
- Choose to run reconciliation only after both med lists are collected
- Decide when it is safe to generate the final summary

The order is **not hardcoded** — the LLM plans dynamically based on what's in memory.

---

## No-Fabrication Guardrail

Every extraction prompt contains explicit rules:
```
1. Extract ONLY information explicitly stated in the text.
2. Never infer, guess, or fill in plausible values.
3. If the information is absent, return exactly: MISSING
4. If a result is pending/awaited, return: PENDING — <describe>
```

`temperature=0.0` is set on all LLM calls to minimize drift.

Any field not found in source documents is marked `MISSING — not found in source documents` in the summary. The output is always labelled **DRAFT FOR CLINICIAN REVIEW**.

---

## Failure Handling

- **LLM failures:** `call_llm()` retries 3 times with exponential backoff. Returns empty string on total failure.
- **Tool failures:** `execute_tool()` wraps every tool in try/except. Failures are caught, logged, and flagged for clinician review — the agent never crashes.
- **Step cap:** `MAX_STEPS = 20` is a hard ceiling. If hit, a flag is added and a partial summary is generated.
- **Empty PDFs:** If OCR returns nothing, the agent reports and aborts cleanly.

---

## Conflict & Reconciliation Handling

**Conflict detection:** `tool_detect_conflicts()` asks the LLM to compare all documents and identify disagreements (different diagnoses, different doses, conflicting values). Every conflict is added to `clinician_flags`.

**Medication reconciliation:** `tool_reconcile_medications()` compares admission vs discharge medication lists and returns:
- Added medications
- Stopped medications  
- Changed medications
- Changes with no documented reason (flagged for review)

**Drug interactions:** `tool_drug_interaction_check()` is a mocked external tool that checks discharge medications for known interaction pairs. In production this would call DrugBank or OpenFDA.

---

## Output Files (per patient)

| File | Contents |
|------|----------|
| `{id}_discharge_summary.txt` | Structured summary draft |
| `{id}_trace.json` | Full step-by-step trace (reasoning → action → result) |
| `{id}_flags.txt` | Clinician review flags only |
| `{id}_memory.json` | Full extracted memory in JSON |

---

## Trace Format

Every step is logged as:
```json
{
  "step": 3,
  "reasoning": "Demographics and dates collected. Diagnoses missing — extract next.",
  "action": "extract_diagnoses",
  "memory_key": "diagnoses",
  "result": "1) Acute Gastroenteritis with Dehydration\n2) Urinary Tract Infection",
  "flags_so_far": 1
}
```

---

## Limitations & What I'd Do with More Time

1. **Context window:** Long PDFs are truncated to 6000 chars per LLM call. With more time I'd chunk documents by section and route each chunk to the relevant extractor.

2. **Planner reliability:** The LLM planner can occasionally output an unrecognised tool name — the fallback catches this but a structured output / function-calling approach would be more robust.

3. **Drug interaction tool is mocked:** A real integration with DrugBank or OpenFDA would cover the full formulary.

4. **No Part 2 (learning from edits):** Would implement a simulated reviewer + DPO fine-tuning or correction-memory injection with before/after edit-distance metrics.

5. **Evaluation:** With more time I'd build a test harness with synthetic patients covering edge cases: missing admission meds, conflicting diagnoses, all-pending labs.
