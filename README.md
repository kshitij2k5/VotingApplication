# Voting Application - Backend (Node.js + Express + MongoDB)

## Project Overview

This is a secure, robust, and well-structured backend for a **Voting Application** built with **Node.js**, **Express**, and **MongoDB (Mongoose)**.
The application allows registered users to vote for listed candidates using their unique government ID (**Aadhar card number**). An admin manages the candidate list but **cannot vote**. The system enforces one vote per user, records vote timestamps, and provides endpoints for live vote counts.

---

## Key Features

* User registration and authentication using **Aadhar card number + password**.
* Secure password storage with hashing (`bcrypt`).
* Role-based access control (user vs admin) using JWT authentication.
* One-time voting per user; votes are recorded with timestamps.
* Candidate management (CRUD) by admin only.
* Live vote counts endpoint sorted by vote count.
* Profile and password management for users.
* Input validation, rate limiting, and basic security best practices.

---

## Models (Mongoose Schemas)

### User

```js
{
  name: { type: String, required: true },
  aadharNumber: { type: String, unique: true, required: true },
  password: { type: String, required: true }, // hashed
  role: { type: String, enum: ['user','admin'], default: 'user' },
  hasVoted: { type: Boolean, default: false },
  votedAt: { type: Date, default: null }
}
```

**Notes:**

* `aadharNumber` is the unique identifier for users and used for login.
* `role` distinguishes between admin and normal user.
* `hasVoted` prevents multiple votes from same user; `votedAt` stores timestamp of vote.

### Candidate

```js
{
  name: { type: String, required: true },
  party: { type: String, required: true },
  age: { type: Number },
  voteCount: { type: Number, default: 0 },
  votes: [
    {
      user: { type: mongoose.Schema.Types.ObjectId, ref: 'User', required: true },
      votedAt: { type: Date, default: Date.now }
    }
  ],
  createdAt: { type: Date, default: Date.now },
  deleted: { type: Boolean, default: false } // optional soft-delete
}
```

**Notes:**

* `voteCount` is stored for fast sorting and querying.
* `votes` array stores a history of voters for audit/traceability (consider size and privacy).
* `deleted` can be used for soft-deletes to preserve audit trails.

---

## Routes (REST API)

### Auth

* `POST /auth/signup`
  Create a new user (body: `name`, `aadharNumber`, `password`). Validate Aadhar format and uniqueness. Hash passwords before storing.
* `POST /auth/login`
  Log in with `aadharNumber` + `password`. Returns JWT access token and user metadata.

### Voting

* `GET /candidates`
  Public: List of candidates (optionally paginated). Exclude sensitive data.
* `POST /vote/:candidateId`
  Protected (user role): Cast vote for candidate with `candidateId`. Steps:

  1. Verify JWT, ensure `role === 'user'`.
  2. Check current user's `hasVoted` flag — reject if `true`.
  3. Atomically increment `candidate.voteCount` and push vote record to `candidate.votes`.
  4. Set user's `hasVoted = true`, `votedAt = now`.
  5. Return success + updated candidate summary.

### Vote Counts

* `GET /vote/counts`
  Public: Returns candidates sorted by `voteCount` descending. Optionally include percentages and total votes.

### User Profile

* `GET /profile`
  Protected: Returns logged-in user's profile (exclude password).
* `PUT /profile/password`
  Protected: Change user's password (body: `currentPassword`, `newPassword`). Re-hash new password securely.

### Admin Candidate Management (Admin-only)

* `POST /candidates`
  Admin: Create new candidate.
* `PUT /candidates/:candidateId`
  Admin: Update candidate details (name, party, age).
* `DELETE /candidates/:candidateId`
  Admin: Remove a candidate from list (soft-delete recommended for audit).

---

## Important Implementation Details & Logic

### 1. Atomic Vote Operation

Use MongoDB transactions (replica set) or atomic update operators to ensure consistency:

* Increment candidate `voteCount` with `$inc: { voteCount: 1 }` and `$push` the vote object in a single update.
* Set user's `hasVoted` flag in the same logical transaction. If transactions are unavailable, use a check-and-update pattern with conditional `findOneAndUpdate`.

Example (Mongoose + transaction):

```js
const session = await mongoose.startSession();
try {
  session.startTransaction();
  const candidate = await Candidate.findByIdAndUpdate(
    candidateId,
    { $inc: { voteCount: 1 }, $push: { votes: { user: userId, votedAt: new Date() } } },
    { new: true, session }
  );
  await User.findByIdAndUpdate(userId, { hasVoted: true, votedAt: new Date() }, { session });
  await session.commitTransaction();
} catch (err) {
  await session.abortTransaction();
  throw err;
} finally {
  session.endSession();
}
```

### 2. Prevent Double Voting

* Primary check: `user.hasVoted` before allowing vote.
* Race condition avoidance: Use transaction or conditional `findOneAndUpdate`:

```js
const userUpdate = await User.findOneAndUpdate(
  { _id: userId, hasVoted: false },
  { $set: { hasVoted: true, votedAt: new Date() } },
  { new: false } // return the old doc; if null, user had already voted
);
if (!userUpdate) throw new Error('User has already voted');
```

Then update candidate.

### 3. Admin Cannot Vote

* Ensure `role` is set to `'admin'` during admin creation.
* In voting middleware, block the route if `req.user.role !== 'user'`.

### 4. Security

* Hash passwords with bcrypt (e.g., `bcrypt.hash(password, 12)`).
* Use JWTs with short expiry for access tokens; optionally implement refresh tokens.
* Validate inputs with `Joi` or `express-validator`.
* Rate-limit auth endpoints (e.g., `express-rate-limit`).
* Sanitize inputs to prevent NoSQL injection.
* Use HTTPS in production and store secrets in environment variables.

### 5. Data Privacy & Audit

* Aadhar numbers are sensitive — consider hashing or encrypting them:

  * Hashing with HMAC using a server-side secret allows uniqueness checks while protecting raw value.
* Keep minimal personal data and consider legal/compliance requirements.
* Keep an audit trail for votes (who voted when) but restrict admin access to raw personal identifiers.

### 6. Candidate Deletion & Audit Trail

* Prefer soft-delete: mark `deleted: true` and keep votes/audit for integrity.
* If hard-delete is necessary, ensure reconciliation of vote counts and user `hasVoted` flags.

---

## Error Handling & Responses

* Use consistent API responses:

```json
{ "success": false, "message": "Reason", "errorCode": "USER_ALREADY_VOTED" }
```

* Use proper HTTP status codes:

  * `200` OK, `201` Created, `400` Bad Request, `401` Unauthorized, `403` Forbidden, `404` Not Found, `409` Conflict, `500` Server Error.

---

## Sample .env (do NOT commit)

```
PORT=8000
MONGO_URI=mongodb://localhost:27017/votingApp
JWT_SECRET=your-strong-secret
JWT_EXPIRES_IN=1h
BCRYPT_SALT_ROUNDS=12
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX=100
```

---

## Testing & Validation

* Write unit tests for controllers, services, and utilities (Jest + Supertest for API).
* Key test cases:

  * Signup/login flow.
  * Vote flow: user votes once, cannot vote again.
  * Admin endpoints protected and only accessible by admin.
  * Concurrent vote attempts to ensure atomicity.
* Use a Postman collection to manually test endpoints and generate example requests.

---

## Deployment & Scaling Notes

* Use a managed MongoDB (Atlas) with replica set for transactions and durability.
* Use clustering or PM2 for Node.js process management.
* Add Redis or another shared store for distributed rate-limiting.
* For very large voter counts:

  * Use denormalized counters (`voteCount`) for read-heavy endpoints.
  * Consider background jobs to reconcile counts if keeping detailed logs.

---

## Postman Example Requests

1. Signup:

```
POST /auth/signup
Body: { "name":"John", "aadharNumber":"123412341234", "password":"Pass@123" }
```

2. Login:

```
POST /auth/login
Body: { "aadharNumber":"123412341234", "password":"Pass@123" }
```

3. Vote (Authenticated):

```
POST /vote/<candidateId>
Headers: Authorization: Bearer <token>
```

---
