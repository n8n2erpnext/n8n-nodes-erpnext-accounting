# Session Log

## 2026-05-14

### Context

- Project: `n8n-nodes-erpnext-accounting`
- Starting point: package only had a planning README.
- HRMS package was used as the implementation reference because it already has working credentials, generic Frappe REST helpers, packaging, and production install lessons.

### Initial Accounting Scaffold

- Added n8n community node package structure:
  - `package.json`
  - `package-lock.json`
  - `tsconfig.json`
  - `eslint.config.js`
  - `gulpfile.js`
  - `.prettierrc`
  - `.gitignore`
  - `LICENSE`
  - `index.ts`
  - `credentials/ErpNextApi.credentials.ts`
  - `credentials/erpnext.svg`
  - `nodes/ErpNextAccounting/ErpNextAccounting.node.ts`
  - `nodes/ErpNextAccounting/GenericFunctions.ts`
  - `nodes/ErpNextAccounting/erpnext-accounting.svg`

### Supported Resources

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

### Operations

- Document operations:
  - Create
  - Get
  - Get Many
  - Update
  - Delete
  - Submit
  - Cancel
- Frappe Method operation:
  - Run Method

### Compatibility Notes

- Supports API v1:

```text
/api/resource/:doctype
```

- Supports API v2:

```text
/api/v2/document/:doctype
```

- Credentials include the same internal URL/public host pattern from HRMS:

```text
Site URL: http://10.192.135.2:8001
Site Host Header: erp.thaiduy.digital
```

### Runtime Dependency Lesson Applied

- `n8n-workflow` is placed in `dependencies`, not only `devDependencies`.
- This avoids the production install issue previously seen in HRMS where a tarball installed with `--omit=dev` could not require `n8n-workflow`.

### Verification

- `npm install` completed and generated `package-lock.json`.
- `npm run build` passed.
- `npm run lint` passed.
- `npm pack` created:

```text
n8n-nodes-erpnext-accounting-0.1.0.tgz
```

- Production install test from tarball with `--omit=dev` succeeded.
- The installed package loads and exports:

```text
ErpNextApi,ErpNextAccounting
```

### Next Suggested Step

- Install the Accounting tarball into the live n8n container and create a GET webhook test for a safe read-only resource first, such as:
  - Account `Get Many`
  - Fiscal Year `Get Many`
  - GL Entry `Get Many` with a small limit

### Working Note

- User reminded: keep this session log updated as Accounting work continues.
- Before/after live n8n install, webhook creation, ERPNext test data checks, or publish steps, add the concrete commands, results, and remaining blockers here.

### 2026-05-14 Logo Standardization

- User requested all module node logos use the same shape as HRMS and differ only by module color.
- Updated Accounting node logo to match the HRMS logo structure exactly:
  - Same 60x60 rounded square.
  - Same white ERPNext `E` mark.
  - Same yellow module dot.
  - Accounting main background color: orange-red `#D94A2B`.
- Updated README with the `n8n2erpnext` node identity convention.

### 2026-05-14 Live n8n Install And Account Webhook

- Rebuilt after logo update:

```text
npm run build
npm run lint
npm pack
```

- Results:
  - Build passed.
  - Lint passed.
  - Recreated `n8n-nodes-erpnext-accounting-0.1.0.tgz`.

- Copied tarball into live `n8n` container and installed it into:

```text
/home/node/.n8n/nodes
```

- Install command result:

```text
added 1 package, changed 1 package, and audited 92 packages
```

- Verified installed package loads inside container:

```text
loaded ErpNextApi,ErpNextAccounting
siteUrl,siteHost,apiKey,apiSecret,allowUnauthorizedCerts
ERPNext Accounting erpNextAccounting salesInvoice,purchaseInvoice,paymentEntry,journalEntry,account,costCenter,fiscalYear,glEntry,customDocType,frappeMethod
```

- Restarted `n8n`.
- Startup log showed no Accounting node load error.
- n8n still shows the known Python task runner warning, unrelated to this node.

### 2026-05-14 Accounting GET Accounts Workflow

- Created workflow artifact:

```text
n8n-webhook-erpnext-accounting-get-accounts.workflow.json
```

- Imported and activated/published workflow in live n8n:

```text
Workflow name: ERPNext Accounting GET Accounts Webhook
Workflow id: accGetAccounts01
Webhook path: erpnext-accounting-get-accounts
Local test URL: http://127.0.0.1:5678/webhook/erpnext-accounting-get-accounts
Public URL: https://n8n.thaiduy.store/webhook/erpnext-accounting-get-accounts
```

- Workflow shape:

```text
GET Webhook -> Get Accounts
```

- ERPNext Accounting node uses:

```text
Resource: Account
Operation: Get Many
API Version: v1
Fields: name,account_name,account_number,company,account_type,is_group
Filters JSON: []
Limit: 20
Order By: modified desc
Credential: ERPNext account
```

- Tested local webhook:

```text
curl -sS -i --max-time 30 --connect-timeout 5 http://127.0.0.1:5678/webhook/erpnext-accounting-get-accounts
```

- Result:

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

- Response returned 20 Account records from ERPNext production, including:

```json
{
  "name": "VAT - TDD",
  "account_name": "VAT",
  "company": "Thái Duy Digital",
  "account_type": "Tax",
  "is_group": 0
}
```

- Tested public webhook:

```text
curl -sS -i --max-time 30 --connect-timeout 5 https://n8n.thaiduy.store/webhook/erpnext-accounting-get-accounts
```

- Result:

```text
HTTP/2 200
Content-Type: application/json; charset=utf-8
```

- Public URL returned the same 20 Account records.
- Updated README with the Accounting GET Accounts webhook guide and the current VPS/LXD credential pattern.

### 2026-05-14 Journal Entry API v2 Nested Accounts Test

- User requested a simple Journal Entry test through n8n to verify whether `GenericFunctions.ts` can POST complex nested objects to ERPNext/Frappe API v2.
- Goal: test `accounts` child table payload, which must balance debit and credit.
- Safety decision:
  - Create a draft Journal Entry only.
  - Do not submit.
  - Verify no GL Entry is created.

- Queried ERPNext Account leaf records in LXD to choose valid accounts:

```text
1110 - Cash - TDD
3300 - Opening Balance Equity - TDD
```

- Created workflow artifact:

```text
n8n-webhook-erpnext-accounting-v2-create-journal-entry.workflow.json
```

- Imported and activated/published workflow in live n8n:

```text
Workflow name: ERPNext Accounting V2 Create Journal Entry Test
Workflow id: accCreateJournalEntryV2Test01
Webhook path: erpnext-accounting-v2-create-journal-entry-test
Local test URL: http://127.0.0.1:5678/webhook/erpnext-accounting-v2-create-journal-entry-test
```

- Workflow shape:

```text
GET Webhook -> Create Draft Journal Entry V2
```

- ERPNext Accounting node uses:

```text
Resource: Journal Entry
Operation: Create
API Version: v2
Credential: ERPNext account
```

- Data JSON:

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

- Tested local webhook:

```text
curl -sS -i --max-time 30 --connect-timeout 5 http://127.0.0.1:5678/webhook/erpnext-accounting-v2-create-journal-entry-test
```

- Result:

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

- ERPNext created draft Journal Entry:

```text
name: ACC-JV-2026-00001
docstatus: 0
voucher_type: Journal Entry
posting_date: 2026-05-14
total_debit: 1000
total_credit: 1000
difference: 0
```

- Response included nested `accounts` rows:
  - Debit row: `1110 - Cash - TDD`, debit `1000`, credit `0`
  - Credit row: `3300 - Opening Balance Equity - TDD`, debit `0`, credit `1000`

- Verified directly in ERPNext DB:

```text
select name, docstatus, voucher_type, posting_date, total_debit, total_credit, difference, user_remark
from `tabJournal Entry`
where name="ACC-JV-2026-00001";
```

- DB result confirmed:

```text
ACC-JV-2026-00001 | docstatus 0 | total_debit 1000 | total_credit 1000 | difference 0
```

- Verified no ledger impact:

```text
select count(*) as gl_entries
from `tabGL Entry`
where voucher_type="Journal Entry"
and voucher_no="ACC-JV-2026-00001";
```

- GL Entry count:

```text
0
```

- Conclusion:
  - `GenericFunctions.ts` handles nested object payloads correctly for API v2 `POST /api/v2/document/Journal%20Entry`.
  - ERPNext accepted and normalized the child table rows.
  - The balanced accounting validation passed.
  - No GL Entry was created because the document is draft.

- Deactivated the Journal Entry test workflow after the successful run to prevent accidental repeated draft creation.
- Restarted n8n so the deactivation took effect.
- Confirmed the webhook no longer creates entries:

```text
HTTP/1.1 404 Not Found
Active version not found for workflow with id "accCreateJournalEntryV2Test01"
```

- Updated README with the Journal Entry API v2 nested payload test and safety notes.

### 2026-05-14 Accounting API v2 GetMany And Get By ID

- Continued following the HRMS session-log pattern after the Journal Entry nested payload test passed.
- Created read-only API v2 Account getMany workflow artifact:

```text
n8n-webhook-erpnext-accounting-v2-get-accounts.workflow.json
```

- Imported and activated/published workflow in live n8n:

```text
Workflow name: ERPNext Accounting V2 GET Accounts Webhook
Workflow id: accGetAccountsV2_01
Webhook path: erpnext-accounting-v2-get-accounts
Local test URL: http://127.0.0.1:5678/webhook/erpnext-accounting-v2-get-accounts
```

- ERPNext Accounting node parameters:

```text
Resource: Account
Operation: Get Many
API Version: v2
Fields: name,account_name,account_number,company,account_type,is_group
Filters JSON: []
Limit: 20
Order By: modified desc
Credential: ERPNext account
```

- Tested local webhook:

```text
curl -sS -i --max-time 30 --connect-timeout 5 http://127.0.0.1:5678/webhook/erpnext-accounting-v2-get-accounts
```

- Result:

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

- Response returned the same 20 Account records as the v1 getMany test.
- Conclusion: API v2 `getMany` works for Account against ERPNext/Frappe v16 production through this node.

- Created read-only Journal Entry get-by-ID workflow artifact:

```text
n8n-webhook-erpnext-accounting-get-journal-entry-by-id.workflow.json
```

- Imported and activated/published workflow in live n8n:

```text
Workflow name: ERPNext Accounting GET Journal Entry By ID Webhooks
Workflow id: accGetJournalEntryById01
Webhook paths:
- erpnext-accounting-get-journal-entry
- erpnext-accounting-v2-get-journal-entry
```

- Workflow shape:

```text
GET Journal Entry Webhook -> Get Journal Entry V1
GET Journal Entry V2 Webhook -> Get Journal Entry V2
```

- Tested v1:

```text
curl -sS -i --max-time 30 --connect-timeout 5 'http://127.0.0.1:5678/webhook/erpnext-accounting-get-journal-entry?name=ACC-JV-2026-00001'
```

- Tested v2:

```text
curl -sS -i --max-time 30 --connect-timeout 5 'http://127.0.0.1:5678/webhook/erpnext-accounting-v2-get-journal-entry?name=ACC-JV-2026-00001'
```

- Results:

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

- Both v1 and v2 returned `ACC-JV-2026-00001`.
- Both responses included the nested `accounts` child table.
- v2 response includes more null-valued fields than v1, as expected from the different Frappe response shape.

- Restarted n8n after activating these workflows.
- Startup log showed:

```text
Activated workflow "ERPNext Accounting V2 GET Accounts Webhook" (ID: accGetAccountsV2_01)
Activated workflow "ERPNext Accounting GET Journal Entry By ID Webhooks" (ID: accGetJournalEntryById01)
```

- Updated README with API v2 Account getMany and Journal Entry get-by-ID examples.

### 2026-05-14 Journal Entry API v2 Update Test

- User requested testing `Update` on the same draft Journal Entry `ACC-JV-2026-00001`.
- Goal: verify document lifecycle through the node up to:

```text
Create -> Get -> Update
```

- Safety decision:
  - Update only `user_remark`.
  - Keep document draft.
  - Do not submit.
  - Verify no GL Entry is created.

- Checked pre-update DB state:

```text
name: ACC-JV-2026-00001
docstatus: 0
total_debit: 1000
total_credit: 1000
difference: 0
gl_entries: 0
user_remark: n8n Accounting node API v2 nested accounts draft test. Do not submit.
```

- Created workflow artifact:

```text
n8n-webhook-erpnext-accounting-v2-update-journal-entry.workflow.json
```

- Imported and activated/published workflow in live n8n:

```text
Workflow name: ERPNext Accounting V2 Update Journal Entry Test
Workflow id: accUpdateJournalEntryV2Test01
Webhook path: erpnext-accounting-v2-update-journal-entry-test
Local test URL: http://127.0.0.1:5678/webhook/erpnext-accounting-v2-update-journal-entry-test
HTTP Method: POST
```

- Workflow shape:

```text
POST Webhook -> Update Draft Journal Entry V2
```

- ERPNext Accounting node uses:

```text
Resource: Journal Entry
Operation: Update
API Version: v2
Document Name: ACC-JV-2026-00001
Credential: ERPNext account
```

- Data JSON:

```json
{
  "user_remark": "n8n Accounting node API v2 update test passed. Draft remains unsubmitted; no GL Entry created."
}
```

- Tested local webhook:

```text
curl -sS -i --max-time 30 --connect-timeout 5 -X POST http://127.0.0.1:5678/webhook/erpnext-accounting-v2-update-journal-entry-test
```

- Result:

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

- Response confirmed:

```text
name: ACC-JV-2026-00001
docstatus: 0
modified: 2026-05-14 04:57:44.219841
total_debit: 1000
total_credit: 1000
difference: 0
user_remark: n8n Accounting node API v2 update test passed. Draft remains unsubmitted; no GL Entry created.
```

- Verified directly in ERPNext DB:

```text
select name, docstatus, total_debit, total_credit, difference, user_remark, modified
from `tabJournal Entry`
where name="ACC-JV-2026-00001";
```

- DB result confirmed:

```text
ACC-JV-2026-00001 | docstatus 0 | total_debit 1000 | total_credit 1000 | difference 0
```

- Verified no ledger impact:

```text
select count(*) as gl_entries
from `tabGL Entry`
where voucher_type="Journal Entry"
and voucher_no="ACC-JV-2026-00001";
```

- GL Entry count:

```text
0
```

- Conclusion:
  - API v2 update works through `GenericFunctions.ts`.
  - The node successfully used `PATCH /api/v2/document/Journal%20Entry/ACC-JV-2026-00001`.
  - Document lifecycle through this node is now verified for `Create -> Get -> Update`.
  - No GL Entry was created because the document remained draft.

- Deactivated the Update workflow after the successful run to prevent accidental repeated writes.
- Restarted n8n so the deactivation took effect.
- Confirmed the webhook no longer updates entries:

```text
HTTP/1.1 404 Not Found
Active version not found for workflow with id "accUpdateJournalEntryV2Test01"
```

- Updated README with the Journal Entry API v2 update test and safety notes.

- Submit/Cancel is the next and highest-risk Accounting test:
  - Submit will post GL Entries.
  - Cancel will reverse/cancel the submitted accounting effect.
  - Do not run Submit/Cancel on production without explicit user confirmation.

### 2026-05-14 Journal Entry Submit/Cancel Test

- User confirmed ERPNext on LXD is for testing node behavior and said to continue without worrying about the database.
- Goal: verify Accounting document lifecycle through the node:

```text
Create -> Get -> Update -> Submit -> Cancel
```

- Pre-submit DB state:

```text
name: ACC-JV-2026-00001
docstatus: 0
total_debit: 1000
total_credit: 1000
difference: 0
gl_entries: 0
```

- Created workflow artifact:

```text
n8n-webhook-erpnext-accounting-submit-cancel-journal-entry.workflow.json
```

- Imported and activated/published workflow in live n8n:

```text
Workflow name: ERPNext Accounting Submit Cancel Journal Entry Test
Workflow id: accSubmitCancelJournalEntry01
Submit path: erpnext-accounting-submit-journal-entry-test
Cancel path: erpnext-accounting-cancel-journal-entry-test
HTTP Method: POST
```

- Workflow shape:

```text
POST Submit Webhook -> Submit Journal Entry
POST Cancel Webhook -> Cancel Journal Entry
```

- ERPNext Accounting node uses:

```text
Resource: Journal Entry
API Version: v2
Document Name: ACC-JV-2026-00001
Operations: Submit and Cancel
Credential: ERPNext account
```

#### Submit Result

- Tested Submit webhook:

```text
curl -sS -i --max-time 30 --connect-timeout 5 -X POST http://127.0.0.1:5678/webhook/erpnext-accounting-submit-journal-entry-test
```

- Result:

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

- Response confirmed:

```text
name: ACC-JV-2026-00001
docstatus: 1
total_debit: 1000
total_credit: 1000
difference: 0
```

- Verified directly in ERPNext DB:

```text
select name, docstatus, total_debit, total_credit, difference, modified
from `tabJournal Entry`
where name="ACC-JV-2026-00001";

select name, account, debit, credit, is_cancelled, posting_date, voucher_no
from `tabGL Entry`
where voucher_type="Journal Entry"
and voucher_no="ACC-JV-2026-00001"
order by account;
```

- DB result after Submit:

```text
Journal Entry docstatus: 1
GL Entry rows: 2
GL debit total: 1000
GL credit total: 1000
Cancelled GL rows: 0
```

- GL rows after Submit:

```text
1110 - Cash - TDD                    debit 1000  credit 0     is_cancelled 0
3300 - Opening Balance Equity - TDD  debit 0     credit 1000  is_cancelled 0
```

#### Cancel Failure And Fix

- First Cancel attempt returned:

```text
HTTP/1.1 500 Internal Server Error
{"message":"Error in workflow"}
```

- n8n execution data showed:

```text
NodeApiError: The service was not able to process your request
Request failed with status code 500
```

- Frappe log showed the request hit:

```text
cmd: frappe.client.cancel
Form Dict: {"doc": {...}}
```

- Root cause:
  - `frappe.client.submit` signature accepts `doc`.
  - `frappe.client.cancel` signature accepts `doctype, name`.
  - Existing `frappeRunDocAction()` sent `{ doc }` for both submit and cancel.

- Confirmed in Frappe v16 source:

```text
def submit(doc)
def cancel(doctype, name)
```

- Fixed `nodes/ErpNextAccounting/GenericFunctions.ts`:

```text
submit -> POST /api/method/frappe.client.submit with { doc }
cancel -> POST /api/method/frappe.client.cancel with { doctype: doc.doctype, name: doc.name }
```

- Rebuilt, linted, repacked, and reinstalled the package into live n8n:

```text
npm run build
npm run lint
npm pack
docker cp n8n-nodes-erpnext-accounting-0.1.0.tgz n8n:/tmp/n8n-nodes-erpnext-accounting-0.1.0.tgz
docker exec n8n sh -lc 'cd /home/node/.n8n/nodes && npm install /tmp/n8n-nodes-erpnext-accounting-0.1.0.tgz --omit=dev'
docker restart n8n
```

- Verification after reinstall:

```text
loaded ErpNextApi,ErpNextAccounting
```

#### Cancel Result After Fix

- Retried Cancel webhook:

```text
curl -sS -i --max-time 30 --connect-timeout 5 -X POST http://127.0.0.1:5678/webhook/erpnext-accounting-cancel-journal-entry-test
```

- Result:

```text
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
```

- Response confirmed:

```text
name: ACC-JV-2026-00001
docstatus: 2
total_debit: 1000
total_credit: 1000
difference: 0
```

- Verified directly in ERPNext DB:

```text
Journal Entry docstatus: 2
GL Entry rows: 4
GL debit total: 2000
GL credit total: 2000
Cancelled GL rows: 4
```

- GL rows after Cancel:

```text
1110 - Cash - TDD                    debit 1000  credit 0     is_cancelled 1
1110 - Cash - TDD                    debit 0     credit 1000  is_cancelled 1
3300 - Opening Balance Equity - TDD  debit 0     credit 1000  is_cancelled 1
3300 - Opening Balance Equity - TDD  debit 1000  credit 0     is_cancelled 1
```

- Conclusion:
  - Full Journal Entry lifecycle through Accounting node is verified:

```text
Create -> Get -> Update -> Submit -> Cancel
```

  - Submit posts GL Entries.
  - Cancel creates reverse GL Entries and marks all related GL rows as cancelled.
  - The node helper now handles Frappe submit/cancel method signature differences correctly.

- Deactivated Submit/Cancel workflow after testing.
- Restarted n8n so deactivation took effect.
- Confirmed both POST endpoints no longer execute:

```text
HTTP/1.1 404 Not Found
Active version not found for workflow with id "accSubmitCancelJournalEntry01"
```

- Updated README with Submit/Cancel test results and implementation note.

## Project Handover For A Fresh AI Session

This section lets a new session continue Accounting work without rereading the whole repo.

### What This Repository Is

This repository is an n8n community node package named:

```text
n8n-nodes-erpnext-accounting
```

Its purpose is to expose ERPNext/Frappe Accounting v15-v16 operations inside n8n. It is scoped to Accounting resources and keeps two escape hatches:

- `Custom DocType` for arbitrary Frappe document doctypes.
- `Frappe Method` for whitelisted method calls.

### Important Files

```text
package.json
package-lock.json
tsconfig.json
gulpfile.js
index.ts
credentials/ErpNextApi.credentials.ts
nodes/ErpNextAccounting/ErpNextAccounting.node.ts
nodes/ErpNextAccounting/GenericFunctions.ts
credentials/erpnext.svg
nodes/ErpNextAccounting/erpnext-accounting.svg
README.md
SESSION_LOG.md
n8n-webhook-erpnext-accounting-get-accounts.workflow.json
n8n-webhook-erpnext-accounting-v2-get-accounts.workflow.json
n8n-webhook-erpnext-accounting-v2-create-journal-entry.workflow.json
n8n-webhook-erpnext-accounting-v2-update-journal-entry.workflow.json
n8n-webhook-erpnext-accounting-get-journal-entry-by-id.workflow.json
n8n-webhook-erpnext-accounting-submit-cancel-journal-entry.workflow.json
n8n-nodes-erpnext-accounting-0.1.0.tgz
dist/
```

What each file does:

- `package.json`: npm metadata, n8n package metadata, scripts, dependencies.
- `package-lock.json`: locked dependency tree.
- `tsconfig.json`: TypeScript compiles source to CommonJS under `dist/`, emits declarations and sourcemaps.
- `gulpfile.js`: copies SVG icons from source folders into `dist/`.
- `index.ts`: exports the credential class and node class.
- `credentials/ErpNextApi.credentials.ts`: defines the n8n credential type `erpNextApi`.
- `nodes/ErpNextAccounting/ErpNextAccounting.node.ts`: defines the n8n node UI and execution logic.
- `nodes/ErpNextAccounting/GenericFunctions.ts`: all low-level Frappe/ERPNext HTTP helper functions.
- `nodes/ErpNextAccounting/erpnext-accounting.svg`: module logo. Must keep same shape as HRMS and only vary module color. Accounting color is `#D94A2B`.
- `README.md`: user/developer guide and tested workflow examples.
- `SESSION_LOG.md`: operational log and handover file.
- `dist/`: generated build output. n8n loads from this folder, not directly from TypeScript source.

### Package Metadata And Build

`package.json` currently has:

```text
name: n8n-nodes-erpnext-accounting
version: 0.1.0
main: dist/index.js
```

The n8n metadata is:

```json
{
  "n8nNodesApiVersion": 1,
  "credentials": [
    "dist/credentials/ErpNextApi.credentials.js"
  ],
  "nodes": [
    "dist/nodes/ErpNextAccounting/ErpNextAccounting.node.js"
  ]
}
```

Scripts:

```text
npm run build       -> tsc && gulp build:icons
npm run lint        -> eslint credentials nodes --ext .ts
npm pack            -> creates n8n-nodes-erpnext-accounting-0.1.0.tgz
npm run dev         -> TypeScript watch mode
```

Important dependency decision:

- `n8n-workflow` is in `dependencies`, not only `devDependencies`.
- This avoids production tarball installs failing with `Cannot find module 'n8n-workflow'`.

Verification that has passed:

```text
npm install
npm run build
npm run lint
npm pack
production install test with --omit=dev
```

### Credential And Network Pattern

Credential class:

```text
credentials/ErpNextApi.credentials.ts
Class: ErpNextApi
n8n credential type name: erpNextApi
displayName: ERPNext API
```

Credential fields:

```text
siteUrl
siteHost
apiKey
apiSecret
allowUnauthorizedCerts
```

Current live n8n credential:

```text
Credential name: ERPNext account
Credential id: 9hFY985G0WpX5Xyt
Credential type: erpNextApi
```

Current working VPS/LXD pattern:

```text
Site URL: http://10.192.135.2:8001
Site Host Header: erp.thaiduy.digital
```

Reason:

- Public `erp.thaiduy.digital` is protected by NetBird proxy/auth/IP policy.
- n8n reaches ERPNext reliably through the internal LXD nginx address while sending the public Host header.

### Live n8n Install

The package tarball was installed into the live `n8n` container:

```text
/home/node/.n8n/nodes
```

Installed package verified inside container:

```text
loaded ErpNextApi,ErpNextAccounting
siteUrl,siteHost,apiKey,apiSecret,allowUnauthorizedCerts
ERPNext Accounting erpNextAccounting salesInvoice,purchaseInvoice,paymentEntry,journalEntry,account,costCenter,fiscalYear,glEntry,customDocType,frappeMethod
```

n8n must be restarted after installing/updating the package or activating workflows by CLI.

Known unrelated n8n warnings:

- Python task runner missing in internal mode.
- `ERR_ERL_UNEXPECTED_X_FORWARDED_FOR` trust proxy warning.

These warnings have not blocked the tested webhooks.

### Supported Resources And Operations

Resources:

```text
Sales Invoice
Purchase Invoice
Payment Entry
Journal Entry
Account
Cost Center
Fiscal Year
GL Entry
Custom DocType
Frappe Method
```

Document operations:

```text
Create
Get
Get Many
Update
Delete
Submit
Cancel
```

Frappe Method operation:

```text
Run Method
```

API versions:

```text
v1: /api/resource/:doctype
v2: /api/v2/document/:doctype
```

### Live Workflows

Active read-only workflows:

```text
ERPNext Accounting GET Accounts Webhook
ID: accGetAccounts01
Path: /webhook/erpnext-accounting-get-accounts
API: v1
Status: tested 200 OK
```

```text
ERPNext Accounting V2 GET Accounts Webhook
ID: accGetAccountsV2_01
Path: /webhook/erpnext-accounting-v2-get-accounts
API: v2
Status: tested 200 OK
```

```text
ERPNext Accounting GET Journal Entry By ID Webhooks
ID: accGetJournalEntryById01
Paths:
- /webhook/erpnext-accounting-get-journal-entry
- /webhook/erpnext-accounting-v2-get-journal-entry
APIs: v1 and v2
Status: tested 200 OK with `ACC-JV-2026-00001`
```

Inactive safety workflow:

```text
ERPNext Accounting V2 Create Journal Entry Test
ID: accCreateJournalEntryV2Test01
Path: /webhook/erpnext-accounting-v2-create-journal-entry-test
Status: intentionally deactivated after one successful draft creation
```

```text
ERPNext Accounting V2 Update Journal Entry Test
ID: accUpdateJournalEntryV2Test01
Path: /webhook/erpnext-accounting-v2-update-journal-entry-test
Status: intentionally deactivated after one successful draft update
```

```text
ERPNext Accounting Submit Cancel Journal Entry Test
ID: accSubmitCancelJournalEntry01
Paths:
- /webhook/erpnext-accounting-submit-journal-entry-test
- /webhook/erpnext-accounting-cancel-journal-entry-test
Status: intentionally deactivated after successful Submit/Cancel lifecycle test
```

### Journal Entry Test Result

The nested `accounts` API v2 create test succeeded.

Created draft:

```text
Journal Entry: ACC-JV-2026-00001
docstatus: 0
total_debit: 1000
total_credit: 1000
difference: 0
GL Entry count: 0
```

Conclusion:

- `GenericFunctions.ts` can POST nested child-table objects through API v2.
- ERPNext accepted and normalized the `accounts` rows.
- Debit/Credit balance validation passed.
- No ledger impact occurred because the document is draft.

### Journal Entry Update Test Result

The API v2 update test succeeded.

Updated draft:

```text
Journal Entry: ACC-JV-2026-00001
docstatus: 0
total_debit: 1000
total_credit: 1000
difference: 0
GL Entry count: 0
```

Updated field:

```text
user_remark: n8n Accounting node API v2 update test passed. Draft remains unsubmitted; no GL Entry created.
```

Conclusion:

- `Update` works through the Accounting node with API v2.
- Document lifecycle through this node is verified for `Create -> Get -> Update`.
- Submit/Cancel was tested afterward and is documented in the next section.

### Journal Entry Submit/Cancel Test Result

The Submit/Cancel test succeeded after fixing Cancel payload handling.

Submitted state:

```text
Journal Entry: ACC-JV-2026-00001
docstatus: 1
GL Entry rows: 2
GL debit total: 1000
GL credit total: 1000
Cancelled GL rows: 0
```

Cancelled state:

```text
Journal Entry: ACC-JV-2026-00001
docstatus: 2
GL Entry rows: 4
GL debit total: 2000
GL credit total: 2000
Cancelled GL rows: 4
```

Implementation fix:

- `frappe.client.submit` expects `{ doc }`.
- `frappe.client.cancel` expects `{ doctype, name }`.
- `frappeRunDocAction()` now sends the correct body for each action.

### Recommended Next Steps

- Test `Purchase Invoice` or `Sales Invoice` create as draft with nested `items`.
- Test `Payment Entry` create as draft.
- Add README sections for ERPNext-to-n8n Accounting webhooks if needed.
- Consider adding safer webhook patterns for create operations, such as POST-only plus a shared secret header.
- Decide whether to keep or delete draft `ACC-JV-2026-00001` from ERPNext production after testing.
