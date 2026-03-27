# 20. Backend Security: Everything You Need to Know*

### *1. The Core Security Mindset*
*   *The Problem:* No application is perfectly secure, but many vulnerabilities stem from a single question: *"Where did the developer make an assumption?"*. Developers often assume the "happy path"—that users will type correct inputs and only click intended buttons.
*   *The Attacker's Mindset:* Attackers do not follow the happy path. They purposefully break inputs and poke at boundaries. As a backend engineer, you must always ask: *"What could go wrong here in terms of security?"*.
*   *Crossing Boundaries:* Almost all vulnerabilities occur when data crosses a boundary (e.g., from the browser into a database query language, or from a user's markdown into HTML) and gets misinterpreted.

### *2. Injection Attacks (SQL & Command Injection)*
Injection attacks happen when an application confuses user-provided *data* with executable **code**. 
*   *SQL Injection:* If you build database queries by concatenating raw user strings, attackers can inject SQL commands. 
    *   The Exploit: If a user types `' OR 1=1 --` into an email login field, the first quote closes your string, `OR 1=1` creates a universally true statement, and `--` comments out the rest of your query. This can trick the database into returning all users or letting the attacker log in without a password. They can even inject `; DROP TABLE users;` to delete your database.
    *   The Fix: *Parameterized Queries* (Prepared Statements). This separates the query template from the data. The database treats whatever is inside the parameter slot purely as a string, making it impossible to execute as a command. 
    *   NoSQL Vulnerability: MongoDB and other NoSQL databases are also vulnerable if you pass raw JSON objects directly from the user to the query, as attackers can inject special operators like `$ne` (not equal).
*   *Command (OS) Injection:* If your server executes shell commands (e.g., using `ffmpeg` to process an uploaded image) and directly concatenates the user's filename, an attacker could input `; rm -rf /` to delete your server's files. 
    *   The Fix: Use programming language APIs that accept the command and the arguments as separate parameters, preventing the input from being processed by the shell interpreter.

### *3. Authentication & Password Storage*
Authentication verifies *who* the user is. Implementing this manually in production is extremely complex, so using a third-party auth provider (like Clerk) is highly recommended. If you must store passwords:
*   *Plain Text (Bad):* Never store passwords in plain text. If your database is breached, attackers will steal them and try those credentials across other sites (Credential Stuffing).
*   *Hashing (Better):* Hashing is a one-way mathematical function that turns any string into a fixed-length output (e.g., taking `password123` and turning it into a random-looking string). You cannot reverse a hash.
    *   The Flaw: Attackers use *Rainbow Tables* (massive pre-computed lists of common passwords and their resulting hashes) to instantly decipher hashed passwords in a breached database.
*   *Salting (Crucial):* To defeat Rainbow Tables, generate a cryptographically random string (a *Salt**) for each user. Concatenate the salt to their password *before hashing it. This ensures that even if two users have the same password, their hashes will look completely different.
*   *Slow Hashing Functions:* Modern GPU graphics cards can guess billions of fast hashes (like MD5 or SHA-256) per second. You must use *slow hashing functions* explicitly designed for passwords, such as *Argon2id* or **Bcrypt**. These include a "cost factor" that forces the calculation to take around 400 milliseconds, slowing down brute-force attacks so much that cracking passwords would take centuries.

### *4. Sessions vs. JWTs (Stateful vs. Stateless Auth)*
Once authenticated, the server must remember the user via a session mechanism.
*   *Stateful Sessions (Recommended):* The server generates a random 128-256 character **Session ID**, saves the user's data in a database/Redis alongside this ID, and sends the ID to the client via a Cookie. 
    *   Benefit: Immediate revocation. If an account is compromised, you instantly delete the session from the database, logging the attacker out.
*   *Stateless Sessions (JWTs):* The server packages the user's data (Claims) into a JSON Web Token (JWT), cryptographically signs it, and gives it to the client. The server stores nothing in the database. 
    *   The Flaw: JWTs cannot easily be revoked before they expire.
    *   The Workaround: Use short-lived Access Tokens (e.g., 5 minutes) paired with longer-lived Refresh Tokens. If the token is stolen, the attacker only has access for a few minutes.
    *   Warning: JWT payloads are just Base64 encoded, not encrypted. Anyone can decode and read them, so never store sensitive data inside a JWT.
*   *Cookie Security Flags:* To safely store Session IDs or JWTs in the browser, always set these cookie flags:
    1.  `HttpOnly`: Prevents malicious JavaScript from reading the cookie.
    2.  `Secure`: Ensures the cookie is only sent over encrypted HTTPS connections.
    3.  `SameSite (Strict or Lax)`: Prevents the cookie from being sent during Cross-Origin requests, mitigating CSRF attacks.

### *5. Rate Limiting*
To prevent attackers from brute-forcing passwords or crashing your server via massive request volumes, implement layered rate limiting:
1.  *Per IP Limit:* Blocks specific IPs sending too many requests (flawed if attackers use botnets/VPNs).
2.  *Per Account Limit:* Locks an account after multiple failed attempts (flawed if attackers try one password across thousands of accounts).
3.  *Global Limit:* A hard system-wide cap on login attempts per minute, preventing distributed brute-force attacks.

### *6. Authorization (BOLA and BFLA)*
Authorization determines *what* an authenticated user is allowed to do.
*   *Horizontal Attacks (BOLA/IDOR):* Broken Object Level Authorization occurs when a user alters an ID in an API request (e.g., fetching `/invoices/5`) to view another user's data. 
    *   The Fix: Do not rely purely on the routing layer for security. At the database layer, append `AND user_id = context.user_id` to your query to ensure the requesting user actually owns the resource. Avoid sequential IDs (101, 102) and use UUIDs to make guessing impossible.
    *   Information Leakage: If a user requests an invoice they don't own, return a `404 Not Found` rather than a `403 Forbidden`. A 403 confirms the invoice exists, allowing attackers to enumerate system data for social engineering attacks.
*   *Vertical Attacks (BFLA):* Broken Function Level Authorization happens when a regular user figures out the URL for an admin endpoint (e.g., `/admin/invoices`) and calls it. Hiding the URL (security through obscurity) is not enough. 
    *   The Fix: Explicitly check the user's role (e.g., `role == admin`) in a middleware before executing sensitive functions.
*   *Authorization Principles:* Centralize your logic, adopt a *Default Deny* policy (block everything unless explicitly allowed), and write automated test suites specifically testing edge-case boundary violations.

### *7. Frontend-Facing Vulnerabilities (XSS & CSRF)*
*   *XSS (Cross-Site Scripting):* Occurs when attackers inject malicious `<script>` tags into your site (e.g., via a markdown comment). When other users view the comment, the script runs in their browser, stealing their session cookies or redirecting them to phishing sites.
    *   The Fix: Heavily sanitize user-provided markup at the validation layer before saving it to the database. Use a *Content Security Policy (CSP)* header to instruct the browser to block inline scripts and only run scripts from trusted domains.
*   *CSRF (Cross-Site Request Forgery):* Tricking a user's browser into executing an action on your site while they are browsing an attacker's site (`evil.com`). 
    *   The Fix: Setting modern `SameSite` cookie flags to `Lax` or `Strict` entirely mitigates this issue for modern applications.

### *8. Misconfigurations and Defense in Depth*
*   *Secrets Management:* Never commit API keys, database URLs, or JWT secrets into your source code (GitHub). Always use environment variables (`.env`) or cloud secret managers like AWS Parameter Store. If you accidentally commit a secret, you must rotate (change) it immediately.
*   *Production Debug Mode:* Do not run your production server on `debug` log levels. Debug mode leaks stack traces, database queries, and sensitive user information to the terminal. Production should always be set to `info`.
*   *Security Headers:* Use modern backend middlewares to automatically inject HTTP security headers. For example, `X-Frame-Options` prevents other malicious websites from embedding your site inside an `iframe` to execute Clickjacking attacks. 
*   *Defense in Depth:* No single layer is perfect. Chain your security: Input Validation $\rightarrow$ Parameterized DB Queries $\rightarrow$ Point-of-Access Authorization $\rightarrow$ Security Headers $\rightarrow$ Monitoring/Audit Logs.

