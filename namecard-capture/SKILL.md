---
name: namecard-capture
description: Capture business card / name card information from photos taken at events or conferences. Use this skill whenever the user uploads a photo of a business card, name card, or contact card, or mentions capturing/scanning/saving a name card or business card. Also trigger when the user says things like "I just met someone at [event]", "save this contact", "log this name card", "I got their card", or uploads a photo that appears to be a business card. The skill extracts contact details, logs them to either a Notion database (preferred, structured) or as labelled Gmail drafts (lightweight fallback), and drafts a follow-up email in Gmail referencing the event where they met. Handles multiple cards within a single session by reusing the event context, but re-asks for event information on a new session or after a long pause of more than 6 hours.
---

# Name Card Capture

Capture name card information from event/conference photos, log them to Notion (or Gmail labels as a fallback), and draft a personalised follow-up email.

## When this skill triggers

- User uploads a photo of a business card / name card
- User says something like "save this contact", "log this card", "just met someone at [event]"
- User mentions capturing card details from a conference or networking event

## Tool reality check

The Claude.ai Gmail connector can **create drafts and apply labels** but **cannot send email directly**. So every follow-up email this skill generates is saved as a draft for the user to review and send.

The Gmail connector cannot label drafts by their draft ID — drafts must be located via `search_threads` (filter `is:draft to:<email>`) and labelled via `label_thread` using the resulting thread ID.

The Google Drive connector is create-only (no append/update), which is why this skill uses Notion or Gmail labels for storage instead of Sheets/Docs.

## Workflow

### Step 1: Establish session context (once per session)

Before processing any cards, check whether session context has already been established **in the current session**. There are two things to establish:

**A. Event context**
- **If no event context yet**: Ask which event/conference this is, and the date if not today.
- **If already established**: Reuse silently. Do NOT re-ask.

**B. Storage choice**
- **If no storage choice set yet**: Ask:
  > "Where should I save these contacts?
  > 1. **Notion database** (recommended — structured, searchable). I'll find or create one called 'Networking Contacts' in your workspace.
  > 2. **Gmail labels** (lightweight). I'll tag each follow-up draft with the event name so you can find them in Gmail later."
- **If already established**: Reuse silently.

You can ask both together in the first message:
> "Before I log this card, two quick questions:
> 1. Which event/conference did you meet them at?
> 2. Where should I save the contacts — Notion database or Gmail labels?"

**Define "session"**: Treat the current chat conversation as one session. If a new chat is started, or if the most recent name-card message was more than 6 hours ago, treat it as a new session and re-ask both questions.

Store the event name, date, and storage choice for use throughout the session. Also remember any IDs you've resolved (the Notion data source ID, the Gmail label IDs) so you don't re-resolve them per card.

### Step 2: Extract name card information

From the uploaded image, extract these fields (leave blank if not present on the card):

- **Name** (full name)
- **Title** (job title / role)
- **Company**
- **Email**
- **Phone**
- **Website**
- **LinkedIn** (if shown — URL or handle)
- **Notes** (anything else relevant: address, secondary phone, tagline, etc.)

If the image is unclear or text is unreadable, surface what you extracted and ask the user to confirm or correct before saving.

### Step 3: Save the contact

Branch based on the storage choice from Step 1.

#### 3A. If storage = Notion

**On the first card of the session**, resolve the database to use:

1. **Search for an existing database** using the Notion `search` tool with query "Networking Contacts" and filter on databases.
2. **If exactly one match is found**: fetch it via `notion-fetch` to get its `data_source_id` (look for the `<data-source>` tag in the response). Cache that ID for the rest of the session.
3. **If multiple matches**: list them and ask the user to confirm which to use, then fetch the chosen one.
4. **If no match is found**: ask the user:
   > "I don't see a 'Networking Contacts' database in your Notion. Should I create one for you?"
   - On confirmation, call `notion-create-database` with **no parent specified** first (creates at workspace root). If that fails because a parent page is required, ask the user to paste the URL of any Notion page they own and use that page's ID as the parent.
   - Use this exact `schema` argument:
     ```sql
     CREATE TABLE (
       "Name" TITLE,
       "Title" RICH_TEXT,
       "Company" RICH_TEXT,
       "Email" EMAIL,
       "Phone" PHONE_NUMBER,
       "Website" URL,
       "LinkedIn" URL,
       "Event" RICH_TEXT,
       "Date Added" DATE,
       "Notes" RICH_TEXT
     )
     ```
   - Title the database **"Networking Contacts"**.
   - The response includes a `<data-source>` tag with the data source ID. Capture that ID.

**For every card** (including the first):

5. Append a row using `notion-create-pages` with:
   - `parent.type = "data_source_id"`
   - `parent.data_source_id` = the cached ID from above
   - `properties` mapping the extracted fields to the columns. Use the exact property names above. For the `Date Added` date property, use the `date:Date Added:start` expanded format (e.g., `"date:Date Added:start": "2026-05-06"`, `"date:Date Added:is_datetime": 0`).

#### 3B. If storage = Gmail labels

**On the first card of the session**, resolve the label IDs:

1. Call `list_labels` and search for a label named exactly `Namecards`. If missing, call `create_label` with `displayName: "Namecards"` and capture the new label ID.
2. Search for `Namecards/[Event Name]` (Gmail nests labels using `/`). If missing, call `create_label` with `displayName: "Namecards/[Event Name]"` and capture the new label ID.
3. Cache both label IDs for the rest of the session.

The actual labelling happens in Step 4 after the draft is created, since Gmail labels apply to messages/threads, not raw drafts.

### Step 4: Create the follow-up email draft

Always create a draft (the connector cannot send directly). The user reviews and sends from Gmail.

**Email parameters:**
- `to`: `[email_from_card]`
- `subject`: `Great meeting you at [Event Name]`
- `body`: short, casual, friendly. 3-4 sentences max.

**Body template:**
```
Hi [First Name],

Great meeting you at [Event Name]! Really enjoyed our chat — would love to stay in touch.

Feel free to connect with me on LinkedIn: https://www.linkedin.com/in/keithteo36

Talk soon,
Keith
```

Vary phrasing slightly per card so the drafts don't read identically. Keep length, tone, and key elements (event reference + LinkedIn URL + casual sign-off as Keith).

Call `create_draft` with the parameters above.

**If storage = Gmail labels**, also apply the event label:

1. Call `search_threads` with query `is:draft to:[email_from_card]` (Gmail search syntax). Take the first/most-recent thread from the result.
2. Call `label_thread` with that `threadId` and the cached `Namecards/[Event Name]` label ID.
3. If the search returns no thread (race condition with draft indexing), wait briefly or note that labelling failed; the draft is still searchable in Gmail by subject text. Don't error out — Notion-mode users don't care, and Gmail-mode users can still find the draft.

If the card has **no email address**, skip Step 4 entirely and tell the user the email step was skipped. (For Notion mode, the contact still gets logged to the database. For Gmail-labels mode, there's nothing to label, so flag the gap.)

### Step 5: Confirm to the user

Give a brief confirmation per card. Include the storage destination so the user knows where to find it.

**Notion mode:**
```
✅ [Name] from [Company]
• Logged to Notion: Networking Contacts (Event: [Event Name])
• Draft email created in Gmail
```

**Gmail-labels mode:**
```
✅ [Name] from [Company]
• Draft email created and labelled "Namecards/[Event Name]"
```

If the email step was skipped (no email on card), say so:
```
✅ [Name] from [Company]
• Logged to Notion: Networking Contacts (Event: [Event Name])
• ⚠️ No email on card — skipped follow-up draft
```

Keep it concise. Don't repeat the entire extracted contact info unless the user asks.

## Handling multiple cards in one session

- If the user uploads several photos at once, process each in turn using the same event/storage context.
- If the user uploads a new card later in the same session (within 6 hours), continue with the established context — do not re-ask.
- Reuse cached IDs (Notion data source ID, Gmail label IDs) for every card. Don't re-resolve per card.
- Only re-ask for context if it's a new chat OR the user clearly indicates a different event.

## Edge cases

- **Multiple cards in one image**: Treat each as a separate row/draft.
- **Card not in English**: Extract what you can; preserve original-language text in the Notes field.
- **Duplicate contact** (same email already in Notion):
  - Notion mode: optionally check via `notion-fetch` on the data source filtered by Email. If a row with the same email exists, still add a new row but flag in confirmation: "⚠️ This email already exists in your Notion database — added as a new row, you may want to dedupe."
  - Gmail mode: don't bother checking — labels are cheap.
- **No email on card**: Skip the email draft step. Tell the user. Notion mode still logs the row.
- **Image too blurry to read**: Ask the user to confirm the fields you weren't sure about, or request a clearer photo.
- **Notion connector not authorised**: Surface a clear error and offer to switch to Gmail-labels mode for this session.
- **Gmail label thread search returns no result**: The draft might still be indexing. Note that labelling didn't succeed but don't fail the whole flow — the draft is searchable in Gmail by subject text or recipient.

## User context

The user is **Keith Teo**, founder of Cclarity. LinkedIn: https://www.linkedin.com/in/keithteo36. All email drafts should be signed off as Keith and reference his LinkedIn URL above.
