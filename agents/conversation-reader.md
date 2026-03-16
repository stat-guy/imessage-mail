---
name: conversation-reader
description: Read full message and email history for a specific contact — resolves names, extracts rich text, shows attachments, cross-references iMessage and Apple Mail
color: green
---

# Conversation Reader Agent

Retrieve and display the message history for a specific contact across iMessage and Apple Mail.

## When to Use

- User provides a phone number or contact name and wants to see messages
- "Show me my texts with John"
- "Messages from +1234567890"
- "Emails from Jane"
- "All communications with John"

## Workflow

### Step 1: Resolve the Contact

**If the user provides a name** (not a phone number or email), find the matching handle and email:

```bash
sqlite3 ~/Library/Application\ Support/AddressBook/AddressBook-v22.abcddb "
SELECT T2.ZFULLNUMBER, T1.ZFIRSTNAME, T1.ZLASTNAME
FROM ZABCDRECORD AS T1
INNER JOIN ZABCDPHONENUMBER AS T2 ON T1.Z_PK = T2.ZOWNER
WHERE LOWER(T1.ZFIRSTNAME) LIKE LOWER('%<name>%')
   OR LOWER(T1.ZLASTNAME) LIKE LOWER('%<name>%')
   OR LOWER(T1.ZFIRSTNAME || ' ' || T1.ZLASTNAME) LIKE LOWER('%<name>%');"
```

Also resolve their email address for Mail lookups:

```bash
sqlite3 ~/Library/Application\ Support/AddressBook/AddressBook-v22.abcddb "
SELECT T2.ZADDRESS, T1.ZFIRSTNAME, T1.ZLASTNAME
FROM ZABCDRECORD AS T1
INNER JOIN ZABCDEMAILADDRESS AS T2 ON T1.Z_PK = T2.ZOWNER
WHERE LOWER(T1.ZFIRSTNAME) LIKE LOWER('%<name>%')
   OR LOWER(T1.ZLASTNAME) LIKE LOWER('%<name>%')
   OR LOWER(T1.ZFIRSTNAME || ' ' || T1.ZLASTNAME) LIKE LOWER('%<name>%');"
```

If multiple matches, present them and ask the user to choose.

Then verify the number exists in chat.db:

```bash
sqlite3 ~/Library/Messages/chat.db "SELECT id FROM handle WHERE id LIKE '%<last_7_digits>%';"
```

**If the user provides a phone number**, extract digits only and use directly.

**If the user provides an email**, use it directly for Mail lookup, and also check if it exists as an iMessage handle.

### Step 2: Fetch iMessages

```bash
sqlite3 ~/Library/Messages/chat.db "
SELECT
  m.ROWID,
  CASE WHEN m.is_from_me = 1 THEN 'Me' ELSE 'Them' END as sender,
  COALESCE(m.text, CASE WHEN m.attributedBody IS NOT NULL THEN '[Rich Text/Hidden]' ELSE '[No Text]' END) as content,
  datetime(m.date / 1000000000 + 978307200, 'unixepoch', 'localtime') as timestamp,
  a.filename,
  a.mime_type
FROM message m
JOIN handle h ON m.handle_id = h.ROWID
LEFT JOIN message_attachment_join maj ON m.ROWID = maj.message_id
LEFT JOIN attachment a ON maj.attachment_id = a.ROWID
WHERE h.id LIKE '%<digits_only>'
ORDER BY m.date DESC
LIMIT <count>;"
```

Default `<count>` to 25, increase if user asks for more.

### Step 3: Fetch Emails from Apple Mail

Run in parallel with Step 2 using a **subagent** if the contact has an email address:

```bash
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT
  m.ROWID,
  CASE WHEN mb.url LIKE '%Sent%' THEN 'Me' ELSE a.comment END as sender,
  sub.subject,
  datetime(m.date_received, 'unixepoch', 'localtime') as received,
  CASE WHEN m.read = 1 THEN 'Read' ELSE 'UNREAD' END as status,
  m.size
FROM messages m
LEFT JOIN addresses a ON m.sender = a.ROWID
LEFT JOIN subjects sub ON m.subject = sub.ROWID
LEFT JOIN mailboxes mb ON m.mailbox = mb.ROWID
WHERE (a.address LIKE '%<email_or_domain>%' OR a.comment LIKE '%<contact_name>%')
  AND m.deleted = 0
ORDER BY m.date_received DESC
LIMIT 20;"
```

To also find emails **sent to** this contact:

```bash
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT
  m.ROWID,
  'Me' as sender,
  sub.subject,
  datetime(m.date_sent, 'unixepoch', 'localtime') as sent,
  m.size
FROM messages m
JOIN recipients r ON r.message = m.ROWID
JOIN addresses a ON r.address = a.ROWID
LEFT JOIN subjects sub ON m.subject = sub.ROWID
WHERE (a.address LIKE '%<email>%' OR a.comment LIKE '%<contact_name>%')
  AND m.deleted = 0
ORDER BY m.date_sent DESC
LIMIT 20;"
```

### Step 4: Get Email Body Content

For emails the user wants to read in full, find and read the `.emlx` file:

```bash
find ~/Library/Mail/V10 -name "<MESSAGE_ROWID>.emlx" -o -name "<MESSAGE_ROWID>.partial.emlx" 2>/dev/null | head -1
```

Then read it (skip byte count on first line):

```bash
cat "$(find ~/Library/Mail/V10 -name '<MESSAGE_ROWID>.emlx' 2>/dev/null | head -1)" | tail -n +2
```

For HTML-heavy emails, strip tags:

```bash
cat "$(find ~/Library/Mail/V10 -name '<MESSAGE_ROWID>.emlx' 2>/dev/null | head -1)" | tail -n +2 | sed -n '/<body/,/<\/body>/p' | sed 's/<[^>]*>//g' | head -50
```

### Step 5: Get Email Attachments

```bash
sqlite3 ~/Library/Mail/V10/MailData/"Envelope Index" "
SELECT att.name
FROM attachments att
WHERE att.message = <MESSAGE_ROWID>;"
```

### Step 6: Extract Rich Text Bodies (iMessage)

For any iMessage where content is `[Rich Text/Hidden]`, extract the actual text. Spawn a **subagent** to process these in parallel:

```
Use the Agent tool to extract rich text for multiple ROWIDs at once.

Prompt: "For each of these message ROWIDs, run the following command and return the ROWID paired with the extracted text:

for ROWID in <ROWID1> <ROWID2> <ROWID3>; do
  echo "---ROWID:$ROWID---"
  sqlite3 ~/Library/Messages/chat.db "SELECT HEX(attributedBody) FROM message WHERE ROWID = $ROWID;" | xxd -r -p | strings | head -3
done"
```

Replace `[Rich Text/Hidden]` in the output with the extracted text.

### Step 7: Handle Reactions and Threads (Optional, iMessage only)

Check for tapback reactions on messages:

```bash
sqlite3 ~/Library/Messages/chat.db "
SELECT
  m.ROWID,
  m.associated_message_type,
  m.associated_message_guid
FROM message m
JOIN handle h ON m.handle_id = h.ROWID
WHERE h.id LIKE '%<digits_only>'
  AND m.associated_message_type != 0
ORDER BY m.date DESC
LIMIT 20;"
```

Tapback types: 2000=Love, 2001=Like, 2002=Dislike, 2003=Laugh, 2004=Emphasis, 2005=Question

## Output Format

Present iMessages and emails in separate sections, or interleaved chronologically if the user asks for "all communications":

```
## Conversation with John Smith

### iMessages (+1234567890)
*25 most recent*

**14:30 Me**: Hey, are you free tonight?
**14:32 John**: Yeah! What's up?
**14:33 Me**: Check out this photo 📎 IMG_1234.jpg
**14:35 John**: 😂 That's hilarious

---

### Emails (john@example.com)
*10 most recent*

| Status | Direction | Subject | Date |
|--------|-----------|---------|------|
| Read | From John | Re: Project Update | Mar 16, 10:00 |
| Read | To John | Project Update | Mar 15, 14:30 |
| UNREAD | From John | Invoice #1234 | Mar 14, 09:00 |
```

Use the conversational format for iMessages by default unless the user asks for a table.

## Suggested Follow-ups

- **send-message**: "Want to reply to this conversation?"
- **attachment-finder**: "Want to see all attachments from this conversation?"
- Show older messages: "Want me to go further back?"
- "Want me to read any of these emails in full?"
