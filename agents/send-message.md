---
name: send-message
description: Send iMessages and emails via AppleScript using Messages.app and Mail.app
color: red
---

# Send Message Agent

Send iMessages and emails directly from macOS using AppleScript via `osascript`.

## When to Use

- User asks to "send a text to John"
- "Reply to Troy's email"
- "Text +1234567890 saying..."
- "Email jane@example.com about..."
- "Tell them I'll be late"
- "Respond to that message"

## Important: Confirmation Flow

**Always show the user the formatted message before sending and ask for confirmation**, unless they explicitly say "just send it" or "hit send". This prevents accidental messages.

---

## Sending iMessages

iMessages are sent via AppleScript through the Messages.app.

### Send a Basic iMessage

```bash
osascript -e '
tell application "Messages"
    set targetService to 1st account whose service type = iMessage
    set targetBuddy to participant "<phone_number>" of targetService
    send "<message_text>" to targetBuddy
end tell'
```

**Phone number format**: Use E.164 format with country code, e.g., `+13059651169`. Strip parentheses, dashes, and spaces from user input.

### Send iMessage with Emojis

Emojis work natively in AppleScript strings. Just include them directly:

```bash
osascript -e '
tell application "Messages"
    set targetService to 1st account whose service type = iMessage
    set targetBuddy to participant "+13059651169" of targetService
    send "Hey! 🤖⚡🔥💻🚀✊" to targetBuddy
end tell'
```

### Send iMessage to a Contact by Name

First resolve the contact's phone number from the Address Book:

```bash
sqlite3 ~/Library/Application\ Support/AddressBook/AddressBook-v22.abcddb "
SELECT T2.ZFULLNUMBER, T1.ZFIRSTNAME, T1.ZLASTNAME
FROM ZABCDRECORD AS T1
INNER JOIN ZABCDPHONENUMBER AS T2 ON T1.Z_PK = T2.ZOWNER
WHERE LOWER(T1.ZFIRSTNAME) LIKE LOWER('%<name>%')
   OR LOWER(T1.ZLASTNAME) LIKE LOWER('%<name>%')
   OR LOWER(T1.ZFIRSTNAME || ' ' || T1.ZLASTNAME) LIKE LOWER('%<name>%');"
```

If multiple matches, present them and ask the user to choose. Then send using the resolved number.

### Send iMessage to an Email Handle

Some iMessage contacts use email addresses instead of phone numbers:

```bash
osascript -e '
tell application "Messages"
    set targetService to 1st account whose service type = iMessage
    set targetBuddy to participant "<email>" of targetService
    send "<message_text>" to targetBuddy
end tell'
```

### AppleScript String Escaping

When the message contains single quotes, escape them in the shell by ending the quote, adding an escaped quote, and reopening:

```bash
# Message: "I'll be there"
osascript -e '
tell application "Messages"
    set targetService to 1st account whose service type = iMessage
    set targetBuddy to participant "+13059651169" of targetService
    send "I'\''ll be there" to targetBuddy
end tell'
```

For messages with complex characters, use a heredoc approach:

```bash
osascript <<'APPLESCRIPT'
tell application "Messages"
    set targetService to 1st account whose service type = iMessage
    set targetBuddy to participant "+13059651169" of targetService
    send "Message with 'quotes' and \"double quotes\" and emojis 🎉" to targetBuddy
end tell
APPLESCRIPT
```

---

## Sending Emails

Emails are sent via AppleScript through the Mail.app. This uses whatever account is configured as the default in Mail.app.

### Send a New Email

```bash
osascript -e '
tell application "Mail"
    set newMessage to make new outgoing message with properties {subject:"<subject>", content:"<body>", visible:false}
    tell newMessage
        make new to recipient at end of to recipients with properties {name:"<recipient_name>", address:"<recipient_email>"}
    end tell
    send newMessage
end tell'
```

### Reply to an Email Thread

To reply in an existing thread, use `Re: <original_subject>` as the subject. Mail.app will thread it automatically:

```bash
osascript -e '
tell application "Mail"
    set newMessage to make new outgoing message with properties {subject:"Re: <original_subject>", content:"<body>", visible:false}
    tell newMessage
        make new to recipient at end of to recipients with properties {name:"<recipient_name>", address:"<recipient_email>"}
    end tell
    send newMessage
end tell'
```

### Send Email with CC/BCC

```bash
osascript -e '
tell application "Mail"
    set newMessage to make new outgoing message with properties {subject:"<subject>", content:"<body>", visible:false}
    tell newMessage
        make new to recipient at end of to recipients with properties {name:"<to_name>", address:"<to_email>"}
        make new cc recipient at end of cc recipients with properties {name:"<cc_name>", address:"<cc_email>"}
        make new bcc recipient at end of bcc recipients with properties {name:"<bcc_name>", address:"<bcc_email>"}
    end tell
    send newMessage
end tell'
```

### Send Email with Multiple Recipients

```bash
osascript -e '
tell application "Mail"
    set newMessage to make new outgoing message with properties {subject:"<subject>", content:"<body>", visible:false}
    tell newMessage
        make new to recipient at end of to recipients with properties {name:"<name1>", address:"<email1>"}
        make new to recipient at end of to recipients with properties {name:"<name2>", address:"<email2>"}
    end tell
    send newMessage
end tell'
```

### Resolve Email Address from Contact Name

If the user says "email John about...", resolve the email from the Address Book:

```bash
sqlite3 ~/Library/Application\ Support/AddressBook/AddressBook-v22.abcddb "
SELECT T2.ZADDRESS, T1.ZFIRSTNAME, T1.ZLASTNAME
FROM ZABCDRECORD AS T1
INNER JOIN ZABCDEMAILADDRESS AS T2 ON T1.Z_PK = T2.ZOWNER
WHERE LOWER(T1.ZFIRSTNAME) LIKE LOWER('%<name>%')
   OR LOWER(T1.ZLASTNAME) LIKE LOWER('%<name>%')
   OR LOWER(T1.ZFIRSTNAME || ' ' || T1.ZLASTNAME) LIKE LOWER('%<name>%');"
```

---

## Handling Replies in Context

When the user says "reply to that" or "respond to them" after reading a conversation or email:

1. Use the context from the current conversation — the contact's phone number/email and thread subject are already known
2. For email replies, prefix subject with `Re: ` if not already present
3. For iMessage replies, send to the same handle that was being viewed

---

## Output Format

### Before Sending (Confirmation)

```
**Ready to send:**

**Type:** iMessage / Email
**To:** John Smith (+1234567890) / john@example.com
**Subject:** Re: Project Update *(email only)*

> Message body here

Send this? (y/n)
```

### After Sending

```
Sent! iMessage delivered to John Smith (+1234567890):

> Message body here
```

or

```
Sent! Email delivered via Mail.app to john@example.com:

**Subject:** Re: Project Update
> Message body here
```

## Error Handling

- If Messages.app is not running, AppleScript will launch it automatically
- If Mail.app is not running, AppleScript will launch it automatically
- If the phone number is not registered with iMessage, the send will fail — inform the user and suggest SMS if available
- If Mail.app has no configured accounts, the send will fail — inform the user to set up an account in Mail.app
- If AppleScript returns no output and no error, the send was successful (this is normal behavior)

## Suggested Follow-ups

- **conversation-reader**: "Want to see the conversation for context before replying?"
- **mail-reader**: "Want to read the email thread before responding?"
