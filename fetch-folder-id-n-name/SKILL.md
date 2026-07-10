---
name: fetch-folder-id-n-name
description: Use this skill whenever a file has been uploaded and needs to be assigned to and uploaded into a Google Drive destination folder — whether the destination is determined automatically from the filename, or the user names a folder directly in their prompt. This skill both resolves the folder AND performs the upload itself.
---

# Fetch Folder ID and Upload File

## Tools this skill uses
- **Search files and folders in Google Drive** — used twice: once to find the target folder, once to find the file to be uploaded.
- **Upload file in Google Drive** — used once, after both the folder and file have been resolved, to actually perform the upload.

## Valid destination folders
Only four folder names are ever valid targets for this skill:
- `folder_1`
- `folder_2`
- `folder_3`
- `default_folder`

If your reasoning ever lands anywhere else, that's a signal you made an error — fall back to `default_folder` instead.

## Step 1 — Determine the target folder name

**Case A: User explicitly names a folder in their prompt**
(e.g. "put this in the Marketing folder", "upload to folder_2")

- Take the folder name the user gave, trimmed and compared case-insensitively.
- Check it against the valid folder list above.
- If it matches one of `folder_1` / `folder_2` / `folder_3` / `default_folder` → that's your target folder name. Skip Case B.
- If it does **not** match any valid folder → target folder name is `default_folder`. Do not create a new folder. Do not guess the closest match. Mark this as a fallback (see Step 5).

**Case B: No folder named by the user — infer from the filename**

Check the filename against these patterns, **in this exact order**, and stop at the first match:

1. Contains `_1` **not** followed by another digit → `folder_1`
2. Contains `_2` **not** followed by another digit → `folder_2`
3. Contains `_3` **not** followed by another digit → `folder_3`

("Not followed by another digit" excludes things like `_10`, `_12`, `_23` from matching `_1`/`_2`/`_3` — e.g. `report_12.pdf` should NOT match `_1`.)

If none of the three patterns match → target folder name is `default_folder`. Do not ask the user to pick a folder — route silently to `default_folder`. Mark this as a fallback (see Step 5).

## Step 2 — Resolve the folder name to a folder id

Use the **Search files and folders in Google Drive** tool to retrieve all folders in Google Drive.

From the results, find the folder whose name matches the target folder name from Step 1, using a **case-insensitive, trimmed** comparison (e.g. `" Folder_1 "` matches `folder_1`).

- If a match is found → that folder's id and name are your resolved folder.
- If no match is found for a non-default target (e.g. `folder_2` doesn't actually exist in Drive) → search again for `default_folder` and use that as the fallback. Mark this as a fallback (see Step 5).
- If even `default_folder` can't be found in the search results → **stop here**. Do not invent a folder id. Do not attempt the upload. Go straight to Step 6 and report the failure.

## Step 3 — Find the file to upload

Use the **Search files and folders in Google Drive** tool again — this time searching for the actual file that needs to be uploaded (by the filename/reference provided in the task input).

- If exactly one matching file is found → that file's id and name are your resolved file.
- If multiple files match → pick the most recently modified one, and note this ambiguity in your final report (Step 6).
- If no file is found → **stop here**. Do not attempt the upload. Go straight to Step 6 and report that the source file could not be located.

## Step 4 — Upload the file

Once you have both the resolved folder (Step 2) and the resolved file (Step 3), use the **Upload file in Google Drive** tool to upload the file into that folder.

- If the upload tool reports success → proceed to Step 6 and report success.
- If the upload tool reports an error → do not retry silently more than once. If it fails again, stop and report the exact failure reason in Step 6.

## Step 5 — Track fallback usage

Keep track internally of whether `default_folder` was used as a fallback at any point (Step 1A mismatch, Step 1B no pattern match, or Step 2 fallback). You'll need this for the final output.

## Step 6 — Return the result

Return **only** a single valid JSON object — no markdown fences, no prose before or after, no explanation of what you did outside this object.

Schema:
```json
{
  "action": "upload",
  "folderId": "string",
  "folderName": "string",
  "usedDefault": true,
  "note": "string — always include; describe what was searched, found, and the upload outcome"
}
```

## Critical field rules
- `action` must always be the literal string `"upload"` when this skill is used — this is what lets the workflow route your output correctly.
- `folderId` and `folderName` refer to the single resolved destination folder only — never return the full list of folders you searched through.
- Set `usedDefault` to `true` whenever `default_folder` was used as a fallback for any reason. Set it to `false` when the originally intended folder was matched directly.
- `note` is **always required** in this skill's output (not just on fallback) — it must summarize what happened: which file was found, which folder was resolved, whether the upload succeeded, and any fallback or ambiguity encountered. If the flow stopped early (no folder found, no file found, or upload failed), `note` must state exactly what failed and why.
- If the flow stopped early and no upload occurred, still return `action: "upload"`, but set `folderId` and/or `folderName` to empty strings for whichever step failed to resolve, and make the failure explicit in `note`.
- Never include `summary` in this output — that field belongs only to the summarize-document skill's output shape.
- Do not nest the object under any outer key such as `"output"`, `"result"`, or `"response"`. Return `action`, `folderId`, `folderName`, `usedDefault`, and `note` directly at the top level, exactly as shown in the schema.
- Do not wrap the JSON in ```json code fences — return raw JSON only, since the downstream Structured Output Parser expects to parse it directly.
- **JSON must be valid** — no trailing commas after the last property in the object.

## Example output (normal match, upload succeeded)

```json
{
  "action": "upload",
  "folderId": "1AbCdEfGhIjKlMnOpQrStUvWxYz",
  "folderName": "folder_2",
  "usedDefault": false,
  "note": "Found file 'invoice_2.pdf' and uploaded it to folder_2 successfully."
}
```

## Example output (fallback used, upload succeeded)

```json
{
  "action": "upload",
  "folderId": "1XyZaBcDeFgHiJkLmNoPqRsTuV",
  "folderName": "default_folder",
  "usedDefault": true,
  "note": "Requested folder name 'invoices_2024' not recognized. Routed to default_folder instead. Found file 'invoice_2024_report.pdf' and uploaded it successfully."
}
```

## Example output (upload failed)

```json
{
  "action": "upload",
  "folderId": "1AbCdEfGhIjKlMnOpQrStUvWxYz",
  "folderName": "folder_1",
  "usedDefault": false,
  "note": "Resolved folder_1 successfully, but the file 'contract_1.pdf' could not be located in Drive search results. Upload was not attempted."
}
```

---

## Appendix — Structured Output Parser schema (reference only)

This skill's output is validated downstream by a Structured Output Parser node using a schema shared with the summarize-document skill. This is shown here for reference so you understand why other fields exist — you must still only populate `action`, `folderId`, `folderName`, `usedDefault`, and `note` as instructed above, never the `summary` field.

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
      "description": "Always populate when action is 'upload' — summarizes what was found and whether upload succeeded. Never used for summarize action."
    }
  },
  "required": ["action"]
}
```