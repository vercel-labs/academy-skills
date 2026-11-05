# Failure labs and escalation

Every lesson breaks something on purpose before it ships. Use this file to run the lab with the student, confirm they saw the failure, and know when the course itself says to stop debugging and contact Vercel.

The "When to ask Vercel" text is quoted from each lesson.

## Section 1: Ship It

### 1.1 Deploy the Starter

- **Outcome:** The starter is live on a Production URL and /api/health reports the deployed commit.
- **Failure lab:** A type error fails the build; read the first useful line of the log.
- **When to ask Vercel:** If a build fails on Vercel but passes locally with the same commit, and the first error in the log points at the platform rather than your code, that's a [support](https://vercel.com/kb/guide/how-to-get-vercel-support) conversation. Include the deployment URL.

### 1.2 Preview Every Change

- **Outcome:** A pull request gets its own Preview with Preview-scoped environment variables and a review comment.
- **Failure lab:** A Preview reads the Production value; fix the environment variable scope.
- **When to ask Vercel:** When you need a long-lived staging environment between Preview and Production, read about [custom environments](https://vercel.com/docs/deployments/environments) and their plan availability before you build one out of branches. If the plan you're on doesn't include them, that's a sales conversation worth having early.

### 1.3 Domains and Rollback

- **Outcome:** Production serves a custom domain, and a bad release is undone with Instant Rollback.
- **Failure lab:** Ship a bad headline to Production, then roll it back.
- **When to ask Vercel:** Moving a domain that already serves real traffic, with email records and an existing host, is worth a plan before you touch DNS. If you're migrating from another provider under a launch deadline, ask Vercel for help with the cutover.

## Section 2: Make It a Product

### 2.1 Save Updates

- **Outcome:** Updates persist to a Marketplace Postgres database through a Function route.
- **Failure lab:** Unset the database URL in Preview and find the failure in the logs.
- **When to ask Vercel:** Pick a database region close to the region your Functions run in, because every query pays for the distance. If your customers' data has to stay in a specific country, ask Vercel and your database provider about region options before you have real data to move.

### 2.2 Add Sign-In

- **Outcome:** Only signed-in founders can read or write their own updates, in the UI and the API.
- **Failure lab:** Call the API signed out, then as a different founder.
- **When to ask Vercel:** When your first enterprise customer asks for SSO and an audit trail of who signed in, that's a different problem from founder sign-in. Bring their requirements to Vercel early, before you promise a date.

### 2.3 Draft in the Background

- **Outcome:** Creating an update returns right away while a Queues consumer finishes the work.
- **Failure lab:** Replay a message; the idempotency key prevents a duplicate.
- **When to ask Vercel:** If you're planning launch-day traffic around Queues, or you need guarantees the docs don't state, ask Vercel before you build something you can't change later.

## Section 3: Add the AI

### 3.1 Stream the Update

- **Outcome:** The draft streams through AI Gateway and validates against a schema.
- **Failure lab:** Malformed output fails the schema and shows a friendly error.
- **When to ask Vercel:** If your customers need their prompts kept out of provider training or logs, read about [zero data retention](https://vercel.com/docs/ai-gateway/security-and-compliance/zdr) on AI Gateway first, and bring enterprise data requirements to Vercel before your first big customer asks.

### 3.2 Give It One Tool

- **Outcome:** The model reads the founder's metrics through a typed tool that checks ownership.
- **Failure lab:** The tool is asked for another founder's metrics and refuses.
- **When to ask Vercel:** When the model needs data from systems you don't own (a CRM or a customer's own database), the same rule applies across every one of them: the application decides who the tool acts for. If your design starts passing customer credentials around, that's a good moment for an architecture review with a Vercel Solutions Architect.

### 3.3 Fallbacks and Attribution

- **Outcome:** Requests fall back to a second model and carry user and feature tags.
- **Failure lab:** The primary model fails and the fallback answers.
- **When to ask Vercel:** If your product can't tolerate a provider outage at all, fallbacks are the start. Talk to Vercel about the provider and region options for your models, and ask before you sign anything that promises uptime to your own customers.

### 3.4 Cap the Spend

- **Outcome:** A project budget and a per-founder rate limit protect the draft route.
- **Failure lab:** Hit the rate limit, then see what the budget does.
- **When to ask Vercel:** Before you price your product, find out what a draft costs you. If you're planning a free tier, bring your expected usage to Vercel so your budget, your rate limits, and your pricing page tell the same story.

### 3.5 Write a 10-Case Eval

- **Outcome:** Ten fixed cases score schema validity, tool authorization, and quality.
- **Failure lab:** Change the prompt and watch a case regress.
- **When to ask Vercel:** When ten cases turn into hundreds, or you want to compare models on your own data before you switch, that's a bigger evaluation setup. Ask Vercel about evaluating models through AI Gateway with your traffic before you commit to a migration.

## Section 4: Let It Act

### 4.1 Add an eve Agent

- **Outcome:** An eve agent runs inside the Next.js app and drafts an update with the existing tool.
- **Failure lab:** A missing Gateway credential; read eve's error.
- **When to ask Vercel:** If you're building a production agent on eve today, tell Vercel what you're building, so breaking changes reach you before they reach your users.

### 4.2 Approve Before Posting

- **Outcome:** Posting to Slack through Vercel Connect waits for the founder's approval.
- **Failure lab:** A second founder with no assigned channel asks to send; the policy denies the call before any approval card. Then approving the same month twice posts it twice, which 4.3 fixes.
- **When to ask Vercel:** Once an agent can act in your customers' tools (their Slack or their CRM), you'll want an answer to "who approved this, where did it go, and who allowed that destination?" for every action. If a customer's security review asks for that audit trail, bring the question to Vercel early.

### 4.3 Survive Interruptions

- **Outcome:** A run interrupted mid-draft resumes without repeating the model call.
- **Failure lab:** Kill the run after the draft step and resume it.
- **When to ask Vercel:** If your agents run long jobs that touch money or customers' systems, design the recovery path before the first incident. Bring the design to a Vercel Solutions Architect while it's still a diagram.

### 4.4 Chart It in a Sandbox

- **Outcome:** Generated chart code runs in an isolated sandbox and returns an image.
- **Failure lab:** Generated code tries to read a secret, then runs forever.
- **When to ask Vercel:** Running untrusted code is a security decision. If your product will run code that your customers or their users write, on top of code your model writes, talk to Vercel about sandbox isolation and limits before you launch it.

## Section 5: Run It in Production

### 5.1 Trace a Failed Request

- **Outcome:** A user-facing error is traced to its deployment and failing tool.
- **Failure lab:** A tool failure, traced from the error message to the cause.
- **When to ask Vercel:** If you already send logs to Datadog, Honeycomb, or another tool, you don't have to pick one place to look. Ask Vercel about [drains](https://vercel.com/docs/drains) so request IDs line up across both.

### 5.2 Protect the AI Route

- **Outcome:** A Firewall rate limit and BotID block a synthetic burst while real founders get through.
- **Failure lab:** Run the burst before and after the rule.
- **When to ask Vercel:** If someone is actively attacking your app (a sustained flood, or someone scraping your AI output at scale), open a support case with the request IDs you've found. Vercel's [DDoS mitigation](https://vercel.com/docs/vercel-firewall/ddos-mitigation) is automatic, and support can help you tune rules during an incident.

### 5.3 Release Behind a Flag

- **Outcome:** A new draft style ships behind a flag and is turned off when it gets worse.
- **Failure lab:** The flagged version raises errors; turn the flag off.
- **When to ask Vercel:** When you want to A/B test drafts against a business metric (more investor replies, more follow-up meetings), that's experiment design on top of a flag. Talk to Vercel about running experiments with Vercel Flags before you pick your own tooling.

### 5.4 Launch Review

- **Outcome:** A cost model, an owner for each failure, and a decision on when to ask Vercel for help.
- **Failure lab:** None; this lesson reviews the evidence from earlier labs.
- **When to ask Vercel:** Not covered in this lesson.
