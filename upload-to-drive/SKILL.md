---
name: upload-to-drive
description: Use this skill whenever the user wants a file saved, uploaded, moved or filed into Google Drive. Trigger on phrases like "upload this," "save it to drive," "file this away," or whenever a file needs to be routed to a specific folder based on its filename or content type. Do not use this skill just to read or summarize a file — use summarize-document for that.
---

# Upload to Google Drive

## When to use
- The user has a file (raw upload or the output of a prior step like summarize-document) that needs to end up in Google Drive
- The destination folder should be chosen automatically based on filename pattern or document type, not asked from the user every time

Determine the destination folder by checking the filename against these patterns, in order. Use the first match. A match requires `_1`, `_2`, or `_3` to appear in the filename immediately followed by a non-digit character or the end of the filename (so `report_1.pdf` matches, but `report_10.pdf` does not):

| Filename pattern | Destination folder |
|---|---|
| contains `_1` (not followed by another digit) | `folder_1` |
| contains `_2` (not followed by another digit) | `folder_2` |
| contains `_3` (not followed by another digit) | `folder_3` |
| *(no match)* | `default_folder` |

Only `folder_1`, `folder_2`, and `folder_3` are valid destination folders — if pattern matching somehow resolves to anything else, fall back to `default_folder` rather than uploading to an unrecognized folder.

Check patterns in the order listed above — `_1` before `_2` before `_3` — and stop at the first match. If a filename matches none of `_1`/`_2`/`_3`, route it to `default_folder` without asking the user for a folder.

## If the user names a folder directly

If the user explicitly asks to upload to a specific folder by name (e.g. "put this in the Marketing folder"), check that name against the valid folder list (`folder_1`, `folder_2`, `folder_3`, `default_folder`). If it matches one of those, use it. If it does **not** match — it's a folder that doesn't exist in this system — do not create a new folder and do not guess the closest match. Upload to `default_folder` instead, and tell the user the requested folder wasn't recognized and the file was placed in `default_folder`.

## Instructions

1. Determine the target folder using the rules above.
2. If a summary was produced by the summarize-document skill in this same request, decide whether the user wants:
   - the original file uploaded, or
   - the summary uploaded as a new file (e.g. `filename-summary.txt`), or
   - both
   If this isn't clear from the request, default to uploading the original file only.
3. Check whether the destination folder already exists before uploading. If it doesn't exist, create it rather than failing or uploading to root.
4. Never overwrite an existing file with the same name — append a short suffix (e.g. `-v2`, or a timestamp) instead, unless the user explicitly asks to replace/overwrite.
5. After uploading, confirm back to the user with the exact folder path and filename used — don't just say "uploaded successfully."

## Output format

After a successful upload, respond with:

```
Uploaded **[filename]** → [Folder Path]
[Drive link if available]
```

If the upload fails, state the specific reason (auth error, folder not found, file too large, etc.) rather than a generic failure message.