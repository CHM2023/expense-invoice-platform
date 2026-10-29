# Architecture

This document describes the Expense & Invoice Platform at the level of components, entities and rules. It contains no code, no schema and no prompt text.

## Components and responsibilities

**Mobile app (React Native on Expo).** Login, company picker for people who belong to more than one company, a category grid drawn from the server, receipt capture (camera or gallery), the scan-or-type choice, a form that the server describes field by field, the payment-method step, the employee's own receipts and bills with balances and statuses, editing or withdrawing a receipt while it is still pending or rejected, and a badge poll for rejected items. It holds a bearer token in secure storage and nothing else.

**JSON API (Laravel).** Login and logout, receipt types for the signed-in user and company, photo upload with optional extraction, receipt and bill create / read / update / delete within the owner's rights, the private photo stream, and the counts behind the badges. Every request names a company and is refused unless the caller is an active member of it. Validation rules for each receipt type live in one configuration that the portal forms also use, so the two channels cannot drift.

**Web portal (Laravel, Blade, Livewire, Alpine.js).** Dashboard, the receipts list with filters and a quick-look popup, receipt pages with approve / reject / reopen, monthly bills with matching, car expenses and fuel consumption, transactions (money without a receipt), the statement with CSV export, and the setup area: companies, users with roles and access, cars, categories, suppliers, page access, audit log. Every destructive action confirms in place; no browser dialogs.

**Extraction service.** Knows how to talk to the vision model and nothing about receipts. The caller passes the image, its real MIME type, the instructions and the JSON schema for the chosen receipt type; the service enforces the time budget, retries only transient failures, rejects truncated or non-JSON answers, logs token usage and the answering model version, and returns the fields.

**Reading mapper.** Turns what the model read (names and numbers as printed) into what the form should show: supplier text to a supplier record, plate text to a car, both by normalised comparison with no fuzzy guessing; unknown names open an "Other" box prefilled with the printed text.

**Access service.** Answers every "may this person" question for categories and pages. Returns "no restriction" for super admins and an explicit list for admins, where an empty list means nothing has been assigned; the two values are deliberately distinct.

**State machines.** One transition table for receipts and one for bills: for each move, the allowed start states, the end state, the effect on the approved balance and the approval right it needs. Controllers re-check the start state inside the UPDATE so two people deciding at the same moment cannot both move money.

**Money helper.** The only place amounts are converted between major and minor units; reads the decimal-place count per currency from configuration and rounds once.

**Duplicate detector.** Computes the two signatures across every receipt type with two grouped queries per page and marks the rows that repeat.

**Reference numbers.** A locked daily counter per company and kind, taken inside the caller's transaction so a rolled-back save burns no number; numbering can never block a save.

**Audit writer.** Called by hand from every controller action that changes state; stores only the fields that changed.

**Scheduler.** Two daily commands: remove abandoned uploads older than a day, and remove the picture files of receipts deleted more than a month ago while keeping their records.

## Data model (entities)

- **Company**: a tenant with a legal identity, a country and a default currency.
- **User**: a person; may belong to several companies.
- **Membership**: user ↔ company with exactly one role in that company (global developer, company super admin, admin, phone-only user). Removed memberships are flagged, not deleted, so they can be revived.
- **Category**: what the money was for (a property, a project, travel, car expenses). System categories back the car receipt types and cannot be renamed or removed.
- **Category access**: per user and category, whether they may review (first stage), approve (second stage) and/or file from the phone. An assignment on the car-expenses parent cascades to car wash, fuel and service, and to handling monthly bills.
- **Page access**: per user, company and page, read-only or read-write.
- **Employee account**: one cash float and one company card per employee per company and currency, each with a stored balance and an approved balance, and a limit on the card.
- **Receipt**: one table per receipt type (general expenses, car wash, fuel, car service) sharing the same core in the same order: owner, account, company, payment method, category, supplier or free-text supplier, amounts in minor units with tax, currency, receipt date, description, status, the two approval stamps, rejection reason, reference number, links to its pictures and to the bill that paid it. Each type adds its own fields (car and slip number; odometer readings and litres; work done). Derived figures such as distance, price per litre and consumption are computed on read, never stored.
- **Monthly bill**: a supplier's invoice for a period, paid by one employee's card or cash, with its own two-stage status, the receipts tied to it, and a note explaining any difference between its total and theirs.
- **Attachment**: a photograph on the private disk with its real MIME type, size and status (pending, kept, purged), plus the raw model output and the model version when it was read. A receipt points to up to three: the paper, the card slip and the proof behind a correction.
- **Account movement**: a deposit to or withdrawal from a cash float, typed by an accountant, with its own reference series.
- **Car**: brand, model, year, plate, the unit its odometer reads in, and a rolling average consumption written only by the fuel tracker.
- **Supplier**: the company's list, matched against the name printed on receipts.
- **Audit entry**: who, which company, action and channel, table and record, old and new values, UTC time. Append-only.
- **Reference counter**: per company, kind and day.

## The receipt state machine (prose)

A receipt is saved as **pending**. If it was paid in cash or with the company card its total is taken off the employee's stored balance at once, because the money has left; a pay-later slip moves nothing.

A first reviewer, assigned to that category, may **approve** it (it becomes **reviewed**), **correct** it while it is on their desk, or **reject** it with a reason of at least a few words. An editing reviewer's change moves the balance by the difference and is audited. Rejecting takes the receipt off the books, so the employee's balance gets the money back; a corrected rejected receipt returns to pending and takes it again. The reviewer can also take back their own approval without accusing anybody (reviewed back to pending, nothing said to the employee).

A second approver, assigned at the second stage, **approves** a reviewed receipt (it becomes **approved**) and only then does the approved balance move. Editing a reviewed receipt sends it back to pending and clears the first stamp. Approved and rejected rows are not edited; an approved receipt can only be **reopened**, which gives the approved balance back, clears both stamps and starts it again from pending. A rejected receipt is re-approved by going through stage one again, never straight to approved.

Deleting is a soft delete that reverses the balance in the same transaction; a slip already on a bill cannot be deleted or have its approval withdrawn until the bill is unmatched.

The phone shows its owner "waiting" for pending and reviewed together, since the owner can do nothing differently between the two.

## Monthly bills and matching (prose)

Some suppliers let drivers sign a slip and invoice the company monthly. Such receipts are filed as **pay later** and move no money. When the invoice arrives, the employee who pays it files it as a **monthly bill**, with the invoice photo and, if paid by card, the card slip. Entering the bill charges that employee's card (or cash) balance, because the payment has happened.

An accountant with rights on fuel opens the bill, picks the supplier and the period, and sees that supplier's pay-later receipts in the window with their review state. Selecting receipts shows the running difference between their sum and the invoice total; **Match and Pay** enables only at zero, or with an explanatory note when a genuine difference remains. Matching is the bill's first approval: it ties the receipts to the bill and marks them paid.

Order of work: receipts pass their first review before they can be matched; the bill's second approval requires every tied receipt to be fully approved; an approved bill locks its receipts; an approval cannot be withdrawn on a receipt that is on a bill (unmatch first). Rejecting a bill does not untie its receipts, because the money still left the card; **Unmatch** is a separate, named click. Reopening an approved bill returns it to matched, not pending, because the receipts are still tied.

Duplicate detection runs on top of all this: the same supplier plus the same printed slip number is a strong match, the same employee, supplier, date and amount a probable one, checked across every receipt type, flagged for a person and never blocked.

## Jobs, scheduling and the absence of a queue

Extraction is **synchronous by design**. The phone uploads the photo, the server stores it and creates the attachment record first, then calls the model within a fixed budget that is shorter than the server's execution limit, which is shorter than the phone's timeout. Within the budget the service retries only transient failures (rate limit, provider busy) and honours the provider's suggested wait only when another attempt still fits. Whatever happens, the response carries the attachment id, so the worst case is a form the employee fills in by hand with the photo already attached. A queue was considered and set aside: the person is standing at the counter with the phone in hand, and a job that answers minutes later would help nobody; the budget and the always-available manual path cover the failure modes a queue would have addressed.

Scheduled work runs through the framework scheduler, triggered by cron every five minutes: a nightly purge of uploads that were scanned but never saved, and a second nightly purge of the picture files of receipts deleted more than thirty days earlier, both with overlap protection. Badge counts are polled by the portal every minute and by the phone every thirty seconds while in the foreground; there is no push notification layer.

## Storage of images

Photographs are written to a private disk that has no public URL, under the company's folder with a random file name, so nothing about the employee or the supplier leaks into a path. The real MIME type is read from the bytes at upload and stored with the record. Images are served only by controller routes that check the company and the caller's rights and stream the file inline with its stored type; the phone fetches them with its bearer token. Replacing a bill photo keeps the old file. Abandoned uploads are hard-deleted after a day; pictures of deleted receipts are deleted after a month while the attachment record stays, marked purged, so the audit trail can still say a photograph existed and when it went.

## Deployment topology

A single Linux web host runs Apache with PHP, serving both the portal and the API over HTTPS with an automatically renewed certificate. MySQL runs on the host's managed database service. Cron invokes the scheduler every five minutes. Outbound SMTP is used for password-reset mail only. The model is reached over HTTPS with the key in a request header. The phone talks to the same host over mobile data; nothing else is exposed.

The deployment unit is a built artefact: front-end assets compiled, production dependencies installed, archived and copied to the host, where the application is put into maintenance mode, the archive unpacked, migrations run, caches rebuilt and the site brought back. Uploaded photographs and the environment file are excluded from every deploy after the first, because the server's copies are the real ones. Because the host's document root is the application root, a hardened rewrite configuration routes every request into the public folder and refuses direct access to the environment file, vendor code and storage.
