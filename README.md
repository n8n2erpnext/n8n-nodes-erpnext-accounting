# n8n-nodes-erpnext-accounting

Community n8n node package for ERPNext/Frappe Accounting v15-v16.

This package is part of the `n8n2erpnext` ecosystem. It focuses on common ERPNext Accounting doctypes and keeps a generic Frappe escape hatch for custom doctypes and whitelisted methods.

## Who This Is For

This package is built for teams that run ERPNext Accounting and want a controlled way to connect finance data with n8n workflows.

Typical users:

- ERP administrators who maintain ERPNext/Frappe.
- Accounting, finance, or operations teams that need invoice, payment, journal, or ledger workflows.
- Integration teams that need repeatable n8n automations without writing custom Frappe client code for every workflow.

The node is intentionally conservative: it exposes standard Accounting document operations, supports Frappe API v1 and v2, and allows controlled fallback access to custom DocTypes and whitelisted Frappe methods.

## Architecture At A Glance

Read workflow from left to right:

```text
ERPNext / Frappe Accounting  <---- API token ---->  n8n ERPNext Accounting node  <---- webhook/API ---->  Client / App / Report
```

Common read pattern:

```text
Client
  -> n8n Webhook
  -> ERPNext Accounting node
  -> Frappe REST API
  -> ERPNext Accounting DocType
  -> filtered JSON response
```

Common accounting lifecycle pattern:

```text
n8n Webhook / Schedule / App Event
  -> validation / mapping / approval logic
  -> ERPNext Accounting node
  -> Invoice, Payment Entry, Journal Entry, or GL readback
  -> safe summary response or downstream system
```

Recommended production network pattern:

```text
Public Client
  -> HTTPS reverse proxy / VPN / allowlist
  -> n8n
  -> private network or internal VPS address
  -> ERPNext / Frappe site
```

## Supported Resources

- Sales Invoice
- Purchase Invoice
- Payment Entry
- Journal Entry
- Account
- Cost Center
- Fiscal Year
- GL Entry
- Custom DocType
- Frappe Method

## Node Identity

All `n8n2erpnext` module nodes use the same ERPNext-style logo shape. Each module changes only the main background color.

| Module | Color | Hex | Reason |
| --- | --- | --- | --- |
| Core | ERPNext blue | `#2490EF` | Foundation package, closest to the ERPNext brand color. |
| HRMS | People green | `#2E7D5F` | Human operations, employees, attendance, leave, payroll. |
| Accounting | Finance orange-red | `#D94A2B` | Ledger, journals, invoices, financial control. |
| Buying | Procurement amber | `#C47F00` | Purchase flow, suppliers, RFQs, purchase orders, spend. |
| Selling | Commerce teal | `#00A6A6` | Customer-facing pipeline, quotations, sales orders, revenue. |
| Stock | Frappe black | `#171717` | Warehouses, items, inventory movement; aligned with Frappe black. |

When building another module, copy the HRMS/Accounting SVG structure and change only the main background fill to that module color.

## Operations

For Accounting doctypes:

- Create
- Get
- Get Many
- Update
- Delete
- Submit
- Cancel

For Frappe methods:

- Run Method

## API Versions

The node supports both ERPNext/Frappe document API styles:

- `v1`: `/api/resource/:doctype`
- `v2`: `/api/v2/document/:doctype`

Use `v1` for broad compatibility. Use `v2` when your ERPNext/Frappe v16 environment is ready for the newer document API behavior.

Reference:

- [Frappe REST API](https://docs.frappe.io/framework/user/en/api/rest)

## Credentials

Create an API key and secret in ERPNext/Frappe, then configure:

- Site URL: `https://erp.example.com`
- Site Host Header, optional: `erp.example.com`
- API Key
- API Secret
- Ignore SSL Issues, optional

The node authenticates with:

```http
Authorization: token api_key:api_secret
```

### Internal URL With Public Host Header

When n8n and ERPNext run on the same VPS, you can point n8n at the internal ERPNext address and still send the public ERPNext host header:

- Site URL: `http://erpnext.internal:8001`
- Site Host Header: `erp.example.com`

This avoids public reverse-proxy authentication while still letting ERPNext receive the expected site host.

For production, create a dedicated ERPNext integration user instead of using a daily admin account. Give that user only the roles required for the workflows it runs.

Official Frappe references:

- [Frappe REST API authentication](https://docs.frappe.io/framework/user/en/api/rest)
- [Frappe token based authentication](https://docs.frappe.io/framework/v15/user/en/guides/integration/rest_api/token_based_authentication)
- [Generate Frappe API key and secret](https://docs.frappe.io/framework/v15/user/en/guides/integration/how_to_setup_token_based_auth)

## Examples

Get submitted sales invoices:

```json
{
  "resource": "salesInvoice",
  "operation": "getMany",
  "fields": "name,customer,posting_date,grand_total,status",
  "filtersJson": "[[\"docstatus\",\"=\",1]]",
  "returnAll": false,
  "limit": 20,
  "orderBy": "posting_date desc"
}
```

Get GL entries for an account:

```json
{
  "resource": "glEntry",
  "operation": "getMany",
  "fields": "name,posting_date,account,debit,credit,voucher_type,voucher_no",
  "filtersJson": "[[\"account\",\"=\",\"1110 - Cash - TDD\"]]",
  "returnAll": false,
  "limit": 50,
  "orderBy": "posting_date desc"
}
```

Run a whitelisted Frappe method:

```json
{
  "resource": "frappeMethod",
  "operation": "runMethod",
  "methodName": "frappe.client.get_value",
  "argumentsJson": {
    "doctype": "Sales Invoice",
    "filters": { "name": "ACC-SINV-2026-00001" },
    "fieldname": ["name", "customer", "grand_total"]
  }
}
```

## Webhook From n8n to ERPNext Accounting

Use this pattern when you want an HTTP GET endpoint in n8n that returns Accounting data from ERPNext.

```text
Client / Browser / BI Tool
  -> GET n8n webhook URL
  -> ERPNext Accounting node
  -> GET /api/resource or /api/v2/document
  -> JSON response
```

### 1. Configure the ERPNext Credential

In n8n, create or edit an `ERPNext API` credential:

- Site URL: `http://erpnext.internal:8001`
- Site Host Header: `erp.example.com`
- API Key: your ERPNext API key
- API Secret: your ERPNext API secret
- Ignore SSL Issues: `false`

For the current VPS/LXD production setup:

```text
Site URL: http://10.192.135.2:8001
Site Host Header: erp.thaiduy.digital
```

### 2. Create the Workflow

Create a workflow with these nodes:

```text
GET Webhook -> ERPNext Accounting
```

Webhook node:

- HTTP Method: `GET`
- Path: `erpnext-accounting-get-accounts`
- Respond: `When Last Node Finishes`
- Response Data: `All Entries`

ERPNext Accounting node:

- Credential: your `ERPNext API` credential
- Resource: `Account`
- Operation: `Get Many`
- API Version: `v1`
- Fields: `name,account_name,account_number,company,account_type,is_group`
- Filters JSON: `[]`
- Return All: `false`
- Limit: `20`
- Order By: `modified desc`

### 3. Activate and Test

Activate the workflow, then call:

```bash
curl -i https://n8n.example.com/webhook/erpnext-accounting-get-accounts
```

On the local VPS, you can test without going through the public proxy:

```bash
curl -i http://127.0.0.1:5678/webhook/erpnext-accounting-get-accounts
```

The working workflow artifact is included in this repository:

```text
n8n-webhook-erpnext-accounting-get-accounts.workflow.json
```

API v2 read workflow artifact:

```text
n8n-webhook-erpnext-accounting-v2-get-accounts.workflow.json
```

API v2 test endpoint:

```bash
curl -i http://127.0.0.1:5678/webhook/erpnext-accounting-v2-get-accounts
```

## Webhook From ERPNext v16 to n8n

Use this pattern when ERPNext should call n8n automatically after an Accounting document is created or updated. For example, ERPNext can call a n8n workflow whenever a `Sales Invoice`, `Purchase Invoice`, or `Payment Entry` changes.

```text
ERPNext Doc Event
  -> Frappe Webhook
  -> POST n8n webhook URL
  -> n8n workflow
  -> validation, notification, sync, approval, audit, or downstream automation
```

### 1. Create The n8n Webhook Receiver

Create a workflow in n8n with a Webhook trigger:

```text
Webhook -> your processing nodes
```

Webhook node:

- HTTP Method: `POST`
- Path: `erpnext-accounting-event`
- Authentication: `None` for a private/internal test, or `Header Auth` for production
- Respond: `Immediately` or `When Last Node Finishes`

The production webhook URL will look like:

```text
https://n8n.example.com/webhook/erpnext-accounting-event
```

On this VPS, if ERPNext and n8n are on the same host/network, you can also use an internal n8n URL from ERPNext.

### 2. Add The Webhook In ERPNext/Frappe v16

In ERPNext/Frappe Desk:

1. Open the global search bar.
2. Search for `Webhook`.
3. Open `Webhook` from the Integrations area.
4. Click `New`.

Configure the Webhook:

- Enabled: checked
- Webhook Doctype: `Sales Invoice`, `Purchase Invoice`, or another Accounting DocType
- Doc Event: `on_submit`, `on_cancel`, `after_insert`, or `on_update` depending on the workflow
- Request URL: your n8n production webhook URL
- Request Method: `POST`
- Request Structure: `JSON`
- Webhook JSON: use an allowlisted body like the example below

Example JSON body:

```json
{
  "event": "sales_invoice_submitted",
  "doctype": "{{ doc.doctype }}",
  "name": "{{ doc.name }}",
  "customer": "{{ doc.customer }}",
  "company": "{{ doc.company }}",
  "posting_date": "{{ doc.posting_date }}",
  "grand_total": "{{ doc.grand_total }}",
  "outstanding_amount": "{{ doc.outstanding_amount }}",
  "status": "{{ doc.status }}",
  "modified": "{{ doc.modified }}"
}
```

For production, add a shared secret header and validate it in n8n:

```text
X-ERPNext-Webhook-Secret: your-long-random-secret
```

If you use Frappe's Webhook Secret field, Frappe adds an `X-Frappe-Webhook-Signature` header generated from the payload and secret. You can verify this signature in n8n with a Code node if needed.

Official Frappe reference:

- [Frappe Webhooks](https://docs.frappe.io/framework/user/en/guides/integration/webhooks)

## Tested Workflow Artifacts

The repository includes workflow artifacts used during live ERPNext LXD testing.

Read-only artifacts:

```text
n8n-webhook-erpnext-accounting-get-accounts.workflow.json
n8n-webhook-erpnext-accounting-v2-get-accounts.workflow.json
n8n-webhook-erpnext-accounting-get-journal-entry-by-id.workflow.json
```

Write-test artifacts:

```text
n8n-webhook-erpnext-accounting-v2-create-journal-entry.workflow.json
n8n-webhook-erpnext-accounting-v2-update-journal-entry.workflow.json
n8n-webhook-erpnext-accounting-submit-cancel-journal-entry.workflow.json
n8n-webhook-erpnext-accounting-v2-create-sales-invoice-draft.workflow.json
n8n-webhook-erpnext-accounting-v2-business-lifecycle-test.workflow.json
```

Stress/security artifact:

```text
n8n-webhook-erpnext-accounting-v2-stress-security-test.workflow.json
```

Accounting write-test workflows create real demo/test documents in the ERPNext LXD test instance. They should be activated only during testing and deactivated after verification unless a trusted operator intentionally keeps them active.

## Journal Entry API v2 Nested Payload Test

This package has been tested with a draft Journal Entry payload that includes the nested `accounts` child table through the Frappe v2 document API.

Workflow artifact:

```text
n8n-webhook-erpnext-accounting-v2-create-journal-entry.workflow.json
```

Test shape:

```text
GET Webhook -> ERPNext Accounting
```

ERPNext Accounting node:

- Resource: `Journal Entry`
- Operation: `Create`
- API Version: `v2`
- Data JSON: balanced Journal Entry object with nested `accounts`

Payload pattern:

```json
{
  "voucher_type": "Journal Entry",
  "company": "Thái Duy Digital",
  "posting_date": "2026-05-14",
  "user_remark": "n8n Accounting node API v2 nested accounts draft test. Do not submit.",
  "accounts": [
    {
      "account": "1110 - Cash - TDD",
      "debit_in_account_currency": 1000,
      "credit_in_account_currency": 0
    },
    {
      "account": "3300 - Opening Balance Equity - TDD",
      "debit_in_account_currency": 0,
      "credit_in_account_currency": 1000
    }
  ]
}
```

Verified result:

- ERPNext created draft Journal Entry `ACC-JV-2026-00001`
- `docstatus`: `0`
- `total_debit`: `1000`
- `total_credit`: `1000`
- `difference`: `0`
- GL Entries created: `0`

The test workflow was deactivated after verification to prevent accidental repeated draft Journal Entry creation.

## Get Journal Entry By ID

Use these read-only webhooks to fetch a Journal Entry by document name.

Workflow artifact:

```text
n8n-webhook-erpnext-accounting-get-journal-entry-by-id.workflow.json
```

API v1 endpoint:

```bash
curl -i 'http://127.0.0.1:5678/webhook/erpnext-accounting-get-journal-entry?name=ACC-JV-2026-00001'
```

API v2 endpoint:

```bash
curl -i 'http://127.0.0.1:5678/webhook/erpnext-accounting-v2-get-journal-entry?name=ACC-JV-2026-00001'
```

Both endpoints return the Journal Entry header and nested `accounts` rows.

## Update Journal Entry API v2 Test

This package has also been tested with an API v2 update against the same draft Journal Entry.

Workflow artifact:

```text
n8n-webhook-erpnext-accounting-v2-update-journal-entry.workflow.json
```

Test shape:

```text
POST Webhook -> ERPNext Accounting
```

ERPNext Accounting node:

- Resource: `Journal Entry`
- Operation: `Update`
- API Version: `v2`
- Document Name: `ACC-JV-2026-00001`
- Data JSON: update `user_remark`

Payload pattern:

```json
{
  "user_remark": "n8n Accounting node API v2 update test passed. Draft remains unsubmitted; no GL Entry created."
}
```

Verified result:

- ERPNext updated draft Journal Entry `ACC-JV-2026-00001`
- `docstatus`: `0`
- `total_debit`: `1000`
- `total_credit`: `1000`
- `difference`: `0`
- GL Entries created: `0`

The test workflow was deactivated after verification to prevent accidental repeated writes.

## Submit And Cancel Journal Entry Test

Submit and Cancel have been tested with the same Journal Entry to verify the full accounting document lifecycle.

Workflow artifact:

```text
n8n-webhook-erpnext-accounting-submit-cancel-journal-entry.workflow.json
```

Test shape:

```text
POST Submit Webhook -> ERPNext Accounting Submit
POST Cancel Webhook -> ERPNext Accounting Cancel
```

ERPNext Accounting node:

- Resource: `Journal Entry`
- API Version: `v2`
- Document Name: `ACC-JV-2026-00001`
- Operations: `Submit`, then `Cancel`

Verified submit result:

- `docstatus`: `1`
- GL Entries created: `2`
- GL debit total: `1000`
- GL credit total: `1000`
- Cancelled GL rows after submit: `0`

Verified cancel result:

- `docstatus`: `2`
- GL Entries after cancel: `4`
- GL debit total: `2000`
- GL credit total: `2000`
- Cancelled GL rows after cancel: `4`

Implementation note:

- `frappe.client.submit` expects `{ doc }`.
- `frappe.client.cancel` expects `{ doctype, name }`.
- The Accounting helper handles these two payload shapes separately.

The Submit/Cancel workflow was deactivated after verification to prevent accidental repeated ledger writes.

## Business Lifecycle And Stress/Security Tests

Accounting write tests are intentionally shipped as inactive workflow artifacts. Import and review them before activation because they create submitted accounting documents and ledger entries.

Lifecycle workflow artifact:

```text
n8n-webhook-erpnext-accounting-v2-business-lifecycle-test.workflow.json
```

Lifecycle test shape:

```text
POST Webhook
-> Build bounded run context
-> Create Customer, Supplier, sales Item, purchase Item
-> Create and submit Sales Invoice
-> Create and submit Receive Payment Entry
-> Create and submit Purchase Invoice
-> Create and submit Pay Payment Entry
-> Create and submit simulated closing Journal Entry
-> Read closing GL Entries
-> Return safe allowlisted summary
```

The test uses unique `N8N-ACC-LIFECYCLE-*` master data per run and keeps the webhook response restricted to document names, totals, counts, and credential leak-scan status. It does not return raw ERPNext documents or credentials.

Default account assumptions:

```text
Cash: 1110 - Cash - TDD
Receivable: 1310 - Debtors - TDD
Payable: 2110 - Creditors - TDD
Sales: 4110 - Sales - TDD
Expense: 5111 - Cost of Goods Sold - TDD
Closing equity: 3300 - Opening Balance Equity - TDD
```

Adjust these account names before running the workflow on another ERPNext company/chart of accounts.

Stress/security workflow artifact:

```text
n8n-webhook-erpnext-accounting-v2-stress-security-test.workflow.json
```

Stress/security shape:

```text
POST Webhook
-> Build 1..25 stress items
-> Read Account getMany through API v2
-> Return safe counts and leak-scan status
```

Example body:

```json
{
  "iterations": 10
}
```

The workflow caps `iterations` at `25`, performs read-only account queries, and scans the response payload for credential-like strings such as API key, API secret, Authorization, token, and password.

### Verified Business Lifecycle Run

The lifecycle and stress/security workflows were imported into the live n8n container, activated temporarily, tested against the ERPNext LXD instance, and deactivated again.

Test timestamp:

```text
2026-05-14 09:58-09:59 UTC
```

Lifecycle result:

```text
Run ID: N8N-ACC-LIFECYCLE-1778752708065
Status: passed
Sales Invoice: ACC-SINV-2026-00002
Receive Payment Entry: ACC-PAY-2026-00001
Purchase Invoice: ACC-PINV-2026-00001
Pay Payment Entry: ACC-PAY-2026-00002
Closing Journal Entry: ACC-JV-2026-00002
Closing GL rows: 3
Closing debit total: 1500
Closing credit total: 1500
Security findings: []
```

ERPNext DB verification:

```text
ACC-SINV-2026-00002 | docstatus 1 | Paid | grand_total 1500 | outstanding 0
ACC-PINV-2026-00001 | docstatus 1 | Paid | grand_total 600 | outstanding 0
ACC-PAY-2026-00001 | docstatus 1 | Receive | paid 1500 | received 1500
ACC-PAY-2026-00002 | docstatus 1 | Pay | paid 600 | received 600
ACC-JV-2026-00002 | docstatus 1 | debit 1500 | credit 1500 | difference 0
```

GL verification:

```text
Sales Invoice ACC-SINV-2026-00002 | 2 rows | debit 1500 | credit 1500 | cancelled 0
Payment Entry ACC-PAY-2026-00001 | 2 rows | debit 1500 | credit 1500 | cancelled 0
Purchase Invoice ACC-PINV-2026-00001 | 2 rows | debit 600 | credit 600 | cancelled 0
Payment Entry ACC-PAY-2026-00002 | 2 rows | debit 600 | credit 600 | cancelled 0
Journal Entry ACC-JV-2026-00002 | 3 rows | debit 1500 | credit 1500 | cancelled 0
```

Stress/security verification:

```text
iterations=3 -> requestedIterations 3, accountRowsReturned 15, leakedPatterns []
iterations=30 -> requestedIterations 25, accountRowsReturned 125, leakedPatterns []
```

After testing, both workflow endpoints returned `404 Active version not found`, confirming the write-test webhook and stress-test webhook were no longer active.

## Reverse Proxy Notes

If `https://erp.example.com` is protected by NetBird or another reverse-proxy auth layer, n8n server-side requests may be blocked before they reach ERPNext. In that case:

- Use the internal ERPNext URL in `Site URL`.
- Set `Site Host Header` to the public ERPNext host.
- Keep the API key and secret from the ERPNext user that has permission to read/write the target DocType.

## Security Baseline

Accounting data includes invoices, payments, ledger rows, tax accounts, payable/receivable balances, and business financial records. Treat every workflow as sensitive by default.

Recommended baseline:

- Use a dedicated ERPNext API user for n8n integrations.
- Avoid using a full Administrator API key in production.
- Scope ERPNext roles to the exact DocTypes and actions needed by the workflow.
- Prefer `Get Many` with explicit `Fields` over `Get` when exposing webhook responses, because `Get` can return the full document including comments, owners, child tables, and accounting metadata.
- Keep n8n webhook URLs private unless they are meant to be public.
- Add authentication to public n8n webhooks, such as header auth, reverse proxy auth, VPN, IP allowlisting, or a shared secret.
- Do not log API keys, API secrets, Authorization headers, raw upstream error bodies, or full accounting documents into external systems unless there is a clear retention policy.
- Use HTTPS for public traffic.
- If n8n and ERPNext are on the same VPS or private network, prefer the internal ERPNext URL plus `Site Host Header`.
- Rotate API keys after testing, after staff changes, and after any suspected exposure.
- Review n8n execution data retention. Disable or reduce saved execution data for workflows that process invoices, payment allocations, journals, GL entries, or tax data.
- Deactivate temporary write-test workflows after verification.

## Security Notice

This package pins transitive `form-data` resolution with an npm `overrides` entry so `npm audit --omit=dev` reports no known vulnerabilities at release time. Keep this override in place until upstream dependencies resolve to a safe version without assistance.

In this package's tested deployment model, security risk is also reduced by:

- Internal network access between n8n and ERPNext.
- Reverse proxy or VPN controls for public endpoints.
- Dedicated ERPNext API credentials with scoped roles.
- Explicit field selection for public webhook responses.
- Safe summary nodes for write-heavy lifecycle tests.
- Avoiding public exposure of generic Custom DocType and Frappe Method workflows.

Do not treat this mitigation as a permanent substitute for dependency maintenance. Re-run `npm audit --omit=dev` before publishing a new package version and upgrade compatible n8n dependencies when the upstream dependency chain allows it without breaking n8n node compatibility.

## Deployment Checklist For SME And Mid-Market Teams

Before going live:

- Confirm ERPNext/Frappe version and choose API `v1` or `v2`.
- Create a dedicated ERPNext integration user.
- Assign only the required Accounting roles and permissions.
- Configure n8n credentials with the ERPNext internal URL when available.
- Set `Site Host Header` if ERPNext is served by a named Frappe site.
- Build and install the packed node package into the n8n custom nodes environment.
- Test `Get Many` for each required resource with limited fields.
- Test write operations in a staging or dedicated ERPNext test site before production.
- Review n8n execution data retention and error logging.
- Protect public webhooks with authentication or network controls.
- Keep workflow JSON exports out of public repositories if they contain real URLs, headers, filters, or business logic.
- Validate chart-of-accounts names before importing workflow artifacts across companies.

Suggested production approach:

- Start with read-only accounting reporting workflows.
- Add invoice/payment/journal write workflows only after role permissions, approval paths, and audit logs are reviewed.
- Keep Custom DocType and Frappe Method workflows limited to trusted internal operators.
- Document each production workflow owner, purpose, data fields, ledger impact, and rollback path.

## Troubleshooting

Common checks:

- `401` or `403`: verify API key, API secret, user roles, and DocType permissions in ERPNext.
- TLS `EPROTO` or `tlsv1 alert internal error`: use the internal ERPNext HTTP URL from n8n when the public domain is protected by a reverse proxy or VPN layer.
- Empty `[]` response: the node is working, but filters may not match any records.
- Frappe site not found or wrong site: set `Site Host Header` to the public ERPNext site name.
- n8n webhook does not run: activate the workflow and use the production `/webhook/` URL, not `/webhook-test/`.
- Unexpected sensitive fields in output: switch from `Get` to `Get Many` and set an explicit `Fields` list.
- Invoice or Payment Entry validation fails: verify party, company, receivable/payable account, cash/bank account, income/expense account, item flags, and fiscal period.
- Journal Entry submit fails: verify total debit equals total credit and all accounts are leaf accounts for the same company.
- GL Entry does not appear: confirm the source document is submitted; drafts should not create GL rows.

## Development

```bash
npm install
npm run build
npm run lint
```

For local n8n testing, link this package into your n8n custom nodes directory or install it from a packed tarball.

Useful n8n references:

- [n8n community nodes installation](https://docs.n8n.io/integrations/community-nodes/installation/)
- [Install community nodes from the n8n GUI](https://docs.n8n.io/integrations/community-nodes/installation/gui-install/)
- [Manual community node installation](https://docs.n8n.io/integrations/community-nodes/installation/manual-install/)
- [Using community nodes](https://docs.n8n.io/integrations/community-nodes/usage/)
- [Creating n8n nodes](https://docs.n8n.io/integrations/creating-nodes/)
- [Using the n8n-node tool](https://docs.n8n.io/integrations/creating-nodes/build/n8n-node/)
- [n8n node linter](https://docs.n8n.io/integrations/creating-nodes/test/node-linter/)
- [Submit community nodes](https://docs.n8n.io/integrations/creating-nodes/deploy/submit-community-nodes/)

## Scope And Roadmap

This package is intentionally Accounting-focused. Other ERPNext modules should live in separate packages so each module can evolve independently:

- `n8n-nodes-erpnext-hrms`
- `n8n-nodes-erpnext-selling`
- `n8n-nodes-erpnext-buying`
- `n8n-nodes-erpnext-stock`

Recommended next hardening tasks before wider public adoption:

- Add automated unit tests for request construction and API v1/v2 endpoint behavior.
- Add credential redaction checks around error messages.
- Add sample workflows that use limited fields by default.
- Add a production security checklist to release notes for every package version.
- Add more focused workflows for tax, cost center, and fiscal-year close scenarios.

## Official References

Frappe / ERPNext:

- [ERPNext introduction](https://docs.frappe.io/erpnext)
- [ERPNext Accounting](https://docs.frappe.io/erpnext/user/manual/en/accounting)
- [Sales Invoice](https://docs.frappe.io/erpnext/user/manual/en/sales-invoice)
- [Purchase Invoice](https://docs.frappe.io/erpnext/user/manual/en/purchase-invoice)
- [Payment Entry](https://docs.frappe.io/erpnext/user/manual/en/payment-entry)
- [Journal Entry](https://docs.frappe.io/erpnext/user/manual/en/journal-entry)
- [Frappe REST API](https://docs.frappe.io/framework/user/en/api/rest)
- [Frappe token based authentication](https://docs.frappe.io/framework/v15/user/en/guides/integration/rest_api/token_based_authentication)
- [Generate Frappe API key and secret](https://docs.frappe.io/framework/v15/user/en/guides/integration/how_to_setup_token_based_auth)
- [Frappe Webhooks](https://docs.frappe.io/framework/user/en/guides/integration/webhooks)

n8n:

- [n8n integrations and nodes overview](https://docs.n8n.io/integrations/)
- [n8n community nodes installation](https://docs.n8n.io/integrations/community-nodes/installation/)
- [Manual community node installation](https://docs.n8n.io/integrations/community-nodes/installation/manual-install/)
- [Using community nodes](https://docs.n8n.io/integrations/community-nodes/usage/)
- [Creating n8n nodes](https://docs.n8n.io/integrations/creating-nodes/)
- [Submit community nodes](https://docs.n8n.io/integrations/creating-nodes/deploy/submit-community-nodes/)

## License

MIT

## Acknowledgement

Prepared and reviewed with care by Codex for the `n8n2erpnext` ERPNext Accounting integration work.

Signed: Codex, May 14, 2026.
