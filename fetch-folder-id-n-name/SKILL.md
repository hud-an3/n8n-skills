---
name: fetch-folder-id-n-name
description: Use this skill whenever a file has been uploaded and needs to be assigned to a Google Drive destination folder — whether the destination is determined automatically from the filename, or the user names a folder directly in their prompt.
---

# Fetch Folder ID and Name

## Valid destination folders
Only four folder names are ever valid outputs of this skill:
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
- If it matches one of `folder_1` / `folder_2` / `folder_3` / `default_folder` → that's your target folder name. Skip Step 1B.
- If it does **not** match any valid folder → target folder name is `default_folder`. Do not create a new folder. Do not guess the closest match. Set `usedDefault` to `true` in your final output and include a `note` explaining the requested folder wasn't recognized.

**Case B: No folder named by the user — infer from the filename**

Check the filename against these patterns, **in this exact order**, and stop at the first match:

1. Contains `_1` **not** followed by another digit → `folder_1`
2. Contains `_2` **not** followed by another digit → `folder_2`
3. Contains `_3` **not** followed by another digit → `folder_3`

("Not followed by another digit" excludes things like `_10`, `_12`, `_23` from matching `_1`/`_2`/`_3` — e.g. `report_12.pdf` should NOT match `_1`.)

If none of the three patterns match → target folder name is `default_folder`. Do not ask the user to pick a folder — route silently to `default_folder`, and set `usedDefault` to `true` with a `note` explaining no filename pattern matched.

## Step 2 — Resolve the folder name to a folder id

Use your Drive folder search tool to retrieve all folders in Google Drive.

From the results, find the folder whose name matches the target folder name from Step 1, using a **case-insensitive, trimmed** comparison (e.g. `" Folder_1 "` matches `folder_1`).

- If a match is found → that folder's id and name are your result.
- If no match is found for a non-default target (e.g. `folder_2` doesn't actually exist in Drive) → look for `default_folder` instead and use that as the fallback. Set `usedDefault` to `true` and include a `note` explaining the originally targeted folder wasn't found in Drive.
- If even `default_folder` can't be found in the search results → stop. Do not invent an id. Return `folderId` as an empty string and put a clear explanation in `note` (e.g. "No matching folder, including default_folder, exists in Drive").

## Step 3 — Return the result

Return **only** a single valid JSON object — no markdown fences, no prose before or after, no explanation of what you did.

Schema:
```json
{
  "action": "upload",
  "folderId": "string",
  "folderName": "string",
  "usedDefault": true,
  "note": "string, optional — only include if usedDefault is true"
}
```

## Critical field rules
- `action` must always be the literal string `"upload"` when this skill is used — this is what lets the workflow route your output correctly.
- `folderId` and `folderName` refer to the single assigned folder only — never return the full list of folders you searched through.
- Set `usedDefault` to `true` whenever `default_folder` was used as a fallback for any reason (Step 1A mismatch, Step 1B no pattern match, or Step 2 fallback). Set it to `false` when the originally intended folder was matched directly.
- Include `note` only when `usedDefault` is `true` — omit it entirely otherwise.
- Never include `summary` in this output — that field belongs only to the summarize-document skill's output shape.
- Do not nest the object under any outer key such as `"output"`, `"result"`, or `"response"`. Return `action`, `folderId`, `folderName`, `usedDefault`, and `note` directly at the top level, exactly as shown in the schema.
- Do not wrap the JSON in ```json code fences — return raw JSON only, since the downstream Structured Output Parser expects to parse it directly.
- **JSON must be valid** — no trailing commas after the last property in the object.

## Example output (normal match)

```json
{
  "action": "upload",
  "folderId": "1AbCdEfGhIjKlMnOpQrStUvWxYz",
  "folderName": "folder_2",
  "usedDefault": false
}
```

## Example output (fallback used)

```json
{
  "action": "upload",
  "folderId": "1XyZaBcDeFgHiJkLmNoPqRsTuV",
  "folderName": "default_folder",
  "usedDefault": true,
  "note": "Requested folder name 'invoices_2024' not recognized, routed to default_folder"
}
```

---

## Appendix — Structured Output Parser schema (reference only), this is just an example 

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
      "description": "Optional explanatory note. Populate ONLY when action is 'upload' and usedDefault is true. Never used for summarize action."
    }
  },
  "required": ["action"]
}
```