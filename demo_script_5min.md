# 5-Minute Demo Script for HealthTrack

## Pre-Demo Setup (DO THIS BEFORE PRESENTATION!)

**Critical:**
- [ ] Backend running (`npm run start:dev` in `app/api/healthtrack-backend`)
- [ ] Frontend running (`npm run dev` in `app/web`)
- [ ] Browser tab open to `http://localhost:5173` (logged out)
- [ ] Have Strava test account credentials ready
- [ ] Clear any existing demo data (optional - fresh start)
- [ ] Browser zoom at 100-125% (readable on projector)
- [ ] Close unnecessary tabs
- [ ] Database viewer ready (Prisma Studio - optional backup)

---

## 5-Minute Demo Flow

### Part 1: Authentication (1.5 minutes)

#### Step 1: Registration (45 seconds)
```
Action: Navigate to Register page
URL: http://localhost:5173/register

1. Enter email: demo@healthtrack.com
2. Enter password: test123
   → SHOW validation error: "Password must contain uppercase"

   SAY: "Notice the real-time password validation"

3. Enter password: Test123456
4. Enter first name: Demo
5. Enter last name: User
6. Click "Register"

   → Success! Redirected to dashboard

   SAY: "We're now logged in with a JWT token stored in localStorage"
```

#### Step 2: Show JWT Token (Optional - 30 seconds, skip if time tight)
```
Action: Open DevTools → Application → localStorage

1. Show auth_token
   SAY: "This JWT token is included in every request"

2. Close DevTools
```

#### Step 3: Quick Protected Endpoint Test (30 seconds, optional)
```
SKIP THIS if running tight on time
Show that without token = 401 error
```

**Running Total: 1.5 minutes**

---

### Part 2: Strava Connection (2 minutes)

#### Step 1: Navigate to Integrations (15 seconds)
```
Action: Click "Integrations" or "Settings" in navigation

SAY: "Let's connect a Strava account to automatically sync fitness data"

Show: Three provider cards (Strava, Fitbit, Lose It)
Point out: Strava shows "Not Connected"
```

#### Step 2: Initiate OAuth Flow (15 seconds)
```
Action: Click "Connect Strava"

SAY: "This initiates the OAuth 2.0 flow. The backend generates an
authorization URL with encoded state for security."

→ Browser redirects to Strava
```

#### Step 3: Strava Authorization (45 seconds)
```
Action: On Strava page

1. Login to Strava (if not already)
   → Use your test credentials

2. Show permissions screen
   SAY: "Strava shows what permissions we're requesting -
   read activities and profile data"

3. Click "Authorize"
   SAY: "User approves, and Strava redirects back with
   an authorization code"

→ Redirect back to app
```

#### Step 4: Callback & Success (30 seconds)
```
Action: Watch the callback

URL will show: /integrations/callback?code=XXX&state=YYY

SAY: "The backend validates the state to prevent CSRF attacks,
exchanges the code for access and refresh tokens, stores them
encrypted in the database, and triggers an initial sync"

→ Redirected to integrations page
→ Strava card now shows "Connected ✓"
→ Shows "Last synced: just now"

SAY: "And we're connected! Data is already syncing in the background"
```

**Running Total: 3.5 minutes**

---

### Part 3: View Synced Data (1 minute)

#### Step 1: Navigate to Dashboard (30 seconds)
```
Action: Click "Dashboard" or "Home"

Point out:
1. Health data chart/table showing activity
2. Entries tagged with "Source: STRAVA"
3. Metrics displayed:
   - Exercise minutes
   - Calories burned
   - Distance
   - Heart rate (if available)

SAY: "This data came directly from Strava's API - no manual entry needed.
The adapter transformed Strava's activity format into our unified
health data model"
```

#### Step 2: Manual Sync (Optional - 30 seconds, skip if time tight)
```
ONLY if you have time:

Action: Go back to Integrations

1. Click "Sync Now" on Strava card
   SAY: "Users can manually trigger sync anytime"

2. Wait for completion
   → Timestamp updates

3. Back to Dashboard - show any new data
```

**Running Total: 4-4.5 minutes**

---

### Part 4: Database View (Optional - 30 seconds)

```
ONLY if you have extra time:

Action: Switch to Prisma Studio or pgAdmin

1. Show Integration table
   - Point out: userId, provider=STRAVA, accessToken (encrypted)

2. Show HealthData table
   - Filter by userId
   - Show records with source="STRAVA"
   - Point out metrics

SAY: "Here's the actual data in PostgreSQL - tokens encrypted,
health data with source tracking"
```

**Total Demo Time: 4-5 minutes**

---

## Demo Script with Narration

### Opening (15 seconds)
"Let me show you this working end-to-end. I'll register a user, connect their Strava account through OAuth, and show the automatic data sync."

### During Registration (45 seconds)
"First, I'll create a new account... Notice if I use a weak password like 'test123', it's rejected in real-time. The backend enforces strength requirements - at least 8 characters with uppercase, lowercase, and a number. With a strong password... success! We're automatically logged in with a JWT token."

### During OAuth Flow (2 minutes)
"Now let's connect Strava. When I click 'Connect Strava', the backend generates an OAuth authorization URL with an encoded state parameter for CSRF protection, and redirects to Strava's site.

[On Strava page] Strava shows exactly what permissions we're requesting - read activities and profile data. The user explicitly approves this access.

[After approval] Strava redirects back with an authorization code. The backend validates the state to prevent attacks, exchanges the code for access and refresh tokens, stores them encrypted in PostgreSQL, and triggers an initial data sync. All of this happens in under a second.

[Back on app] And we're connected! The integration is active and data is already synced."

### Viewing Data (1 minute)
"On the dashboard, you can see the fitness data that just synced from Strava - exercise minutes, calories burned, distance. Each entry is tagged with the source, so users know it came from Strava, not manual entry.

The Strava adapter fetched activities from their API, grouped them by date, aggregated the metrics, transformed them into our unified format, and saved to the database. Users get automatic data import with zero manual work."

### Closing (15 seconds)
"That's the complete flow - from authentication to OAuth integration to automatic data sync. Any questions about the implementation?"

---

## Timing Breakdown

| Section | Time | Cumulative |
|---------|------|------------|
| Registration + Login | 1:00 | 1:00 |
| Show JWT (optional) | 0:30 | 1:30 |
| Navigate to Integrations | 0:15 | 1:45 |
| Click Connect | 0:15 | 2:00 |
| Strava Authorization | 0:45 | 2:45 |
| Callback & Success | 0:30 | 3:15 |
| View Dashboard Data | 0:30 | 3:45 |
| Manual Sync (optional) | 0:30 | 4:15 |
| Database View (optional) | 0:30 | 4:45 |
| Buffer | 0:15 | 5:00 |

---

## If Demo Fails - Backup Plan

### Backend/Frontend Not Running
**Backup:** Use screenshots of each step
- Have screenshots ready of: registration, OAuth, connected state, dashboard with data
- Walk through screenshots explaining what would happen

### OAuth Fails
**Backup:** Use Postman to show API calls
```
1. POST /users/register → show JWT response
2. POST /integrations/connect → show authUrl response
3. GET /integrations → show connected integration
4. Show database directly
```

### Internet Issues
**Backup:**
- Show local authentication (doesn't need internet)
- Show code in IDE instead
- Walk through architecture diagram

---

## Pro Tips for Smooth Demo

**Before Starting:**
1. Practice the demo 3-4 times
2. Know your Strava login by heart
3. Have a backup tab ready
4. Clear browser history (fast autocomplete)

**During Demo:**
1. **Narrate everything** - say what you're doing as you do it
2. **Slow down** - It feels slow to you, but right pace for audience
3. **Point with cursor** - Highlight what you're showing
4. **Pause for loading** - Don't rush, let things load
5. **If something breaks** - Stay calm, switch to backup

**Common Mistakes to Avoid:**
- ❌ Going too fast (most common!)
- ❌ Not explaining what's happening
- ❌ Assuming audience sees what you see
- ❌ Panicking when something loads slowly
- ❌ Skipping validation error demo (it's impressive!)

**Energy Points:**
- Show excitement when password validation works
- "And we're in!" after successful registration
- "Watch this..." before OAuth redirect
- "Pretty cool, right?" after seeing synced data

---

## Condensed 3-Minute Version (If Needed)

If you need to shorten further:

**1. Registration (45 sec)**
- Show password validation error → success

**2. Connect Strava (1:30 min)**
- Click connect → OAuth page → Approve → Connected

**3. Show Data (45 sec)**
- Dashboard with Strava data tagged

Skip:
- Showing JWT token
- Manual sync
- Database view

---

## Questions You Might Get During Demo

**Q: What if token expires?**
A: "Great question - the backend automatically refreshes tokens before they expire. Users never see errors."

**Q: What if Strava API is down?**
A: "The integration status would show ERROR with a message. Users can retry later. Historical data remains safe."

**Q: Can you show the database?**
A: [Switch to Prisma Studio] "Sure! Here's the Integration table with encrypted tokens, and HealthData with the synced metrics."

**Q: What about Fitbit?**
A: "Same flow! Fitbit makes multiple API calls per date - activity, sleep, heart rate, weight - using Promise.allSettled for resilience."

---

## Technical Details to Mention (If Asked)

**During Registration:**
- "Password hashed with bcrypt, 10 salt rounds"
- "JWT signed with secret from environment variables"
- "Token expires in 24 hours"

**During OAuth:**
- "State parameter encodes userId, provider, timestamp, and nonce"
- "Validated on callback to prevent CSRF"
- "Tokens stored encrypted in JSON field"

**During Data View:**
- "Adapter pattern transforms Strava format to unified model"
- "Single database schema works for all providers"
- "Duplicate prevention via unique constraints"

---

## Post-Demo Transition

After demo, transition smoothly:

**If went well:**
"That's the complete OAuth flow working end-to-end. As you can see, it's secure, seamless, and automatic. Any questions about what we just saw or the implementation?"

**If had issues:**
"Even with the [connection issue], you can see the core flow - OAuth authorization, token exchange, and data sync. The actual implementation handles errors gracefully. Any questions?"

**Always end with:**
"I'm happy to show specific code, explain the architecture in more detail, or answer any questions about security, scalability, or the implementation approach."

---

## Final Checklist Before Demo

- [ ] Services running and verified
- [ ] Test account credentials memorized
- [ ] Browser at correct URL, logged out
- [ ] Zoom level readable (100-125%)
- [ ] Unnecessary tabs closed
- [ ] Strava test account works (verify login)
- [ ] Practiced narration 2-3 times
- [ ] Backup screenshots ready
- [ ] Know where to find code (if asked)
- [ ] Water nearby
- [ ] Confident mindset! 💪

Good luck! 🚀