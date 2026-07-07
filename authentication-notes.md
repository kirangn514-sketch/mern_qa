# Authentication Complete Notes

---

## 1. What is Authentication?

**Simple definition:** Authentication is the process of verifying **who a user is** — proving identity, typically through credentials like a username/password, a token, or a third-party login (Google, GitHub, etc.).

**Why it matters:** Without authentication, anyone could pretend to be anyone else — access other people's data, perform actions on their behalf, or bypass restrictions entirely. Authentication is the foundation that authorization (what a user is *allowed* to do) builds on top of.

**Two broad approaches:**
1. **Stateful (session-based):** Server stores session data; client holds a reference (session ID/cookie).
2. **Stateless (token-based):** Server stores nothing; client holds a self-contained token (JWT) proving identity on every request.

---

## 2. Session

**Simple definition:** A session is a way for the server to **remember a logged-in user** across multiple requests, by storing user data on the server and giving the client a small reference ID (usually via a cookie) to look it up.

### How it works
1. User logs in with credentials.
2. Server creates a **session object** (e.g., `{ userId: 123, role: "admin" }`) and stores it — in memory, a database, or a store like Redis.
3. Server generates a unique **session ID** and sends it to the client in a cookie.
4. On every subsequent request, the browser automatically sends that cookie back.
5. Server looks up the session ID in its store, retrieves the user data, and knows who's making the request.

```js
// Express example using express-session
const session = require("express-session");

app.use(session({
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: { maxAge: 1000 * 60 * 60, httpOnly: true }, // 1 hour
}));

app.post("/login", (req, res) => {
  // after verifying credentials...
  req.session.userId = user.id;
  res.json({ message: "Logged in" });
});

app.get("/profile", (req, res) => {
  if (!req.session.userId) return res.status(401).json({ message: "Not logged in" });
  res.json({ userId: req.session.userId });
});
```

### Session vs Token-based auth

| | Session-based | Token-based (JWT) |
|---|---|---|
| Where state lives | Server (memory/DB/Redis) | Client (in the token itself) |
| Scalability | Harder — needs shared session store across servers | Easier — any server can verify the token independently |
| Revocation | Easy — just delete the session from the store | Harder — token is valid until it expires (needs extra logic to revoke early) |
| Typical use case | Traditional server-rendered web apps | APIs, mobile apps, microservices, SPAs |

---

## 3. JWT (JSON Web Token)

**Simple definition:** JWT is a compact, **self-contained** token format used to prove identity without the server needing to store anything. All the necessary information is packed and cryptographically signed inside the token itself.

### Structure
A JWT has three parts, separated by dots: `header.payload.signature`

```
eyJhbGciOiJIUzI1NiJ9.eyJpZCI6MSwicm9sZSI6ImFkbWluIn0.4f8s...
```

| Part | Contains | Notes |
|---|---|---|
| **Header** | Algorithm + token type (e.g., `{ "alg": "HS256", "typ": "JWT" }`) | Base64-encoded, not encrypted |
| **Payload** | Claims — the actual data (`userId`, `role`, `exp`, etc.) | Base64-encoded, **not encrypted** — readable by anyone with the token |
| **Signature** | A cryptographic signature of header + payload, signed with a secret/private key | Ensures the token hasn't been **tampered with** |

⚠️ **Important:** JWT payloads are encoded, not encrypted. Never store passwords, secrets, or sensitive personal data inside a JWT payload — anyone can decode and read it (they just can't *modify* it without invalidating the signature).

### Issuing and verifying a JWT
```js
const jwt = require("jsonwebtoken");

// Issue a token after successful login
const token = jwt.sign(
  { id: user.id, role: user.role },
  process.env.JWT_SECRET,
  { expiresIn: "15m" }
);

// Verify a token on protected routes
function authenticate(req, res, next) {
  const token = req.headers.authorization?.split(" ")[1];
  if (!token) return res.status(401).json({ message: "No token provided" });

  jwt.verify(token, process.env.JWT_SECRET, (err, decoded) => {
    if (err) return res.status(403).json({ message: "Invalid or expired token" });
    req.user = decoded;
    next();
  });
}
```

### Signing algorithms
- **HS256 (symmetric):** Same secret used to sign and verify — simple, but every service that verifies tokens must share the secret.
- **RS256 (asymmetric):** Private key signs, public key verifies — safer for distributed systems since only one service needs the private key, while others can verify with the public key alone.

---

## 4. Access Token

**Simple definition:** A **short-lived** token used to authenticate API requests. It's what you actually send with each request to prove "I'm allowed to access this resource right now."

- Typically a JWT, but doesn't have to be.
- **Short expiry (15 minutes to 1 hour)** is intentional — it limits the damage if the token is ever stolen (e.g., via XSS). Even if leaked, it becomes useless quickly.
- Sent with every request, usually in the `Authorization` header:
  ```
  Authorization: Bearer <access_token>
  ```

```js
app.get("/dashboard", authenticate, (req, res) => {
  res.json({ message: `Welcome, user ${req.user.id}` });
});
```

**The tradeoff:** Short expiry is safer but means the user would have to log in again frequently — this is where **refresh tokens** come in.

---

## 5. Refresh Token

**Simple definition:** A **long-lived** token used solely to obtain a **new access token** once the old one expires — without forcing the user to log in again.

### Why both access + refresh tokens exist
- Access tokens are short-lived for security.
- But constantly re-entering a password every 15 minutes would be a terrible user experience.
- **Solution:** Issue a long-lived refresh token (days/weeks) alongside the short-lived access token. When the access token expires, the client silently uses the refresh token to get a new one — no password re-entry needed.

### Flow
1. User logs in → server issues both an **access token** (short-lived) and a **refresh token** (long-lived).
2. Client uses the access token for API calls.
3. When the access token expires, the client calls a `/refresh` endpoint, sending the refresh token.
4. Server verifies the refresh token, and if valid, issues a **new access token** (and often a new refresh token too — called **rotation**).
5. If the refresh token itself is invalid/expired/revoked, the user must log in again.

```js
app.post("/refresh", async (req, res) => {
  const refreshToken = req.cookies.refreshToken;
  if (!refreshToken) return res.status(401).json({ message: "No refresh token" });

  // Check it's still valid AND not revoked (e.g., stored in DB / not blacklisted)
  const storedToken = await RefreshToken.findOne({ token: refreshToken });
  if (!storedToken) return res.status(403).json({ message: "Invalid refresh token" });

  jwt.verify(refreshToken, process.env.REFRESH_SECRET, (err, decoded) => {
    if (err) return res.status(403).json({ message: "Expired refresh token" });

    const newAccessToken = jwt.sign(
      { id: decoded.id, role: decoded.role },
      process.env.JWT_SECRET,
      { expiresIn: "15m" }
    );
    res.json({ accessToken: newAccessToken });
  });
});
```

### Key security practices
- **Store refresh tokens server-side** (DB) so they can be **revoked** (e.g., on logout, or if compromised) — unlike access tokens, which can't easily be invalidated before they expire.
- Store refresh tokens in **httpOnly cookies**, not `localStorage`, to reduce exposure to XSS attacks.
- Use **refresh token rotation**: every time a refresh token is used, issue a new one and invalidate the old — if a stolen refresh token is reused after rotation, it signals theft and all sessions can be revoked.

| | Access Token | Refresh Token |
|---|---|---|
| Lifespan | Short (minutes) | Long (days/weeks) |
| Purpose | Authorize API requests | Get a new access token |
| Sent with every request? | Yes | No — only sent to the refresh endpoint |
| Storage | Memory or httpOnly cookie | httpOnly cookie (never localStorage) |
| Revocable? | Hard (valid until expiry) | Easy (stored & checked server-side) |

---

## 6. Cookies

**Simple definition:** A cookie is a small piece of data that the server sends to the browser, which the browser then **automatically attaches** to every future request to that same domain. Commonly used to store session IDs or tokens.

```js
res.cookie("token", jwtToken, {
  httpOnly: true,
  secure: true,
  sameSite: "strict",
  maxAge: 1000 * 60 * 60, // 1 hour
});
```

### Key cookie attributes
| Attribute | Purpose |
|---|---|
| `httpOnly` | Prevents JavaScript from accessing the cookie (see below) |
| `secure` | Cookie only sent over HTTPS |
| `sameSite` | Controls cross-site sending behavior (see below) |
| `maxAge` / `expires` | How long the cookie persists |
| `domain` / `path` | Which domain/paths the cookie is sent to |

### Why cookies vs storing tokens in localStorage?
- Cookies (with `httpOnly`) **can't be read by JavaScript**, making them more resistant to **XSS (Cross-Site Scripting)** attacks that try to steal tokens.
- `localStorage` is fully accessible to any JavaScript running on the page — if an attacker manages to inject a malicious script (XSS), they can read and steal anything stored there, including tokens.
- **Best practice:** Store sensitive tokens (especially refresh tokens) in `httpOnly` cookies, not `localStorage`.

---

## 7. HttpOnly

**Simple definition:** `HttpOnly` is a cookie flag that tells the browser: **"This cookie should never be accessible via JavaScript"** (`document.cookie` won't show it). It can only be sent automatically by the browser in HTTP requests.

```js
res.cookie("refreshToken", token, {
  httpOnly: true, // JavaScript (document.cookie) cannot read this
});
```

**Why it matters:** It's one of the most effective defenses against **XSS-based token theft**. Even if an attacker injects malicious JavaScript into your page, they cannot read an `httpOnly` cookie — so they can't steal the token that way.

**What it does NOT protect against:** CSRF (Cross-Site Request Forgery) — since the browser still sends the cookie automatically with requests to your domain, even from a malicious third-party site. This is where `SameSite` comes in.

---

## 8. SameSite

**Simple definition:** `SameSite` is a cookie attribute that controls whether a cookie is sent along with **cross-site requests** — i.e., requests initiated from a different website than the one that set the cookie. It's the main defense against **CSRF attacks**.

### Values

| Value | Behavior |
|---|---|
| `Strict` | Cookie is **never** sent on cross-site requests, even when following a link from another site. Most secure, but can break flows like clicking an email link that should auto-log you in. |
| `Lax` (default in modern browsers) | Cookie is sent on top-level navigations (e.g., clicking a link) but **not** on cross-site subrequests like images, iframes, or AJAX/fetch calls from other sites. Good balance of security and usability. |
| `None` | Cookie is sent on all cross-site requests — required for legitimate cross-site use cases (e.g., third-party embedded widgets), but **must** be paired with `Secure` (HTTPS only). |

```js
res.cookie("token", jwtToken, {
  httpOnly: true,
  secure: true,
  sameSite: "strict", // won't be sent if request originates from another domain
});
```

**Example CSRF scenario `SameSite` prevents:**
A malicious site `evil.com` embeds a hidden form that auto-submits to `yourbank.com/transfer`. Without `SameSite` protection, the browser would still attach your `yourbank.com` session cookie to that request — letting the attacker perform actions as you. `SameSite=Strict` or `Lax` blocks the cookie from being sent in this cross-site context.

---

## 9. OAuth

**Simple definition:** OAuth (2.0) is an **authorization framework** that lets a user grant a third-party application limited access to their data on another service — **without sharing their password**. It's the protocol behind "Login with Google/GitHub/Facebook."

**Important distinction:** OAuth is technically about **authorization** ("this app can access your Google Calendar"), but it's very commonly used as the basis for **authentication** too (via OpenID Connect, an identity layer built on top of OAuth 2.0) — that's what powers "Sign in with Google."

### Key roles
| Role | Description |
|---|---|
| **Resource Owner** | The user who owns the data (e.g., you) |
| **Client** | The application requesting access (e.g., your app) |
| **Authorization Server** | Issues tokens after the user grants permission (e.g., Google's OAuth server) |
| **Resource Server** | Hosts the protected data (e.g., Google's API servers) |

### Simplified OAuth 2.0 flow (Authorization Code flow — most common & secure)
1. User clicks "Login with Google" on your app.
2. Your app redirects the user to Google's authorization server with your `client_id`, requested `scopes` (permissions), and a `redirect_uri`.
3. User logs in to Google and approves the permissions requested.
4. Google redirects back to your app with a temporary **authorization code**.
5. Your backend exchanges that code (plus your `client_secret`) for an **access token** (and often a refresh token) directly with Google's server.
6. Your app uses that access token to call Google's APIs on the user's behalf (e.g., fetch their profile info) — this profile info is then used to log the user into your own app.

```js
// Simplified example using an OAuth library (e.g., Passport.js with Google strategy)
app.get("/auth/google", passport.authenticate("google", { scope: ["profile", "email"] }));

app.get(
  "/auth/google/callback",
  passport.authenticate("google", { failureRedirect: "/login" }),
  (req, res) => {
    // req.user now contains the authenticated Google profile
    res.redirect("/dashboard");
  }
);
```

**Why OAuth is valuable:**
- User never shares their Google password with your app.
- User can revoke access at any time from their Google account settings.
- Access can be scoped narrowly (e.g., "read-only calendar access" rather than full account access).

---

## How It All Fits Together — A Typical Modern Auth Flow

1. User logs in (either with email/password, or via **OAuth** like "Sign in with Google").
2. Server verifies credentials and issues a **JWT access token** (short-lived) + a **refresh token** (long-lived).
3. Both tokens are stored in **httpOnly, secure, SameSite cookies** — protecting against XSS and CSRF.
4. Access token is sent automatically with each request; server verifies it on protected routes.
5. When the access token expires, the client silently calls `/refresh` using the refresh token to get a new access token — no re-login needed.
6. On logout, the refresh token is deleted/revoked from the server's store, and cookies are cleared.

---

## Quick Summary Cheat Sheet

| Concept | One-liner |
|---|---|
| Session | Server stores user state; client holds a reference cookie |
| JWT | Self-contained, signed token proving identity — stateless |
| Access Token | Short-lived token sent with every API request |
| Refresh Token | Long-lived token used only to get a new access token |
| Cookies | Browser-stored data automatically sent with requests to a domain |
| HttpOnly | Cookie flag blocking JavaScript access — defends against XSS token theft |
| SameSite | Cookie flag controlling cross-site sending — defends against CSRF |
| OAuth | Framework for granting third-party access without sharing passwords (e.g., "Login with Google") |

## Interview-Style Quick Answers

**Q: Why use both access and refresh tokens instead of just one long-lived token?**
A single long-lived token would be extremely dangerous if stolen — it stays valid for a long time with no easy way to revoke it. Splitting into a short-lived access token (limits damage if stolen) and a revocable, server-tracked refresh token (only used occasionally, over a more controlled endpoint) balances security with user convenience.

**Q: Is JWT encrypted?**
No — it's only base64-encoded and signed. Anyone can decode and read the payload; the signature only proves it hasn't been altered. Never put secrets in a JWT payload.

**Q: localStorage vs httpOnly cookies for storing tokens — which is safer?**
httpOnly cookies are safer against XSS, since JavaScript cannot read them at all. localStorage is fully readable by any script on the page, making it vulnerable if an attacker manages to inject malicious JavaScript.

**Q: What's the difference between authentication via OAuth and traditional login?**
Traditional login verifies credentials you created directly with your app (e.g., email + password stored in your database). OAuth delegates that verification to a trusted third party (Google, GitHub) — your app never sees or stores the user's password, and instead receives a token proving the third party has already verified them.

**Q: How does SameSite protect against CSRF specifically?**
CSRF attacks rely on the browser automatically attaching cookies to requests, even ones triggered by a malicious third-party site. `SameSite=Strict` or `Lax` stops the browser from sending your cookies along with such cross-site requests, breaking the attack.
