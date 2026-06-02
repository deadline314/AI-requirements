---
name: email-categorize-rules
description: Rules for assigning exactly one business category to a Gmail message. Use as a sub-skill of email-classify.
---

# Email Categorize Rules

Assign **exactly one** category from this set, or return `null` if no rule fires confidently:

```
contract | sales | support | finance | hr | newsletter | internal
```

Apply rules in the order below. The **first** rule that matches wins — order matters because higher-priority rules represent more downstream / higher-stakes states.

## Rule 0 — Strong override: `newsletter`

If the message has any of these, classify as `newsletter` and stop:

- `List-Unsubscribe` header is present
- `List-Id` header is present
- `Precedence: bulk` header
- Sender address contains `noreply`, `no-reply`, `newsletter`, `marketing`, `mailer`, `notifications@`

## Rule 1 — `contract`

Subject or body contains (case-insensitive):

- English: `contract`, `agreement`, `MSA`, `NDA`, `SOW`, `statement of work`, `terms and conditions`, `signed`, `DocuSign`, `Adobe Sign`, `countersign`, `addendum`, `amendment`
- Chinese: `合約`, `合同`, `協議`, `保密協議`, `簽署`, `用印`, `用章`

PDF attachment named `*contract*`, `*agreement*`, `*MSA*`, `*NDA*` also counts.

## Rule 2 — `finance`

- Subject or body contains: `invoice`, `receipt`, `payment`, `billing`, `wire transfer`, `remittance`, `purchase order`, `PO #`, `quote #`, `tax`, `VAT`, `expense report`
- Chinese: `發票`, `帳單`, `付款`, `匯款`, `請款`, `對帳`, `報帳`, `稅`
- Sender domain is a known accounting tool: `quickbooks`, `xero`, `stripe`, `paypal`, `wise`, `bill.com`

## Rule 3 — `hr`

- Subject or body contains: `offer letter`, `onboarding`, `payroll`, `benefits`, `PTO`, `leave request`, `performance review`, `1:1`, `headcount`, `recruiting`, `interview`, `resume`, `CV`, `job application`
- Chinese: `錄取`, `到職`, `薪資`, `薪水`, `福利`, `請假`, `年假`, `考核`, `面試`, `履歷`
- Sender contains `hr@`, `people@`, `talent@`, `recruiting@`, or matches an HRIS tool (`workday`, `bamboohr`, `lever`, `greenhouse`, `ashby`)

## Rule 4 — `support`

- Subject contains a ticket/case marker: `[ticket #`, `case #`, `incident`, `RMA`, `bug report`
- Body language: `having an issue`, `not working`, `error`, `broken`, `please help`, `escalation`
- Chinese: `故障`, `無法使用`, `客訴`, `案件編號`
- Sender from a helpdesk tool: `zendesk`, `freshdesk`, `intercom`, `helpscout`, `jira service`

## Rule 5 — `sales`

- Subject or body contains: `proposal`, `pricing`, `quote`, `demo`, `lead`, `prospect`, `pipeline`, `discovery call`, `RFP`, `RFQ`, `sales`, `interested in your product`
- Chinese: `報價`, `提案`, `銷售`, `業務`, `合作`
- Sender from a CRM: `salesforce`, `hubspot`, `pipedrive`, `apollo`, `outreach`

## Rule 6 — `internal`

Only fires when **all** of these are true:

- `EMAIL_INTERNAL_DOMAIN` is configured
- Sender's domain equals `EMAIL_INTERNAL_DOMAIN`
- None of rules 0–5 fired

This means the message is from a colleague but isn't already a contract / finance / HR / etc. discussion. Catch-all team chatter, internal announcements, code review pings.

## Default — return `null`

If no rule matches, return `null`. The message stays unlabeled. Do **not** force a category — better to leave 30% unclassified than to mislabel.

## Tie-breaking

If two rules in the same priority tier could match, the rule listed first wins. Within Rule 1 vs Rule 5 (contract vs sales), contract always wins because it represents a later deal stage.

## Output

Return one of: `"contract"`, `"sales"`, `"support"`, `"finance"`, `"hr"`, `"newsletter"`, `"internal"`, or `null`.

Also return a 1-line `reason` string for logging (e.g. `"Rule 1: subject contains 'NDA'"`).
