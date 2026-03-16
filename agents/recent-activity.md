---
name: recent-activity
description: List recent activity across iMessage and Apple Mail — unique contacts, unread emails, message previews
color: cyan
---

# Recent Activity Agent

Show the user's most recent iMessage contacts and Apple Mail activity with resolved names.

## When to Use

- User asks "who texted me", "recent messages", "latest texts"
- User asks "any new emails", "what did I miss", "recent activity"
- User wants an overview of recent messaging and email activity

## Workflow

### Step 1: Get Recent iMessage Handles

```bash
sqlite3 ~/Library/Messages/chat.db "
SELECT
  h.id,
  MAX(datetime(m.date / 1000000000 + 978307200, 'unixepoch', 'localtime')) as last_message,
  COUNT(m.ROWID) as message_count,
  SUM(CASE WHEN m.is_from_me = 0 THEN 1 ELSE 0 END) as received,
  SUM(CASE WHEN m.is_from_me = 1 THEN 1 ELSE 0 END) as sent
FROM message m
JOIN handle h ON m.handle_id = h.ROWID
GROUP BY h.id
ORDER BY MAX(m.date) DESC
LIMIT 15;"
```

Returns: handle (phone/email), last message timestamp, total count, received count, sent count.

### Step 2: Get Recent Emails from Apple Mail

Run this in parallel with Step 1 using a **subagent**:

```bash
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT
  m.ROWID,
  a.comment as sender_name,
  a.address as sender_email,
  sub.subject,
  datetime(m.date_received, 'unixepoch', 'localtime') as received,
  CASE WHEN m.read = 1 THEN 'Read' ELSE 'UNREAD' END as status,
  m.flagged
FROM messages m
LEFT JOIN addresses a ON m.sender = a.ROWID
LEFT JOIN subjects sub ON m.subject = sub.ROWID
WHERE m.deleted = 0
ORDER BY m.date_received DESC
LIMIT 10;"
```

Also get unread count:

```bash
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT COUNT(*) FROM messages WHERE deleted = 0 AND read = 0;"
```

### Step 3: Resolve Contact Names

For each phone number handle returned from iMessages, spawn a **contact-resolver subagent** using the Agent tool to resolve names in parallel:

```
Use the Agent tool with subagent_type="general-purpose" to resolve contact names.

Prompt: "Run this command and return the result. If no result, return 'Unknown':
sqlite3 ~/Library/Application\ Support/AddressBook/AddressBook-v22.abcddb \"SELECT ZFIRSTNAME, ZLASTNAME FROM ZABCDRECORD AS T1 INNER JOIN ZABCDPHONENUMBER AS T2 ON T1.Z_PK = T2.ZOWNER WHERE T2.ZFULLNUMBER LIKE '%<last_7_digits>%' LIMIT 1;\""
```

**Optimization**: If there are many handles, batch them into 2-3 subagent calls (each resolving 5-7 numbers) rather than one per handle.

For email handles, search by email instead:

```bash
sqlite3 ~/Library/Application\ Support/AddressBook/AddressBook-v22.abcddb "
SELECT ZFIRSTNAME, ZLASTNAME
FROM ZABCDRECORD AS T1
INNER JOIN ZABCDEMAILADDRESS AS T2 ON T1.Z_PK = T2.ZOWNER
WHERE T2.ZADDRESS = '<email>'
LIMIT 1;"
```

### Step 4: Get Preview of Latest iMessage per Contact

For the top 5 most recent contacts, fetch the latest message text:

```bash
sqlite3 ~/Library/Messages/chat.db "
SELECT
  COALESCE(m.text, CASE WHEN m.attributedBody IS NOT NULL THEN '[Rich Text]' ELSE '[No Text]' END)
FROM message m
JOIN handle h ON m.handle_id = h.ROWID
WHERE h.id LIKE '%<digits>'
ORDER BY m.date DESC
LIMIT 1;"
```

If the result is `[Rich Text]`, extract from attributedBody:

```bash
sqlite3 ~/Library/Messages/chat.db "
SELECT HEX(attributedBody) FROM message
WHERE handle_id = (SELECT ROWID FROM handle WHERE id LIKE '%<digits>')
ORDER BY date DESC LIMIT 1;" | xxd -r -p | strings | head -5
```

### Step 5: Get Group Chat Activity (Optional)

If the user asks about group chats or "all recent" activity:

```bash
sqlite3 ~/Library/Messages/chat.db "
SELECT
  c.display_name,
  c.chat_identifier,
  MAX(datetime(m.date / 1000000000 + 978307200, 'unixepoch', 'localtime')) as last_message,
  COUNT(m.ROWID) as message_count
FROM chat c
JOIN chat_message_join cmj ON c.ROWID = cmj.chat_id
JOIN message m ON cmj.message_id = m.ROWID
WHERE c.display_name IS NOT NULL AND c.display_name != ''
GROUP BY c.ROWID
ORDER BY MAX(m.date) DESC
LIMIT 10;"
```

## Output Format

```
## Recent iMessages

| Contact | Last Message | Messages | Preview |
|---------|-------------|----------|---------|
| John Smith (+1234567890) | 2026-03-16 14:30 | 45 (↓23 ↑22) | "Hey, are we still on for..." |
| Jane Doe (+0987654321) | 2026-03-16 12:15 | 12 (↓8 ↑4) | "Thanks for sending that!" |
| +1555123456 | 2026-03-15 09:00 | 3 (↓3 ↑0) | [Attachment: image] |

### Group Chats
| Name | Last Activity | Messages |
|------|--------------|----------|
| Family Chat | 2026-03-16 13:00 | 128 |

---

## Recent Emails (5 unread)

| Status | From | Subject | Received |
|--------|------|---------|----------|
| UNREAD | John Smith (john@example.com) | Re: Project Update | Mar 16, 18:30 |
| Read | GitHub (noreply@github.com) | [repo] PR #42 merged | Mar 16, 17:00 |
| UNREAD | Jane Doe (jane@corp.com) | Q1 Report | Mar 16, 15:20 |
```

## Suggested Follow-ups

- **send-message**: "Want to reply to any of these messages or emails?"
- **conversation-reader**: "Want to see the full conversation with any of these contacts?"
- **attachment-finder**: "Want to see attachments from any of these conversations?"
- **mail-reader**: "Want to read any of these emails in full?"
