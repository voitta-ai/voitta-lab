# agent-relay

Relay messages between Claude Code sessions on different machines, using a
private Slack channel as the bus. This is a prototype for #4 and #5.

Each machine runs one `relay` Claude Code session. It polls the channel for
messages addressed to its machine and hands them to local sessions with
`SendMessage`. It also posts local sessions' outgoing messages to the channel.
Every connection is outbound HTTPS to Slack, so it works from a machine that
refuses inbound connections.

## Terms

- **Machine**: named by its role, in `~/.agent-relay/self`. Use `work` for a
  managed work machine (outbound only) and `personal` for a personal one. Add a
  device suffix only when two machines share a role.
- **Address**: `<session-name>@<machine>`, e.g. `blog:drafts@personal`.
- **Relay session**: the Claude Code session named `relay` on each machine,
  following [`RELAY.md`](RELAY.md).

## How a message travels

1. A session sends `SendMessage` to `relay`. The first line is
   `to: <session>@<machine>`, the rest is the body.
2. The relay runs `bin/relay-post`:
   - It checks that both names match the allowlist.
   - It scrubs the body with voitta-yolt's `secret_redact`, and refuses to post
     if that cannot be loaded.
   - It posts `to:` and `from:` header lines plus the body to the channel.
3. On the target machine, `bin/relay-slack-poll` runs under the relay's
   `Monitor`. It turns messages addressed to that machine into inbox files and
   prints their paths.
4. That relay delivers each one with `SendMessage`, labelled as peer authority
   rather than the user's.

## Properties and limits

- **Scope**: sender and target session names must both match
  `~/.agent-relay/allow`. Keep work-related sessions out of it.
- **Secrets**: the token is read from the macOS Keychain through a file
  descriptor. Message bodies travel on stdin only, so neither appears on a
  command line.
- **Authority**: a relayed message is a peer's request, not the user's. Receiving
  sessions treat it the way they treat any `SendMessage`.
- **Latency**: polls every 15 s. Each poll reads one page of 100 messages, so a
  burst larger than that would drop the oldest.
- **Delivery**: the last Slack ts is saved after each message, so a restart
  neither loses nor repeats messages. A crash between writing the inbox file and
  saving the ts can deliver a message twice.

## Setup, per machine

1. **Slack app.** Create it from [`slack-app-manifest.yml`](slack-app-manifest.yml),
   giving each machine a distinct name. Install it, then store the bot token
   (the command prompts for it without echoing):
   `security add-generic-password -a "$USER" -s agent-relay-slack -w`
2. **Channel.** Create one private channel and invite every machine's bot to it.
3. **Files:**

   ```bash
   mkdir -p ~/.agent-relay/{bin,inbox,done,dead} && chmod 700 ~/.agent-relay ~/.agent-relay/*
   cp bin/* ~/.agent-relay/bin/ && cp RELAY.md ~/.agent-relay/
   cp allow.example ~/.agent-relay/allow      # then edit it
   echo personal > ~/.agent-relay/self       # this machine's role
   ln -s /path/to/voitta-yolt/hooks/secret_redact.py ~/.agent-relay/secret_redact.py
   ```

4. **Channel id.** Once the bot is in the channel:

   ```bash
   ~/.agent-relay/bin/slack-api conversations.list types=private_channel \
     | python3 -c 'import json,sys; print(next(c["id"] for c in json.load(sys.stdin)["channels"] if c["name"]=="<channel>"))' \
     > ~/.agent-relay/slack-channel
   ```

5. **Relay session:**

   ```bash
   cd ~/.agent-relay && claude -n relay "Read RELAY.md in this directory and follow it as your standing brief. Start now by arming the Monitor on bin/relay-slack-poll, then wait for events."
   ```

A Keychain item with another name works too: put its name in
`~/.agent-relay/keychain-item`.

## Test on one machine

From any allowed session, send a message to yourself through the relay:

```bash
echo 'loopback test' | ~/.agent-relay/bin/relay-post <your-session>@<this-machine> <your-session>
```

The relay should hand it back to you within about 15 s. To test the relay's
outbound path, `SendMessage` the relay with `to: <your-session>@<this-machine>`
on the first line.

## Requirements

bash, curl, python3, and the macOS `security` command, plus a voitta-yolt
checkout for `secret_redact.py`.
