# n8n Blog Image Pipeline

Batch-generate blog hero images with n8n + the OpenAI image API, driven by a Google Sheet manifest, with a mandatory human review gate before anything ships.

Built in a day. Generated 28 hero images for roughly $5. The images are AI-generated; the taste decisions are not.

## Architecture

```
Google Sheet (manifest)
        |
        v
       n8n
        |
        v
OpenAI Images API (gpt-image-1)
        |
        v
Google Drive staging folder ("generated")
        |
        v
Human review  -->  approved folder
        |
        v
Manual copy into the website repo
```

n8n never touches the website repo. Nothing is published automatically. API keys live only in n8n's credential store.

## The manifest

One spreadsheet row per image. Columns:

| Column | Purpose |
|---|---|
| `slug` | Unique ID for the post/image; used to match write-backs |
| `title` | Post title (human reference) |
| `lane` | Style lane: `photo` or `illustration` (see below) |
| `filename` | Output PNG filename |
| `scene_prompt` | ALL per-image direction: who's in frame, emotion, composition |
| `alt_text` | Accessibility text for the website |
| `status` | Lifecycle: `ready` → `generated` → `approved` |
| `drive_link` | Written back by the workflow after upload |

**Status lifecycle:** only rows with status `ready` are processed. After generation, the workflow sets status to `generated` and fills in `drive_link`. A human reviews and either sets `approved`, or flips the row back to `ready` (optionally with an edited `scene_prompt`) to regenerate. Rejects cost cents to redo.

See `manifest-template.csv` for a starter with example rows.

## The two-lane style system

Series consistency does not require one uniform style. It requires a consistent *palette, light, and mood*. This pipeline uses two lanes:

- **`photo`** — photorealistic, for human-story posts
- **`illustration`** — warm textured illustration, for conceptual posts that don't photograph well

The `lane` column on each row selects one of two style blocks defined in a single Code node ("Build Prompt"). The style block is appended identically to every request, so there is zero prompt drift across the series. Per-image direction lives entirely in `scene_prompt`.

**Key lesson:** image "policy" lives in the prompts, not in model settings. Between our worst run (technically perfect, emotionally dead) and our best run, nothing in the workflow changed — only the words. Scene prompts that name who's in frame and direct the emotion ("gap-toothed first-day grin," "mid-laugh") are what separate "contains a face" from "alive."

## The node chain

```
Manual Trigger
  → Google Sheets (read manifest)
  → Filter (status = ready)
  → SplitInBatches (1 at a time)
  → Code: "Build Prompt" (append per-lane style block)
  → HTTP Request: POST https://api.openai.com/v1/images/generations
      model: gpt-image-1 | size: 1536x1024 | quality: high
      response: base64 | 3 retries | 180s timeout
  → Convert base64 → binary PNG (filename from the row)
  → Google Drive upload (staging folder)
  → Google Sheets write-back (match on slug; set status=generated + drive_link)
  → Wait 2s
  → loop
```

## Setup checklist

1. **Import** `workflow.json` into n8n.
2. **Credentials** — create and attach your own:
   - OpenAI API key (in n8n's credential store, nowhere else)
   - Google Sheets OAuth
   - Google Drive OAuth
   (Every credential in the shipped workflow is a `REPLACE_ME_AFTER_IMPORT` placeholder.)
3. **Sheet ID** — replace `REPLACE_ME_SHEET_DOCUMENT_ID` in **both** Google Sheets nodes (read and write-back). Missing one is a classic silent failure.
4. **Drive folder ID** — replace `REPLACE_ME_GENERATED_FOLDER_ID` in the Drive upload node.
5. **Pick the sheet tab From-list, not By-Name.** If you imported a CSV, the tab is named after the file, and "Sheet1" won't exist (see Troubleshooting).
6. **Write-back mapping** — in the Sheets write-back node: match on `slug`, and confirm the mapped fields are in **Expression mode** (you should see a resolved preview under each field). Pasted `{{ }}` in a fixed-mode field sends the literal braces.
7. **Pilot rule:** first run = 5 images only. API rendering can differ from chat-UI rendering; validate before spending on the full batch.

## Running it

1. Fill manifest rows; set status to `ready`.
2. Hit the Manual Trigger in n8n.
3. Watch the staging folder fill. ~20–40s per image; a 28-image run takes ~12–17 minutes with the 2-second spacing.
4. Review in Drive: move keepers to your `approved` folder, flip rejects back to `ready` (edit the scene prompt if needed), re-run.
5. Manually copy approved images into your site repo.

## Cost & timing

- ~20–40 seconds per image at 1536×1024, high quality
- ~$4–6 for a full 28-image run
- Regenerating a single reject: cents

## Troubleshooting (every one of these was actually hit)

**OpenAI: "project does not have access to model gpt-image-1."** This can be *three stacked causes*, and fixing one may not be enough:
1. Organization not verified — complete the Persona ID check (takes minutes).
2. The project's model **allowlist** excludes the model — add it in project settings.
3. An API key minted *before* verification never inherits the new entitlement — mint a **fresh key**. This was the final fix.

**Google Drive OAuth: "authorization grant invalid / expired / revoked."** The refresh token is dead; delete and recreate the credential. Self-hosted n8n note: a Google Cloud consent screen left in "Testing" status expires refresh tokens **every 7 days** — publish the consent screen to stop the churn.

**"Column to Match On is required."** Swapping the manifest to a new spreadsheet silently **wipes the Sheets node's column mapping**. Re-map (match on `slug`) after any document change.

**Loop halts after image one, with weird literal `{{ }}` values.** The field is in fixed mode. Toggle it to **Expression mode**; you should see a resolved preview under the field.

**"Sheet1 not found" after CSV import.** The tab gets named after the imported file. Pick the tab via **From-list**, not By-Name.

## Sample outputs

Two sample images are in `samples/` — one from each lane. All outputs are AI-generated (OpenAI gpt-image-1) and reviewed by a human before use.

## License

MIT — see `LICENSE`.
