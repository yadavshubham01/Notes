# 16. Error Handling and Building Fault-Tolerant Systems

### *1. The Fault-Tolerant Mindset*
*   *The Reality of Backend Engineering:* Errors are not just problems; they are a normal, guaranteed part of building applications. Database queries will fail, external APIs will time out, and users will send bad data.
*   *The Goal:* The question is not if errors will happen, but how your system handles them when they do. A backend engineer must design systems preparing for the worst-case scenarios so that errors do not crash the application or corrupt data.

### *2. The 5 Major Types of Backend Errors*
Backend errors typically fall into five main categories:

#### *A. Logic Errors*
*   *What they are:* These are the most dangerous and sneaky errors because they **do not crash the application**. The code runs fine, but it produces the wrong business result (e.g., applying a discount twice, resulting in negative shipping costs).
*   *Why they happen:* Usually caused by misunderstanding business requirements, implementing complex algorithms incorrectly, or failing to anticipate user behavior edge cases.
*   *Impact:* They can go unnoticed for weeks, silently corrupting data or causing massive financial loss.

#### *B. Database Errors*
*   *Connection Errors:* Happens when the app cannot talk to the database (e.g., network down, or connection pool exhaustion).
*   *Constraint Violations:* Trying to perform an action that breaks database rules, such as a *Unique Constraint* (creating a user with an email that already exists) or a *Foreign Key Constraint* (inserting an order for a customer ID that doesn't exist).
*   *Query Errors & Deadlocks:* Malformed SQL queries (typos in table names) or circular dependencies where multiple operations wait on each other indefinitely.

#### *C. External Service Errors*
*   *The Risk:* Modern apps rely heavily on third parties (Stripe for payments, AWS S3 for storage, Auth0/Clerk for authentication). Every external service is a point of failure you do not control.
*   *Rate Limiting:* If you hit an external API too many times, they will block you and return a `429 Too Many Requests` error. You must handle this using an *exponential backoff strategy* (e.g., wait 1 min, then 2 mins, then 4 mins before retrying).
*   *Service Outages:* When an external cloud provider goes down, your app must handle it gracefully using fallbacks (e.g., switching to an in-memory cache if Redis fails) so core features stay alive.

#### *D. Input Validation Errors*
*   *What they are:* These occur when users send bad or malicious data that doesn't meet system requirements (e.g., wrong email format, a string that is too long, or missing mandatory fields).
*   *How to handle:* These are the easiest errors to handle. You should catch them at the very entry point of your app and immediately throw a `400 Bad Request`.

#### *E. Configuration Errors*
*   *What they are:* Missing or incorrect environment variables (like an OpenAI API key) when moving code between development, staging, and production.
*   *Best Practice:* Always validate your configuration variables **before the server starts**. If a key is missing, intentionally crash the app immediately. It is much better for a deployment to fail to start than for the app to randomly throw `500 Internal Server Errors` to users at runtime.

### *3. Proactive Error Detection*
The best error handling starts before the error causes damage.
*   *Health Checks:* Don't just check if the HTTP server is returning a `200 OK`. You must verify core functionality: run test queries to ensure database connectivity, send test emails to verify your email provider, and validate authentication tokens.
*   *Monitoring & Observability:* Track error rates, but more importantly, track *performance metrics* (like sudden drops in response times) and *business metrics* (like a sudden drop in successful checkouts). Performance degradation is often an early warning sign before a total system failure.

### *4. Error Handling Philosophies*
When an error occurs, your system's immediate response dictates if it becomes a minor hiccup or a major incident.
*   *Recoverable Errors:* (e.g., network timeouts, DB pool limits). Handle these automatically with retry mechanisms and exponential backoff, but be careful not to overwhelm an already stressed system.
*   *Non-Recoverable Errors:* Handle these with containment and graceful degradation (e.g., disabling non-essential features while keeping the core app alive).
*   *Error Propagation (Bubbling Up):* Low-level errors (like a database failure) should be caught and passed up to higher-level processes (using `try/catch` or returning errors) so they can be handled with proper business context.

### *5. The "Global Error Handler" (The Final Safety Net)*
A massive best practice in backend architecture is implementing a Global Error Handling middleware.
*   *How it Works:* Instead of writing repetitive `try/catch` error logic in every single repository or service file, you "bubble up" all errors to a single, centralized middleware at the end of the request cycle.
*   *Routing the Error:* This middleware inspects the bubbled-up error and formats the correct HTTP response:
    *   If it's a Validation error $\rightarrow$ returns a `400 Bad Request` with form details.
    *   If it's a Unique Constraint DB error (e.g., email already exists) $\rightarrow$ returns a `400 Bad Request`.
    *   If it's a "No Rows" DB error (e.g., requesting a user ID that doesn't exist) $\rightarrow$ returns a `404 Not Found`.
*   *Advantages:* It drastically reduces code redundancy and guarantees that if a developer forgets to handle an edge case, the system gracefully catches it instead of throwing a generic, confusing `500 Internal Server Error` to the user.

### *6. Security in Error Handling*
Error messages and logs are prime targets for malicious users. You must strictly control what data leaves your backend.
*   *Never Leak Internal Details:* If a database query fails, never send the raw database error back to the user. It can expose table names, index names, and constraint logic, which attackers use to craft SQL injection attacks. Always intercept these and return a generic "Something went wrong" (`500`) message.
*   *Prevent Enumeration Attacks:* In authentication (login) endpoints, if a user inputs the wrong email, never return "Email does not exist." If you do, attackers will automate scripts to figure out which emails belong to real accounts. Always use a generic message like **"Invalid username or password"**.
*   *Sanitize Your Logs:* When logging errors for debugging, never log sensitive user data like plain-text passwords, API keys, credit card numbers, or even emails. Always use non-sensitive identifiers like User IDs and Correlation IDs.
