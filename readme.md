


# lib_MailListener

Mail listener library for Convertigo: it polls an IMAP mailbox, stores every new mail in the FullSync database `maillistener`, routes it to business sequences with rules, stores the attachment files of routed mails, keeps a full trace (status, errors, retries) and purges old mails.

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

Run options (variables of `mail_poll`, can be set on the Scheduler job): `max_per_poll` (default 50), `lock_ttl_minutes` (default 60). `mail_replay`: `max_attempts` (default 3).

## Routing

`mail_route` evaluates the rules of its `read_mail_fields` step in order; the first match gives the route:

| Route | Rule |
|---|---|
| `rule_urgent` | The subject contains `URGENT` (case-insensitive), whatever the sender. |
| `rule_domain` | The sender is verified and its address domain is exactly `convertigo.com`. |

Sender verification is done once, by `mail_ingest`: the mail must have a single From address, the topmost `Authentication-Results` header must come from `lib_maillistener.imap.authserv_id`, and it must report `dmarc=pass` for the From domain. The result is stored in `fromAddress` and `senderVerified`; the display name is never used. A content rule such as `rule_urgent` can be triggered by anyone: do not attach sensitive actions to it without checking `senderVerified`.

Outcome written on the mail document (`status`, `route`, `routedAt`, `attempts`, `error`):

- `routed`: a rule matched, the attachment files are stored and the business sequence succeeded;
- `unmatched`: no rule matched. The mail is kept for traceability (attachment files not stored) until the purge;
- `error`: the attachment storage or the business sequence failed. `mail_replay` retries it up to `max_attempts`, then it stays in `error` for a manual look;
- `store_error`: the database rejected the full mail; a minimal trace is kept so the cursor can move on. It is not retried.

## How the business sequence is launched

For a matched route, `mail_route`:

1. stores the attachment files (`mail_store_attachments`): the message is fetched again from IMAP by UID and each file becomes a CouchDB attachment of `mail-<uid>`, named `<index>-<safe name>` (`attachments[i].storedName`; the original name stays in `attachments[i].name`). If this fails, the business sequence is not called and the mail goes to `error`;
2. calls the business sequence with a Call Sequence step (`call_business_urgent`, `call_business_domain`), synchronously, inside the `mail_poll` run, with two variables: `docid` (`mail-<uid>`) and `route`;
3. writes the status. If the business sequence throws an exception or returns an error structure (`XMLErrorStep`), the mail goes to `error` with the message and the business sequence is called again by `mail_replay`. A business sequence must therefore be idempotent, and quick (it runs inside the polling run and its lock).

The business sequence reads the mail with `lib_MailListener.maillistener.get_document` (`_use_docid` = `docid`): `from`, `fromAddress`, `senderVerified`, `subject`, `body`, `bodyTruncated`, `receivedAt`, `attachments`. Attachment files are read with a GetDocumentAttachment transaction on the `maillistener` connector (document `docid`, attachment name `storedName`).

`demo_business` is a placeholder. To plug your own logic:

- change an existing route: point its Call Sequence step to your sequence (a sequence of another loaded project can be targeted; add a project reference to it);
- add a route: add a condition in `read_mail_fields` that sets `decided = "rule_x"`, then copy the `is_rule_domain` step (jIf `route_var == "rule_x" && String(attachments_error || "") == ""`, its Call Sequence step and its two error sources) and point it to your sequence.

## Data

Database `maillistener` (FullSync connector): one `mail-<uid>` document per mail (sender, subject, body up to 200 000 characters, attachment metadata and files, routing status) and the `state-inbox` document (cursor, UIDVALIDITY, lock, last run, last error). All sequences are Private: only `mail_poll` is meant to be scheduled, the others are called by it and can be run alone from the Studio.



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



