# n8n-nodes-erpnext-accounting

Community n8n node package for ERPNext/Frappe Accounting v15-v16.

This package is part of the `n8n2erpnext` ecosystem. It focuses on common ERPNext Accounting doctypes and keeps a generic Frappe escape hatch for custom doctypes and whitelisted methods.

## Who This Is For

This package is built for teams that run ERPNext Accounting and want a controlled way to connect finance data with n8n workflows.

Typical users:

- ERP administrators who maintain ERPNext/Frappe.
- Accounting, finance, or operations teams that need invoice, payment, journal, or ledger workflows.
- Integration teams that need repeatable n8n automations without writing custom Frappe client code for every workflow.

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
  "filtersJson": "[[\"account\",\"=\",\"Cash - TDD\"]]",
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

## Local Development

Install dependencies:

```bash
npm install
```

Build:

```bash
npm run build
```

Lint:

```bash
npm run lint
```
