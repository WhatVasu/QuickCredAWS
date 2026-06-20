# QuickCred — P2P Lending Platform API

A serverless peer-to-peer lending platform built on AWS Lambda + API Gateway + DynamoDB. Students can post loan requests, get funded by other users, and build a credit score through repayment history.

**Base URL:** `https://d3nvtc3it8.execute-api.us-east-1.amazonaws.com`

---

## Table of Contents

- [Architecture Overview](#architecture-overview)
- [Business Logic](#business-logic)
- [Event Flow](#event-flow)
- [Services & Endpoints](#services--endpoints)
  - [User Service](#user-service)
  - [Loan Service](#loan-service)
  - [Credit Score Service](#credit-score-service)
  - [Repayment Service](#repayment-service)
  - [Notification Service](#notification-service)
- [Error Responses](#error-responses)
- [Status Reference](#status-reference)

---

## Architecture Overview

```
Client
  │
  ├──▶ API Gateway (HTTP API)
  │         │
  │    ┌────┴────────────────────────────────────────┐
  │    │              Lambda Functions                │
  │    │  UserService  LoanService  RepaymentService  │
  │    │  CreditScoreService  NotificationService     │
  │    └────┬────────────────────────────────────────┘
  │         │
  │    ┌────┴──────────────────┐
  │    │      DynamoDB         │
  │    │  Users  Loans         │
  │    │  Repayments           │
  │    │  CreditScores         │
  │    │  Notifications        │
  │    └───────────────────────┘
  │
  └──▶ EventBridge (quickcred-bus)
            │
            ├──▶ NotificationService  (LoanCreated, LoanFunded)
            └──▶ CreditScoreService   (RepaymentCompleted, RepaymentDefaulted)
```

---

## Business Logic

### Users
- Every user gets a **credit profile initialized at score 50** upon registration.
- Email addresses are **unique** — duplicate registrations are rejected with 409.
- Deleting a user also removes their associated credit profile.

### Loans
- Any registered user can **post a loan request** with a desired amount and interest.
- A loan moves through statuses: `REQUESTED → FUNDED`.
- **A borrower cannot fund their own loan** — the lender must be a different user.
- Once a loan is `FUNDED`, its **terms (amount, interest) are locked** and cannot be updated.
- A `FUNDED` loan **cannot be deleted** — it is a financial record for the lender.

### Repayments
- Repayments can only be created for **loans in `FUNDED` status**.
- Only the **actual borrower** of that loan can create a repayment for it.
- **Only one active (PENDING) repayment** is allowed per loan at a time.
- A repayment marked `PAID` **cannot be updated or deleted** — it is a permanent record.
- Valid repayment statuses: `PENDING`, `PAID`, `DEFAULTED`.

### Credit Scores
- Score starts at **50** for every new user.
- Each successful repayment (`PAID`) adds **+5** to the score (max 100).
- Each default (`DEFAULTED`) deducts **-40** from the score (min 0).
- Scores update automatically via EventBridge — no manual intervention needed.

### Notifications
- Notifications are created **automatically** by EventBridge events:
  - `LoanCreated` → borrower is notified.
  - `LoanFunded` → both borrower and lender are notified.
  - `RepaymentCompleted` / `RepaymentDefaulted` → credit score service updates silently.
- Manual notifications can also be created directly via `POST /api/notifications`.

---

## Event Flow

```
POST /api/loans
  └──▶ EventBridge: LoanCreated
            └──▶ NotificationService
                      └──▶ "Loan request created for ₹{amount}" → borrower

PUT /api/loans/{loanId}/fund
  └──▶ EventBridge: LoanFunded
            └──▶ NotificationService
                      ├──▶ "Your loan has been funded for ₹{amount}" → borrower
                      └──▶ "You have successfully funded a loan of ₹{amount}" → lender

PUT /api/repayments/{repaymentId}  (status: PAID)
  └──▶ EventBridge: RepaymentCompleted
            └──▶ CreditScoreService → score +5, successfulRepayments +1

PUT /api/repayments/{repaymentId}  (status: DEFAULTED)
  └──▶ EventBridge: RepaymentDefaulted
            └──▶ CreditScoreService → score -40, defaults +1
```

---

## Services & Endpoints

---

### User Service

#### Create User
```
POST /api/users
```

**Request Body**
```json
{
  "name": "Vasudev",
  "email": "vasu@example.com",
  "college": "Vasavi College",
  "department": "CSE",
  "year": 2
}
```

**Response** `201 Created`
```json
{
  "userId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "email": "vasu@example.com",
  "name": "Vasudev",
  "college": "Vasavi College",
  "department": "CSE",
  "year": 2,
  "reputation": 50
}
```

> Also initializes a credit profile with `score: 50` in CreditScores table.

**Errors**
| Status | Message |
|--------|---------|
| 400 | `Missing required field: {field}` |
| 409 | `A user with this email already exists` |

---

#### Get All Users
```
GET /api/users
```

**Response** `200 OK`
```json
[
  {
    "userId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
    "name": "Vasudev",
    "email": "vasu@example.com",
    "college": "Vasavi College",
    "department": "CSE",
    "year": 2,
    "reputation": 50
  }
]
```

---

#### Get User by ID
```
GET /api/users/{userId}
```

**Response** `200 OK`
```json
{
  "userId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "name": "Vasudev",
  "email": "vasu@example.com",
  "college": "Vasavi College",
  "department": "CSE",
  "year": 2,
  "reputation": 50
}
```

**Errors**
| Status | Message |
|--------|---------|
| 404 | `User not found` |

---

#### Update User
```
PUT /api/users/{userId}
```

**Request Body**
```json
{
  "name": "Vasudev K",
  "college": "Vasavi College",
  "department": "CSE",
  "year": 3
}
```

**Response** `200 OK`
```json
{
  "message": "User Updated"
}
```

**Errors**
| Status | Message |
|--------|---------|
| 404 | `User not found` |

---

#### Delete User
```
DELETE /api/users/{userId}
```

**Response** `200 OK`
```json
{
  "message": "User Deleted"
}
```

> Also deletes the user's credit profile.

**Errors**
| Status | Message |
|--------|---------|
| 404 | `User not found` |

---

### Loan Service

#### Create Loan
```
POST /api/loans
```

**Request Body**
```json
{
  "borrowerId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "borrowerEmail": "vasu@example.com",
  "amount": 5000,
  "interest": 500
}
```

**Response** `201 Created`
```json
{
  "loanId": "4644c03e-6128-45a0-8730-ed7b00c49c0c",
  "borrowerId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "borrowerEmail": "vasu@example.com",
  "amount": 5000,
  "interest": 500,
  "status": "REQUESTED"
}
```

> Fires `LoanCreated` event → notification sent to borrower.

**Errors**
| Status | Message |
|--------|---------|
| 400 | `Missing required field: {field}` |

---

#### Get All Loans
```
GET /api/loans
```

**Response** `200 OK`
```json
[
  {
    "loanId": "4644c03e-6128-45a0-8730-ed7b00c49c0c",
    "borrowerId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
    "borrowerEmail": "vasu@example.com",
    "amount": 5000,
    "interest": 500,
    "status": "REQUESTED"
  }
]
```

---

#### Get Loan by ID
```
GET /api/loans/{loanId}
```

**Response** `200 OK`
```json
{
  "loanId": "4644c03e-6128-45a0-8730-ed7b00c49c0c",
  "borrowerId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "borrowerEmail": "vasu@example.com",
  "amount": 5000,
  "interest": 500,
  "status": "REQUESTED"
}
```

**Errors**
| Status | Message |
|--------|---------|
| 404 | `Loan not found` |

---

#### Update Loan
```
PUT /api/loans/{loanId}
```

> Only allowed when loan status is `REQUESTED`. Funded loans are locked.

**Request Body**
```json
{
  "amount": 5500,
  "interest": 550,
  "status": "REQUESTED"
}
```

**Response** `200 OK`
```json
{
  "message": "Loan Updated",
  "loan": {
    "loanId": "4644c03e-6128-45a0-8730-ed7b00c49c0c",
    "borrowerId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
    "borrowerEmail": "vasu@example.com",
    "amount": 5500,
    "interest": 550,
    "status": "REQUESTED"
  }
}
```

**Errors**
| Status | Message |
|--------|---------|
| 400 | `Missing required field: {field}` |
| 400 | `Cannot modify a funded loan` |
| 404 | `Loan not found` |

---

#### Fund Loan
```
PUT /api/loans/{loanId}/fund
```

> The `lenderId` must be different from the loan's `borrowerId`.

**Request Body**
```json
{
  "lenderId": "abc12345-0000-0000-0000-000000000000",
  "lenderEmail": "lender@example.com"
}
```

**Response** `200 OK`
```json
{
  "loanId": "4644c03e-6128-45a0-8730-ed7b00c49c0c",
  "lenderId": "abc12345-0000-0000-0000-000000000000",
  "status": "FUNDED"
}
```

> Fires `LoanFunded` event → notifications sent to both borrower and lender.

**Errors**
| Status | Message |
|--------|---------|
| 400 | `Missing required field: {field}` |
| 400 | `Loan is not available for funding` (already funded) |
| 400 | `Borrower cannot fund their own loan` |
| 404 | `Loan not found` |

---

#### Delete Loan
```
DELETE /api/loans/{loanId}
```

> Only `REQUESTED` loans can be deleted. `FUNDED` loans cannot be removed.

**Response** `200 OK`
```json
{
  "message": "Loan Deleted"
}
```

**Errors**
| Status | Message |
|--------|---------|
| 400 | `Cannot delete a funded loan` |
| 404 | `Loan not found` |

---

### Credit Score Service

#### Get Credit Score
```
GET /api/creditscores/{userId}
```

**Response** `200 OK`
```json
{
  "userId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "score": 55,
  "completedLoans": 0,
  "successfulRepayments": 1,
  "defaults": 0
}
```

**Score Calculation**

| Event | Score Change |
|-------|-------------|
| Starting score | 50 |
| Successful repayment (`PAID`) | +5 (max 100) |
| Default (`DEFAULTED`) | −40 (min 0) |

**Errors**
| Status | Message |
|--------|---------|
| 404 | `Credit Profile Not Found` |

---

#### Update Credit Score (Manual)
```
PUT /api/creditscores/{userId}
```

> Score is recalculated from the provided counters. Normally updated automatically via EventBridge.

**Request Body**
```json
{
  "completedLoans": 1,
  "successfulRepayments": 3,
  "defaults": 0
}
```

**Response** `200 OK`
```json
{
  "userId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "score": 75,
  "completedLoans": 1,
  "successfulRepayments": 3,
  "defaults": 0
}
```

**Errors**
| Status | Message |
|--------|---------|
| 404 | `Credit Profile Not Found` |

---

### Repayment Service

#### Create Repayment
```
POST /api/repayments
```

> Loan must be in `FUNDED` status. Only the loan's borrower can create a repayment. Only one `PENDING` repayment allowed per loan.

**Request Body**
```json
{
  "loanId": "4644c03e-6128-45a0-8730-ed7b00c49c0c",
  "borrowerId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "borrowerEmail": "vasu@example.com",
  "amount": 500,
  "dueDate": "2026-08-01"
}
```

**Response** `201 Created`
```json
{
  "repaymentId": "fbb6dca7-4312-4660-9cc5-a6ac70124df9",
  "loanId": "4644c03e-6128-45a0-8730-ed7b00c49c0c",
  "borrowerId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "borrowerEmail": "vasu@example.com",
  "amount": 500,
  "dueDate": "2026-08-01",
  "status": "PENDING"
}
```

**Errors**
| Status | Message |
|--------|---------|
| 400 | `Missing required field: {field}` |
| 400 | `Repayments can only be created for funded loans` |
| 403 | `BorrowerId does not match the loan borrower` |
| 404 | `Loan not found` |
| 409 | `A pending repayment already exists for this loan` |

---

#### Get All Repayments
```
GET /api/repayments
```

**Response** `200 OK`
```json
[
  {
    "repaymentId": "fbb6dca7-4312-4660-9cc5-a6ac70124df9",
    "loanId": "4644c03e-6128-45a0-8730-ed7b00c49c0c",
    "borrowerId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
    "borrowerEmail": "vasu@example.com",
    "amount": 500,
    "dueDate": "2026-08-01",
    "status": "PENDING"
  }
]
```

---

#### Get Repayment by ID
```
GET /api/repayments/{repaymentId}
```

**Response** `200 OK`
```json
{
  "repaymentId": "fbb6dca7-4312-4660-9cc5-a6ac70124df9",
  "loanId": "4644c03e-6128-45a0-8730-ed7b00c49c0c",
  "borrowerId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "borrowerEmail": "vasu@example.com",
  "amount": 500,
  "dueDate": "2026-08-01",
  "status": "PENDING"
}
```

**Errors**
| Status | Message |
|--------|---------|
| 404 | `Repayment not found` |

---

#### Update Repayment
```
PUT /api/repayments/{repaymentId}
```

> Cannot update a repayment that is already `PAID`. Marking as `PAID` fires `RepaymentCompleted` event (+5 score). Marking as `DEFAULTED` fires `RepaymentDefaulted` event (−40 score).

**Request Body**
```json
{
  "amount": 500,
  "status": "PAID"
}
```

Valid status values: `PENDING`, `PAID`, `DEFAULTED`

**Response** `200 OK`
```json
{
  "message": "Repayment Updated"
}
```

**Errors**
| Status | Message |
|--------|---------|
| 400 | `Missing required field: {field}` |
| 400 | `Repayment is already marked as PAID` |
| 400 | `Invalid status. Must be one of: PENDING, PAID, DEFAULTED` |
| 404 | `Repayment not found` |

---

#### Delete Repayment
```
DELETE /api/repayments/{repaymentId}
```

> Cannot delete a `PAID` repayment — it is a permanent financial record.

**Response** `200 OK`
```json
{
  "message": "Repayment Deleted"
}
```

**Errors**
| Status | Message |
|--------|---------|
| 400 | `Cannot delete a completed repayment` |
| 404 | `Repayment not found` |

---

### Notification Service

#### Create Notification (Manual)
```
POST /api/notifications
```

**Request Body**
```json
{
  "userId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "message": "Your account has been reviewed."
}
```

**Response** `201 Created`
```json
{
  "notificationId": "4a7d8e30-3614-41ad-b621-b7eeb2d27063",
  "userId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "message": "Your account has been reviewed.",
  "status": "UNREAD",
  "createdAt": "2026-06-20T12:26:57.980668"
}
```

**Errors**
| Status | Message |
|--------|---------|
| 400 | `Missing required field: userId` |
| 400 | `Missing required field: message` |

---

#### Get All Notifications
```
GET /api/notifications
```

**Response** `200 OK`
```json
[
  {
    "notificationId": "4a7d8e30-3614-41ad-b621-b7eeb2d27063",
    "userId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
    "message": "Your loan has been funded for ₹5500",
    "status": "UNREAD",
    "createdAt": "2026-06-20T12:26:51.203217"
  }
]
```

---

#### Get Notification by ID
```
GET /api/notifications/{notificationId}
```

**Response** `200 OK`
```json
{
  "notificationId": "4a7d8e30-3614-41ad-b621-b7eeb2d27063",
  "userId": "1cb07abb-8646-42c8-8e42-4bea49a22b81",
  "message": "Your loan has been funded for ₹5500",
  "status": "UNREAD",
  "createdAt": "2026-06-20T12:26:51.203217"
}
```

**Errors**
| Status | Message |
|--------|---------|
| 404 | `Notification Not Found` |

---

## Error Responses

All errors follow a consistent structure:

```json
{
  "message": "Description of what went wrong"
}
```

| HTTP Status | Meaning |
|-------------|---------|
| 400 | Bad Request — missing or invalid field |
| 403 | Forbidden — action not permitted for this user |
| 404 | Not Found — resource does not exist |
| 405 | Method Not Allowed |
| 409 | Conflict — duplicate resource |
| 500 | Internal Server Error |

---

## Status Reference

**Loan Status**
| Status | Description |
|--------|-------------|
| `REQUESTED` | Loan posted, awaiting a lender |
| `FUNDED` | Loan funded by a lender, terms locked |

**Repayment Status**
| Status | Description |
|--------|-------------|
| `PENDING` | Repayment scheduled, not yet paid |
| `PAID` | Repayment completed — score +5 |
| `DEFAULTED` | Repayment missed — score −40 |

**Notification Status**
| Status | Description |
|--------|-------------|
| `UNREAD` | Notification not yet read by the user |
