# Q&A Preparation - Authentication & Integrations

## Your Implementation-Specific Questions & Answers

---

## AUTHENTICATION QUESTIONS

### Q1: Why did you choose JWT over session-based authentication?

**Answer:**
"JWT is stateless and scalable. Unlike session-based auth where you store session data on the server, JWT tokens contain all the user information needed for authentication. This means no database lookups on every request, no session storage to manage, and it scales horizontally easily. It's also perfect for modern single-page applications like our React frontend."

**Follow-up if asked:** "Plus, JWTs work great with mobile apps and third-party integrations since the token can be passed around independently."

---

### Q2: How do you prevent JWT token theft?

**Answer:**
"Several ways. First, we use HTTPS in production so tokens are encrypted in transit. Second, tokens expire after 24 hours, limiting the window of vulnerability. Third, we store tokens in localStorage but could use httpOnly cookies for extra security. Fourth, the secret key for signing tokens is stored in environment variables, never in code. And we validate the signature on every request, so forged tokens are rejected."

**Short version:** "HTTPS encryption, 24-hour expiration, secret key in environment variables, and signature validation on every request."

---

### Q3: What happens if someone steals a JWT token?

**Answer:**
"That's a valid concern. If a token is stolen, the attacker has access until it expires - that's why we set a 24-hour expiration. For additional security, we could implement token refresh patterns with short-lived access tokens and longer-lived refresh tokens. We could also maintain a token blacklist for logout functionality. But the 24-hour expiration limits the damage window significantly."

---

### Q4: Why bcrypt instead of other hashing algorithms?

**Answer:**
"Bcrypt is specifically designed for password hashing. It's intentionally slow - we use 10 salt rounds which means 2^10 iterations - making brute-force attacks impractical. It automatically handles salting, so no two users with the same password get the same hash. And it's an industry standard that's stood the test of time. Alternatives like SHA-256 are too fast for passwords - attackers could try billions of combinations per second."

**Short version:** "Bcrypt is designed for passwords with built-in salting and computational cost that defeats brute-force attacks."

---

### Q5: How do you validate password strength?

**Answer:**
"We enforce several rules: minimum 8 characters, at least one uppercase letter, one lowercase letter, and one number. This is implemented in the UserService using regular expressions. The validation happens both on the frontend for user experience and on the backend for security - never trust client-side validation alone."

**If asked about special characters:** "We could add special character requirements, but research shows length and complexity balance is more important than requiring symbols."

---

### Q6: Is the Google OAuth implementation complete?

**Answer:**
"About 70% complete. The infrastructure is fully built - database schema supports Google IDs, user model allows OAuth users without passwords, all the service methods exist, the endpoint is created. What's missing is the Google token verification library integration. Someone needs to implement the Passport Google Strategy, verify the ID token from Google, extract the user profile, and wire it to the existing loginWithGoogle service method. It's maybe 2-4 hours of work."

**Short version:** "Infrastructure is complete, token verification pending. It's about 70% done."

---

## OAUTH/INTEGRATION QUESTIONS

### Q7: How does the OAuth flow work?

**Answer:**
"Four main steps. First, user clicks 'Connect Strava' and we generate an authorization URL with our client ID and a secure state parameter. Second, user is redirected to Strava where they approve permissions. Third, Strava redirects back with an authorization code. Fourth, we exchange that code for access and refresh tokens, store them encrypted, and trigger an initial data sync. The user never sees the tokens - it's all handled in the background."

**With diagram:** [Draw flow: User → Our App → Strava → User Approves → Callback → Token Exchange → Store & Sync]

---

### Q8: What's the state parameter and why is it important?

**Answer:**
"The state parameter prevents CSRF attacks. We generate a JSON object containing the user ID, provider name, timestamp, and a random nonce, then encode it as Base64 and send it to the provider. The provider returns it unchanged. On callback, we validate that the state is properly formatted, less than 10 minutes old, and matches the current user. This prevents attackers from tricking users into connecting their accounts to someone else's HealthTrack account."

**Short version:** "It's CSRF protection. We encode user context into it and validate everything on callback to prevent attack vectors."

---

### Q9: How do you handle expired OAuth tokens?

**Answer:**
"Completely automatically. Every time we're about to sync data, we check if the access token is expired or expiring within 5 minutes. If it is, we use the refresh token to get a new access token from the provider's refresh endpoint. We update the database with new credentials and proceed with the sync. Users never see token expiration errors - it's totally transparent."

**If asked about refresh token expiration:** "Refresh tokens usually last months or years. If one expires, we'd update the integration status to EXPIRED and prompt the user to reconnect."

---

### Q10: Why did you use the adapter pattern for integrations?

**Answer:**
"Three reasons. First, extensibility - adding a new provider means creating one new adapter class, no changes to existing code. Second, each provider has different APIs and data formats. Strava returns activities, Fitbit has four separate endpoints. The adapter pattern isolates those differences. Third, testability - we can mock the adapter interface for unit tests without hitting real APIs. It follows the Open/Closed principle - open for extension, closed for modification."

**With example:** "For example, when we added Fitbit after Strava, we didn't touch a single line of existing service code. Just created FitbitAdapter implementing the same five methods."

---

### Q11: What are the five methods in the adapter interface?

**Answer:**
"Every adapter implements IHealthDataProvider with these five:
1. generateAuthUrl() - Creates the OAuth authorization URL
2. exchangeCodeForToken() - Exchanges authorization code for tokens
3. fetchHealthData() - Retrieves user's health data for a date range
4. refreshAccessToken() - Gets new access token using refresh token
5. revokeAccess() - Revokes tokens when user disconnects

Each provider implements these differently, but the service layer doesn't care - it just calls the interface methods."

---

### Q12: How is Strava integration different from Fitbit?

**Answer:**
"The key difference is API structure. Strava has one endpoint that returns all activities - I fetch them, group by date, and aggregate metrics like exercise minutes and calories. It's exercise-focused.

Fitbit requires four separate API calls per date - one for activity data, one for sleep, one for heart rate, one for weight. I make these in parallel using Promise.allSettled for resilience. If sleep data isn't available, the other three still save. Fitbit also has stricter rate limits, so I added 100ms delays between dates.

Strava uses client credentials for token exchange. Fitbit uses Basic Auth with base64-encoded client ID and secret. Despite these differences, both implement the same five adapter methods."

---

### Q13: What data do you sync from each provider?

**Answer:**
"From Strava: exercise minutes, calories burned, distance, and heart rate from activities.

From Fitbit: steps, active minutes, calories, sleep minutes, resting heart rate, weight, and distance.

All of this gets transformed into our unified ExternalHealthData format with nine possible fields: steps, weight, calories burned, exercise minutes, sleep minutes, heart rate, distance, active minutes, and resting heart rate. The database doesn't care which provider it came from."

---

### Q14: How do you prevent duplicate data syncs?

**Answer:**
"Database unique constraint on three fields: userId, date, and source. If we try to insert data for user 123, date 2025-12-07, source STRAVA, and it already exists, the database rejects it. This prevents duplicate records when users manually sync multiple times. If they want to force resync - maybe Strava updated their data - we could delete existing records first then insert fresh data."

---

### Q15: What happens if Strava's API is down during sync?

**Answer:**
"The integration status would change to ERROR and we'd store the error message in the syncErrorMessage field. The user would see an error in the UI with a message like 'Unable to sync. Try again later.' Their existing data remains safe - we don't delete anything. They can retry when Strava is back up. We could enhance this with retry logic with exponential backoff."

---

## SECURITY QUESTIONS

### Q16: How do you store OAuth tokens securely?

**Answer:**
"OAuth credentials are stored in a JSON field in PostgreSQL in the Integration table. In production, you'd want database-level encryption. The tokens themselves are random strings generated by the providers, not passwords. We also set appropriate database permissions so only the application can access them. And we never return tokens in API responses - they're strictly backend-only."

**If asked about encryption at rest:** "For production, we'd implement database encryption at rest or use a secrets management service like AWS Secrets Manager or HashiCorp Vault."

---

### Q17: Could an attacker access someone else's Strava data?

**Answer:**
"No, because of several protections. First, OAuth state parameter validation ensures the user who initiated the connection is the same one completing it - prevents CSRF. Second, the integration record is tied to the authenticated user's ID from the JWT token. Third, when fetching data, we only query integrations where userId matches the authenticated user. Fourth, Strava's authorization requires the actual user to approve - you can't connect someone else's account without their Strava login."

---

### Q18: What about XSS or SQL injection vulnerabilities?

**Answer:**
"For XSS prevention, we sanitize user inputs using the sanitize-html library before storing in the database. And React escapes output by default.

For SQL injection, we use Prisma ORM which automatically uses prepared statements. We never concatenate SQL strings. All database queries go through Prisma's type-safe query builder, so injection isn't possible.

We also validate all inputs using class-validator decorators on our DTOs, checking types, formats, and ranges before data even reaches the database layer."

---

### Q19: How do you handle rate limiting from external APIs?

**Answer:**
"For Fitbit specifically, I implemented 100ms delays between each date iteration to avoid hitting rate limits. I also use Promise.allSettled instead of Promise.all, so if one API call fails due to rate limiting, the others still succeed. The system detects 429 responses and logs warnings.

For production, we could implement more sophisticated strategies like exponential backoff, request queuing, or caching recent data. We could also track rate limits in Redis and throttle requests proactively."

---

## TECHNICAL IMPLEMENTATION QUESTIONS

### Q20: Walk me through your code structure.

**Answer:**
"I follow NestJS layered architecture. Controllers handle HTTP requests and responses. Services contain business logic like user registration or data syncing. Repositories abstract database operations using Prisma. Adapters handle external API calls to Strava and Fitbit.

For example, when a user registers: UserController receives the request, validates the DTO, calls UserService.register(), which validates the password, hashes it with bcrypt, calls UserRepository.create() to save to the database, generates a JWT token, and returns it to the controller.

Each layer has a single responsibility and depends on interfaces, not concrete implementations. This makes testing easy - we can mock repositories and adapters."

---

### Q21: How did you test this?

**Answer:**
"Unit tests for services by mocking repository interfaces - no real database needed. Integration tests for the full flow using a test database. For OAuth, we'd mock the external APIs since we can't rely on Strava being available during tests.

The dependency injection pattern makes everything mockable. For example, to test IntegrationService, we inject a mock HealthDataProviderFactory that returns a mock adapter with predefined responses. We can test token refresh logic, error handling, data transformation, all without hitting real APIs."

**If asked for specifics:** "I didn't write comprehensive tests due to time constraints, but the architecture is test-ready."

---

### Q22: How does dependency injection work in your implementation?

**Answer:**
"NestJS has a built-in dependency injection container. I define interfaces like IUserRepository and IHealthDataProvider. Then in the infrastructure module, I register concrete implementations - UserRepository implements IUserRepository, StravaAdapter implements IHealthDataProvider.

In service classes, I inject these through the constructor using the @Inject decorator. For example, UserService constructor takes IUserRepository. At runtime, NestJS automatically provides the concrete UserRepository instance.

This decouples services from implementations. If I want to swap from Prisma to TypeORM, I just create a new repository implementation and update the module registration. Service code doesn't change."

---

### Q23: Why did you choose NestJS over Express?

**Answer:**
"NestJS provides structure and features that raw Express doesn't. Built-in dependency injection, decorators for clean code, modules for organization, guards for authentication, pipes for validation, and it's TypeScript-first.

For authentication, I used @Injectable, @UseGuards, and custom decorators like @Public(). For integrations, the module system helped organize adapters and factories. And the built-in JWT module simplified token generation and validation.

Express is more flexible but requires building all this structure yourself. For a team project with multiple contributors, NestJS's opinions and structure help maintain consistency."

---

### Q24: Explain your database schema for integrations.

**Answer:**
"The Integration model has: id, userId (foreign key to User), provider enum (STRAVA/FITBIT/LOSE_IT), accessToken, refreshToken, expiresAt timestamp, credentials JSON field for full OAuth response, isActive boolean, status enum (ACTIVE/EXPIRED/REVOKED/ERROR), lastSyncedAt, syncErrorMessage, and timestamps.

There's a unique constraint on userId and provider - users can only connect one account per provider. There's an index on userId and isActive for fast queries.

The HealthData model stores synced metrics with a source field tagging STRAVA, FITBIT, or MANUAL, and a unique constraint on userId, date, and source to prevent duplicates."

---

### Q25: How does the factory pattern work in your code?

**Answer:**
"The HealthDataProviderFactory class has a getProvider() method that takes a provider enum. Inside, it's a switch statement: if STRAVA, return the StravaAdapter instance; if FITBIT, return FitbitAdapter; etc.

The factory is registered as a singleton in the infrastructure module and injected into IntegrationService. When syncing, the service calls factory.getProvider(integration.provider) and gets back the correct adapter without knowing which one.

This centralizes instantiation logic. If we need to add configuration or logging to adapter creation, we do it once in the factory. And it makes testing easy - we can inject a mock factory."

---

## ARCHITECTURE/DESIGN QUESTIONS

### Q26: What design patterns did you use?

**Answer:**
"Five main patterns. Adapter Pattern for provider integrations - each implements IHealthDataProvider. Repository Pattern for data access - services depend on repository interfaces. Factory Pattern for creating adapters and strategies. Guard Pattern for authentication - JwtAuthGuard protects routes. And DTO Pattern for data validation and transfer between layers.

These aren't just buzzwords - each solves a specific problem. Adapters isolate provider differences. Repositories make the database swappable. Factory centralizes creation. Guards handle cross-cutting auth logic. DTOs enforce validation."

---

### Q27: How would you add a new provider like Garmin?

**Answer:**
"Four steps, takes maybe 2-4 hours. First, create GarminAdapter class implementing IHealthDataProvider with the five methods. Second, register Garmin in the provider enum and add credentials to environment variables. Third, register GarminAdapter in the infrastructure module. Fourth, add GARMIN case in the factory's getProvider() method.

That's it. No changes to services, controllers, or database schema. The unified data model already handles all health metrics. The factory returns the new adapter. The service layer doesn't know or care that Garmin was added. That's the power of the adapter pattern."

---

### Q28: What would you do differently if you rebuilt this?

**Answer:**
"Three things. First, I'd complete the Google OAuth implementation - finish the token verification. Second, I'd implement webhook listeners instead of manual sync. Strava and Fitbit support webhooks that notify us when new data is available, enabling real-time sync instead of polling. Third, I'd add more comprehensive error handling with retry logic using exponential backoff for transient API failures.

Maybe also implement the OAuth PKCE flow for enhanced security and add request/response logging for debugging. But overall, the architecture is solid."

---

### Q29: How does this scale to thousands of users?

**Answer:**
"The architecture is inherently scalable. JWT is stateless - no server-side session storage, so we can scale horizontally across multiple servers. Each integration is independent - syncing user A doesn't affect user B.

For production at scale, we'd add: Redis for token caching to reduce database lookups, a message queue like RabbitMQ or Bull for background sync jobs instead of synchronous API calls, database connection pooling which Prisma provides, and possibly a CDN for static assets.

The adapter pattern helps too - we could shard providers across different servers if one provider becomes a bottleneck."

---

### Q30: What about database performance with lots of health data?

**Answer:**
"We have indexes on userId and date for fast queries. The unique constraint on userId, date, and source also creates an index. For dashboards, we'd typically query a date range like the last 30 days, which is fast with proper indexing.

For larger scale, we could implement: pagination for API responses, data aggregation tables for common queries, archiving old data to cold storage after a year, or time-series database optimization. Prisma's query builder is efficient and uses connection pooling. But with proper indexes, PostgreSQL handles millions of health data records easily."

---

## CHALLENGE/DEBUGGING QUESTIONS

### Q31: What was the hardest part of this implementation?

**Answer:**
"The Fitbit integration was challenging because it requires four separate API calls per date, each with different response formats. I had to figure out how to make parallel calls efficiently without hitting rate limits, handle partial failures gracefully, and transform four different JSON structures into our unified model.

I solved it with Promise.allSettled for resilience - if sleep data fails, we still save activity data. Added 100ms delays to respect rate limits. And wrote transformation functions for each endpoint.

OAuth state security was also tricky - encoding the right information, validating it properly, handling expiration edge cases."

---

### Q32: How did you debug OAuth issues?

**Answer:**
"Lots of logging. I logged the authorization URL being generated, the state parameter before encoding, the callback parameters, the token exchange request and response. When something failed, I could trace exactly where.

I also used Strava's OAuth playground to test the flow manually and understand their API responses. For token refresh, I artificially expired tokens in the database to test the refresh logic without waiting 24 hours.

And I used Postman to test API endpoints independently before integrating them into the full flow."

---

### Q33: Did you encounter any API documentation issues?

**Answer:**
"Yes. Fitbit's documentation isn't always clear about error responses or rate limits. I had to experiment to figure out that their token endpoint requires Basic Auth in a specific format: base64(clientId:clientSecret). Their docs mentioned it but not prominently.

Strava's documentation is better, but their activity data structure has many optional fields. I had to handle cases where average heart rate doesn't exist for activities without a heart rate monitor.

That's why the adapter pattern is valuable - all those provider quirks are isolated in one place."

---

### Q34: How do you handle timezone differences?

**Answer:**
"Good question. We store dates as DateTime in PostgreSQL which includes timezone information. Strava returns timestamps in ISO 8601 format with timezone. Fitbit's API uses the user's profile timezone.

When transforming data, I normalize everything to UTC in the database. The frontend converts to the user's local timezone for display. This prevents issues where a midnight run shows up on the wrong date.

For date ranges in API queries, I ensure we're comparing dates in the same timezone to avoid off-by-one errors."

---

### Q35: What security vulnerabilities did you consider?

**Answer:**
"Several. CSRF attacks on OAuth - solved with state parameter validation. Token theft - mitigated with expiration and HTTPS. SQL injection - prevented with Prisma's prepared statements. XSS - handled with input sanitization and React's escaping. Password attacks - defeated with bcrypt's computational cost.

I also considered: timing attacks on password comparison (bcrypt.compare is timing-safe), replay attacks (state expiration), session fixation (new token on each login), and privilege escalation (userId from JWT token, not request parameters).

Security is layered - no single point of failure."

---

## FUTURE/IMPROVEMENT QUESTIONS

### Q36: What features would you add next?

**Answer:**
"Three priorities. First, complete Google OAuth - it's 70% done. Second, implement webhook listeners for Strava and Fitbit so data syncs in real-time when users complete activities, instead of manual sync. Third, add more providers - Garmin, Apple Health, Google Fit - since the adapter pattern makes this straightforward.

Other ideas: two-factor authentication for enhanced security, OAuth PKCE flow, data export functionality, activity-level detail views instead of just daily summaries, and scheduling automatic syncs at certain times."

---

### Q37: How would you implement real-time sync with webhooks?

**Answer:**
"Strava and Fitbit both support webhooks. I'd create webhook endpoints in the controller that receive POST requests when users have new data. Verify the webhook signature for security. Then queue a background job to sync that specific user's data.

We'd need a message queue like Bull for job processing, Redis for tracking processed events to prevent duplicates, and proper error handling with retries. The adapter pattern already supports fetching data - we'd just trigger it from webhooks instead of manual clicks.

This would make the user experience seamless - run tracked on Strava, data appears in HealthTrack within seconds."

---

### Q38: How would you handle user data privacy and GDPR?

**Answer:**
"Several measures. First, users explicitly authorize data access through OAuth consent screens - they see exactly what we're requesting. Second, we only fetch data they've authorized, nothing more. Third, users can disconnect integrations anytime, which revokes tokens with the provider.

For GDPR compliance, we'd implement: right to access (export all their data), right to deletion (cascade delete user and all related data), data minimization (only store necessary fields), encryption at rest, audit logging of data access, and clear privacy policy.

The DELETE /users/me endpoint already deletes the account and cascades to integrations and health data. We'd enhance it with data export before deletion."

---

---

## QUICK REFERENCE STATS

Memorize these for confidence:

- **16 API endpoints** (8 auth, 8 integrations)
- **3 providers** (Strava, Fitbit, Lose It infrastructure)
- **9 data types** (steps, weight, calories, exercise, sleep, HR, distance, active mins, resting HR)
- **5 adapter methods** (auth URL, token exchange, fetch data, refresh, revoke)
- **2 main systems** (authentication, integrations)
- **24-hour** JWT expiration
- **10 salt rounds** for bcrypt
- **10-minute** state parameter expiration
- **~3,500 lines** of code

---

## ANSWERING STRATEGY

**If you don't know:**
- "That's a great question. I'd need to research the best approach, but my initial thought is..."
- "I didn't implement that specific feature, but based on the architecture, we could..."
- "Let me think about that for a second..." [pause] "I believe..."

**If it's outside your scope:**
- "That's related to [other team member]'s work on the frontend/goals system. They could answer that better."
- "I focused on the backend auth and integrations. The dashboard visualization was [name]'s contribution."

**If you made a mistake:**
- "You're right, I should have considered that. In hindsight, I would..."
- "That's a valid concern. If I rebuilt this, I'd implement..."

**Keep answers:**
- Concise (30-60 seconds)
- Confident (you built this!)
- Honest (admit unknowns)
- Technical (show depth)
- Value-focused (explain why, not just what)

---

Good luck with Q&A! You know this inside and out. 🚀