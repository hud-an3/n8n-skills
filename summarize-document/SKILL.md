---
name: summarize-document
description: Use this skill whenever the user uploads, references, or attaches a document (PDF, DOCX, TXT, or plain text) and wants a summary, recap, TL;DR, or key-points extraction. Trigger this even if the user just says "summarize this" or "what's in this file" without more detail. Do not use this skill for tasks that only involve moving or fetching id and name a file with no summarization requested — use fetch-folder-id-n-name for that instead.
---

# Summarize Document

## When to use
- The user has attached or referenced a file and wants its content condensed
- The user asks for "key points," "highlights," "TL;DR," or "what does this say"
- A document needs a summary before being routed/uploaded (see upload-to-drive)

## Instructions

1. **Read the full document content** before summarizing. Do not summarize based on the filename alone, and do not summarize based on partial content if more is available — if the input appears truncated, note that in your summary rather than treating it as the whole document.
2. **Hard rule — no filename-based summaries:** A filename (e.g. "Noor_resume.pdf") tells you nothing about what's inside the file. You must base every sentence of the summary on the actual extracted text/pages of the document, not on what the filename implies the document is likely to contain. If you catch yourself writing a summary that could apply to any file with that name — without citing specific content from inside it — stop and re-read the actual document content before continuing.
3. If the file is a PDF or image-based document, read it directly (Gemini accepts PDFs/images as multimodal input) — do not guess at contents. If no document content/binary was actually passed to you (only a filename or reference), do not summarize — follow instruction 6 instead.
4. Identify the document type first (e.g., invoice, report, contract, meeting notes, article, research paper) **based on what the content actually is**, not what the filename suggests. Adjust the summary style to fit:
   - **Reports/articles** → thematic summary + key findings
   - **Contracts/invoices** → key terms, amounts, dates, obligations
   - **Meeting notes** → decisions made, action items, owners
   - **Research/technical docs** → problem, method, result, in that order
5. **Grounding rule (strict):** Every fact, number, name, date, or claim in your summary must be traceable to text that actually appears in the document. If you are not sure a detail is present, leave it out rather than include it. Do not fill gaps with plausible-sounding inference, and do not fill gaps with assumptions based on the filename or file type.
6. If the document is very long (multi-page), summarize section by section internally, then produce one consolidated summary — do not truncate and summarize only the first page.
7. If the document is unreadable, corrupted, empty, or you were only given a filename/ID with no actual content, set `summary` to a plain statement of that fact (e.g. "The document could not be read — no content was provided") instead of producing a fabricated summary.
8. Keep the summary shorter than the original document — never reproduce large verbatim blocks of the source text.

## Response format — STRICT

You must respond with **only** a single valid JSON object. No markdown fences, no prose before or after, no explanation of what you did.

Schema:
```json
{
  "action": "summarize",
  "summary": "string — the full formatted summary, see template below"
}
```

The `summary` field must contain this exact template as a single string (use `\n` for line breaks):

If the document has no action items, dates, or amounts, omit that section entirely rather than leaving it empty.

## Critical field rules
- `action` must always be the literal string `"summarize"` when this skill is used — this is what lets the workflow route your output correctly.
- The actual summary content goes **only** in the `summary` field — never in a field called `note`.
- Do **not** include `folderId`, `folderName`, `usedDefault`, or `note` in your response under any circumstances. Those fields belong exclusively to a different task (folder lookup) and must never appear in output from this skill, even if you see them listed as optional in a schema.
- Do not nest the object under any outer key such as `"output"`, `"result"`, or `"response"`. Return `action` and `summary` directly at the top level, exactly as shown in the schema.
- Do not wrap the JSON in ```json code fences — return raw JSON only, since the downstream Structured Output Parser expects to parse it directly.

## Example output

```json
{
  "action": "summarize",
  "summary": "**Document:** Noor_resume.pdf\n**Type:** other\n\n**Summary:**\nA one-page resume for a computer engineering graduate pursuing an MS in AI, with experience in AI/ML engineering, full-stack development, and freelance project delivery.\n\n**Key points:**\n- B.E. in Computer Engineering, MS in AI in progress\n- Core stack: PyTorch, TensorFlow, Hugging Face, LangChain, FastAPI\n- Portfolio includes DDPM, RT-DETR, and RAG-based projects"
}
```

## Notes
- This skill only produces the summary text. If the user also wants the file (or the summary) uploaded somewhere, hand off to the fetch-folder-id-n-name skill after this step.

---

## Appendix — Structured Output Parser schema (reference only), this is just an example 

This skill's output is validated downstream by a Structured Output Parser node using a schema shared with the fetch-folder-id-n-name skill. This is shown here for reference so you understand why other fields exist — you must still only populate `action` and `summary` as instructed above, never the upload-related fields.

```json
{
  "type": "object",
  "properties": {
    "action": {
      "type": "string",
      "enum": ["summarize", "upload"],
      "description": "Which task was performed"
    },
    "summary": {
      "type": "string",
      "description": "The full formatted document summary. Populate ONLY when action is 'summarize'. Leave absent otherwise."
    },
    "folderId": {
      "type": "string",
      "description": "Google Drive folder ID. Populate ONLY when action is 'upload'. Leave absent otherwise."
    },
    "folderName": {
      "type": "string",
      "description": "Google Drive folder name. Populate ONLY when action is 'upload'. Leave absent otherwise."
    },
    "usedDefault": {
      "type": "boolean",
      "description": "Whether the default folder was used. Populate ONLY when action is 'upload'."
    },
    "note": {
      "type": "string",
      "description": "Optional explanatory note. Populate ONLY when action is 'upload' and usedDefault is true. Never used for summarize action."
    }
  },
  "required": ["action"]
}
```