Google Services
===============

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

## GMail Search Operators

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

## `gcloud`

- free tier
  - <https://cloud.google.com/free/docs/free-cloud-features#free-tier>
  - compute engine
    - `e2-micro`
    - `us-west1`
    - 30 GB-months standard persistent disk
- <https://cloud.google.com/sdk/docs>
  - untar the tarball
  - `install.sh` or use `google-cloud-sdk/bin/gcloud` directly
- components
  - `gcloud components list`
  - `gcloud components update`
- init
  - `gcloud init`
- auth
  - `gcloud auth list` lists accounts
  - `gcloud auth login` logs in
- compute
  - os login
    - `gcloud compute project-info add-metadata --metadata=enable-oslogin=true`
    - `gcloud compute os-login ssh-keys add --key-file ~/.ssh/id_rsa.pub`
    - the user name is the email with special characters (`@` or `.`) replaced
      by `_`
  - quick ssh
    - `gcloud compute ssh <vm-instance>`
    - this adds the ssh key to the vm temporarily and ssh to it

## Gemini CLI

- <https://geminicli.com/docs/get-started/>
  - install
  - auth
  - `gemini -s [-m gemini-3.1-pro-preview] [-r latest]`
- cli options
  - <https://geminicli.com/docs/cli/cli-reference/>
  - `gemini` is interactive
  - `gemini <query>` is one-shot
    - `-i` to force interactive after one-shot
  - `-m` selects the model
  - `-s` sandboxes gemini for safer execution
  - `extensions` manages extensions
  - `mcp list` manages mcp
  - `skills` manages skills
  - sessions
    - `--list-sessions` lists sessions
    - `-r <session>` resumes a session
      - `-r latest` resumes the lastest session
- commands
  - `/about` shows version info
  - `/commands` manages custom commands
    - `~/.gemini/commands/*.toml` or `<project>/.gemini/commands/*.toml`
  - `/extensions` manages extensions
    - an extension is a package of custom commands, hooks, skills, etc.
  - `/hooks` manages hooks
    - `settings.json` has a `hooks` key to describe what hooks to run on what events
      - `~/.gemini/hooks/*` are hook executables
  - `/init` creates project `GEMINI.md` automatically
  - `/mcp` manages mcp
    - an mcp server is an external service that gemini cli can make use of
  - `/memory`
    - `~/.gemini/GEMINI.md` or `<project>/GEMINI.md` sets the context for gemini
  - `/model` selects a model
    - the latest is `gemini-3.1-pro-preview`
  - `/permissions` manages permissions
    - `~/.gemini/trustedFolders.json` is trusted folders
      - it prevents a malicious dir from tricking gemini cli
  - `/plan` switches to plan mode
    - to outline a plan of modifications before executing them
  - `/policies`
    - `~/.gemini/policies/*.toml` or `<project>/.gemini/policies/*.toml`
    - they define policies (allow, deny, ask, etc.) for tools
  - `/quit` quits gemini cli
  - `/settings`
    - `~/.gemini/settings.json` or `<workspace>/.gemini/settings.json`
  - `/skills`
    - `~/.gemini/skills/` or `<workspace>/.gemini/skills/`
      - `<skill>/SKILL.md` defines a skill
  - `/stats` shows session/model/tool stats
  - `/tools` lists available built-in tools
    - e.g., list dir, read file, write file, grep, etc.
  - `!<shell command>` runs a shell command
  - session-related
    - `/chat` saves/resumes a checkpoint
    - `/clear` clears the current session (and screen)
    - `/compress` compresses the current session to a summary
    - `/directory` includes extra directories for the current session
    - `/rewind` jumps back and restarts from the specified message
    - `/resume` resumes a selected session
