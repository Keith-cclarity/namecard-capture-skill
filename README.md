# Name Card Capture — Claude Skill

A Claude Skill that turns photos of business cards into structured contact records and follow-up emails — without breaking your flow at an event.

Snap a photo of someone's name card at a conference. Claude extracts the contact details, logs them to your **Notion** database (or labels them in **Gmail** if you don't use Notion), and drafts (or sends) a personalised follow-up email referencing the event you met at.

Built for founders, salespeople, and anyone who collects more cards at events than they ever follow up with.

---

## What it does

When you upload a photo of a business card, the skill will:

1. **Ask once per session**: which event you're at, where to save contacts (Notion or Gmail labels), and whether to draft or send follow-up emails
2. **Extract** name, title, company, email, phone, website, LinkedIn, and notes from the card
3. **Save the contact** — either as a new row in your `Networking Contacts` Notion database (auto-created the first time), or by labelling each follow-up draft in Gmail under `Namecards/[Event Name]`
4. **Draft or send** a short, casual follow-up email with your LinkedIn URL
5. **Confirm** with a one-line summary per card

For multiple cards in the same session, the event, storage choice, and email-handling preference are reused silently — no repeated questions until you start a new session or pause for more than 6 hours.

> **Why two storage options?** Claude.ai's Google Drive connector can only *create* files, not append rows to existing Sheets/Docs. Notion has full read/write support so it's the recommended path. Gmail labels are the fallback for users who don't use Notion — your follow-up drafts become a searchable record by event.

---

## Requirements

- A Claude account (Free, Pro, Max, Team, or Enterprise) — see [claude.ai](https://claude.ai)
- **Code execution and file creation** enabled in Claude settings
- **Gmail** connector enabled (for email drafts / sending — required either way)
- **Notion** connector enabled (recommended, for structured storage) — _or_ skip Notion and use Gmail labels only

All connectors are available out of the box on claude.ai.

---

## Quick install (60 seconds)

If you just want it working on [claude.ai](https://claude.ai), do this:

1. **Download the skill file:** [namecard-capture.skill](https://github.com/Keith-cclarity/namecard-capture-skill/releases/latest/download/namecard-capture.skill)
2. **Go to [claude.ai](https://claude.ai)** → click your profile (top right) → **Settings** → **Capabilities** → turn on **Code execution and file creation** → then go to **Skills** → **+ Upload skill** → pick the file you just downloaded → toggle it **on**
3. **Connect Gmail** (and **Notion** if you have it): Settings → **Connectors** → enable both

That's it. Open any chat, upload a photo of a business card, and the skill triggers automatically.

> 💡 If your browser rejects the `.skill` file, rename it to `.zip` (same format) and try again. **Don't** use the green "Code → Download ZIP" button on this repo — that's the source code, not the skill.

---

## Installation (detailed)

Three install paths, depending on where you use Claude. Most people only need Option A.

### Option A — Claude.ai (web or desktop)

This is the most common path. Once installed via the web app, the skill works on the **mobile app** too (since it's tied to your account).

1. Download the skill file: **[namecard-capture.skill](https://github.com/Keith-cclarity/namecard-capture-skill/releases/latest/download/namecard-capture.skill)** (always points to the latest release).
   - Alternative sources: the [Releases](../../releases) page, or the [`dist/namecard-capture.skill`](https://github.com/Keith-cclarity/namecard-capture-skill/raw/main/dist/namecard-capture.skill) file in this repo.
   - **Do not** use GitHub's green "Code → Download ZIP" button — that gives you the repo source, not a valid skill package, and Claude will reject it with "SKILL.md must be in the top-level folder."
2. Go to [claude.ai](https://claude.ai) and sign in
3. Click your profile icon (top right) → **Settings**
4. Under **Capabilities**, make sure **Code execution and file creation** is toggled **on** (skills won't work without this)
5. Go to **Skills** in the sidebar
6. Click **+ Upload skill** and select `namecard-capture.skill`
   - If your browser rejects the `.skill` extension, rename the file to `namecard-capture.zip` and try again — it's the same archive format
7. Toggle the skill **on** in your skills list
8. Make sure your **Gmail** connector is enabled under **Settings → Connectors**. If you use Notion, enable that too.

You're done. Open any chat (web or mobile), upload a name card photo, and the skill will trigger.

### Option B — Claude Code (CLI)

For developers who want the skill available in the terminal.

```bash
# Clone the repo
git clone https://github.com/Keith-cclarity/namecard-capture-skill.git
cd namecard-capture-skill

# Copy the skill folder to your Claude Code skills directory
mkdir -p ~/.claude/skills/
cp -r namecard-capture ~/.claude/skills/
```

Verify it's loaded:

```bash
claude
> /skills
```

You should see `namecard-capture` in the list. Note that Claude Code doesn't have native Notion/Gmail connectors the same way claude.ai does — you'll need to wire those up via [MCP servers](https://docs.claude.com/en/docs/agents-and-tools/mcp) for the storage and email steps to work.

### Option C — Claude API (developers)

If you're building on the Anthropic API and want to use this skill programmatically:

1. Read the skill content from `namecard-capture/SKILL.md`
2. Pass it into your system prompt or via the Skills feature (see [Anthropic API docs on Skills](https://docs.claude.com/en/docs/build-with-claude/skills))
3. Provide tool access to whatever Notion / Gmail integration you're using (e.g., custom MCP servers or function-calling tools)

The skill itself is a portable instruction file — it doesn't depend on claude.ai-specific infrastructure, only on the **tool capabilities** (Notion or Gmail labels for storage, Gmail for email, image input for cards) being available in the runtime.

---

## Usage

After install, open a chat in Claude (web, desktop, or mobile) and upload a photo of a business card. The skill triggers automatically.

**First card of a session:**

> You: *(uploads photo of business card)*
>
> Claude: Before I log this card, three quick questions:
> 1. Which event/conference did you meet them at?
> 2. Where should I save the contacts — Notion database or Gmail labels?
> 3. Should follow-up emails be saved as drafts or sent directly?

After you answer, Claude processes the card and confirms:

> ✅ Jane Smith from Acme Corp
> • Logged to Notion: Networking Contacts → TechCon 2026
> • Draft email created in Gmail

**Subsequent cards in the same session:** No repeated questions — Claude just processes and confirms.

**New session (new chat or after 6+ hours):** All three questions are re-asked.

### Notion database (recommended)

The first time you pick Notion, the skill searches your workspace for a database called **Networking Contacts**. If it doesn't exist, it offers to create one with these properties:

```
Name | Title | Company | Email | Phone | Website | LinkedIn | Event | Date Added | Notes
```

Every subsequent card appends a new row. From session two onward, the skill finds the database silently — you never have to paste a URL.

### Gmail labels (fallback)

If you don't use Notion, the skill creates the labels `Namecards` and `Namecards/[Event Name]` in your Gmail and tags every follow-up draft with the event label. To find everyone you met at SaaStr 2026, open Gmail → Drafts → filter by label `Namecards/SaaStr 2026`.

> ⚠️ Gmail-labels mode is intentionally lightweight — your "record" is the labelled draft itself (To: field has the contact, body has the follow-up message). Use Notion if you want a structured table you can sort, filter, or sync to a CRM.

### Manual triggers

If the skill doesn't auto-trigger, you can force it:

> "Use the namecard-capture skill — I'm at SaaStr 2026, log to Notion, save follow-ups as drafts."

---

## Customisation

Fork this repo and edit `namecard-capture/SKILL.md` to adapt it to your workflow. Common tweaks:

- **Change the email body** — edit the template in Step 4 to match your tone
- **Update the LinkedIn URL** — replace the URL in Step 4 and the user context section with your own
- **Add or rename Notion properties** — modify Step 3A to capture different fields
- **Change the session timeout** — currently 6 hours; edit the "session" definition in Step 1

After editing, repackage the skill so that `SKILL.md` sits at the **root** of the archive (Claude rejects nested skill files):

```bash
cd namecard-capture
zip ../namecard-capture.skill SKILL.md
```

Then upload `namecard-capture.skill` to Claude.

> ⚠️ Don't use GitHub's "Download ZIP" button to install the skill — that gives you the repo source, not a valid skill package. Use the file in `dist/` (or the [Releases](../../releases) page) instead.

---

## File structure

```
namecard-capture-skill/
├── README.md                      ← this file
├── LICENSE                        ← MIT
├── namecard-capture/
│   └── SKILL.md                   ← the skill itself
└── dist/
    └── namecard-capture.skill     ← packaged for upload
```

---

## Troubleshooting

**Skill doesn't trigger when I upload a card photo**
- Confirm it's toggled on in Settings → Skills
- Make sure code execution is enabled
- Try a manual trigger: "Use the namecard-capture skill on this image"

**"Cannot find Notion database" / Notion errors**
- Check that the Notion connector is enabled and authenticated under Settings → Connectors
- The skill creates the `Networking Contacts` database the first time you use Notion mode — open Notion and check that it appeared
- If you'd rather not deal with Notion, switch to Gmail-labels mode at the start of the session

**Email step skipped unexpectedly**
- The skill skips the email if no email address was readable on the card
- Check the extracted info; you may need a clearer photo

**Gmail labels don't appear**
- Make sure the Gmail connector is enabled and authenticated
- Refresh Gmail — labels may take a moment to show up in the sidebar

**Mobile upload of skill file fails**
- Skill installation isn't supported on the mobile app — install via claude.ai on desktop/web. Once installed, the skill works on mobile.

---

## Contributing

PRs welcome. If you build a useful variant (e.g., LinkedIn-only mode, CRM integration, multi-language extraction), open a PR or fork freely.

---

## License

MIT — see [LICENSE](LICENSE).

---

## Author

**Keith Teo** — Founder of [Cclarity](https://cclarity.io)

- LinkedIn: [linkedin.com/in/keithteo36](https://www.linkedin.com/in/keithteo36)
- GitHub: [@Keith-cclarity](https://github.com/Keith-cclarity)

If this saves you time at your next conference, a star on the repo is appreciated. If you build something cool on top of it, tag me on LinkedIn — always keen to see how people remix these.
