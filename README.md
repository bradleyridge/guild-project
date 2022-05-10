# Guild Expense System

This project is for demonstration purposes only.

## How Far I Made It

To demonstrate a variety of skills, I've included information pertaining to the domain model, multiple microservices, use cases, and core database queries that will be needed to empower this project.

I did not get to:
- Authorization model (RBAC vs. ABAC)
- How requests are authenticated (via JWT minted by auth service, including information in scopes)
- Complete details/responsibilities of Reports Service
  - I lumped in CSV processing and report generation here, but ideally those would have two distinct deployments/services as well as SQS queues

## Prompt

**TuitionCo** is a new ecosystem of frontend web applications and backend microservices empowering enterprise **partners** to manage expense items and manage aggregate **expense reports** by **user**. TuitionCo administrators can additionally approve/deny users with overruns/exceptions and release invoices to customers (partners).

> The `CRUD` representation (create, read, update, delete) dictates which operations are included as use cases. The `_` character indicates that operation is not applicable.

### Assumptions

The following assumptions have been made, indicating clarifying questions that would be asked during a formal requirements gathering and/or prototyping process.

**Partners are Businesses or Organizations**

Each user belongs to one partner, representing a business or organization.

**Users are Unique by Email Address**

Email addresses are unique identifiers for both platform users and expense users.

**Admin Portal is for TuitionCo Users Only**

The admin portal allows TuitionCo administrators to view invoices by partner and by user, set thresholds at the partner and user level, approve/deny individual expenses, and release invoices to customers.

**Uploaded CSV Reports Include User External ID or Email**

Expenses include a user external ID OR unique user email address representative of a unique user in that organization.

**Expenses are in US Dollars**

Expenses are only in US dollars, and do not need to be converted to foreign currencies.

**Authorization Uses ABAC and is Least-Privilaged**

Authentication & identity management is out of scope for this proposal, and assumes the OAuth 2.0 protocol is applied throughout the system with frontends leveraging token-based authentication. An `access_token` should accompany every request to the API via the auth header, including the user's external ID provided by Auth0 or an SSO connection. The Auth0 user ID attribute maps to User.id in the TuitionCo database.

**Expense Report Generation is Automatic and Manual**

Each partner configures when expense reports are generated in order to ensure customers can base these *triggers* on their own business processes. Each partner can configure up to 4 expense reports (weekly). The **generation day** is representative of how many days should pass beyond the **range end day** before the expense report is generated automatically.

| Configurable Field | Default | Description                                                |
|--------------------|---------|------------------------------------------------------------|
| Generation Day     | 4       | Day of month to generate invoice (of previous time period) |
| Range Start Day    | 1       | Day of month to start filtering expenses by (inclusive)    |
| Range End Day      | null    | Day of month to end filtering expenses by (inclusive)      |

Examples:
```js
const monthly = [{ generation_day: 4 }];

const biweekly = [
    { generation_day: 4, /* range_start: 1 */ range_end: 1 },
    { generation_day: 4, range_start: 15, /* range_end: (last day) */ },
];

const weekly = [
    { generation_day: 7, /* range_start: 1 */, range_end: 7 },
    { generation_day: 7, range_start: 8, range_end: 14 },
    { generation_day: 7, range_start: 15, range_end: 21 },
    { generation_day: 7, range_start: 21, /* range_end: (last day) */ },
];
```

Users are also able to generate expense reports for all users OR for a specific user manually by configuring the start/end dates (inclusive) on respective frontend pages.

---

## Architecture

The frontend applications will be koa webservers with nuxt/vue middleware wrapped in OIDC auth middleware - configured to an Auth0 OIDC connection leveraging token-based auth.

Backend applications are also koa webservers, with the exception of the REPORTS_SERVICE - responsible for executing jobs from an SQS queue to **process expense list CSVs** and **generate expense reports**.

![Architecture](images/guild2.png?raw=true)

## Domain Model

The domain model below represents the relational entities needed to track all the information required to empower the core use cases. It was defined using [this online tool](https://start.jhipster.tech/jdl-studio/). It is defined in [this text file](/domainModelDefinition.txt).

![Domain Model](images/jhipster-jdl.png?raw=true)

## Use Cases

The following use cases outline the high level functionality of different components of the ecosystem.

### Actors

**Partner**: enterprise customer, representing an organization or enterprise.

**User**: each user belongs to a single *partner*, and is an end user of the customer-facing frontend web application and/or API.

**Admin**: End user of the administrative portal. Inherits all functionality of the User.

**Client**: a client application consuming the externally facing REST API

**Expense List**: a collection of *expenses* corresponding to a CSV file uploaded by a *user* or records created via the customer-facing API.

**Expense**: a single line item including a dollar amount, external user ID/email, and status (pending, approved, denied).

**Expenses API**: a microservice API responsible for managing all data/records.

**Reports Service**: a microservice API schedulding APIs

**Admin Portal**: frontend vue/nuxt application serving admin features.

**Partner Portal**: frontend vue/nuxt application serving partner/end-user features.

**Auth 0**: External authentication service and identity provider. For this project, all identities are managed in Auth0.

- USER authenticates
- USER uploads expense list as CSV in PARTNER_PORTAL
- USER views expense lists in PARTNER_PORTAL
- USER views expense list details in PARTNER_PORTAL
- USER views expenses in PARTNER_PORTAL
- ADMIN updates expense in ADMIN_PORTAL

### 1) USER Authenticates

This use case is a prerequisite to all subsequent use cases. It is assumed a [PKCE flow](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-proof-key-for-code-exchange-pkce) will be used.

> The below flow also applies to ADMINs using the ADMIN_PORTAL.

**Primary Flow**

1. USER navigates to PARTNER_PORTAL
2. PARTNER_PORTAL finds no valid token-based auth session
3. PARTNER_PORTAL generates `code verifier` and `code challenge`
4. PARTNER_PORTAL sends `code request` and `code challenge` to AUTH_0
5. PARTNER_PORTAL redirects USER to AUTH_0
6. USER completes authentication & consent flow in AUTH_0
7. AUTH_0 redirects USER to PARTNER_PORTAL with `code`
8. PARTNER_PORTAL sends `code` and `code verifier` to AUTH_0
9. AUTH_0 validates inputs and returns `id token`, `access token`, and `refresh token`
10. PARTNER_PORTAL manages token-based sessions for USER

> Now that the user has a valid auth session, the access token will be used to authenticate requests to the EXPENSES_API. When an access token expires, the above flow will be repeated OR PARTNER_PORTAL will use a `refresh token` to obtain a valid `access token`.

### 2) USER Uploads Expense List as CSV

This assumes users are authenticated in the PARTNER_PORTAL.

**Primary Flow: File is Formatted Properly**

1. USER navigates to `Expense Lists` page
2. USER clicks `Upload Expense List` button
3. Device prompts USER to select file (only one `.csv` file can be). For post-MVP, this can include multiple files
4. USER selects file
5. PARTNER_PORTAL displays modal with loading progress of file being uploaded
6. PARTNER_PORTAL uploads file to EXPENSES_API
7. PARTNER_PORTAL updates modal to show file is being validated
8. EXPENSES_API parses file contents and validates headers
   1. validates `user_id || email || user_email` value/column exists
   2. validates `amount || total || expense` value/column exists
   3. validates `currency` (defaults to `USD`) value/column exists
9.  EXPENSES_API creates `expense_list` record in DB for corresponding file (id is object key)
10. EXPENSES_API passes file onto AWS S3 bucket in the `/expense_lists` directory
11. EXPENSES_API adds job to SQS queue to process the list
12. EXPENSES_API responds with new expense list record and result
13. PARTNER_PORTAL updates modal to verify file has been uploaded successfully
14. USER dismisses modal (or modal auto-dismisses 10 seconds later)

**Secondary Flow: File is Missing Fields**

> Repeat primary flow steps 1 - 7.

8. EXPENSES_API discoveres missing fields/columns in CSV data
9. EXPENSES_API creates `expense_list` record with `status: FAILED` and `statusMessage: ['some error here on missing field']`
10. EXPENSES_API responds with new expense list record
11. PARTNER_PORTAL updates modal to display errors

### 3) USER Views Expense Lists

This assumes users are authenticate and have expense lists uploaded with a variety of states.

1. USER navigates to `Expense Lists` page
2. PARTNER_PORTAL displays list of expense lists
   1. Clickable to view list of all expenses from that report
   2. Filterable by status (pending, succeedd, needs attention, failed)
      - `Pending`: file is uploaded but not processed yet
      - `Succeeded`: all expenses were valid
      - `Needs Attention`: invalid expenses were found
      - `Failed`: file was not formatted properly (errors displayed on hover or details view)
   3. Paginated (10 - 20, configurable by user)
   4. Sorted by last updated (processed or uploaded)
3. USER clicks on `View Details` button
4. PARTNER_PORTAL navigates user to `Expenses` view with expense list filter applied

### 4) USER Views Expense List Details

1. USER navigates to `Expenses` page in PARTNER_PORTAL
2. PARTNER_PORTAL displays expenses
   - amount/currency
   - expense list parent (if applicable)
   - sorted by created at timestamps
   - expense date (of transaction/expense)
   - filterable by expense list
   - paginated (10 - 20, configurable by user, can show all as well)
   - status
     - `Pending`: pending review by admin
     - `Approved`: approved by admin
     - `Denied`: denied by admin
     - `Needs Attention`: needs attention from admin for missing information

### 5) USER Views Expense Reports

1. USER navigates to `Expense Reports` page in PARTNER_PORTAL
2. PARTNER_PORTAL displays `Batches` tab (by default), displaying list of generated expense reports grouped by batch
   - number of total expenses
   - user applicable to
   - date range (exact & relative)
   - filterable, sortable, & paginated
   - whether batch was generated automatically or manually
   - when batch/report was generated
3. USER clicks on `Generate New` button
4. PARTNER_PORTAL displays modal to configure filters for the new expense report batch
   1. which users (all, or specific)
   2. date range of expenses
5. USER configures options and clicks `Submit` button
6. PARTNER_PORTAL adds jobs to SQS queue for every partnerId/userId permutation
7. (see use case for reports service)

### 5) Expense List is Processed by REPORTS_SERVICE

The REPORTS_SERVICE executes jobs from an SQS queue. This use case assumes a `event list processing` job has been added to the SQS queue with an `event_list_id` attribute.

1. REPORTS_SERVICE gets `.csv` file from AWS S3, at `/event_lists/${event_list_id}.csv`
2. REPORTS_SERVICE parses CSV to JSON
3. REPORTS_SERVICE generates `expense` record via EXPENSES_API for every record found
4. REPORTS_SERVICE updates `expense_list` record via EXPENSES_API
   1. status (`pending` -> `['succeeded', 'attention']`)


### 6) Expense Report is Generated by REPORTS_SERVICE

This use case assumes jobs exist in the SQS queue with `{ expense_report_id }`, pertaining to valid expense report records in the database.

1. REPORTS_SERVICE opens expense report page in headless browser, at `GET /partners/${partnerId}/expense-reports/users${userId}`
2. REPORTS_SERVICE captures screenshot of page and converts it to PDF
3. REPORTS_SERVICE uploads screenshot to AWS S3 bucket at `/expense_reports/${expenseReportId}/${userId}.pdf`
   1. this can be updated to include partner name and expense user ID/email, etc.
4. If `ExpenseReport.requestedByUserId` exists, email user via Twilio API with link to view newly generated reports
   1. feature can be expanded post-MVP to always email partner email lists

## Common Queries

These are rough queries that have not been validate and may be incomplete. I'd use an ORM like Bookshelf.js or Knex to make these queries more human readable and include pagination out of the box without requiring additional implementation.

### Get Expenses by Date Range and User

This query will have performance issues through time, may need a mechanism to store these in AWS S3 beyond a certain amount of time, to prevent DB from growing infinitely.

```js
const query = `SELECT * FROM expenses 
    WHERE status = ${status.APPROVED}
    AND WHERE partner_id = ${partnerId}
    AND WHERE expense_date > ${getStartDate(expenseReport.range_start, month)}
    AND WHERE expense_date < ${getEndDate(expenseReport.range_end, month)}
    AND WHERE external_user_id = ${userId}
`
```