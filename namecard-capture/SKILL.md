---
name: namecard-capture
description: Capture business card / name card information from photos taken at events or conferences. Use this skill whenever the user uploads a photo of a business card, name card, or contact card, or mentions capturing/scanning/saving a name card or business card. Also trigger when the user says things like "I just met someone at [event]", "save this contact", "log this name card", "I got their card", or uploads a photo that appears to be a business card. The skill extracts contact details, logs them to either a Notion database (preferred, structured) or Gmail labels (lightweight fallback), and drafts a follow-up email in Gmail referencing the event where they met. Handles multiple cards within a single session by reusing the event context, but re-asks for event information on a new session or after a long pause of more than 6 hours.
---

# Name Card Capture

Capture name card information from event/conference photos, log them to Notion (or Gmail labels as a fallback), and draft a personalised follow-up email.

## When this skill triggers

- User uploads a photo of a business card / name card
- User says something like "save this contact", "log this card", "just met someone at [event]"
- User mentions capturing card details from a conference or networking event

## Workflow

### Step 1: Establish session context (once per session)

Before processing any cards, check whether session context has already been established **in the current session**. There are three things to establish:

**A. Event context**
- **If no event context yet**: Ask which event/conference this is, and the date if not today.
- **If already established**: Reuse silently. Do NOT re-ask.

**B. Storage choice**
- **If no storage choice set yet**: Ask:
  > "Where should I save these contacts?
  > 1. **Notion database** (recommended — structured, searchable). I'll find or create one called 'Networking Contacts' in your workspace.
  > 2. **Gmail labels only** (lightweight). I'll tag each follow-up draft with the event name so you can search them later in Gmail."
- **If already established**: Reuse silently.

**C. Email handling preference**
- **If no email preference set yet**: Ask whether to save follow-up emails as drafts (so the user can review) or send them directly.
- **If already established**: Reuse silently.

You can ask all three together in the first message:
> "Before I log this card, three quick questions:
> 1. Which event/conference did you meet them at?
> 2. Where should I save the contacts — Notion database or Gmail labels?
> 3. Should follow-up emails be saved as drafts or sent directly?"

**Define "session"**: Treat the current chat conversation as one session. If a new chat is started, or if the most recent name-card message was more than 6 hours ago, treat it as a new session and re-ask all three questions.

Store the event name, date, storage choice, and email preference for use throughout the session.

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

1. **Search for an existing database** using the Notion `search` tool with query "Networking Contacts".
2. **If found**: use that database's ID for the rest of the session.
3. **If not found**: ask the user:
   > "I don't see a 'Networking Contacts' database in your Notion. Should I create one for you?"
   - On confirmation, use `create-database` (or `create-pages` with a database parent) to create a new database titled **"Networking Contacts"** at the user's workspace root with these properties:
     - `Name` (title)
     - `Title` (rich text)
     - `Company` (rich text)
     - `Email` (email)
     - `Phone` (phone)
     - `Website` (URL)
     - `LinkedIn` (URL)
     - `Event` (rich text)
     - `Date Added` (date)
     - `Notes` (rich text)
4. **Append the new contact** as a row using `create-pages` with `data_source_id` set to the database's data source ID. Set `Date Added` to today's date and `Event` to the event name.

If the user already had multiple databases that could match, ask them to confirm which one.

#### 3B. If storage = Gmail labels

The Gmail draft itself is the record — there is no separate "row" to append. Instead:

1. **Ensure the parent label exists**: list labels and check for `Namecards`. If missing, create it via Gmail's `create_label` tool.
2. **Ensure the event sub-label exists**: check for `Namecards/[Event Name]` (Gmail nests labels with `/`). If missing, create it.
3. The actual labelling happens in Step 4, after the draft is created.

### Step 4: Draft (or send) the follow-up email

Based on the email preference from Step 1.

**Email parameters (same for both modes):**
- **To**: Email address from the card
- **Subject**: `Great meeting you at [Event Name]`
- **Body style**: Short, casual, friendly. 3-4 sentences max.

**Body template:**
```
Hi [First Name],

Great meeting you at [Event Name]! Really enjoyed our chat — would love to stay in touch.

Feel free to connect with me on LinkedIn: https://www.linkedin.com/in/keithteo36

Talk soon,
Keith
```

Adjust phrasing naturally so each email doesn't read identically. Keep length, tone, and key elements (event reference + LinkedIn URL + casual sign-off as Keith).

**Mode:**
- **Draft mode**: Use Gmail's `create_draft` tool — do NOT send.
- **Send-directly mode**: Use Gmail's send tool to send immediately.

**If storage = Gmail labels**, after creating the draft (or sending), apply the `Namecards/[Event Name]` label to that message using `label_message` with the message ID returned from the draft/send call.

If the card has no email address, skip this step and tell the user the email step was skipped. (For Notion mode, the contact still gets logged to the database. For Gmail-labels mode, there is nothing to label, so flag it as a gap.)

### Step 5: Confirm to the user

Give a brief confirmation per card. Include the storage destination so the user knows where to find it.

**Notion mode, draft email:**
```
✅ [Name] from [Company]
• Logged to Notion: Networking Contacts → [Event Name]
• Draft email created in Gmail
```

**Notion mode, sent email:**
```
✅ [Name] from [Company]
• Logged to Notion: Networking Contacts
• Follow-up email sent
```

**Gmail-labels mode, draft email:**
```
✅ [Name] from [Company]
• Draft email created and labelled "Namecards/[Event Name]"
```

**Gmail-labels mode, sent email:**
```
✅ [Name] from [Company]
• Follow-up email sent and labelled "Namecards/[Event Name]"
```

Keep it concise. Don't repeat the entire extracted contact info unless the user asks.

## Handling multiple cards in one session

- If the user uploads several photos at once, process each in turn using the same event/storage/email context.
- If the user uploads a new card later in the same session (within 6 hours), continue with the established context — do not re-ask.
- Only re-ask for event context if it's a new chat OR the user clearly indicates a different event.

## Edge cases

- **Multiple cards in one image**: Treat each as a separate row/draft.
- **Card not in English**: Extract what you can; preserve original-language text in the Notes field.
- **Duplicate contact** (same email already in Notion):
  - Notion mode: search the database first; if a row with the same email exists, still add a new row but flag in confirmation: "⚠️ This email already exists in your Notion database — added as a new row, you may want to dedupe."
  - Gmail mode: don't bother checking — labels are cheap.
- **No email on card**: Skip the email step. Tell the user. Notion mode still logs the row.
- **Image too blurry to read**: Ask the user to confirm the fields you weren't sure about, or request a clearer photo.
- **Notion connector not authorised**: Surface a clear error and offer to switch to Gmail-labels mode for this session.

## User context

The user is **Keith Teo**, founder of Cclarity. LinkedIn: https://www.linkedin.com/in/keithteo36. All email drafts should be signed off as Keith and reference his LinkedIn URL above.
