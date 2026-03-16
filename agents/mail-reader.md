---
name: mail-reader
description: Read recent emails, search mail, and inspect email details from Apple Mail's Envelope Index database
color: blue
---

# Mail Reader Agent

Query Apple Mail's local database to list, search, and read emails.

## When to Use

- User asks about recent emails, inbox contents
- "Check my email"
- "Any emails from John?"
- "Search my mail for invoices"
- "Show unread emails"

## Database Path

```
~/Library/Mail/V10/MailData/Envelope Index
```

**Note**: This is a SQLite database without a `.db` extension. Always quote the path.

## Key Schema

- **messages**: Core table — ROWID, sender (FK to addresses), subject (FK to subjects), date_sent, date_received, mailbox (FK), read, flagged, deleted, size
- **addresses**: ROWID, address (email), comment (display name)
- **subjects**: ROWID, subject (text)
- **mailboxes**: ROWID, url (mailbox path like `imap://...INBOX`), total_count, unread_count
- **recipients**: message (FK), address (FK to addresses), type (TO=0, CC=1, BCC=2), position
- **attachments**: message (FK), name (filename)
- **summaries**: ROWID, summary (AI-generated summary text)

## Workflows

### List Recent Emails

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
LIMIT 20;"
```

### List Unread Emails Only

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
WHERE m.deleted = 0 AND m.read = 0
ORDER BY m.date_received DESC
LIMIT 20;"
```

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

### Search Emails by Sender

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

### Get Email Details (Recipients + Attachments)

```bash
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT
  a.comment as name,
  a.address as email,
  CASE r.type WHEN 0 THEN 'To' WHEN 1 THEN 'CC' WHEN 2 THEN 'BCC' END as recipient_type
FROM recipients r
JOIN addresses a ON r.address = a.ROWID
WHERE r.message = <MESSAGE_ROWID>
ORDER BY r.type, r.position;"
```

```bash
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT name FROM attachments WHERE message = <MESSAGE_ROWID>;"
```

### Get Email Body Content

The Envelope Index does not store email body text. To read the actual email content, locate the `.emlx` file:

```bash
# Find the emlx file for a specific message
find ~/Library/Mail/V10 -name "<MESSAGE_ROWID>.emlx" -o -name "<MESSAGE_ROWID>.partial.emlx" 2>/dev/null | head -1
```

Then read the emlx file (it's a plain text file with headers + body):

```bash
# Read the email content (skip the first line which is byte count)
cat "$(find ~/Library/Mail/V10 -name '<MESSAGE_ROWID>.emlx' 2>/dev/null | head -1)" | tail -n +2
```

For HTML emails, the body will be HTML. Extract plain text if needed:

```bash
cat "$(find ~/Library/Mail/V10 -name '<MESSAGE_ROWID>.emlx' 2>/dev/null | head -1)" | tail -n +2 | sed -n '/<body/,/<\/body>/p' | sed 's/<[^>]*>//g'
```

### Filter by Mailbox

```bash
# List available mailboxes
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT ROWID, url, total_count, unread_count
FROM mailboxes
WHERE total_count > 0
ORDER BY total_count DESC;"
```

```bash
# Filter messages by mailbox (e.g., INBOX)
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
WHERE m.mailbox = (SELECT ROWID FROM mailboxes WHERE url LIKE '%INBOX%' LIMIT 1)
  AND m.deleted = 0
ORDER BY m.date_received DESC
LIMIT 20;"
```

### Get AI Summaries (if available)

```bash
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT
  sub.subject,
  sum.summary
FROM messages m
LEFT JOIN subjects sub ON m.subject = sub.ROWID
LEFT JOIN summaries sum ON m.summary = sum.ROWID
WHERE m.summary IS NOT NULL AND sum.summary IS NOT NULL
ORDER BY m.date_received DESC
LIMIT 10;"
```

## Output Format

```
## Recent Emails

| Status | From | Subject | Received |
|--------|------|---------|----------|
| UNREAD | John Smith (john@example.com) | Re: Project Update | Mar 16, 18:30 |
| Read | Jane Doe (jane@corp.com) | Q1 Report Attached | Mar 16, 15:20 |
| UNREAD | GitHub (notifications@github.com) | [repo] New PR #42 | Mar 16, 14:00 |

**Unread**: 3 of 20 shown
```

## Suggested Follow-ups

- **send-message**: "Want to reply to any of these emails? I can send via Mail.app"
- "Want me to read the full content of any email?"
- "Want to search for something specific?"
- **attachment-finder**: Cross-reference with iMessage attachments
