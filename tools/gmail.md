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
  - MTAs can add even more header fields automatically
    - e.g., `Message-ID:`, `Received:`, `Delivered-To:`, etc.
- envelope is used by MTA
  - `MAIL FROM:<bob@example.org>` tells the MTA the return address in case the
    message cannot be delivered
    - the recipient MTA adds `Return-Path: <bob@example.org>` based on this
  - `RCPT TO:<bob@example.org>` tells the MTA the destination address(es)
  - `DATA` tells the MTA the message
  - the local MTA can generate the envelope from the message header, or be
    instructed to use a specific envelope
    - e.g., `RCPT TO:` and `To:` can be unrelated

## Modern SMTP Flow

- sender composes email using MUA
  - email consists of message header and message body
  - message header consists of fields explicitly added by sender or
    automatically added by MUA
- sender sends email to local MTA (e.g., msmtp)
- local MTA relays email to its configured relay MTA
  - this talks to the configured relay MTA over SMTP with TLS
  - `AUTH PLAIN <base64-encoded-username-and-password>` authenticates
  - `MAIL FROM:<sender>` is the envelope-from addr
  - `RCPT TO:<recipient>` is the envelope-to addr
  - `DATA` is followed by email data
  - relay MTA might reject based on any of above
- relay MTA relays email to recipient MTA
  - relay MTA looks up MX records for the recipients
  - relay MTA talks to recipient MTA over SMTP with TLS
  - recipient MTA might reject based on
    - IP-based blacklist of relay MTAs
    - `MAIL FROM` / `RCPT TO`
    - SPF / DKIM / DMARC
- Sender Policy Framework (SPF)
  - SPF prevents forged `MAIL FROM` (but not forged `From:`)
    - e.g., `bob@example.org` uses `smtp.example.org` as the relay MTA which
      rewrites the sender to `MALI FROM:<bob@bounce.example.org>`
    - recipient MTA looks up SPF record for `bounce.example.org`, which
      specifies `smtp.example.org` as the valid relay MTA
    - recipient accepts the email only if it passes SPF
      - if a bad actor uses a different relay MTA but the same `MALI FROM`,
        the email fails SPF
    - IOW, the domain owner of `example.org` adds a SPF record to specify
      valid relay MTAs for the domain
  - third-party relay MTA
    - e.g., `bob@example.org` uses `smtp.thirdparty.com` as the relay MTA
      which rewrites the sender to `MALI FROM:<bob@bounce.example.org>`
    - the third party requires the domain owner of `example.org` to add a
      record, `CNAME bounce.example.org bounce.thirdparty.com`
    - this way, recipient MTA will look up the SPF record for
      `bounce.thirdparty.com`, which is controlled by third party
  - SPF record
    - e.g., `v=spf1 include:spf.thirdparty.com ~all`
    - `v=spf1` means spf version 1
    - `include:spf.thirdparty.com` means use the spf record for
      `spf.thirdparty.com`
    - `~all` means all other MTAs should be soft-rejected
- DomainKeys Identified Mail (DKIM)
  - DKIM prevents forged email
    - e.g., `bob@example.org` uses `smtp.example.org` as the relay MTA which
      adds `DKIM-Signature:` field as the signature of the email
    - recipient MTA looks up DKIM record for `default._domainkey.example.org`,
      which specifies the pubkey for signature verification
    - recipient accepts the email only if it passes DKIM
      - if a bad actor uses a different relay MTA, the email fails DKIM
    - IOW, the domain owner of `example.org`
      - configures its relay MTAs to sign emails
      - adds a DKIM record to specify the pubkey
  - third-party relay MTA
    - e.g., `bob@example.org` uses `smtp.thirdparty.com` as the relay MTA
    - the third party requires the domain owner of `example.org` to add a
      record, `CNAME default._domainkey.example.org dkim.thirdparty.com`
    - this way, recipient MTA will look up the DKIM record for
      `dkim.thirdparty.com`, which is controlled by third party
  - `DKIM-Signature:`
    - `v=1` is DKIM v1
    - `a=rsa-sha256` is the signing algorithm
    - `d=example.org; s=default` is the signing domain and selector
      - recipient MTA will look up the DKIM record for `<s>._domainkey.<d>`
    - `h=from : subject : to : message-id : date` is an ordered list of fields
      that are hashed
    - `bh=<hash>` is the message body hash
    - `b=<signature>` is the signature
  - DKIM record
    - e.g., `v=DKIM1; k=rsa; p=<pubkey>`
    - `v=DKIM1` is DKIM v1
    - `k=rsa` is rsa key
    - `p=...` is pubkey
- Domain-based Message Authentication, Reporting, and Conformance (DMARC)
  - DMARC defines the SPF/DKIM policy
    - e.g., `bob@example.org` uses `smtp.example.org` as the relay MTA
    - recipient MTA looks up DMARC record for `_dmarc.example.org`, which
      specifies the policy
    - recipient accepts the email only if it passes DMARC
  - DMARC record
    - e.g., `v=DMARC1; p=quarantine; rua=mailto:example@mxtoolbox.dmarc-report.com; ruf=mailto:example@forensics.dmarc-report.com; fo=1`
    - `v=DMARC1` is DMARC v1
    - `p=quarantine` means to treat the email as spam if it fails SPF/DKIM
    - `rua=` and `ruf` means to send aggregate and failure reports to the
      specified addresses
    - `fo=1` means to report when either SPF or DKIM fails
    - `adkim=r` is the default
      - it means `DKIM-Signature: d=<domain>; ...` and `From: foo@<domain>`
        must have aligned domain
    - `aspf=r` is the default
      - it means `MAIL FROM:<foo@domain>` and `From: foo@<domain>` must have
        aligned domain

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
