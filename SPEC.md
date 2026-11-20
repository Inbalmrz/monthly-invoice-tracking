# Monthly Invoice Tracking – Make Automation Spec

## Purpose
Scan Gmail for emails from a list of sender addresses (stored in Google Sheets), detect whether each email has a PDF attachment, check whether it was already forwarded to a fixed address, and log the result in a separate tab of the same spreadsheet.

## Stack
- Make (scenario, scheduled every 15 minutes)
- Gmail (connection: owner's account)
- Google Sheets

## Google Sheets structure
**Tab `addresses`** – column B: sender email addresses (header row in row 1).

**Tab `log`** – columns:
| Col | Name | Value |
|---|---|---|
| A | Sender | From (email) of the message |
| B | Subject | Message subject |
| C | Received date | Message date |
| D | PDF exists | `yes` / `no` |
| E | Status | `sent` / `not sent` |
| F | Sent date | Date of the forwarded email (empty if not sent) |

## Scenario modules
| # | Module | Settings |
|---|---|---|
| 2 | Google Sheets – Search Rows | Tab `addresses`, table contains headers. Filter after: column B exists. |
| 3 | Gmail – Search emails | Simple filter, Folder: All Mail, Sender = column B from module 2, Date from/to empty, Limit 500. |
| 11 | Router | Route 1: filter `Has attachment = true` (module 3). Route 2: fallback (no attachment). |
| 25 | Gmail – List email attachments and media (route 1) | Message ID from module 3. Filter after: `MIME type` contains (case insensitive) `pdf`. |
| 26 | Gmail – Search emails (already forwarded?) | Filter type: Gmail filter. Query: `to:<FIXED_ADDRESS> subject:"<Subject from module 3>"`. |
| 27 | Tools – Numeric aggregator | Source module: 26, Count, Value `1`. |
| 31 | Router | Route 1: filter `result > 0` (already sent). Route 2: fallback (not sent). |
| 29 | Google Sheets – Add a Row (sent) | Tab `log`: PDF exists = `yes`, Status = `sent`, Sent date = date of the sent message. |
| 32 | Google Sheets – Add a Row (not sent) | Tab `log`: PDF exists = `yes`, Status = `not sent`, Sent date empty. |
| 13 | Google Sheets – Add a Row (route 2 of first router) | Tab `log`: PDF exists = `no`, Status empty. |

In all Add a Row modules: Sender, Subject and Received date are mapped from module 3.

## Flow
1. Read sender addresses from `addresses`.
2. For each address, search Gmail for its messages.
3. Router 1: messages with attachments go to route 1; others go to route 2 and are logged as `PDF exists = no`.
4. Route 1: list attachments and keep only PDFs.
5. Search the fixed address for a message with the same subject; aggregate the count.
6. Router 2: count > 0 → log as `sent`; otherwise log as `not sent`.

## Known limitations
- Messages with only non-PDF attachments are stopped at the PDF filter and are not logged.
- Messages are logged on every scheduled run; limit module 3 with `Date from` to avoid duplicate rows.
- Matching "already forwarded" relies on an identical subject line.

## Export
Scenario blueprint: in Make, open the scenario, then **⋮ → Export Blueprint**. Store the `.json` file next to this spec in the repository.
