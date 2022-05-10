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

Authentication & identity management is out of scope for this proposal, and assumes the OAuth 2.0 protocol is applied throughout the system with frontends leveraging token-based authentication. An `access_token` should accompany every request to the API via the auth header, and 

---

## Domain Model

![Domain Model](images/jhipster-jdl.png?raw=true "Test Title")


The domain model below represents the relational entities needed to track all the information required to empower the core use cases. It was defined using [this online tool](https://start.jhipster.tech/jdl-studio/).

**Partner**: enterprise customer, representing an organization or enterprise.

**User**: each user belongs to a single *partner*, and is an end user of the customer-facing frontend web application and/or API.

**Expense List**: a collection of *expenses* corresponding to a CSV file uploaded by a *user* or records created via the customer-facing API.

**Expense**: a single line item including a dollar amount, external user ID/email, and status (pending, approved, denied).

### Model JSON Definition

```
entity Partner {
	id UUID,
	name String,
    createdAt Timestamp,
    updatedAt Timestamp
}

entity User {
	id UUID,
    externalId String,
    partnerId UUID,
    firstName String,
    lastName String,
    email String,
    createdAt Timestamp,
    updatedAt Timestamp
}

// defining multiple OneToMany relationships
relationship OneToMany {
	// one partner to many users
	Partner to User{partnerId}
    
    // one expense list to many expenses
    ExpenseList to Expense{expenceListId}
}

entity ExpenseList {
	id UUID, // object key in AWS S3 bucket
    status ExpenseListStatus,
    statusMessage String, // errors, etc.
    // optionally defined by user
    externalId String,
    name String,
    description String,
    // automatically created
    createdAt Timestamp,
    updatedAt Timestamp
}

enum ExpenseListStatus {
	PENDING, // uploaded but not processed
    SUCCEEDED, // files created successfully
    ATTENTION, // some expenses need attention
    FAILED // failed to be processed
}

entity Expense {
	id UUID,
    amount Double,
    currency Currency,
    expenceListId UUID,
    status ExpenseStatus, // defaults to PENDING
    statusMessage String,
    externalUserId String, // user_id || user_email || email 
    externalId String, // optionally defiend by user
    createdAt Timestamp,
    updatedAt Timestamp
}

// can expand to additional currencies
enum Currency {
    USD, EUR, JPY
}

// favoring an enum for readability
enum ExpenseStatus {
	PENDING,
    APPROVED,
    DENIED,
    ATTENTION // needs attention, missing information
}
```

