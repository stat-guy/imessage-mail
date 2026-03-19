---
name: imessage-mail
description: Read, search, and SEND iMessages and emails on macOS. Extracts iMessage history, contacts, and attachments from chat.db, reads/searches Apple Mail, and sends iMessages and emails via AppleScript. Use when the user asks for recent messages, texts, emails, to send a message or email, or to find specific attachments.
argument-hint: <contact name, phone number, "recent", "email", "send", or search query>
user-invocable: true
---

# iMessage & Mail — Orchestrator

Read, search, and send iMessages and emails on macOS. Cross-references data from macOS Messages, Apple Mail, and the Address Book.

## Prerequisites

- **Full Disk Access**: Terminal must have Full Disk Access in macOS System Settings > Privacy & Security.
- **Paths**:
  - Messages DB: `~/Library/Messages/chat.db`
  - Address Book: `~/Library/Application Support/AddressBook/AddressBook-v22.abcddb`
  - Mail DB: `~/Library/Mail/V10/MailData/Envelope Index`
- **AppleScript**: Used to send iMessages (via Messages.app) and emails (via Mail.app)

## Timestamp Reference

iMessage uses Mac CoreData (Cocoa) epoch (seconds since Jan 1, 2001). To convert to Unix epoch, add `978307200`. In recent macOS versions, `message.date` is in nanoseconds, so divide by `1,000,000,000` first:

```sql
datetime(m.date / 1000000000 + 978307200, 'unixepoch', 'localtime')
```

Apple Mail uses standard Unix epoch for `date_received` and `date_sent`:

```sql
datetime(m.date_received, 'unixepoch', 'localtime')
```

## Rich Text (attributedBody) Reference

Many iMessages store content as an `NSAttributedString` blob in the `attributedBody` column rather than `text`. If a message appears empty or as `[Rich Text/Hidden]`, extract readable text with:

```bash
sqlite3 ~/Library/Messages/chat.db "SELECT HEX(attributedBody) FROM message WHERE ROWID = <ROWID>;" | xxd -r -p | strings
```

## Pre-loaded Context

The following data is injected at skill load time via `!`command`` shell injection — no tool call needed. Use it directly; do NOT re-run the same queries with a Bash tool call.

### Recent iMessage Contacts — Last 10 Active
!`sqlite3 ~/Library/Messages/chat.db 'SELECT h.id, datetime(MAX(m.date)/1000000000+978307200,"unixepoch","localtime"), COUNT(*), SUM(m.is_from_me=0) FROM message m JOIN handle h ON m.handle_id=h.ROWID GROUP BY h.id ORDER BY MAX(m.date) DESC LIMIT 10' 2>/dev/null | awk -F'|' '{printf "%d. %s | last: %s | %s msgs (%s received)\n", NR, $1, $2, $3, $4}'`

Use these handles for "recent messages" / "who texted me" queries — skip Step 1 in `recent-activity.md`. Still run Steps 2-4 to resolve names and fetch message previews.

### Unread Emails — Count + Top 8 Most Recent
!`{ n=$(sqlite3 "$HOME/Library/Mail/V10/MailData/Envelope Index" 'SELECT COUNT(*) FROM messages WHERE deleted=0 AND read=0' 2>/dev/null); echo "$n unread emails"; sqlite3 "$HOME/Library/Mail/V10/MailData/Envelope Index" 'SELECT COALESCE(a.comment,a.address,"?")||" | "||COALESCE(sub.subject,"(no subject)")||" | "||datetime(m.date_received,"unixepoch","localtime") FROM messages m LEFT JOIN addresses a ON m.sender=a.ROWID LEFT JOIN subjects sub ON m.subject=sub.ROWID WHERE m.deleted=0 AND m.read=0 ORDER BY m.date_received DESC LIMIT 8' 2>/dev/null | awk '{print NR". "$0}'; } 2>/dev/null`

Use this for "unread emails" / "check my email" queries — the count and top 8 unread are already here. Only re-query if the user wants more results, a different filter, or specific email body content.

## Intent Classification

Classify the user's input and `Read` the matched agent file, then follow its workflow step-by-step.

| Input Pattern | Agent File | Description |
|---|---|---|
| "recent messages", "who texted me", "latest texts" | `agents/recent-activity.md` | List recent unique iMessage contacts with names |
| Contact name or phone number + "messages"/"texts" | `agents/conversation-reader.md` | Read iMessage history for a specific contact |
| "find attachment", "photos from", "files from" | `agents/attachment-finder.md` | Find attachments in iMessage conversations |
| "search for", "find messages about", keyword search | `agents/message-search.md` | Full-text search across all iMessages |
| "email", "mail", "inbox", "unread emails" | `agents/mail-reader.md` | Read, search, and inspect Apple Mail emails |
| "send", "text", "reply", "respond", "tell them" | `agents/send-message.md` | Send iMessages or emails via AppleScript |

### Input Detection Logic

1. **Send/reply keywords**: If input contains "send", "text [someone]", "reply", "respond", "tell them", "message [someone] saying", route to **send-message**
2. **Email/mail keywords**: If input contains "email", "mail", "inbox", "unread email", "newsletter", or sender addresses with `@`, route to **mail-reader** (unless also contains send intent, then route to **send-message**)
3. **Phone number detection**: If input contains a phone number pattern (digits, +, dashes, parens), check for send intent first, otherwise extract digits and route to conversation-reader
4. **Attachment keywords**: If input contains "attachment", "photo", "image", "file", "video", "pdf", route to attachment-finder
5. **Search keywords**: If input contains "search", "find messages", "look for", route to message-search
6. **Recent/activity keywords**: If input contains "recent", "latest", "who texted", "new messages", route to recent-activity
7. **Contact name**: If input is a person's name (possibly with "messages"/"texts"), route to conversation-reader (resolve name first)
8. **Ambiguous "messages from [name]"**: If unclear whether user means iMessage or email, check both — spawn **conversation-reader** and **mail-reader** subagents in parallel via the Agent tool, then combine results
9. **Default**: Route to recent-activity

## Execution Instructions

1. Classify the intent using the table above
2. `Read` the matched agent `.md` file from `~/.claude/skills/imessage-mail/agents/`
3. Follow that agent's workflow step-by-step, executing SQL queries via `sqlite3` or AppleScript via `osascript` in Bash
4. When resolving contacts, spawn subagents via the Agent tool for parallel name lookups
5. Always present results in a clean, readable format (tables or formatted lists)
6. For conversations, clearly distinguish between sent ("Me") and received messages
7. For ambiguous queries about a person, spawn **parallel subagents** to search both iMessage and Mail simultaneously, then merge results
8. **For send actions**: Always show the user the formatted message before sending and ask for confirmation, unless they explicitly say to send immediately

## Subagent Patterns

### Parallel Contact Resolution
When you have multiple phone numbers to resolve, batch them into 2-3 Agent tool calls (each resolving 5-7 numbers) running in parallel rather than sequentially.

### Cross-Source Search
When searching for messages from a specific person, spawn two parallel subagents:
- One searching iMessage via conversation-reader
- One searching Mail via mail-reader
Then combine and present results chronologically.

### Rich Text Extraction
When multiple messages have `[Rich Text/Hidden]` content, spawn a single subagent to extract all of them in one batch rather than querying one at a time.
