
# ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/core/images/project_color_16x16.png?raw=true "Project") lib_MailListener

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

<details><summary><span style="color:DarkGoldenRod"><i>Connectors</i></span></summary><blockquote><p>


<details><summary><b>maillistener</b></summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/connectors/images/fullsyncconnector_color_16x16.png?raw=true "FullSyncConnector") maillistener



<details><summary><span style="color:DarkGoldenRod"><i>Transactions</i></span></summary><blockquote><p>


<details><summary><b>delete_documents</b> : Bulk write used by mail_purge to delete documents: _use_json_base is an array of {_id, _rev, _deleted: true}</summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/postbulkdocuments_color_16x16.png?raw=true "PostBulkDocumentsTransaction") delete_documents

Bulk write used by mail_purge to delete documents: _use_json_base is an array of {_id, _rev, _deleted: true}.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_json_base
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>find_expired</b> : Mango query for mail_purge: mail and processing documents older than the retention (selector built by mail_purge and passed in _use_selector)</summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/postfind_color_16x16.png?raw=true "PostFindTransaction") find_expired

Mango query for mail_purge: mail and processing documents older than the retention (selector built by mail_purge and passed in _use_selector). The default selector matches nothing.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_selector
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>find_mails</b> : Mango query for the admin list: every processing document (one per mail and listener), plus the mail documents of the former single-listener format that still carry a status (until the purge)</summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/postfind_color_16x16.png?raw=true "PostFindTransaction") find_mails

Mango query for the admin list: every processing document (one per mail and listener), plus the mail documents of the former single-listener format that still carry a status (until the purge). 10 000 at most; sorting, filtering and paging are done by admin_list_mails.
</p></blockquote></details>

<details><summary><b>find_retry_candidates</b> : Mango query for mail_replay: processing documents of one listener left pending (interrupted run, attachment files still being stored) or in error (business sequence failure)</summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/postfind_color_16x16.png?raw=true "PostFindTransaction") find_retry_candidates

Mango query for mail_replay: processing documents of one listener left pending (interrupted run, attachment files still being stored) or in error (business sequence failure). The selector (with the listener) is built by mail_replay and passed in _use_selector.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_selector
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>get_attachment</b> : Reads one attachment file of a mail document for business sequences: attachment name = attachments[i]</summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/getdocumentattachment_color_16x16.png?raw=true "GetDocumentAttachmentTransaction") get_attachment

Reads one attachment file of a mail document for business sequences: attachment name = attachments[i].storedName. Without _use_attpath the file comes back as base64 in couchdb_output/data (keep it for small files); with _use_attpath it is written to that path.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_attname
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_attpath
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_docid
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>get_document</b></summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/getdocument_color_16x16.png?raw=true "GetDocumentTransaction") get_document



<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_docid
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>get_view</b></summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/getview_color_16x16.png?raw=true "GetViewTransaction") get_view



<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_ddoc
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_endkey
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_include_docs
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_key
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_startkey
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_view
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>post_document</b></summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/postdocument_color_16x16.png?raw=true "PostDocumentTransaction") post_document



<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_id
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_json_base
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>put_attachment</b> : Uploads one attachment file (temp file written by mail_poll from the IMAP part) onto a mail document</summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/putdocumentattachment_color_16x16.png?raw=true "PutDocumentAttachmentTransaction") put_attachment

Uploads one attachment file (temp file written by mail_poll from the IMAP part) onto a mail document. Requires the current _rev.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_attbase64
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_attcontent_type
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_attname
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_attpath
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_docid
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_rev
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>update_document</b> : Updates an existing document by merging the posted fields (state-<listener> cursor, processing status, attachment metadata)</summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/postdocument_color_16x16.png?raw=true "PostDocumentTransaction") update_document

Updates an existing document by merging the posted fields (state-<listener> cursor, processing status, attachment metadata). The attachments array is replaced, not merged (p_merge). CouchDB attachments (_attachments) are kept. post_document keeps policy none so creating an existing document stays a 409 (store once, claim once).

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_id
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;_use_json_base
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>
</p></blockquote></details>
</p></blockquote></details>

<details><summary><b>void</b> : Placeholder SQL connector for template projects</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/connectors/images/sqlconnector_color_16x16.png?raw=true "SqlConnector") void

Placeholder SQL connector for template projects

<details><summary><span style="color:DarkGoldenRod"><i>Transactions</i></span></summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/images/sqltransaction_color_16x16.png?raw=true "SqlTransaction") void

Placeholder transaction that intentionally cancels execution
</p></blockquote></details>
</p></blockquote></details>
</p></blockquote></details>

<details><summary><span style="color:DarkGoldenRod"><i>Sequences</i></span></summary><blockquote><p>


<details><summary><b>admin_list_mails</b> : Admin list for the NGX admin page: one line per mail and listener (processing documents), newest first, one page at a time</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") admin_list_mails

Admin list for the NGX admin page: one line per mail and listener (processing documents), newest first, one page at a time. Only answers a browser session logged into the Convertigo administration console as an administrator (WEB_ADMIN role), in the Studio too; any other caller gets an error structure (code 403) and nothing is read. Variables: listener (empty = all), page (from 1), page_size (25, max 100), statuses (comma-separated, empty = all, 'none' = nothing). Returns {items: [{docid, listener, from, subject (80 chars max), receivedAt, status, route, hasAttachments, attachmentCount}], total (after the filters), page, pageSize, pageCount, first, last, all (lines of the kept listener, before the status filter), capped (true when the 10 000 documents limit was reached), listeners (every listener found), counts: {routed, unmatched, error, pending, store_error}, statusCounts: [{status, count}] (same counts, as a list)}.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;listener
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;page
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;page_size
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;statuses
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>demo_business</b> : Reference implementation of the business sequence contract, default value of the business_sequence variable of mail_poll</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") demo_business

Reference implementation of the business sequence contract, default value of the business_sequence variable of mail_poll. Input: docid (mail-<key>). Steps: init, get_mail, read_mail and prepare_mail read the mail and expose its fields (subject, body, from_address, from_domain, sender_verified, files); if_mail_missing answers an error structure for a missing mail; rules is the part to customize: it sets route (empty = not handled) and action; decision returns the route to mail_route (contract, keep it); if_handled holds the treatment of a handled mail (the demo reads the stored files with maillistener.get_attachment, logs the action and returns a summary; replace it by your own work). Idempotent. Copy it into your project, change rules and if_handled, then set business_sequence to yourProject.yourSequence.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;docid
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>mail_ingest</b> : Stores a batch of fetched mails for one listener</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") mail_ingest

Stores a batch of fetched mails for one listener. Each mail is stored once in the database as mail-<key>, shared by every listener (409 = already stored by a previous run or by another listener; rejected document = minimal store_error trace), with its attachment files when this read creates it. It is then claimed for this listener (proc-<listener>-<key>, 409 = already processed by this listener) and each newly claimed mail goes to mail_route. Finally the state-<listener> cursor is committed; it only advances over mails stored (or traced) and claimed. Called by mail_poll under its lock; callable alone in Studio with a synthetic batch.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;business_sequence
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;has_more
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;last_uid
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;listener
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;mails
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;uid_next
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;uid_validity
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>mail_poll</b> : Scheduled entry point, one Scheduler job per listener</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") mail_poll

Scheduled entry point, one Scheduler job per listener. Takes the state-<listener> lock (skips while another run of the same listener is live, on any Convertigo instance), polls the IMAP mailbox of the listener (imap_* variables, defaults = lib_maillistener.imap.* symbols) for mails with uid > lastUid (at most max_per_poll) and writes their attachment files to temp files, hands the batch to mail_ingest (store once, claim for this listener, business sequence, cursor), retries the pending / error mails of the listener with mail_replay, purges old documents with mail_purge, deletes the temp files, then releases the lock and records lastError. First run (or UIDVALIDITY change) starts from the current UIDNEXT. Returns a run summary (key result: status ok / failed / skipped, listener, one-line summary, mailbox cursor, counts, one line per mail read, replay, purge, errors) and logs the one-line summary at INFO.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;attachments_max_mb
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;business_sequence
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;imap_folder
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;imap_host
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;imap_password
</td>
<td>
Masked in logs, Studio and platform. The project file only holds the symbol reference; the secret itself lives in the engine symbol lib_maillistener.imap.password.secret (stored ciphered thanks to the .secret suffix).
</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;imap_port
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;imap_user
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;listener
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;lock_ttl_minutes
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;max_per_poll
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>mail_purge</b> : Retention: deletes mail and processing documents (any status, any listener) stored more than retention_days ago (lib_maillistener</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") mail_purge

Retention: deletes mail and processing documents (any status, any listener) stored more than retention_days ago (lib_maillistener.retention_days, default 30), 200 per run. State documents are never touched. Purges run by other listeners or instances at the same time are tolerated: conflict / not_found answers are counted as conflicts, not failures. Called by mail_poll under its lock; callable alone in Studio.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;retention_days
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>mail_replay</b> : Retries the processing documents of one listener left pending (interrupted run, attachment files still being stored by another listener) or in error (business sequence failure) through mail_route, until max_attempts business attempts</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") mail_replay

Retries the processing documents of one listener left pending (interrupted run, attachment files still being stored by another listener) or in error (business sequence failure) through mail_route, until max_attempts business attempts. store_error mails are not retried. Called by mail_poll under its lock; callable alone in Studio.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;business_sequence
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;listener
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;max_attempts
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>mail_route</b> : Hands a stored mail to the business sequence of one listener, which decides how it is handled, then writes status/route/routedAt/attempts/error on the processing document of that listener</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") mail_route

Hands a stored mail to the business sequence of one listener, which decides how it is handled, then writes status/route/routedAt/attempts/error on the processing document of that listener. Input: docid (mail-<key>), procid (proc-<listener>-<key>), business_sequence (project.sequence, default lib_MailListener.demo_business). The business sequence returns a decision element: route not empty = routed with that route, empty = unmatched. While another listener is still uploading the attachment files of the mail (attachmentsStatus storing, less than one hour), the business sequence is not called and the status stays pending. Business failure (exception, XMLErrorStep, sequence not found, project not loaded, no decision element): error, retried by mail_replay.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;business_sequence
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;docid
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;procid
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>
</p></blockquote></details>

<details><summary><span style="color:DarkGoldenRod"><i>Mobile Application</i></span></summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/core/images/mobileapplication_color_16x16.png?raw=true "MobileApplication") Application

Describes the mobile application global properties

<details><summary><span style="color:DarkGoldenRod"><i>Pages</i></span></summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/ngx/components/images/pagecomponent_color_16x16.png?raw=true "PageComponent") Page

Admin list of the stored mails. Calls the Private sequence admin_list_mails: on a server engine it only answers a browser session already logged into the Convertigo administration console.
</p></blockquote></details>

<details><summary><span style="color:DarkGoldenRod"><i>Shared Actions</i></span></summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/ngx/components/images/uiactionstack_color_16x16.png?raw=true "UIActionStack") LoadMailsPage

Loads one page of the admin mail list (admin_list_mails) with the checked status filters into page.local.results. Used by page enter, Refresh, Previous/Next and the filter checkboxes.

<span style="color:DarkGoldenRod">Variables</span>

<table>
<tr>
<th>
name
</th>
<th>
comment
</th>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/ngx/components/images/uistackvariable_16x16.png?raw=true "  alt="UIStackVariable" >&nbsp;page
</td>
<td>
Page to load, from 1
</td>
</tr>
</table>

</p></blockquote></details>
</p></blockquote></details>
