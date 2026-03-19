# imessage-mail

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) skill for reading, searching, and sending iMessages and emails on macOS. Queries the local Messages and Apple Mail SQLite databases and sends messages via AppleScript.

## Features

- **Recent Activity** — See who texted or emailed you, with message previews and unread counts
- **Conversation Reader** — Read full iMessage and email history for any contact
- **Message Search** — Full-text search across all iMessage conversations and Apple Mail
- **Attachment Finder** — Find photos, videos, PDFs, and other files from messages and emails
- **Mail Reader** — Browse, search, and read Apple Mail emails directly
- **Send Messages** — Send iMessages and emails via AppleScript (with confirmation prompt)

## Installation

### 1. Copy the skill into your Claude Code skills directory

```bash
# Clone this repo
git clone https://github.com/stat-guy/imessage-mail.git

# Copy the skill files
mkdir -p ~/.claude/skills/imessage-mail
cp -r imessage-mail/SKILL.md ~/.claude/skills/imessage-mail/
cp -r imessage-mail/agents ~/.claude/skills/imessage-mail/
```

### 2. Grant Full Disk Access to your terminal

The skill reads local SQLite databases that macOS protects. You must grant **Full Disk Access** to whichever terminal app you use with Claude Code:

1. Open **System Settings** > **Privacy & Security** > **Full Disk Access**
2. Click the **+** button
3. Add your terminal app (e.g., Terminal.app, iTerm2, Warp, Ghostty, etc.)
4. Restart the terminal

### 3. Apple Permission Prompts

When sending iMessages or emails for the first time, macOS will display a permission dialog asking whether to allow your terminal to control **Messages.app** or **Mail.app** via AppleScript. **Click "Allow"** — this is a one-time prompt. Once allowed, the skill will send messages without further prompts going forward. If you accidentally deny it, you can re-enable it in **System Settings** > **Privacy & Security** > **Automation**.

## Usage

Once installed, use the skill in Claude Code with natural language:

```
/imessage-mail recent
/imessage-mail messages from Sarah
/imessage-mail search "project deadline"
/imessage-mail email
/imessage-mail send a text to Mike saying I'm running late
/imessage-mail find attachments from David
```

### Example Interactions

**Check recent messages:**
```
> /imessage-mail who texted me

## Recent iMessages

| Contact              | Last Message        | Messages    | Preview                        |
|----------------------|---------------------|-------------|--------------------------------|
| Sarah Chen           | 2026-03-16 14:30    | 45 (↓23 ↑22) | "Hey, are we still on for..." |
| Mike Johnson         | 2026-03-16 12:15    | 12 (↓8 ↑4)   | "Thanks for sending that!"    |
| +1555000111          | 2026-03-15 09:00    | 3 (↓3 ↑0)    | [Attachment: image]           |

## Recent Emails (5 unread)

| Status | From                        | Subject              | Received         |
|--------|-----------------------------|----------------------|------------------|
| UNREAD | Sarah Chen (sarah@corp.com) | Re: Project Update   | Mar 16, 18:30    |
| Read   | GitHub (noreply@github.com) | [repo] PR #42 merged | Mar 16, 17:00    |
```

**Read a conversation:**
```
> /imessage-mail texts with Sarah

## Conversation with Sarah Chen

### iMessages (+15551234567)
*25 most recent*

**14:30 Me**: Hey, are we still on for dinner tonight?
**14:32 Sarah**: Yeah! What time works?
**14:33 Me**: How about 7?
**14:35 Sarah**: Perfect, see you then!
```

**Send a message:**
```
> /imessage-mail text Sarah saying I'll be 10 minutes late

**Ready to send:**

**Type:** iMessage
**To:** Sarah Chen (+15551234567)

> I'll be 10 minutes late

Send this? (y/n)
```

**Search across messages and email:**
```
> /imessage-mail search "invoice"

## Search Results for "invoice"

### iMessages — 2 matches
**Mike Johnson (+15559876543)**
- **Mar 14, 12:00 — Mike**: "Can you send me the invoice?"
- **Mar 14, 12:05 — Me**: "Just emailed it over"

### Emails — 3 matches
| Status | From                  | Subject                | Received         |
|--------|-----------------------|------------------------|------------------|
| Read   | QuickBooks            | Invoice #1042 paid     | Mar 15, 10:00    |
| Read   | Mike Johnson          | Re: Invoice request    | Mar 14, 12:30    |
| UNREAD | Vendor Inc.           | Invoice #8837 due      | Mar 12, 09:00    |
```

## Architecture

```
imessage-mail/
├── SKILL.md                    # Orchestrator — intent classification and routing
└── agents/
    ├── recent-activity.md      # Recent contacts and email overview
    ├── conversation-reader.md  # Full message history for a contact
    ├── message-search.md       # Cross-source keyword search
    ├── attachment-finder.md    # Find files and media from messages/emails
    ├── mail-reader.md          # Apple Mail database queries
    └── send-message.md         # Send iMessages and emails via AppleScript
```

### How It Works

1. **SKILL.md** classifies user intent (recent activity, conversation, search, send, etc.)
2. **Pre-loaded context** is injected at skill load time via shell commands — the top 10 recent iMessage contacts and top 8 unread emails are available instantly with zero tool calls
3. Routes to the appropriate **agent** markdown file
4. The agent executes **SQLite queries** against local macOS databases:
   - `~/Library/Messages/chat.db` — iMessage history
   - `~/Library/Mail/V10/MailData/Envelope Index` — Apple Mail metadata
   - `~/Library/Application Support/AddressBook/AddressBook-v22.abcddb` — Contacts
5. For sending, it uses **AppleScript** via `osascript` to control Messages.app and Mail.app
6. Contact resolution and cross-source searches use **parallel subagents** for performance

### Pre-loaded Context

At skill load time, two shell injections run automatically and embed live data directly into the prompt — no Bash tool call needed to answer the most common queries:

| Injection | Data | Eliminates |
|-----------|------|-----------|
| Recent iMessage contacts | Top 10 active handles w/ timestamps + message counts | Initial sqlite3 query in `recent-activity.md` Step 1 |
| Unread emails | Total unread count + top 8 most recent unread (sender, subject, date) | Unread listing queries in `mail-reader.md` |

For "who texted me" and "check my email" — the two most common queries — responses start instantly from pre-loaded data rather than waiting for a Bash tool call to complete.

### Database Paths

| Database | Path | Contains |
|----------|------|----------|
| Messages | `~/Library/Messages/chat.db` | iMessage/SMS history, attachments, handles |
| Mail | `~/Library/Mail/V10/MailData/Envelope Index` | Email metadata, senders, subjects, mailboxes |
| Contacts | `~/Library/Application Support/AddressBook/AddressBook-v22.abcddb` | Names, phone numbers, email addresses |

## Requirements

- **macOS** (tested on macOS 15+ / Apple Silicon)
- **Claude Code** CLI
- **Full Disk Access** granted to your terminal application
- **Messages.app** signed into iMessage (for iMessage features)
- **Mail.app** with at least one account configured (for email features)

## Session Trace

This repo includes an [Entire CLI](https://entire.io) session trace in `.entire/trace.log`, capturing the agent lifecycle events from the Claude Code session that built this skill. Entire is an observability tool for AI-assisted development.

## Privacy

This skill runs entirely locally on your Mac. No message data, contacts, or email content is sent to any external service beyond what Claude Code normally processes. All database queries are read-only (except for the send functionality which uses AppleScript to send via your local apps).

## License

MIT
