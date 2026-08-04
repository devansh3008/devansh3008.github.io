![](attachments/Pasted%20image%2020260804014343.png)

*By [Devansh](https://devansh3008.github.io) & [Aniket](https://awwfensive.github.io)*

AI-assisted development has fundamentally changed how software is built. Founders can now go from an idea to a production-ready SaaS application in hours instead of weeks, with AI generating everything from the frontend to the database schema and deployment configuration.

That shift has also changed the security landscape. Over the past year, security researchers have repeatedly uncovered AI-generated applications exposing customer data, secrets, and business logic through the same handful of implementation mistakes. The question is no longer _whether_ AI can build software, it is whether the software it builds is secure enough for production.

To answer that question, we set out to evaluate the security posture of publicly accessible vibe-coded SaaS applications. Rather than looking for novel exploits or zero-days, the goal was to identify the security issues an attacker would realistically encounter during a routine black-box assessment.

The results were strikingly consistent. The same **10 vulnerability patterns** appeared repeatedly across different applications, industries, and founders. Every application evaluated exhibited at least one issue from this recurring set.

None of the findings required source code, privileged access, or sophisticated exploitation. They were identified using standard black-box application testing with nothing more than a browser and an intercepting proxy.

This research documents those ten recurring vulnerability patterns, why they continue to appear across AI-generated applications, and what they reveal about the current state of AI-assisted software development.

## The Stack We Kept Seeing

![](attachments/Pasted%20image%2020260803171918.png)

- **Frontend:** Next.js (App Router), React, TypeScript, Tailwind
- **Backend:** Next.js API Routes / Server Actions, middleware for auth & rate limiting
- **Data & Auth:** Supabase (Postgres, Auth, Storage, Realtime)
- **AI:** OpenAI / Anthropic / Gemini APIs
- **Integrations:** Stripe, Resend, PostHog
- **Deploy:** Vercel, GitHub, GitHub Actions

Every finding below maps to a specific layer of that stack.

---

## **Security Risks Introduced by AI Development Tools**

AI-assisted development tools such as Cursor, Claude Code, Windsurf, OpenCode, and full-stack AI app builders like Lovable, Bolt.new, and v0 significantly reduce development time by generating complete features, or entire applications, from user prompts. However, these tools optimize primarily for functional correctness rather than "secure-by-default" implementations. Unless security requirements are explicitly included in the prompt or enforced through project rules, the generated code frequently omits critical security controls.

Not every tool in that list fails the same way, and the difference comes down to how much of the stack each one actually touches:

- **IDE-embedded assistants** (Cursor, Claude Code, Windsurf, OpenCode) work inside an existing codebase, generating or modifying individual features, functions, or files at a developer's direction. The security gaps here tend to show up at the _logic_ level — a new endpoint that skips an ownership check, a form handler that trusts a client-submitted field, because the assistant is solving the prompt in front of it, not auditing the surrounding system.
- **Full-stack AI app builders** (Lovable, Bolt.new, v0) generate an entire application, frontend, backend, and often the database schema, from a single natural-language description, with little to no existing codebase to anchor against. The security gaps here tend to show up at the _architecture_ level instead: database tables created without Row-Level Security, API keys embedded directly in generated client bundles, storage buckets left public by default, because there was no existing pattern for the tool to inherit security posture from in the first place.

Both categories share the same root cause, generated code is optimized to satisfy the prompt, not to satisfy a threat model, but the blast radius differs. A missed ownership check from an IDE assistant usually affects one endpoint. A missing RLS policy from a full-stack builder usually affects every table the generated backend touches, which is why database-layer misconfigurations in tools like Lovable and Bolt.new have produced some of the more widely reported incidents in this space (more on that later in this post).

In practice, across both categories, this leads to patterns such as:

- Authorization logic being implemented only in the UI while backend endpoints remain directly accessible.
- Request bodies being written directly into ORM models without allowlisting fields, resulting in mass assignment vulnerabilities.
- Missing ownership validation on API routes, leading to Broken Object Level Authorization (BOLA/IDOR).
- Secrets accidentally embedded into frontend bundles through incorrect environment variable usage, most visibly in full-stack builders where the generated client code calls a backend service (like Supabase) directly.
- Database tables and storage buckets provisioned by full-stack builders left without Row-Level Security or access-control policies, since these are opt-in configurations rather than defaults.
- AI proxy endpoints and authentication routes being generated without rate limiting or abuse protection.
- Missing validation, logging and secure defaults because the generated implementation focuses only on making the feature work.

These issues rarely originate from a flaw in the AI tool itself. Instead, they arise because generated code is often accepted with minimal manual review, allowing insecure implementation patterns to propagate consistently across projects built using the same development workflow, and, in the case of full-stack builders, across every project built on the same underlying template.

---

## A Security Taxonomy of AI Development Tools

Lumping Cursor, Claude Code, Windsurf, OpenCode, Lovable, Bolt.new, and v0 into one "AI dev tools" bucket hides a real split. Once you look at what's actually been publicly disclosed about each, they fall into two genuinely different risk categories, not just different brand names on the same problem.

| Tool                    | Category                     | What it generates                                                         | Where the risk actually lives                                                                   | What's publicly documented                                                                                                                                                                                                                                                                                                                |
| ----------------------- | ---------------------------- | ------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Cursor**              | IDE-embedded assistant       | Code inside an existing repo                                              | The editor itself, not just its output                                                          | Tracked as CVE-2026-50548 / CVE-2026-50549 ("DuneSlide"), a zero-click prompt-injection chain that escapes the IDE's sandbox for OS-level remote code execution. A separate, earlier flaw involved Workspace Trust being disabled by default, letting a malicious repo auto-execute a task the moment it's opened, no prompt, no consent. |
| **Claude Code**         | IDE-embedded assistant       | Code inside an existing repo                                              | Its automated security-review step can itself be socially engineered                            | Checkmarx reported that a carefully worded code comment could convince Claude Code's automated review that plainly dangerous code was safe, letting insecure code pass review via prompt injection.                                                                                                                                       |
| **Windsurf / OpenCode** | IDE-embedded assistants      | Code inside an existing repo                                              | Same general class as above (logic-level omissions in generated code)                           | No independently documented CVE-level incident specific to either tool was found at the time of writing.                                                                                                                                                                                                                                  |
| **Lovable**             | Full-stack app builder       | Entire application: frontend, backend, and database schema, from a prompt | The generated _application's_ access-control layer, not the builder tool itself                 | CVE-2025-48757: a missing Row-Level-Security misconfiguration confirmed across 170+ apps. A separate 2026 report documented 16 vulnerabilities, 6 critical, in a single Lovable-hosted app that leaked 18,000+ users' data.                                                                                                               |
| **Bolt.new**            | Full-stack app builder       | Entire application: frontend, backend, and database schema, from a prompt | Same layer as Lovable, generated keys and access controls                                       | No public CVE on file, but independent scanners consistently report the same failure pair, hardcoded provider keys shipped in client bundles and RLS left off by default on the Supabase backend it generates.                                                                                                                            |
| **v0 (Vercel)**         | Frontend component generator | React/Next.js UI components, not a full backend                           | The client-side rendering and validation layer, since it doesn't generate a database or backend | Documented issues skew toward XSS via unescaped output and missing input validation. Vercel's own engineering blog states it blocked 17,000+ deployments in a 30-day period specifically for exposed secrets.                                                                                                                             |

The distinction between these tools comes down to **where failures propagate**.

For **IDE assistants** such as Cursor, Claude Code, Windsurf, and OpenCode, the primary risk is the developer’s environment. A compromised assistant or unsafe suggestion can affect local code, secrets, or the development workflow.

For **full-stack builders** such as Lovable, Bolt.new, and v0, the risk shifts to the deployed application. Insecure generated code or backend configuration can expose customer data, business logic, or authentication systems.

Neither category is inherently less secure—they simply present different attack surfaces. This is why the findings in this post focus on Lovable-, Bolt.new-, and v0-style tools: the research evaluates **deployed applications**, not IDE sessions.

Because these applications consistently converged on the same technology stack, we stopped treating each as a unique target and instead applied the same security checklist across every assessment. No source code, no privileged access, and no custom tooling—just black-box testing with an intercepting proxy, a free-tier account, and requests beyond what the UI exposed.

The remainder of this post breaks down the ten recurring vulnerability patterns that emerged from that process.

---

## How We Tested Each Layer of the Stack

Since the apps we looked at were all built on more or less the same stack, we stopped treating each one as a fresh target and started treating it as the same research checklist, run six times, once per layer. No source access, no special credentials, just a proxy, a free-tier signup, and the willingness to send requests the UI never intended to send. Here's the full breakdown, layer by layer.

### Frontend : Next.js (App Router), React, TypeScript, Tailwind

The App Router blurs the line between "what the server computed" and "what the browser needs," and that blur is exactly where over-fetching lives.

**What we tested:**

- Clicked through every feature once with Burp/Caido sitting in the background, logging every request, then diffed the raw network response against what the UI actually rendered
- Paid close attention to React Server Component payloads specifically, since RSC streams serialize the full data object passed to the component tree, not just the props the JSX consumes
- Checked `/api/*` routes hit directly by client components vs. Server Actions invoked via form submission, since the two often have different (and inconsistently applied) validation
- Harvested every extra field visible in any response (`role`, `credits`, `isVerified`, `internalNotes`) into a running list, for reuse as mass-assignment candidates against the backend later
- Checked client bundles for feature flags or admin routes referenced in code but not linked in the UI, dead giveaways of functionality that's "hidden" rather than actually authorized

**Typically vulnerable to:** Broken Object Property Level Authorization. The frontend rarely filters what the backend sends it, so any over-fetched field just gets discarded silently client-side, and any client-only "hidden" route is trivially reachable by anyone who reads the bundle.

**Reference:** This category merges the older Excessive Data Exposure and Mass Assignment classes in the 2023 OWASP API Security Top 10, since both stem from a lack of authorization validation at the object property level. [OWASP](https://owasp.org/www-project-api-security/)

---

### Backend : Next.js API Routes / Server Actions, middleware

This is where authorization is supposed to actually happen, and where "the demo works" most often substitutes for "the request is checked."

**What we tested:**

- Replayed captured `PATCH`/`POST` requests padded with extra fields the UI never exposed, one field at a time, to isolate exactly which ones the ORM `update()` call would silently accept
- Swapped resource IDs (`orderId`, `documentId`, `invoiceId`) for sequential neighbors and for IDs leaked elsewhere in the app (support tickets, invite links, public review author fields)
- Checked whether Server Actions validated the caller's session independently, or just trusted that the action was only reachable from an authenticated page (it usually isn't enforced, Server Actions are still just POST endpoints under the hood)
- Burst-tested login, signup, password reset, and any `/api/ai/*` proxy route (~200 requests in a tight loop, watching for `429`s or account lockouts)
- Looked at middleware config specifically, since Next.js middleware is where rate limiting and auth gating are meant to live, and it's the first thing skipped when a solo dev is racing to ship a feature

**Typically vulnerable to:**

- ID-swapping → Broken Object Level Authorization (BOLA)
- Field-padding → Mass Assignment
- Burst testing → Unrestricted Resource Consumption / credential stuffing

**Reference:** BOLA has been reported present in roughly 40% of all API attacks and has held the number-one spot on the OWASP API Security list since 2019. The standard classifications are CWE-639 (Authorization Bypass Through User-Controlled Key) for IDOR and CWE-915 for mass assignment. [Salt Security](https://salt.security/blog/owasp-api-security-top-10-explained)

---

### Data & Auth : Supabase (Postgres, Auth, Storage, Realtime)

Supabase auto-generates a REST API directly over Postgres, which means the database's authorization model _is_ the application's authorization model. If RLS is wrong, there is no second line of defense.

**What we tested:**

- Queried PostgREST tables directly with a captured anon key + a low-privilege JWT, bypassing the Next.js layer entirely: `GET /rest/v1/<table>?select=*`
- Tried this against every table name we could infer from API responses or client bundle strings, not just the obvious ones like `orders`
- Checked whether RLS was enabled per table at all (it's off by default per table in Supabase, not just per project), and whether existing policies were scoped to `auth.uid()` or left as permissive catch-alls
- Guessed storage object paths using the `{userId}/{filename}` convention visible in upload responses, and checked bucket-level visibility (public vs. private)
- Left Realtime/WebSocket channels open during actual usage, logged the full synced state object, then edited our own copy locally and re-sent it to see if the server recomputed the state or just accepted it as ground truth
- Where Realtime synced other players'/users' objects through the same channel, tried the same edit against someone else's record, not just our own

**Typically vulnerable to:** missing or misconfigured Row-Level Security (cross-tenant data exposure), public storage buckets, and client-authoritative real-time state.

**Reference:** Supabase's own [Row Level Security documentation](https://supabase.com/docs/guides/database/postgres/row-level-security) is explicit that RLS has to be enabled and scoped per table, it isn't a project-wide default and won't retroactively apply to new tables created later in the project's life.

---

### AI : OpenAI / Anthropic / Gemini APIs

LLM calls are metered, per-token infrastructure bolted onto apps that otherwise treat "backend" as free. Nobody budgets for what an unthrottled proxy route actually costs at scale.

**What we tested:**

- Grepped shipped JS bundles (`_next/static/chunks/*.js`) for key-shaped strings matching provider prefixes, to catch client-side calls using credentials that should never have left the server
- Burst-tested `/api/ai/*` or equivalent proxy routes the same way as auth endpoints, unauthenticated and authenticated, to see if per-user or per-IP throttling existed
- Checked whether the proxy route validated the caller's plan/tier before forwarding the request, or just passed every authenticated request through regardless of subscription status
- Looked at whether prompt construction on the server concatenated user input directly into system-level instructions in a way that could be manipulated to change the app's own behavior (prompt injection into the app's own control flow, not just output quality)

**Typically vulnerable to:** uncapped cost exposure via unthrottled proxy routes. Less a classic OWASP category, more a cost-abuse vector specific to LLM-backed products, an open proxy route is an unattended tab running at someone else's bar, and it doesn't show up as a "hack" in any log, just a bill.

---

### Integrations : Stripe, Resend, PostHog

**Why this layer:** this is the layer where a security bug stops being a security bug and starts being an accounting discrepancy.

**What we tested:**

- Located webhook endpoints (`/api/webhooks/stripe`, `/api/webhooks/resend`) via client bundle strings or predictable naming
- Posted hand-crafted events (`checkout.session.completed`, `subscription.updated`, `invoice.paid`) with no `Stripe-Signature` header, to see if the handler trusted the raw JSON body before checking anything
- Checked whether the handler, if it did verify signatures, also correctly matched the event's customer/email against the account being upgraded, rather than trusting whatever email was in the forged payload
- Checked whether Resend or PostHog keys leaked into client-side bundles where they had no business being, since both are commonly initialized with a single shared key rather than separate public/private ones

**Typically vulnerable to:** missing webhook signature verification, a direct path to fraudulent "paid" account state with zero dollars actually collected.

**Reference:** Stripe's own [webhook documentation](https://docs.stripe.com/webhooks) is explicit that signature verification via the `Stripe-Signature` header and webhook secret is the only thing distinguishing a real event from anyone who has discovered the endpoint URL.

---

### Deploy : Vercel, GitHub, GitHub Actions

This is where the app's infrastructure identity lives, and it's the layer most founders assume is "someone else's problem" because Vercel manages the hosting.

**What we tested:**

- Pulled apart deployed `_next/static` bundles for hardcoded secrets using a simple grep for key-shaped strings
- Checked whether `.env` values were correctly scoped, `NEXT_PUBLIC_`-prefixed variables ship to the browser by design, and it's common to see a variable prefixed that way out of habit when it should never have left the server
- Pointed any "fetch this URL" feature (webhook testers, PDF/screenshot generators, link-preview generators, "import from URL") at `169.254.169.254` and a handful of internal-looking hostnames and RFC 1918 ranges
- Checked whether GitHub Actions workflows (where visible, e.g. on public repos or via leaked CI logs) exposed any secrets in build logs or artifact outputs

**Typically vulnerable to:** Server-Side Request Forgery (SSRF), classified as CWE-918, particularly dangerous on cloud-hosted deployments because the same request path used for a legitimate feature can just as easily reach the instance metadata service and pull IAM credentials, turning an app-level bug into a full cloud-account compromise.

---

None of this needed custom tooling or source access, just the same six checklists, run in the same order, against every target, adjusted only for the framework-specific quirks noted above.

So yes, we ran this exact process across the full pool of applications, and these were the patterns that kept showing up, again and again, in the order we tend to hit them:

## Fingerprinting the Stack

One thing became obvious early in the research: most targets shared the same underlying architecture. Manually confirming the stack and checking for the same recurring misconfigurations quickly became repetitive, so we built `vibe_stack_scanner.py` to automate the reconnaissance.

Given a target URL, the scanner fingerprints the application stack and performs a series of passive security checks in seconds. Passive mode uses only standard **GET** requests—the same traffic generated by a browser while loading the application—making it suitable for safely identifying common security signals without modifying application state.

The scanner automates the repetitive parts of the assessment, allowing manual effort to focus on validating findings and investigating application-specific behavior.

**What it checks, and how much of the assessment is automated:**

|Check|What it does|Coverage|
|---|---|---|
|Stack fingerprint|Detects Next.js, Vercel, Supabase, Stripe from headers/HTML|4 signals|
|Secret scan|Greps every shipped `/_next/static/*.js` bundle|7 secret patterns (OpenAI, Anthropic, Stripe live/restricted, Supabase service-role JWT, Google API key, generic secret regex)|
|RLS probe|Hits `/rest/v1/<table>` with the public anon key|14 common table names|
|Public bucket probe|Checks `/storage/v1/object/public/<bucket>/`|6 common bucket names|
|Over-fetch scan|Flags sensitive field names in any JSON response|14 field names (password, role, is_admin, token, etc.)|
|Rate limiting _(active, opt-in)_|Bursts a login/reset endpoint, checks for a 429|configurable request count|
|Webhook signature _(active, opt-in)_|Sends one synthetic, unsigned event|checks for improper 2xx acceptance|
|SSRF _(active, opt-in)_|Fires canary URLs at any "fetch this URL" feature|2 canaries (cloud metadata, localhost)|
|IDOR _(active, opt-in, your own session)_|Requests a list of IDs, flags any 200|as many IDs as you supply|
|Mass assignment _(active, opt-in, your own session)_|PATCHes one extra field at a time|as many fields as you supply|

That's 5 checks that run automatically and passively on any target, plus 5 more that are gated behind `--active` and, for the last two, require your own authenticated session, because they send state-changing or unusual traffic. All ten map one-to-one to the findings below.

Because the majority of applications followed nearly identical architectural patterns, this fingerprinting step significantly reduced the time spent on initial reconnaissance. Once the stack was identified, we could apply the same structured testing methodology to each layer instead of treating every application as an entirely new target. The tool acted as a starting point of the research; the findings discussed throughout this article were ultimately confirmed through manual verification.

Vibe-stack-scanner Tool: `<link here>`

---

After stack confirmation and manually identifying vulnerabilities, these were the patterns that kept recurring across the applications we looked at:

## 1. Over-Fetching: APIs That Return the Whole Database Row

The first thing we do on any Next.js target is read the raw network responses, not the rendered page — React Server Components serialize way more than the UI displays. On a review page that only rendered a username and avatar, the actual response looked like this:

```json
"user": {
    "id": "...",
    "username": "...",
    "email": "...",
    "birthday": "...",
    "role": "USER",
    "passwordHash": "$2b$...",
    ...
}
```

The frontend needed two fields. The API shipped the entire `User` model, bcrypt hash and all. This happens because the handler is `return NextResponse.json(user)` on the raw Prisma/Supabase object instead of a mapped DTO, the fastest way to get a feature working, and the AI agent's default unless told otherwise.

ref: https://specopssoft.com/blog/hashing-algorithm-cracking-bcrypt-passwords/

**Why it matters:** Every extra field is free reconnaissance, internal IDs for IDOR testing, roles for privilege-escalation targeting, timestamps for account enumeration, and in the worst case, credential material that should never leave the server.

**Business impact:** Password hashes and PII in API responses are a reportable data exposure under GDPR/CCPA the moment they're discovered, regardless of whether anyone actually cracked a hash. That's a mandatory disclosure conversation with customers and, for regulated verticals, a regulator. It also fails outright on any SOC 2 or vendor-security questionnaire an enterprise buyer sends before signing a contract.

**Fix:** never `res.json(model)`. Map to an explicit response DTO per endpoint and allowlist outbound fields.

## 2. Mass Assignment: Endpoints That Accept Whatever You Send

Once we see an API over-returning data, we test whether it also over-accepts it. On one app, the profile update endpoint looked ordinary:

```http
PATCH /api/users/me
{
    "username": "newuser",
    "birthday": null,
    "socialLink": null
}
```

We added a field the UI never exposed:

```http
PATCH /api/users/me
{
    "username": "newuser",
    "birthday": null,
    "socialLink": null,
    "profilePicUrl": "https://example.com/image.png"
}
```

Request succeeded. Avatar changed instantly, no dedicated endpoint, no UI control for it. The handler was passing `req.body` straight into an ORM `update()` call, so any field matching the model's schema got persisted, whether or not the frontend ever intended to expose it.

Chained with an upload endpoint that returned a **public** blob URL, the full path to persistent, unauthorized state modification was, upload any file → grab its public URL → smuggle it into `profilePicUrl` → done. On a different endpoint, the same pattern could just as easily hit `role`, `isVerified`, or `subscriptionTier`, the test is identical, only the field name changes.

**Business impact:** The same mechanism that flips an avatar flips a billing tier. If `subscriptionTier` or `credits` lives on the same model, this is a direct path to unauthorized feature/paid-tier access, lost revenue that's invisible in your metrics because the "upgrade" never touched Stripe. It's also the kind of bug that, once posted publicly, gets every free-tier user in your database self-upgrading within hours.

**Fix:** validate every request body against an explicit schema (Zod/Yup) that allowlists exactly the fields that endpoint accepts. Never spread a raw body into an ORM write.

## 3. Trusting Client-Reported State in Real-Time Apps

The mass-assignment pattern above shows up in an even more dangerous form in real-time apps. On a multiplayer quiz game built on Firebase, we watched the WebSocket traffic right after joining a lobby, and the server pushed down the **entire quiz dataset**, every question and every correct answer, before the round even started. The frontend just wasn't rendering it yet.

Digging into the synchronized player object, fields like `score`, `powerUps`, and `currentQuestion` weren't just readable, they were writable from the client and accepted back by the server as ground truth. Editing our own player object directly and re-syncing it let us set:

```json
{
   "score":100,
   "powerUps":[
      "shield",
      "doublePoints"
   ],
   "answered":true
}
```

![](attachments/Pasted%20image%2020260803172026.png)
...without answering a single question. Because every participant's state synced through the same unauthenticated channel, the same edit worked on **other players' records**, meaning we could hand ourselves the win, or take it away from someone else, mid-match.
![](attachments/Pasted%20image%2020260803172043.png)
**Business impact:** For any product where outcomes carry stakes, leaderboards, cash prizes, ranked matchmaking, loyalty points redeemable for real value, client-writable state is a direct fraud vector, not just a fairness bug. It's the difference between "a customer complained" and "a customer won a cash prize by editing a JSON object," which is a very different conversation with your payments processor and your legal team.

**Fix:** the server must be the sole authority over game/application state. Clients send _intents_ ("I selected option B"); the server validates and computes the resulting state, never the reverse.

## 4. Missing or Misconfigured Row-Level Security (RLS)

Supabase gives you Postgres with RLS built in, but it's **off by default** on new tables, and it's common to see it enabled with a policy like `USING (true)` just to unblock a feature during development, then never revisited. We test this by hitting the auto-generated **PostgREST** endpoint directly with a low-privilege session token and querying tables that should be scoped to the current user.

```http
GET /rest/v1/orders?select=*
apikey: <very secret anon key>
Authorization: Bearer <any authenticated user's JWT>
```

If RLS isn't enforced, this returns every row in the table, not just the caller's own.

**Business impact:** This is the difference between a single-tenant incident and a full-tenant data breach. A missing RLS policy on an `orders` or `documents` table means every customer's data is exposed to every other customer with a login, the exact scenario cyber-insurance underwriters and enterprise security reviews are built to catch, and the exact scenario that ends multi-tenant SaaS contracts overnight.

**Fix:** default-deny. Enable RLS on every table and scope every policy to `auth.uid()` explicitly.

## 5. IDOR in API Routes and Server Actions

Classic, and still everywhere: `/api/orders/[id]` or a server action `getOrder(orderId)` fetches by primary key with no ownership check. We test this by taking any resource ID that legitimately belongs to us, incrementing/decrementing it, or swapping it for an ID leaked elsewhere in the app (support tickets, invite links, review authors), and replaying the request as ourselves.

**Business impact:** Sequential or guessable IDs turn a single account into a scraper for your entire customer base, invoices, support conversations, uploaded documents. For B2B products, this is precisely the finding that shows up in a prospective enterprise customer's pentest before purchase report and stalls a deal in procurement.

**Fix:** every "get/update/delete by ID" query needs a `WHERE user_id = current_user` clause, or an equivalent RLS policy, the ID alone is never sufficient authorization.

## 6. Hardcoded Secrets and Over-Privileged Service Keys

Next.js exposes any `NEXT_PUBLIC_`-prefixed variable straight to the browser bundle. Pulling apart the client JS with a quick `strings`/grep pass on the deployed bundle turns up API keys more often than it should:

```bash
curl -s https://target.app/_next/static/chunks/*.js | grep -Eo "sk-[a-zA-Z0-9]{20,}|sb_secret_[a-zA-Z0-9]+"
```

The other version of this: the backend uses the Supabase **service-role key** (which bypasses RLS entirely) as its default client, because writing correct RLS policies is slower than just skipping them. Every route built on that client inherits full database access regardless of who's calling it.

**Business impact:** A leaked OpenAI/Anthropic key is a direct, uncapped operating expense, attackers resell access to stolen LLM keys, and the bill lands on the founder's card before anyone notices. A leaked Stripe secret key or Supabase service-role key is worse: it's the difference between "we had a bug" and "an outside party had root access to our production database and payment processor," which is a mandatory incident-response and customer-notification event, not a quiet patch.

**Fix:** anything with write/admin privilege never ships to the client, and the service-role key is scoped to specific, audited server operations, never the default client for user-facing routes.

## 7. Public Storage Buckets and Unrestricted Uploads

Supabase Storage buckets are private by default, but we regularly find them flipped to **public** to sidestep a CORS/signed-URL headache during development. Combined with predictable object paths (`/uploads/{userId}/{filename}`), enumerating other users' uploaded files is trivial. Upload handlers also frequently skip MIME-type and size validation, and, per finding `#2`, happily accept any URL an upload endpoint returns as input to unrelated, sensitive fields.

**Business impact:** For any product handling ID documents, contracts, medical records, or financial statements, an increasingly common "AI features" use case, a public bucket is an unencrypted, unauthenticated leak of exactly the documents customers trusted you to protect. That's the category of incident that ends up in a breach-notification letter with your company's name on it, not a GitHub issue.

**Fix:** keep buckets private, serve objects through signed URLs scoped to their owner, validate MIME type/extension/size server-side, and never let a client-supplied URL become application state without verifying its provenance.

## 8. No Rate Limiting on Auth, Reset, or AI-Proxy Endpoints

Middleware is where rate limiting is supposed to live, and it's the layer most often skipped — it has zero visible effect while a solo dev is building the app. We test login, signup, password-reset, and any `/api/ai/*` proxy route with a simple burst:

```bash
for i in {1..200}; do curl -s -o /dev/null -w "%{http_code}\n" -X POST https://target.app/api/auth/login -d '{"email":"a@a.com","password":"guess'"$i"'"}'; done
```

No `429s` means credential stuffing is unthrottled, and on AI-proxy routes, it means anyone can run up the app owner's OpenAI/Anthropic bill for free.

**Business impact:** unthrottled AI-proxy routes are a direct margin problem. every unauthenticated request is COGS with no corresponding revenue, and it scales exactly as fast as an attacker's script does. Unthrottled login endpoints turn every leaked password list on the internet into an automated account-takeover attempt against your customer base, with your brand attached to every resulting complaint.

**Fix:** rate limit by IP and by user on every auth and AI-proxy route (Vercel Edge Middleware + Upstash Redis, or Supabase's built-in limits).

## 9. SSRF and Injection via "Fetch This URL" Features

Any feature that takes a URL and fetches it server-side, webhook testers, "import from URL," PDF/screenshot generators, is worth pointing at internal infrastructure:

```json
{ "url": "http://169.254.169.254/latest/meta-data/" }
```

If the response comes back, the server just proxied a request to cloud metadata (or an internal service) on the attacker's behalf. Rarely is there an allowlist on scheme or destination host.

**Business impact:** cloud metadata endpoints hand over the IAM role credentials your infrastructure runs as, from there, the blast radius is your entire cloud account, not just the application. This converts a "medium" web bug into a full infrastructure compromise, which is the scenario cyber-insurance policies specifically underwrite premiums against and the one that turns a single incident into weeks of downtime.

**Fix:** allowlist destination hosts/schemes for any server-initiated outbound request and block private/link-local IP ranges before the fetch happens.

## 10. Missing Webhook Signature Verification

Stripe/Resend webhook handlers at `/api/webhooks/*` frequently trust the JSON body without verifying the provider's signature header, because the demo works fine without it. That means anyone who finds (or guesses) the endpoint can POST a fake `checkout.session.completed` or `subscription.updated` event and grant themselves paid access for free:

```bash
curl -X POST https://target.app/api/webhooks/stripe \
  -H "Content-Type: application/json" \
  -d '{"type":"checkout.session.completed","data":{"object":{"customer_email":"me@me.com"}}}'
```

**Fix:** always verify the signature (`Stripe-Signature` header + webhook secret) server-side before trusting the payload, using the provider's SDK, never parse the raw body first.

**Business impact:** This is a direct revenue-loss bug, not a data-exposure one, every forged "payment succeeded" event is a customer with full paid access and zero dollars collected. Left unnoticed, it shows up months later as an unexplained gap between MRR reported by Stripe and the number of "paying" accounts in your own database, which is an awkward number to explain in a fundraising data room.

---

## Impact Summary

|#|Finding|Primary Business Risk|
|---|---|---|
|1|Over-fetching / excessive data exposure|Regulatory disclosure (GDPR/CCPA), failed SOC 2 / vendor security review|
|2|Mass assignment|Unauthorized paid-tier access, direct revenue leakage|
|3|Client-authoritative real-time state|Fraud on stakes-bearing outcomes (prizes, leaderboards, rankings)|
|4|Missing/weak RLS|Full cross-tenant data breach, lost enterprise contracts|
|5|IDOR|Bulk customer data scraping, failed pre-purchase security review|
|6|Hardcoded/over-privileged secrets|Uncapped API cost exposure, full infrastructure/database compromise|
|7|Public storage buckets|Unauthenticated leak of sensitive documents, breach notification|
|8|No rate limiting|Margin erosion via API abuse, mass account takeover|
|9|SSRF|Full cloud account compromise via metadata theft|
|10|Missing webhook verification|Direct MRR fraud, inaccurate revenue reporting|

## The Pattern Behind the Pattern

- None of these findings required custom tooling, a proxy, dev tools, and the willingness to send a request the UI never intended to send got us all ten. That's the point, **AI coding agents optimize for "the feature works," not "the feature is safe against a hostile client."**
- Authorization checks, response DTOs, and server-side state validation are invisible in a working demo. They only matter once someone stops using the app the way it was designed to be used which is exactly when nobody's watching for them, and exactly when it shows up as a support ticket, a chargeback, or a lost deal instead of a line in a code review.

## Fixing the Workflow, Not Just the Output

- The default AI-assisted build loop is: `write feature → run app → fix bugs → deploy.`
- Security is an afterthought, caught (if at all) in a post-hoc review of a diff nobody has time to fully read.

The higher-leverage fix is pushing these rules into the agent's context before it writes a line of code. Claude, Cursor, Cline, Goose, and OpenCode all support project-level instruction files loaded automatically into context. A condensed version of what we now drop into every vibe-coded project before letting an agent touch the backend:

```markdown
# AI Security Skill

## Never Trust the Client
The client is always untrusted. Never allow it to determine
permissions, roles, prices, balances, scores, or business state.
The server is always authoritative.

## Explicit Input Validation
Never update database models directly from request bodies.
Reject unknown fields. Use allowlists for every API endpoint.

## Never Serialize ORM Models
Return response DTOs. Never expose password hashes, emails
(unless required), internal IDs, timestamps, feature flags,
or authentication metadata.

## Keep Business Logic on the Server
Clients request actions. Servers validate actions.
Servers update state. Clients render state.

## Hidden UI Is Not Security
If the browser has the data, assume the user already has it.
Never send future answers, hidden admin data, internal prompts,
or privileged application state ahead of when it's needed.

## Secure File Uploads
Treat uploads as untrusted. Validate ownership, MIME type, and
authorization. Never let an arbitrary storage URL become
application state without verifying its origin.

## Perform a Security Self-Review
Before completing any implementation, verify: unknown fields
rejected, authorization enforced, least privilege maintained,
sensitive data not exposed, business logic server-side, input
validated, secrets protected.
```

This doesn't replace a real assessment, but it closes off the most common findings before they're ever written, which is a far cheaper place to catch them than in a pentest report after launch.

However, for a more detailed guide, ref: https://www.reddit.com/r/vibecoding/comments/1tlvpjr/security_and_maintainability_guide_for_vibecoders/

## Conclusion

The patterns covered here weren't tied to a specific product or codebase. They appeared across different applications built with similar AI-assisted workflows and modern web stacks, often stemming from the same underlying assumptions about trust, authorization, and data exposure.

As AI-assisted development becomes more common, reviewing these security fundamentals early in the development process can help reduce the likelihood of introducing common implementation flaws. Security reviews remain valuable not only for finding vulnerabilities before deployment, but also for identifying recurring patterns that can be addressed systematically.
