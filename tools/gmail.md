Gmail
=====

## SMTP

- Port 25
  - RFC 788: SIMPLE MAIL TRANSFER PROTOCOL, 1981
  - RFC 821: SIMPLE MAIL TRANSFER PROTOCOL, 1982
  - RFC 2821: Simple Mail Transfer Protocol, 2001
  - RFC 5321: Simple Mail Transfer Protocol, 2008
- Port 587
  - RFC 2476, Message Submission, 1998
  - RFC 4409, Message Submission for Mail, 2006
  - RFC 6409, Message Submission for Mail, 2011
  - require explicit `STARTTLS` for TLS
- Port 465
  - RFC 8341, Cleartext Considered Obsolete, 2018
  - all connections are TLS

## Message Header and Envelope

- message header is a part of the message
  - user can add header fields directly
    - e.g., `To:`, `Cc:`, `Subject:`, etc.
  - MUA can add more header fields automatically
    - e.g., `From:`, `Date:`, `Content-Type:`, etc.
  - MTA and mail servers can add even more header fields automatically
    - e.g., `Message-ID:`, `Received:`, `Delivered-To:`, etc.
- envelope is used by MTA
  - `MAIL FROM:<bob@example.org>` tells the MTA the return address in case the
    message cannot be delivered
  - `RCPT TO:<bob@example.org>` tells the MTA the destination address(es)
  - `DATA` tells the MTA the message
  - the local MTA can generate the envelope from the message header, or be
    instructed to use a specific envelope
    - e.g., `RCPT TO:` and `To:` can be unrelated

## Search Operators

- search by header fields
  - `from:`, `to:`, `cc:`, `bcc:`, `list:`
    - `to:` implies `cc:` and `bcc:`
    - `list:` implies `to:`
  - `subject:`
  - `after:`, `before:`
- search by derived data
  - `larger:`, `smaller:`
  - `filename:`
  - `has:attachment`, `has:drive`, `has:youtube`
- search by metadata
  - `is:starred`, `is:unread`
  - `label:`, `has:userlabels`, `has:nouserlabels`
- operators
  - `AND` is logical and: `this AND that`
    - it is the default and can be omitted
  - `OR` is logical or: `this OR that`
    - `{this that}` can be used as well
  - `NOT` is logical not: `this NOT that`
    - `-` can be used as well
  - `()` is grouping: `subject:(this AND that)`
  - `+` is exact search: `+regressed`
  - `AROUND`: `this AROUND 30 that`
    - find `this` and `that` that are within 30 words apart
- `@` is special
  - `abc@xyz` matches `abc@xyz` or `abc.foo@xyz`
    - but not `foo.abc@xyz` nor `abc@foo.xyz`
  - otherwise, `@` seem to match anything
    - `xyz@` and `@xyz` are the same as `xyz`
      - they can match `foo.xyz.bar@foo.xyz.bar` for example
    - `{abc}@xyz` is the same as `{abc}.xyz` which is the same as `abc.xyz`
      - it matches `foo.abc.xyz.bar`
- types of group mails
  - traditional mailing list
    - sender is on `From:`
    - list is on `To:`, `Cc:`, or `Bcc:`
    - list is also on `List-Id:`
  - ads
    - sender is on `From:`
    - receiver is on `To:`, `Cc:`, or `Bcc:`
    - there is no list
  - gitlab notifications
    - sender is on `From:`
    - receiver is on `To:`, `Cc:`, or `Bcc:`
    - list is also on `List-Id:`
  - aliases
    - sender is on `From:`
    - alias is on `To:`, `Cc:`, or `Bcc:`
    - there is no list
  - because `list:` matches all relevant fields, we can always use `list:`
    - but watch out for when `@` is special

## Inbox Zero

- the first principle: skip indox if I am not on `to:` or `cc:`
  - this alone makes inbox manageable
  - if a mail is directly addressed to me and hits the inbox,
    - if it is from offending senders (e.g., ads), unsubscribe or add a
      specific `from:` skip rule
    - if it is to project lists where I am a project reviewer, add a specific
      `to:` skip rule
- the second principle: label every mail
  - label every mail that is directly addressed to me
  - label every mail that is sent to known lists
  - how about mails that are sent to unknown lists?
    - they can be important sometimes
    - find them with `has:nouserlabels` and make sure the lists are known (or
      unsubscribe)
- the third principle: star important mails
- multiple inboxes
  - the first one is the regular inbox
  - the second one is `is:starred`
  - the third one is `has:nouserlabels`
