# Name Card Capture — Claude Skill

A Claude Skill that turns photos of business cards into structured contact records and follow-up emails — without breaking your flow at an event.

Snap a photo of someone's name card at a conference. Claude extracts the contact details, logs them to a Google Sheet in your Drive, and drafts (or sends) a personalized follow-up email referencing the event you met at.

Built for founders, salespeople, and anyone who collects more cards at events than they ever follow up with.

---

## What it does

When you upload a photo of a business card, the skill will:

1. **Ask once per session**: which event you're at, and whether to save follow-up emails as drafts or send them directly
2. **Extract** name, title, company, email, phone, website, LinkedIn, and notes from the card
3. **Append** a row to a Google Sheet in your Drive (existing sheet or a new one named `Networking - [Event] - [Date]`)
4. **Draft or send** a short, casual follow-up email with your LinkedIn URL
5. **Confirm** with a one-line summary per card

For multiple cards in the same session, the event and email-handling choice are reused silently — no repeated questions until you start a new session or pause for more than 6 hours.

---

## Requirements

- A Claude account (Free, Pro, Max, Team, or Enterprise) — see [claude.ai](https://claude.ai)
- **Code execution and file creation** enabled in Claude settings
- **Google Drive** connector enabled (for the sheet)
- **Gmail** connector enabled (for email drafts / sending)

Both connectors are available out of the box on claude.ai.

---

## Installation

There are three ways to install this skill depending on where you use Claude.

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
8. Make sure your **Google Drive** and **Gmail** connectors are enabled under **Settings → Connectors**

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

You should see `namecard-capture` in the list. Note that Claude Code doesn't have native Google Drive/Gmail connectors the same way claude.ai does — you'll need to wire those up via [MCP servers](https://docs.claude.com/en/docs/agents-and-tools/mcp) for the sheet and email steps to work.

### Option C — Claude API (developers)

If you're building on the Anthropic API and want to use this skill programmatically:

1. Read the skill content from `namecard-capture/SKILL.md`
2. Pass it into your system prompt or via the Skills feature (see [Anthropic API docs on Skills](https://docs.claude.com/en/docs/build-with-claude/skills))
3. Provide tool access to whatever Google Drive / Gmail integration you're using (e.g., custom MCP servers or function-calling tools)

The skill itself is a portable instruction file — it doesn't depend on claude.ai-specific infrastructure, only on the **tool capabilities** (Drive, Gmail, image input) being available in the runtime.

---

## Usage

After install, open a chat in Claude (web, desktop, or mobile) and upload a photo of a business card. The skill triggers automatically.

**First card of a session:**

> You: *(uploads photo of business card)*
>
> Claude: Before I log this card, two quick questions:
> 1. Which event/conference did you meet them at?
> 2. Should follow-up emails be saved as drafts or sent directly?

After you answer, Claude processes the card and confirms:

> ✅ Jane Smith from Acme Corp
> • Added to sheet: Networking - TechCon 2026 - 2026-05-04
> • Draft email created in Gmail

**Subsequent cards in the same session:** No repeated questions — Claude just processes and confirms.

**New session (new chat or after 6+ hours):** The event and email preference are re-asked.

### Sheet structure

Cards are logged with these columns:

```
Date Added | Event | Name | Title | Company | Email | Phone | Website | LinkedIn | Notes
```

### Manual triggers

If the skill doesn't auto-trigger, you can force it:

> "Use the namecard-capture skill — I'm at SaaStr 2026, save follow-ups as drafts."

---

## Customization

Fork this repo and edit `namecard-capture/SKILL.md` to adapt it to your workflow. Common tweaks:

- **Change the email body** — edit the template in Step 4 to match your tone
- **Update the LinkedIn URL** — replace the URL in Step 4 and the user context section with your own
- **Adjust the columns** — modify Step 3 to capture different fields
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

**"Cannot create sheet" error**
- Check that the Google Drive connector is enabled and authenticated
- Try with an existing sheet — paste a URL when asked

**Email step skipped unexpectedly**
- The skill skips the email if no email address was readable on the card
- Check the extracted info; you may need a clearer photo

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
