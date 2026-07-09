---
name: summarize-document
description: Use this skill whenever the user uploads, references, or attaches a document (PDF, DOCX, TXT, or plain text) and wants a summary, recap, TL;DR, or key-points extraction. Trigger this even if the user just says "summarize this" or "what's in this file" without more detail. Do not use this skill for tasks that only involve moving or uploading a file with no summarization requested — use upload-to-drive for that instead.
---

# Summarize Document

## When to use
- The user has attached or referenced a file and wants its content condensed
- The user asks for "key points," "highlights," "TL;DR," or "what does this say"
- A document needs a summary before being routed/uploaded (see upload-to-drive)

## Instructions

1. **Read the full document content** before summarizing. Do not summarize based on the filename alone.
2. If the file is a PDF or image-based document, read it directly (Gemini accepts PDFs/images as multimodal input) — do not guess at contents.
3. Identify the document type first (e.g., invoice, report, contract, meeting notes, article, research paper). Adjust the summary style to fit:
   - **Reports/articles** → thematic summary + key findings
   - **Contracts/invoices** → key terms, amounts, dates, obligations
   - **Meeting notes** → decisions made, action items, owners
   - **Research/technical docs** → problem, method, result, in that order
4. Never invent facts, numbers, or names not present in the source document.
5. If the document is very long (multi-page), summarize section by section internally, then produce one consolidated summary — do not truncate and summarize only the first page.
6. If the document is unreadable, corrupted, or empty, say so plainly instead of producing a fabricated summary.

## Output format

Always return the summary in this structure:

```
**Document:** [filename or inferred title]
**Type:** [invoice / report / contract / notes / article / other]

**Summary:**
[2-5 sentence plain-language summary]

**Key points:**
- [point 1]
- [point 2]
- [point 3]

**Action items / dates / amounts (if applicable):**
- [item]
```

If the document has no action items, dates, or amounts, omit that section entirely rather than leaving it empty.

## Notes
- Keep the summary shorter than the original document — never reproduce large verbatim blocks of the source text.
- This skill only produces the summary text. If the user also wants the file (or the summary) uploaded somewhere, hand off to the upload-to-drive skill after this step.
