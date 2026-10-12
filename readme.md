


# lib_MailListener

Mail listener library for Convertigo: it polls IMAP mailboxes, stores every new mail once in the FullSync database `maillistener` with its attachment files, hands it to the business sequence of each listener, which decides how it is handled, keeps a full trace per listener (status, errors, retries) and purges old mails. Several listeners (Scheduler jobs, possibly on several Convertigo instances) can share the database.

## How it works

Schedule `mail_poll` (administration console > Scheduler, every 5 minutes; avoid sub-minute polling, IMAP servers throttle it), one job per listener. Each run:

1. takes the lock of its listener in the `state-<listener>` document: a second run of the same listener skips while the first one is live, a lock older than `lock_ttl_minutes` is considered stale and taken over;
2. fetches the mails whose UID is greater than the stored cursor (at most `max_per_poll` per run) and writes their attachment files to temp files while the mailbox is open. The first run, or a mailbox renumbering (UIDVALIDITY change), starts from "now": the mailbox history is never replayed;
3. `mail_ingest` stores each mail once as `mail-<key>` (the key comes from its Message-ID) with its attachment files, claims it for the listener in `proc-<listener>-<key>`, hands each newly claimed mail to the business sequence through `mail_route`, then moves the cursor forward over the mails actually stored and claimed;
4. `mail_replay` retries the processing documents of the listener left `pending` or in `error` (up to `max_attempts`);
5. `mail_purge` deletes the mail and processing documents older than the retention;
6. deletes the temp files, releases the lock, writes `lastRunAt` / `lastError` in `state-<listener>`, logs a one-line summary and returns the run summary (see below). The run fails (visible in the Scheduler log, with the one-line summary as message) when something failed.

The mailbox is opened read-only: mails are neither flagged as read nor moved.

## Symbols

Define them in the administration console (Symbols), then reload the project. The IMAP ones and `attachments.max_mb` are the defaults of the `mail_poll` variables of the same name: a Scheduler job overrides them to watch another mailbox (see Run options).

| Symbol | Default | Purpose |
|---|---|---|
| `lib_maillistener.imap.host` | `imap.gmail.com` | IMAP server. |
| `lib_maillistener.imap.port` | `993` | IMAPS port (implicit TLS 1.2/1.3, server name checked). |
| `lib_maillistener.imap.user` | `changeme` | Mailbox login (full address). |
| `lib_maillistener.imap.password.secret` | *(empty)* | Mailbox password. Gmail / Google Workspace: an app password (2-Step Verification required, IMAP allowed by the Workspace admin). The `.secret` suffix makes the engine store it ciphered; it is masked in logs. Required: an empty value fails the run. |
| `lib_maillistener.imap.folder` | `INBOX` | Folder to watch. With Gmail a label can be used: a Gmail filter that labels the relevant mails means only those are read. |
| `lib_maillistener.imap.authserv_id` | `mx.google.com` | authserv-id of the receiving mail server: only the topmost `Authentication-Results` header written by this server is trusted for sender verification. |
| `lib_maillistener.retention_days` | `30` | Mail and processing documents (any status, with their attachment files) stored longer than this are deleted. Decimals allowed; an invalid or non-positive value means 30. |
| `lib_maillistener.attachments.max_mb` | `500` | Maximum size of one stored attachment file. Bigger files are listed as skipped (`too_large`). Note: Gmail refuses incoming mails above 25 MB. |

## Run options

Variables of `mail_poll`, set them on the Scheduler job:

| Variable | Default | Purpose |
|---|---|---|
| `listener` | `inbox` | Listener id (1 to 40 letters, digits, `_` or `-`), one per job. It owns `state-<listener>` (cursor, lock) and its processing documents. |
| `business_sequence` | `lib_MailListener.demo_business` | Business sequence that decides how every new mail is handled, as `project.sequence`. It can be a sequence of any project loaded in the engine; a `mail_*` or `admin_*` sequence of this library is refused. |
| `imap_host`, `imap_port`, `imap_user`, `imap_folder`, `imap_password` | the `lib_maillistener.imap.*` symbols | Mailbox of the listener. |
| `attachments_max_mb` | the `lib_maillistener.attachments.max_mb` symbol | Maximum size of one stored attachment file. |
| `max_per_poll` | `50` | Mails fetched per run; the rest is picked up by the next run. |
| `lock_ttl_minutes` | `60` | Age after which a lock is considered left by a crashed run. |

`mail_replay` also has `max_attempts` (default 3).

## Several listeners

Several listeners can share the database `maillistener`, each with its own Scheduler job, mailbox and business sequence, on one Convertigo instance or on several (pods):

- each job sets its `listener` and its `business_sequence`, and overrides `imap_user` / `imap_password` (or `imap_host`, `imap_folder`) when it watches another mailbox or folder. Scheduler job values are stored in clear: to keep a password out of them, schedule instead a small sequence of your project that calls `lib_MailListener.mail_poll` with a Call Sequence step whose `imap_password` step variable takes its value from a `.secret` symbol of your project (for example a symbol named `myproject.support.imap.password.secret`, referenced with an empty default);
- a mail is stored once: its document id comes from its Message-ID (`mail-<key>`), so the same mail read by several listeners, or read twice, is not stored again (creating an existing document answers 409). Its attachment files are stored by the read that creates it;
- each listener claims the mail in its own processing document `proc-<listener>-<key>` (status, route, attempts, error), created only when absent: its business sequence gets the mail once, whatever the other listeners do. Replay and the admin page work on these processing documents;
- concurrency: a second run of the same listener (the same job on another instance) skips while the `state-<listener>` lock is held (compare-and-set write). Different listeners never write the same document, except the shared mail document: it is created once (atomic create) and completed with its files by its creator; a listener that reads the mail meanwhile does not call its business sequence (status `pending`, retried by `mail_replay`) until the files are stored, or until one hour has passed. Purges running at the same time are tolerated;
- two listeners on the same mailbox and folder each read every mail (stored once, decided by each business sequence): use a folder or a Gmail label per listener to avoid useless reads.

## Run summary

`mail_poll` returns one `result` object per run and writes its `summary` line in the engine log (Context logger, INFO level), for example `mail_poll inbox ok: 3 mails read (1 routed, 1 unmatched, 1 already processed), 2 stored for the first time; replay: 1 retried (1 routed, 0 unmatched, 0 pending, 0 in error); 2.4 s`. The log line holds counts and errors only, never a sender or a subject.

| Field | Content |
|---|---|
| `status` | `ok`, `failed`, or `skipped` when another run of the listener holds the lock (`reason` says why). |
| `listener` | Listener of the run. |
| `summary` | The one-line summary; it starts with `FAILED:` and the errors when the run failed. |
| `startedAt`, `finishedAt`, `durationMs` | Run timing. |
| `businessSequence` | Business sequence used by the run. |
| `mailbox` | `account`, `folder`, `uidValidity`, `uidNext`, `cursorBefore` / `cursorAfter` (last handled UID), `firstRun`, `hasMore` (mails left for the next run). |
| `counts` | `read`, `stored` (stored for the first time by this run), then the number of read mails per status: `routed`, `unmatched`, `error`, `pending`, `alreadyProcessed`, `storeError`, `notSaved`. |
| `mails` | One line per mail read: `uid`, `docid`, `procid`, `receivedAt`, `from` (bare address), `senderVerified`, `subject` (80 characters), `attachments` (count), `stored`, `status`, `route`, `error`. Never the body. |
| `replay` | `candidates`, `retried`, `routed`, `unmatched`, `pending`, `errors`, `skippedMaxAttempts` (mails left in `error` after `max_attempts`), and one line per retried mail: `docid`, `procid`, `from`, `senderVerified`, `subject`, `status`, `route`, `attempts`, `error`. |
| `purge` | `retentionDays`, `cutoff`, `expired`, `deleted`, `conflicts` (handled by another run at the same time), `failed`, `hasMore`. |
| `errors` | Run errors, prefixed by their stage (`fetch:`, `ingest:`, `replay:`, `purge:`, lock release). |

Status of a mail read: `routed`, `unmatched` or `error`, decided for this listener (see Decision); `pending` when the business sequence was not called yet (attachment files still being stored by another listener, or `mail_route` failed; `mail_replay` retries it); `already_processed` when this listener had handled the mail before; `store_error` when the database rejected the full mail (minimal trace kept); `not_saved` when it could not be saved or claimed (the cursor stays before it and the next run reads it again).

A failed run ends with an exception so that the Scheduler shows it: the engine then returns the error instead of the `result` object, and the error message is the `summary` line.

## Decision

The library applies no matching rule: every mail claimed by a listener goes to the business sequence of that listener, which decides how it is handled and returns its decision. The outcome is written on the processing document of the listener (`status`, `route`, `routedAt`, `attempts`, `error`):

- `routed`: the business sequence handled the mail and returned a non-empty `route`, its own label for the treatment, shown in the admin page and the run summary;
- `unmatched`: the business sequence returned an empty `route`: the mail is not for it. It is kept for traceability until the purge;
- `error`: the business sequence failed (see below). `mail_replay` retries it up to `max_attempts`, then it stays in `error` for a manual look;
- `store_error`: the database rejected the full mail; a minimal trace is kept so the cursor can move on. It is not retried.

Sender verification is done once, by `mail_ingest`: the mail must have a single From address, the topmost `Authentication-Results` header must come from `lib_maillistener.imap.authserv_id`, and it must report `dmarc=pass` for the From domain. The result is stored in `fromAddress` and `senderVerified`, for the business sequence to use; the display name is never used. A rule on the content (subject, body) can be triggered by anyone: do not attach a sensitive action to it without checking `senderVerified`.

## How the business sequence is launched

For every mail newly claimed by the listener, and every mail retried by `mail_replay`, `mail_route`:

1. calls the sequence named by `business_sequence` (passed from `mail_poll` through `mail_ingest` / `mail_replay`), synchronously, inside the polling run, with one variable: `docid` (`mail-<key>`). Technically, the `call_business` Call Sequence step keeps `demo_business` as design-time target and its `__project` / `__sequence` step variables redirect the call at runtime;
2. reads its decision and writes the status on the processing document. The mail goes to `error`, and `mail_replay` calls the business sequence again later, when the business sequence throws an exception or returns an error structure (`XMLErrorStep`), when it returns no `decision` element, when the sequence does not exist in the project, or when the project is not loaded ("could not be called"). A business sequence must therefore be idempotent, and quick (it runs inside the polling run and its lock).

Business sequence contract:

- input variable: `docid`;
- read the mail with `lib_MailListener.maillistener.get_document` (`_use_docid` = `docid`; a project calling it needs a reference to `lib_MailListener`): `from`, `fromAddress`, `senderVerified`, `to`, `cc`, `replyTo`, `deliveredTo`, `subject`, `body`, `bodyTruncated`, `sentAt`, `receivedAt`, `attachments`, `attachmentsStatus`. The recipient fields are lists of bare lowercase addresses (at most 100 each); `replyTo` is the sender when the mail has no Reply-To header; `deliveredTo` is the address the mailbox received the mail for (alias or group), topmost first, and the only trace of a Bcc recipient. Mails stored before these fields existed do not have them;
- decide with your own rules and return a top-level `decision` element with a `route`, for example a JSON to XML step with key `decision` and value `{ route: "open_ticket" }`. A non-empty route means handled (status `routed`), an empty route means not handled (status `unmatched`);
- the attachment files are stored with the mail before the business sequence is called. Read one with `lib_MailListener.maillistener.get_attachment` (`_use_docid` = `docid`, `_use_attname` = `attachments[i].storedName`, only for `stored: true` entries). Without `_use_attpath` the file comes back in `couchdb_output/data`: text types as text, binary types as base64; keep that for small files. With `_use_attpath` the file is written to that path. `attachmentsStatus` is `stored` (every file), `partial` (some files skipped: too large or unreadable), `error` (an upload failed) or `none`;
- report a failure by throwing or with an `XMLErrorStep` (then stop the sequence with a Return step).

`demo_business` is the reference implementation of this contract, laid out so that only two places change:

- `init`, `get_mail`, `read_mail` and `prepare_mail` read the mail and expose its fields: `subject`, `body`, `from_address`, `from_domain`, `sender_verified`, `to`, `cc`, `reply_to`, `delivered_to`, `sent_at`, `files` (stored attachment files), `attachments_status`, `mail`. `if_mail_missing` answers an error structure for a missing mail;
- `rules` is the part to customize: it holds the matching rules and sets `route` (empty = not handled) and `action`. The demo has `rule_urgent` (the subject contains `URGENT`, any sender) and `rule_domain` (verified sender from exactly `convertigo.com`), plus a commented example of a rule on the recipient alias (`delivered_to`);
- `decision` returns the route to `mail_route` (contract: keep it);
- `if_handled` holds the treatment of a handled mail: the demo reads the stored files (up to 5 MB in memory), logs the action and returns a summary. Replace these steps by your own work.

Copy it into your project, change `rules` and the steps of `if_handled`, and set `business_sequence` to it, for example `myProject.process_mail`.

## Admin page

The NGX application of the project is an admin page that lists one line per mail and listener (the processing documents), newest first, 25 per page: subject (shortened to 80 characters), sender, reception date, status (coloured pill) and route, and a paperclip with the number of attachments when the mail has some. One checkbox per status, with its count, filters the list; Previous / Next buttons page through it and Refresh reloads the current page.

The page loads through the shared action `LoadMailsPage`, which calls `admin_list_mails` (variables `listener` (empty = all, the page shows all listeners for now), `page`, `page_size` (max 100), `statuses`: comma-separated, empty = all). Each line carries its `listener`, and the result lists every listener found. Mail documents of the former single-listener format are listed as listener `inbox` until the purge. That sequence only answers a browser session logged into the Convertigo administration console as an administrator (WEB_ADMIN role). It checks the session itself: Private alone is not enough, because the Studio lets anyone run Private sequences and a server lets any console user with the test platform role run them. Log into the console first, then open the application in the same browser (`.../convertigo/projects/lib_MailListener/DisplayObjects/mobile/` on a server, the viewer URL in the Studio). Anybody else gets "Access denied" (error code 403) and no data. The list reads at most 10 000 documents; a note is shown when that limit is reached.

## Data

Database `maillistener` (FullSync connector):

- `mail-<key>`: one document per mail, shared by the listeners (Message-ID, mailbox, sender, recipients, subject, sent and received dates, body up to 200 000 characters, attachment metadata and files, `storedBy`);
- `proc-<listener>-<key>`: processing of a mail by a listener (status, route, attempts, error, and the fields shown by the admin page);
- `state-<listener>`: cursor, UIDVALIDITY, lock, last run and last error of a listener.

All sequences are Private: only `mail_poll` is meant to be scheduled, the others are called by it (or by the admin page) and can be run alone from the Studio.


For more technical informations : [documentation](./project.md)

- [Installation](#installation)
- [Mobile Library](#mobile-library)


## Installation

1. In your Convertigo Studio click on ![](https://github.com/convertigo/convertigo/blob/develop/eclipse-plugin-studio/icons/studio/project_import.gif?raw=true "Import a project in treeview") to import a project in the treeview
2. In the import wizard

   ![](https://github.com/convertigo/convertigo/blob/develop/eclipse-plugin-studio/tomcat/webapps/convertigo/templates/ftl/project_import_wzd.png?raw=true "Import Project")
   
   paste the text below into the `Project remote URL` field:
   <table>
     <tr><td>Usage</td><td>Click the copy button at the end of the line</td></tr>
     <tr><td>To contribute</td><td>

     ```
     lib_MailListener=https://github.com/convertigo/c8oprj-lib-mail-listener.git:branch=master
     ```
     </td></tr>
     <tr><td>To simply use</td><td>

     ```
     lib_MailListener=https://github.com/convertigo/c8oprj-lib-mail-listener/archive/master.zip
     ```
     </td></tr>
    </table>
3. Click the `Finish` button. This will automatically import the __lib_MailListener__ project


## Mobile Library

Describes the mobile application global properties



