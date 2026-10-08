


# lib_MailListener

Mail listener library for Convertigo: it polls an IMAP mailbox, stores every new mail in the FullSync database `maillistener`, routes it with rules to a configurable business sequence, stores the attachment files of routed mails, keeps a full trace (status, errors, retries) and purges old mails.

## How it works

Schedule `mail_poll` (administration console > Scheduler, every 5 minutes; avoid sub-minute polling, IMAP servers throttle it). Each run:

1. takes a lock in the `state-inbox` document: a second run skips while the first one is live, a lock older than `lock_ttl_minutes` is considered stale and taken over;
2. fetches the mails whose UID is greater than the stored cursor (at most `max_per_poll` per run). The first run, or a mailbox renumbering (UIDVALIDITY change), starts from "now": the mailbox history is never replayed;
3. `mail_ingest` stores each mail as a `mail-<uid>` document (status `pending`), routes each new one through `mail_route`, then moves the cursor forward over the mails actually stored;
4. `mail_replay` retries the mails left `pending` or in `error` (up to `max_attempts`);
5. `mail_purge` deletes the mail documents older than the retention;
6. releases the lock and writes `lastRunAt` / `lastError` in `state-inbox`. The run fails (visible in the Scheduler log) when something failed.

The mailbox is opened read-only: mails are neither flagged as read nor moved.

## Symbols

Define them in the administration console (Symbols), then reload the project.

| Symbol | Default | Purpose |
|---|---|---|
| `lib_maillistener.imap.host` | `imap.gmail.com` | IMAP server. |
| `lib_maillistener.imap.port` | `993` | IMAPS port (implicit TLS 1.2/1.3, server name checked). |
| `lib_maillistener.imap.user` | `changeme` | Mailbox login (full address). |
| `lib_maillistener.imap.password.secret` | *(empty)* | Mailbox password. Gmail / Google Workspace: an app password (2-Step Verification required, IMAP allowed by the Workspace admin). The `.secret` suffix makes the engine store it ciphered; it is masked in logs. Required: an empty value fails the run. |
| `lib_maillistener.imap.folder` | `INBOX` | Folder to watch. With Gmail a label can be used: a Gmail filter that labels the relevant mails means only those are read. |
| `lib_maillistener.imap.authserv_id` | `mx.google.com` | authserv-id of the receiving mail server: only the topmost `Authentication-Results` header written by this server is trusted for sender verification. |
| `lib_maillistener.retention_days` | `30` | Mail documents (any status, with their attachment files) stored longer than this are deleted. Decimals allowed; an invalid or non-positive value means 30. |
| `lib_maillistener.attachments.max_mb` | `500` | Maximum size of one stored attachment file. Bigger files are listed as skipped (`too_large`). Note: Gmail refuses incoming mails above 25 MB. |

## Run options

Variables of `mail_poll`, set them on the Scheduler job:

| Variable | Default | Purpose |
|---|---|---|
| `business_sequence` | `lib_MailListener.demo_business` | Business sequence called for every matched mail, as `project.sequence`. It can be a sequence of any project loaded in the engine. |
| `max_per_poll` | `50` | Mails fetched per run; the rest is picked up by the next run. |
| `lock_ttl_minutes` | `60` | Age after which a lock is considered left by a crashed run. |

`mail_replay` also has `max_attempts` (default 3).

## Routing

`mail_route` evaluates the rules of its `read_mail_fields` step in order; the first match gives the route:

| Route | Rule |
|---|---|
| `rule_urgent` | The subject contains `URGENT` (case-insensitive), whatever the sender. |
| `rule_domain` | The sender is verified and its address domain is exactly `convertigo.com`. |

Sender verification is done once, by `mail_ingest`: the mail must have a single From address, the topmost `Authentication-Results` header must come from `lib_maillistener.imap.authserv_id`, and it must report `dmarc=pass` for the From domain. The result is stored in `fromAddress` and `senderVerified`; the display name is never used. A content rule such as `rule_urgent` can be triggered by anyone: do not attach sensitive actions to it without checking `senderVerified`.

To add a route, add a condition in `read_mail_fields` that sets `decided = "rule_x"`. Nothing else is needed: every route goes to the business sequence with its name.

Outcome written on the mail document (`status`, `route`, `routedAt`, `attempts`, `error`):

- `routed`: a rule matched, the attachment files are stored and the business sequence succeeded;
- `unmatched`: no rule matched. The mail is kept for traceability (attachment files not stored) until the purge;
- `error`: the attachment storage or the business sequence failed. `mail_replay` retries it up to `max_attempts`, then it stays in `error` for a manual look;
- `store_error`: the database rejected the full mail; a minimal trace is kept so the cursor can move on. It is not retried.

## How the business sequence is launched

For a matched route, `mail_route`:

1. stores the attachment files (`mail_store_attachments`): the message is fetched again from IMAP by UID and each file becomes a CouchDB attachment of `mail-<uid>`, named `<index>-<safe name>` (`attachments[i].storedName`; the original name stays in `attachments[i].name`). If this fails, the business sequence is not called and the mail goes to `error`;
2. calls the sequence named by `business_sequence` (passed from `mail_poll` through `mail_ingest` / `mail_replay`), synchronously, inside the polling run, with two variables: `docid` (`mail-<uid>`) and `route` (the matched route name, to branch on). Technically, the `call_business` Call Sequence step keeps `demo_business` as design-time target and its `__project` / `__sequence` step variables redirect the call at runtime;
3. writes the status. The mail goes to `error`, and `mail_replay` calls the business sequence again later, when the business sequence throws an exception or returns an error structure (`XMLErrorStep`), when the sequence does not exist in the project, or when the project is not loaded ("could not be called"). A business sequence must therefore be idempotent, and quick (it runs inside the polling run and its lock).

Business sequence contract:

- input variables: `docid` and `route`;
- read the mail with `lib_MailListener.maillistener.get_document` (`_use_docid` = `docid`; a project calling it needs a reference to `lib_MailListener`): `from`, `fromAddress`, `senderVerified`, `subject`, `body`, `bodyTruncated`, `receivedAt`, `attachments`;
- read an attachment file with `lib_MailListener.maillistener.get_attachment` (`_use_docid` = `docid`, `_use_attname` = `attachments[i].storedName`, only for `stored: true` entries). Without `_use_attpath` the file comes back in `couchdb_output/data`: text types as text, binary types as base64; keep that for small files. With `_use_attpath` the file is written to that path;
- report a failure by throwing or with an `XMLErrorStep` (then stop the sequence with a Return step).

`demo_business` is the reference implementation of this contract: it reads the mail, rejects a missing mail or a `rule_domain` mail without verified sender with an error structure, picks an action per route (`open_ticket`, `file_documents`, `log_only`), reads the stored attachment files (up to 5 MB in memory), logs the action and returns a summary. Copy it into your project, replace the action by your own logic, and set `business_sequence` to it, for example `myProject.process_mail`.

## Admin page

The NGX application of the project is an admin page that lists the stored mails, newest first, 25 per page: subject (shortened to 80 characters), sender, reception date, routing status (coloured pill) and route, and a paperclip with the number of attachments when the mail has some. One checkbox per routing status, with its count, filters the list; Previous / Next buttons page through it and Refresh reloads the current page.

The page loads through the shared action `LoadMailsPage`, which calls `admin_list_mails` (variables `page`, `page_size` (max 100), `statuses`: comma-separated, empty = all). That sequence only answers a browser session logged into the Convertigo administration console as an administrator (WEB_ADMIN role). It checks the session itself: Private alone is not enough, because the Studio lets anyone run Private sequences and a server lets any console user with the test platform role run them. Log into the console first, then open the application in the same browser (`.../convertigo/projects/lib_MailListener/DisplayObjects/mobile/` on a server, the viewer URL in the Studio). Anybody else gets "Access denied" (error code 403) and no data. The list reads at most 10 000 mail documents; a note is shown when that limit is reached.

## Data

Database `maillistener` (FullSync connector): one `mail-<uid>` document per mail (sender, subject, body up to 200 000 characters, attachment metadata and files, routing status) and the `state-inbox` document (cursor, UIDVALIDITY, lock, last run, last error). All sequences are Private: only `mail_poll` is meant to be scheduled, the others are called by it (or by the admin page) and can be run alone from the Studio.


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



