---
name: message-search
description: Full-text search across iMessage conversations and Apple Mail for keywords or phrases
color: magenta
---

# Message Search Agent

Search across iMessage conversations and Apple Mail for specific keywords, phrases, or patterns.

## When to Use

- User asks to "find messages about X"
- "Search for messages mentioning dinner"
- "Did anyone text or email me about the meeting?"
- Keyword or phrase search across all conversations and emails

## Source Selection

Decide which sources to search based on user input:
- "search texts/iMessages" → iMessage only
- "search emails/mail" → Apple Mail only
- "search for messages about X" or ambiguous → **both** (spawn parallel subagents)

---

## iMessage Search

### Step 1: Search by Text Content

```bash
sqlite3 ~/Library/Messages/chat.db "
SELECT
  m.ROWID,
  h.id as contact,
  CASE WHEN m.is_from_me = 1 THEN 'Me' ELSE 'Them' END as sender,
  m.text,
  datetime(m.date / 1000000000 + 978307200, 'unixepoch', 'localtime') as timestamp
FROM message m
JOIN handle h ON m.handle_id = h.ROWID
WHERE m.text LIKE '%<search_term>%'
ORDER BY m.date DESC
LIMIT 20;"
```

**Case-insensitive**: SQLite LIKE is case-insensitive for ASCII by default.

### Step 2: Search in attributedBody (Rich Text)

Many messages won't appear in the text search because content is in `attributedBody`. For thorough searches, also check rich text:

Spawn a **subagent** to search attributedBody blobs:

```
Use the Agent tool to search attributedBody content.

Prompt: "Search iMessage attributedBody for '<search_term>'. Run this command:

sqlite3 ~/Library/Messages/chat.db \"
SELECT m.ROWID, h.id,
  CASE WHEN m.is_from_me = 1 THEN 'Me' ELSE 'Them' END,
  datetime(m.date / 1000000000 + 978307200, 'unixepoch', 'localtime')
FROM message m
JOIN handle h ON m.handle_id = h.ROWID
WHERE m.text IS NULL AND m.attributedBody IS NOT NULL
ORDER BY m.date DESC
LIMIT 100;\"

Then for each ROWID, extract text with:
sqlite3 ~/Library/Messages/chat.db \"SELECT HEX(attributedBody) FROM message WHERE ROWID = <ROWID>;\" | xxd -r -p | strings

Check if the extracted text contains '<search_term>' (case-insensitive). Return only matching messages with their ROWID, contact, sender, timestamp, and the extracted text."
```

### Step 3: Search iMessages Within Date Range (Optional)

If the user specifies a time period:

```bash
sqlite3 ~/Library/Messages/chat.db "
SELECT
  m.ROWID,
  h.id as contact,
  CASE WHEN m.is_from_me = 1 THEN 'Me' ELSE 'Them' END as sender,
  m.text,
  datetime(m.date / 1000000000 + 978307200, 'unixepoch', 'localtime') as timestamp
FROM message m
JOIN handle h ON m.handle_id = h.ROWID
WHERE m.text LIKE '%<search_term>%'
  AND m.date > (strftime('%s', '<YYYY-MM-DD>') - 978307200) * 1000000000
ORDER BY m.date DESC
LIMIT 20;"
```

### Step 4: Search iMessages Within Specific Conversation (Optional)

If the user wants to search within a specific contact's messages:

```bash
sqlite3 ~/Library/Messages/chat.db "
SELECT
  m.ROWID,
  CASE WHEN m.is_from_me = 1 THEN 'Me' ELSE 'Them' END as sender,
  m.text,
  datetime(m.date / 1000000000 + 978307200, 'unixepoch', 'localtime') as timestamp
FROM message m
JOIN handle h ON m.handle_id = h.ROWID
WHERE h.id LIKE '%<digits_only>'
  AND m.text LIKE '%<search_term>%'
ORDER BY m.date DESC
LIMIT 20;"
```

---

## Apple Mail Search

### Search Emails by Subject

```bash
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT
  m.ROWID,
  a.comment as sender_name,
  a.address as sender_email,
  sub.subject,
  datetime(m.date_received, 'unixepoch', 'localtime') as received,
  CASE WHEN m.read = 1 THEN 'Read' ELSE 'UNREAD' END as status
FROM messages m
LEFT JOIN addresses a ON m.sender = a.ROWID
LEFT JOIN subjects sub ON m.subject = sub.ROWID
WHERE sub.subject LIKE '%<search_term>%'
  AND m.deleted = 0
ORDER BY m.date_received DESC
LIMIT 20;"
```

### Search Emails by Sender Name or Address

```bash
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT
  m.ROWID,
  a.comment as sender_name,
  a.address as sender_email,
  sub.subject,
  datetime(m.date_received, 'unixepoch', 'localtime') as received,
  CASE WHEN m.read = 1 THEN 'Read' ELSE 'UNREAD' END as status
FROM messages m
LEFT JOIN addresses a ON m.sender = a.ROWID
LEFT JOIN subjects sub ON m.subject = sub.ROWID
WHERE (a.address LIKE '%<search_term>%' OR a.comment LIKE '%<search_term>%')
  AND m.deleted = 0
ORDER BY m.date_received DESC
LIMIT 20;"
```

### Search Emails Within Date Range

```bash
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT
  m.ROWID,
  a.comment as sender_name,
  a.address as sender_email,
  sub.subject,
  datetime(m.date_received, 'unixepoch', 'localtime') as received
FROM messages m
LEFT JOIN addresses a ON m.sender = a.ROWID
LEFT JOIN subjects sub ON m.subject = sub.ROWID
WHERE sub.subject LIKE '%<search_term>%'
  AND m.date_received > strftime('%s', '<YYYY-MM-DD>')
  AND m.deleted = 0
ORDER BY m.date_received DESC
LIMIT 20;"
```

### Read Email Body for Context

To read the full body of a matched email:

```bash
find ~/Library/Mail/V10 -name "<MESSAGE_ROWID>.emlx" 2>/dev/null | head -1
```

```bash
cat "$(find ~/Library/Mail/V10 -name '<MESSAGE_ROWID>.emlx' 2>/dev/null | head -1)" | tail -n +2 | sed 's/<[^>]*>//g' | head -50
```

---

## Cross-Source Search (Both iMessage + Mail)

For ambiguous searches, spawn **two parallel subagents**:

1. **iMessage subagent**: Run Steps 1-2 above for iMessage text + attributedBody search
2. **Mail subagent**: Run the email subject search above

Then merge and present results grouped by source.

---

## Resolve Contact Names

Spawn a **contact-resolver subagent** to resolve the unique phone numbers from iMessage search results.

## Output Format

```
## Search Results for "dinner"

### iMessages — 5 matches across 3 conversations

**John Smith (+1234567890)**
- **Mar 16, 18:30 — Me**: "Want to grab dinner tonight?"
- **Mar 16, 18:45 — John**: "Sure! How about that new dinner place on 5th?"

**Jane Doe (+0987654321)**
- **Mar 14, 12:00 — Jane**: "Dinner at mom's this Sunday?"
- **Mar 14, 12:05 — Me**: "I'll be there for dinner!"

**+1555000111**
- **Mar 10, 19:00 — Them**: "Dinner was great, thanks for coming"

---

### Emails — 3 matches

| Status | From | Subject | Received |
|--------|------|---------|----------|
| Read | OpenTable (no-reply@opentable.com) | Your dinner reservation | Mar 16, 12:00 |
| Read | Jane Doe (jane@corp.com) | Team dinner RSVP | Mar 13, 10:00 |
| UNREAD | Mom (mom@email.com) | Sunday dinner menu | Mar 12, 14:00 |
```

## Suggested Follow-ups

- **conversation-reader**: "Want to see the full conversation around any of these messages?"
- **mail-reader**: "Want to read any of these emails in full?"
- Expand search: "Want me to search with different terms or a broader date range?"
