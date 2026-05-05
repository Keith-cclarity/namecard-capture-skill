---
name: namecard-capture
description: Capture business card / name card information from photos taken at events or conferences. Use this skill whenever the user uploads a photo of a business card, name card, or contact card, or mentions capturing/scanning/saving a name card or business card. Also trigger when the user says things like "I just met someone at [event]", "save this contact", "log this name card", "I got their card", or uploads a photo that appears to be a business card. The skill extracts contact details, appends them to a Google Sheet in the user's Google Drive, and drafts a follow-up email in Gmail referencing the event where they met. Handles multiple cards within a single session by reusing the event context, but re-asks for event information on a new session or after a long pause of more than 6 hours.
---

# Name Card Capture

Capture name card information from event/conference photos, log to Google Sheets, and draft a personalized follow-up email.

## When this skill triggers

- User uploads a photo of a business card / name card
- User says something like "save this contact", "log this card", "just met someone at [event]"
- User mentions capturing card details from a conference or networking event

## Workflow

### Step 1: Establish session context (once per session)

Before processing any cards, check whether session context has already been established **in the current session**. There are two things to establish:

**A. Event context**
- **If no event context yet**: Ask:
  > "Which event or conference did you meet them at? (And the date if it's not today.)"
- **If already established**: Do NOT ask again. Reuse silently.

**B. Email handling preference**
- **If no email preference set yet**: Ask:
  > "For the follow-up emails — should I save them as drafts in Gmail (so you can review before sending), or send them directly?"
- **If already established**: Do NOT ask again. Apply the same choice for every card in the session.

You can ask both questions together in the first message if no context is set yet, e.g.:
> "Before I log this card, two quick questions:
> 1. Which event/conference did you meet them at?
> 2. Should follow-up emails be saved as drafts or sent directly?"

**Define "session"**: Treat the current chat conversation as one session. If a new chat is started, or if the most recent name-card message was more than 6 hours ago, treat it as a new session and re-ask both questions.

Store the event name, date, and email preference for use throughout the session.

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

### Step 3: Append to Google Sheet

The user's preference is **per-session sheet**: ask the user once at the start of the session whether to:
- (a) Append to an existing sheet (ask for the sheet name or have them paste the URL), OR
- (b) Create a new sheet for this event

If creating a new sheet, name it: `Networking - [Event Name] - [YYYY-MM-DD]`

**Required columns** (create headers if the sheet is new):
`Date Added | Event | Name | Title | Company | Email | Phone | Website | LinkedIn | Notes`

Use the Google Drive / Google Sheets tools available in the environment to:
1. Create the sheet (if new) in the user's Google Drive root, OR locate the existing one
2. Append a row with the extracted info, using today's date for "Date Added"

If Google Sheets tooling isn't directly available, fall back to creating/updating an `.xlsx` file or use the Google Drive `create_file` tool with a Sheets MIME type.

### Step 4: Handle the follow-up email

Based on the email preference established in Step 1:

- **If "draft"**: Use Gmail's `create_draft` tool — do NOT send.
- **If "send directly"**: Use Gmail's send tool to send the email immediately.

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

Adjust the wording naturally — don't make every email read like a copy-paste. Vary phrasing slightly while keeping the same length, tone, and key elements (event reference + LinkedIn URL + casual sign-off as Keith).

If the card has no email address, skip this step and tell the user the email step was skipped.

### Step 5: Confirm to the user

After processing each card, give a brief confirmation. Use the wording that matches the email preference:

If draft mode:
```
✅ [Name] from [Company]
• Added to sheet: [sheet name]
• Draft email created in Gmail
```

If send-directly mode:
```
✅ [Name] from [Company]
• Added to sheet: [sheet name]
• Follow-up email sent
```

Keep it concise. Don't repeat the entire extracted contact info unless the user asks.

## Handling multiple cards in one session

- If the user uploads several photos at once, process each one in turn using the same event context
- If the user uploads a new card later in the same session (within 6 hours), continue using the established event — do not re-ask
- Only re-ask for event context if it's a new chat OR clearly a new event ("just got to a different event now…")

## Edge cases

- **Multiple cards in one image**: Treat each as a separate row and a separate email draft.
- **Card not in English**: Extract what you can; preserve original-language text in the Notes field.
- **Duplicate contact** (same email already in sheet): Still add the row, but flag in confirmation: "⚠️ This email already exists in the sheet — added as a new row, you may want to dedupe."
- **No email on card**: Skip the email draft step. Tell the user.
- **Image too blurry to read**: Ask the user to confirm the fields you weren't sure about, or request a clearer photo.

## User context

The user is **Keith Teo**, founder of Cclarity. LinkedIn: https://www.linkedin.com/in/keithteo36. All email drafts should be signed off as Keith and reference his LinkedIn URL above.
