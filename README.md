# Guild Expense System

This project is for demonstration purposes only.

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

## Domain Model

The domain model below represents the relational entities needed to track all the information required to empower the core use cases. It was defined using [this online tool](https://start.jhipster.tech/jdl-studio/). It is defined in [this text file](/domainModelDefinition.txt).

![Domain Model](images/jhipster-jdl.png?raw=true "Test Title")

**Partner**: enterprise customer, representing an organization or enterprise.

**User**: each user belongs to a single *partner*, and is an end user of the customer-facing frontend web application and/or API.

**Expense List**: a collection of *expenses* corresponding to a CSV file uploaded by a *user* or records created via the customer-facing API.

**Expense**: a single line item including a dollar amount, external user ID/email, and status (pending, approved, denied).
