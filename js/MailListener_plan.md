# Action plan — lib_MailListener

Mail "listener" connector for Convertigo: receive emails on a dedicated address
and trigger sequences based on message content.

- Status: **awaiting confirmation before implementation**
- Approved decisions: IMAP polling (embedded javax.mail, Rhino step), FullSync storage (embedded CouchDB)
- Default target: Gmail (IMAPS + app password) — to be confirmed

## 1. Goal

A Convertigo library `lib_MailListener` provides:

1. an IMAP polling sequence scheduled by the Scheduler,
2. a deduplication mechanism based on IMAP UIDs,
3. a router sequence dispatching to business sequences based on rules,
4. full traceability of processed emails (status, errors, replay).

Consumer projects reference the library and declare their own business
sequences and routing rules.

## 2. Architecture

```
Scheduler (every 5 min by default)
   └── mail_poll                     [Private]
        ├── GetDocument("state-inbox")            → lastUid cursor
        ├── JavaStep/Rhino javax.mail IMAP        → mails uid > lastUid
        ├── for each mail:
        │     ├── PostDocument(_id="mail-<uid>", status="pending")
        │     ├── PutAttachment (attachments if any)
        │     ├── CallSequence(mail_route, mail)   → routed | error
        │     └── PostDocument(final status)
        └── PostDocument("state-inbox", lastUid=max)
```

```
mail_route                          [Private]
   ├── IfStep rule 1 (sender / subject / body regex)
   │     └── CallSequence(business sequence A)
   ├── IfStep rule 2
   │     └── CallSequence(business sequence B)
   └── default: mark as "unmatched" (trace, no processing)
```

## 3. FullSync data model (database via FullSync connector)

One document per email (native dedup through deterministic `_id`):

```json
{
  "_id": "mail-<imapUID>",
  "type": "mail",
  "mailbox": "process@mydomain.com",
  "from": "", "subject": "", "body": "",
  "receivedAt": "",
  "status": "pending | routed | error | unmatched",
  "route": "", "routedAt": "", "error": null
}
```

State document:

```json
{ "_id": "state-inbox", "type": "state", "lastUid": 0, "lastPollAt": "" }
```

Design document `_design/listener`:

- view `by_status`: `emit([doc.status, doc.receivedAt], 1)` — error queue, orphan pending, statistics
- view `by_route`: `emit([doc.route, doc.receivedAt], 1)`

## 4. Technical implementation

- **IMAP step**: Rhino JavaScriptStep/JavaStep calling `javax.mail` (Session, IMAPStore,
  `com.sun.mail.imap.IMAPFolder` for UIDs: `getUIDNext()`, `getUID(message)`).
  Body extraction prefers `text/plain`, falls back to `multipart/*`.
  No JAR to add: the library is embedded in the engine (same stack as SMTPSendStep).
- **Credentials**: symbols with harmless default values, so the project runs out of the box
  in dev and the administration console overrides them per environment:
  `${imap.host=imap.gmail.com}`, `${imap.port=993}`, `${imap.user=changeme}`,
  `${imap.password=changeme}`, `${imap.folder=INBOX}` —
  password masked everywhere (`visibility`), real value provided at deployment time,
  never stored in the project. `changeme` is a placeholder, not a secret.
- **Routing rules**: in V1, rules coded in `mail_route` (IfStep on subject/sender).
  Possible V2: a `routing-rules` configuration document in the database (data-driven rules).
- **Error handling**: a failing email is marked `error` (error message traced),
  the poll continues with the next ones. Manual/scheduled replay: QueryView(by_status, key=error)
  then CallSequence(mail_route).
- **Anti-overlap lock**: the Scheduler may restart while a cycle is running; V1 = the state
  document carries `running=true/false` (the poll refuses to start if a cycle is running or if a
  `pending` document is older than a threshold).
- **Mailbox marking**: `moveToProcessed` option — moves processed mails to an IMAP
  sub-folder `Processed` (disabled by default, not needed thanks to UID dedup).

## 5. Phases

| Phase | Content | Validation |
|---|---|---|
| P1 | FullSync connector + design document + transactions (Get/Post/QueryView) | `requestable-execute` on transactions with test documents |
| P2 | Rhino IMAP step (connection, UID fetch, body/attachment extraction) | Manual run against the target mailbox, log review |
| P3 | Complete `mail_poll` sequence (cursor, dedup, loop, state) | Send a test email → `mail-<uid>` document created, status `routed` |
| P4 | `mail_route` sequence + 2 sample rules + demo business sequence | 2 test emails matching each rule → correct dispatch |
| P5 | Error replay + anti-overlap lock | Simulated failure → replay OK |
| P6 | Hardening: symbols, variable visibility, `project-security-audit`, `project-save` | Audit green, project saved |

The Scheduler (administration console) is configured **outside the project** at delivery,
on the `mail_poll` sequence, every 5 minutes.

## 6. Points to confirm before implementation

1. Target mailbox: Gmail (IMAPS, app password) or another IMAP server?
2. Dedicated mailbox address (for the `mailbox` field and tests)?
3. Two sample routing rules wanted for V1 (e.g. subject contains "URGENT" → sequence A;
   sender = a given domain → sequence B)?
4. Called business sequences will belong to a consumer project (reference to the library)
   — will V1 ship a demo sequence inside the library itself?
5. Polling interval (5 min by default)?

## 7. Gmail configuration (step by step)

### 7.1 Google account prerequisites

1. **2-Step Verification** must be enabled on the Google account
   (myaccount.google.com → Security). App passwords only exist on 2FA-enabled accounts.
2. Create an **App Password**: myaccount.google.com → Security → 2-Step Verification →
   App passwords (or directly: https://myaccount.google.com/apppasswords).
   - Select "Mail", device "Other (custom name, e.g. `lib_MailListener`)".
   - Google displays a 16-character password (`abcd efgh ijkl mnop`): save it now,
     it is shown only once. Spaces are ignored by the IMAP login.
3. **Enable IMAP** in Gmail: Settings (gear) → "See all settings" → "Forwarding and POP/IMAP"
   tab → "IMAP access" → "Enable IMAP" → Save.
4. Google Workspace accounts: the administrator may enforce policies that disable app
   passwords or IMAP; check with the domain admin if the connection is refused.

### 7.2 Convertigo symbols

Symbols ship with safe default values in the project; the administration console
(Symbols) overrides them per environment:

| Symbol | Project default | Production value (Gmail example) |
|---|---|---|
| `imap.host` | `imap.gmail.com` | `imap.gmail.com` (or your IMAP server) |
| `imap.port` | `993` | `993` (implicit TLS, protocol `imaps`) |
| `imap.user` | `changeme` | the dedicated Gmail address, e.g. `process@mydomain.com` |
| `imap.password` | `changeme` | the 16-character app password (not the account password) |
| `imap.folder` | `INBOX` | `INBOX` |

Notes:
- `imap.password` visibility must be masked everywhere (value ciphered in the project file,
  hidden in logs). The `changeme` default is a placeholder, not a secret: the real app
  password is a secret and is injected through the administration console per environment,
  never inside the project sources.

### 7.3 Validation checklist (Gmail)

1. Run `mail_poll` manually once (test button / `requestable-execute`).
2. Send a test email to the dedicated address from another account.
3. Re-run `mail_poll`; expect a `mail-<uid>` document with status `routed` (or `unmatched`
   if no rule matches), and `state-inbox.lastUid` updated.
4. If the connection fails with `AUTHENTICATIONFAILED [ALERT] ...`:
   - app password typo, or app passwords disabled on the account;
   - IMAP disabled in Gmail settings;
   - Google can require "Less secure app" alternatives — the app password is the
     supported path, "less secure apps" no longer exists.
5. Gmail rate limits: IMAP is throttled for continuous polling; a 5-minute interval is
   safe. Avoid sub-minute polling.
6. Note: Google has announced deprecation of app passwords for some account types;
   if app passwords disappear, the fallback is **XOAUTH2** (OAuth2 access token via
   SASL) — same `javax.mail` client, only the authentication layer changes
   (see §8).

## 8. Known accepted limitations

- Polling (minutes), no real-time push — approved decision.
- Incoming email is not authenticated: triggered business sequences must not perform
  sensitive actions without sender validation (allow-list in the rules).
- Gmail: IMAP access + app password must be enabled on the account; if Google policy
  changes, XOAUTH2 switch is possible with the same javax.mail client.
