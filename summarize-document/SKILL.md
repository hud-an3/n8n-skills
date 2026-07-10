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
2. If the file is a PDF or image-based document, read it directly (Gemini accepts PDFs/images as multimodal input) — do not guess at contents.
3. Identify the document type first (e.g., invoice, report, contract, meeting notes, article, research paper). Adjust the summary style to fit:
   - **Reports/articles** → thematic summary + key findings
   - **Contracts/invoices** → key terms, amounts, dates, obligations
   - **Meeting notes** → decisions made, action items, owners
   - **Research/technical docs** → problem, method, result, in that order
4. **Grounding rule (strict):** Every fact, number, name, date, or claim in your summary must be traceable to text that actually appears in the document. If you are not sure a detail is present, leave it out rather than include it. Do not fill gaps with plausible-sounding inference.
5. If the document is very long (multi-page), summarize section by section internally, then produce one consolidated summary — do not truncate and summarize only the first page.
6. If the document is unreadable, corrupted, empty, or you were only given a filename/ID with no actual content, set `summary` to a plain statement of that fact (e.g. "The document could not be read — no content was provided") instead of producing a fabricated summary.
7. Keep the summary shorter than the original document — never reproduce large verbatim blocks of the source text.

## Output format — STRICT

You must respond with **only** a single valid JSON object. No markdown fences, no prose before or after, no explanation of what you did.

Schema:
```json
{
  "action": "summarize",
  "summary": "string — the full formatted summary as described below"
}
```

The `summary` field itself should contain the following structure as a single string (use `\n` for line breaks):

If the document has no action items, dates, or amounts, omit that section entirely rather than leaving it empty.

`action` must always be the literal string `"summarize"` when this skill is used — this is what lets the workflow route your output correctly. Never include `file_id` or `folder_name` fields; those belong only to the fetch-folder-id-n-name skill's output shape.

## Notes
- This skill only produces the summary text. If the user also wants the file (or the summary) uploaded somewhere, hand off to the fetch-folder-id-n-name skill after this step.
- Do not wrap the JSON in ```json code fences — return raw JSON only, since the downstream Structured Output Parser expects to parse it directly.