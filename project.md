
# ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/core/images/project_color_16x16.png?raw=true "Project") lib_MailListener

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
- read attachment files with a GetDocumentAttachment transaction on the `maillistener` connector (document `docid`, attachment name `storedName`);
- report a failure by throwing or with an `XMLErrorStep`.

`demo_business` is a placeholder: set `business_sequence` to your own sequence, for example `myProject.process_mail`.

## Data

Database `maillistener` (FullSync connector): one `mail-<uid>` document per mail (sender, subject, body up to 200 000 characters, attachment metadata and files, routing status) and the `state-inbox` document (cursor, UIDVALIDITY, lock, last run, last error). All sequences are Private: only `mail_poll` is meant to be scheduled, the others are called by it and can be run alone from the Studio.


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

<details><summary><b>find_expired</b> : Mango query for mail_purge: mail documents older than the retention (selector built by mail_purge and passed in _use_selector)</summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/postfind_color_16x16.png?raw=true "PostFindTransaction") find_expired

Mango query for mail_purge: mail documents older than the retention (selector built by mail_purge and passed in _use_selector). The default selector matches nothing.

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

<details><summary><b>find_retry_candidates</b> : Mango query: mails left pending (interrupted run) or in error (business sequence failure), for mail_replay</summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/postfind_color_16x16.png?raw=true "PostFindTransaction") find_retry_candidates

Mango query: mails left pending (interrupted run) or in error (business sequence failure), for mail_replay.
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

<details><summary><b>put_attachment</b> : Uploads one attachment file (temp file written from the IMAP part) onto a mail document</summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/putdocumentattachment_color_16x16.png?raw=true "PutDocumentAttachmentTransaction") put_attachment

Uploads one attachment file (temp file written from the IMAP part) onto a mail document. Requires the current _rev.

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

<details><summary><b>update_document</b> : Updates an existing document by merging the posted fields (state-inbox cursor, mail status, attachment metadata)</summary><blockquote><p>


### ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/transactions/couchdb/images/postdocument_color_16x16.png?raw=true "PostDocumentTransaction") update_document

Updates an existing document by merging the posted fields (state-inbox cursor, mail status, attachment metadata). The attachments array is replaced, not merged (p_merge). CouchDB attachments (_attachments) are kept. post_document keeps policy none so creating an existing mail-<uid> stays a 409 (dedup).

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


<details><summary><b>demo_business</b> : Placeholder business sequence, default value of the business_sequence variable of mail_poll</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") demo_business

Placeholder business sequence, default value of the business_sequence variable of mail_poll. Receives docid (mail-<uid>) and route. Replace it by setting business_sequence to your own sequence (project.sequence).

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
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;route
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>mail_ingest</b> : Stores a batch of fetched mails as mail-<uid> documents (409 = already stored; rejected document = minimal store_error trace), routes each new one through mail_route, then commits the state-inbox cursor</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") mail_ingest

Stores a batch of fetched mails as mail-<uid> documents (409 = already stored; rejected document = minimal store_error trace), routes each new one through mail_route, then commits the state-inbox cursor. The cursor only advances over mails that are stored or traced. Called by mail_poll under its lock; callable alone in Studio with a synthetic batch.

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

<details><summary><b>mail_poll</b> : Scheduled entry point</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") mail_poll

Scheduled entry point. Takes the state-inbox lock (skips when another run is live), polls the IMAP inbox for mails with uid > lastUid (at most max_per_poll), hands the batch to mail_ingest (store, route, cursor), retries pending / error mails with mail_replay, purges mail documents older than the retention with mail_purge, then releases the lock and records lastError. First run (or UIDVALIDITY change) starts from the current UIDNEXT. Scheduled outside the project (administration console).

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
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;imap_password
</td>
<td>
Masked in logs, Studio and platform. The project file only holds the symbol reference; the secret itself lives in the engine symbol lib_maillistener.imap.password.secret (stored ciphered thanks to the .secret suffix).
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

<details><summary><b>mail_purge</b> : Retention: deletes mail documents (any status) stored more than retention_days ago (lib_maillistener</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") mail_purge

Retention: deletes mail documents (any status) stored more than retention_days ago (lib_maillistener.retention_days, default 30), 200 per run. state-inbox is never touched. Called by mail_poll under its lock; callable alone in Studio.

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

<details><summary><b>mail_replay</b> : Retries mails left pending (interrupted run) or in error (business sequence failure) through mail_route, until max_attempts routing attempts</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") mail_replay

Retries mails left pending (interrupted run) or in error (business sequence failure) through mail_route, until max_attempts routing attempts. store_error mails are not retried. Called by mail_poll under its lock; callable alone in Studio.

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
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;max_attempts
</td>
<td>

</td>
</tr>
</table>

</p></blockquote></details>

<details><summary><b>mail_route</b> : Routes a stored mail document based on rules, then writes status/route/routedAt/attempts/error back on the document</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") mail_route

Routes a stored mail document based on rules, then writes status/route/routedAt/attempts/error back on the document. Input: docid (mail-<uid>), business_sequence (project.sequence, default lib_MailListener.demo_business). V1 rules: rule_urgent (subject contains URGENT, any sender), rule_domain (DMARC-authenticated sender from convertigo.com, exact domain). A matched mail first gets its attachment files stored (mail_store_attachments), then business_sequence is called with docid and route. No match: unmatched (attachments not stored). Attachment or business failure: error, retried by mail_replay.

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
</table>

</p></blockquote></details>

<details><summary><b>mail_store_attachments</b> : Stores the attachment files of a routed mail as CouchDB attachments of mail-<uid> (name: <index>-<safe name>, original name kept in the attachments metadata)</summary><blockquote><p>


## ![](https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/sequences/images/genericsequence_color_16x16.png?raw=true "GenericSequence") mail_store_attachments

Stores the attachment files of a routed mail as CouchDB attachments of mail-<uid> (name: <index>-<safe name>, original name kept in the attachments metadata). The message is fetched again from IMAP by UID (UIDVALIDITY checked) and each part goes through a temp file. Parts above lib_maillistener.attachments.max_mb (default 500) are listed as skipped (too_large). Already stored parts are skipped, so a retry resumes. Fatal problems fail the run. Called by mail_route before the business sequence.

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
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;imap_password
</td>
<td>

</td>
</tr>
<tr>
<td>
<img src="https://github.com/convertigo/convertigo/blob/develop/engine/src/com/twinsoft/convertigo/beans/variables/images/variable_color_16x16.png?raw=true "  alt="RequestableVariable" >&nbsp;max_mb
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

Default home page
</p></blockquote></details>
</p></blockquote></details>
