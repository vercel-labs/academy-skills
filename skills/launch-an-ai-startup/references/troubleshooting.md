# Troubleshooting by lesson

Every troubleshooting entry from the course, verbatim, grouped by lesson. Match the student's symptom to the bold line, then walk them through the fix. Links point to the Vercel docs pages that own anything that changes (limits, labels, CLI flags).

## 1.1 Deploy the Starter

**The Production response still shows an old commit.** `/api/health` reads only build-time values, so Next.js prerenders it once per deployment. Each deployment answers with its own commit, forever. If the commit is old, the newest deployment is still building or it failed. Open the deployment list and check that the top Production deployment shows **Ready**.

**Production says `"development"` and `"local"`.** The deployment didn't get Vercel's system environment variables. In the project's environment variable settings, turn on **Enable access to System Environment Variables** (the [system environment variables](https://vercel.com/docs/environment-variables/system-environment-variables) page has the steps), then redeploy.

## 1.2 Preview Every Change

**The banner didn't change after you edited the variable.** Old deployments keep their old values. Every deployment is built once with the variables that existed at that moment, so redeploy to pick up the change.

**The Preview asks you to log in to Vercel.** That's [Vercel Authentication](https://vercel.com/docs/deployment-protection/methods-to-protect-deployments/vercel-authentication) protecting the URL. Anyone on your Vercel team gets in. For an outside reviewer, share the deployment from the Toolbar instead of turning protection off.

## 1.3 Domains and Rollback

**The domain says it's misconfigured.** DNS changes can take a while to spread. Compare the record at your provider with the one the dashboard shows, character for character. A missing trailing dot or an old `A` record from a previous host is the usual cause.

**You pushed a fix and Production didn't change.** The project is still in a rolled-back state, so new deployments wait. Use **Undo Rollback**, or run `vercel promote` with the deployment URL from the [CLI](https://vercel.com/docs/cli/promote).

## 2.1 Save Updates

**`db:setup` prints `ECONNREFUSED 127.0.0.1:5432` on your laptop.** `.env.local` doesn't have `DATABASE_URL` yet. Run `vercel env pull` and check the file.

**Saving works locally but returns a 500 on Preview.** The database is connected to Development but not Preview. Open it from **Storage**, check the connected environments, and redeploy after you add Preview.

## 2.2 Add Sign-In

**Sign-in sends you back to `/sign-in` every time.** Your Slack email isn't in `ALLOWED_FOUNDER_EMAILS`, or the dev server started before you added it. The match ignores case and spaces, so check for a typo, then restart.

**Sign-in on a Preview lands on Production afterward.** `BETTER_AUTH_URL` is set for Preview. Remove it from Preview so the code falls back to that Preview's branch URL, then redeploy.

**Sign-in fails and the server log shows `Vercel Connect sign-in failed`.** The `onError` handler in `lib/auth.ts` logs the Connect error's `code` and `status`. The usual causes are a `CONNECTOR_SLACK` UID that doesn't match the connector, or a connector that isn't attached to the environment you're running. On your laptop, a stale `VERCEL_OIDC_TOKEN` does the same thing; `vercel env pull` refreshes it.

## 2.3 Draft in the Background

**Notes stay at `crunching` forever on your laptop.** `send` couldn't authenticate or the consumer isn't registered. Check the terminal for an `UnauthorizedError` (run `vercel env pull` again), and check that the path in `vercel.json` matches `app/api/queues/highlights/route.ts` exactly.

**It works locally but not on a deployment.** `vercel.json` didn't ship with the commit, so the deployment has no trigger for the topic. Commit it and redeploy.

## 3.1 Stream the Update

**The draft fails with an authentication error locally.** `AI_GATEWAY_API_KEY` is missing from `.env.local`, or the dev server started before you added it. [AI Gateway authentication](https://vercel.com/docs/ai-gateway/authentication-and-byok) covers keys and OIDC.

**Nothing streams, and the panel jumps from empty to finished.** Something between the route and the browser is buffering. Check that the route returns `createTextStreamResponse` directly, and isn't wrapped in code that awaits the whole stream first.

## 3.2 Give It One Tool

**The draft comes back without any numbers.** The model wrote the draft without calling the tool, usually because the run stopped after one step. Check for `stopWhen: isStepCount(3)`.

**TypeScript says `toolsContext` is missing `getMetrics`.** Once a tool declares `contextSchema`, the route has to supply it. The key in `toolsContext` must match the key in `tools`.

## 3.3 Fallbacks and Attribution

**The fallback never answers and the draft fails anyway.** The fallback has to support everything the request uses. A model without tool calling or structured output fails the same request the primary did. Check both models' capabilities in [Models and providers](https://vercel.com/docs/ai-gateway/models-and-providers).

**The log says `servedBy: undefined`.** Routing metadata comes from AI Gateway. If you replaced the model string with a direct provider package, the request never went through the Gateway, so there's no routing record. Keep the plain `"creator/model"` string.

## 3.4 Cap the Spend

**You can't trigger the limit on a Preview.** Preview deployments need [Protection Bypass for Automation](https://vercel.com/docs/deployment-protection/methods-to-bypass-deployment-protection/protection-bypass-automation) enabled for the rate limiting SDK to run there, and [system environment variables](https://vercel.com/docs/environment-variables/system-environment-variables) exposed. Test on Production, or turn both on.

**Every founder hits the limit at once.** The rule is counting one shared bucket. Check that `rateLimitKey` is `founder.id` and not a constant string, which would make the limit global.

## 3.5 Write a 10-Case Eval

**Every case fails with `no valid draft`.** The runner can't reach the model or the database. Check that `.env.local` has `AI_GATEWAY_API_KEY` and `DATABASE_URL`, then run `vercel env pull` if the database credentials are missing.

**A check fails on wording you'd accept.** Some checks match patterns, like a percentage for MRR growth. If the model wrote "fourteen percent," the draft is fine and the check is too strict. Loosen the check, but only when you'd defend the draft to an investor.

## 4.1 Add an eve Agent

**`eve info` doesn't list `getMetrics`.** The tool file isn't in `agent/tools/`, or it doesn't `export default` a `defineTool(...)`. The filename becomes the tool name, so `getMetrics.ts` is `getMetrics`.

**Tool calls fail with `Sign in through Up and to the Right to use this agent.`** The caller isn't a signed-in founder. In local dev, a request that skips `appSession` gets eve's stand-in `local-dev` caller, which `requireFounder` rejects on purpose. Sign in through the app and try again.

## 4.2 Approve Before Posting

**`channel:set` says `No founder has signed in with ... yet.`** The founder account is created on first sign-in. Sign in once, then run the script again.

**The tool fails with `Slack rejected the post: not_in_channel`.** The bot isn't a member of the channel. Invite it from the channel in Slack and try again.

## 4.3 Survive Interruptions

**The second approval posts again anyway.** The claim isn't keyed the way you think. Check that `claimPost` uses the founder from `requireFounder(ctx)` and the `month` from the tool input, and that `slack_posts` has `primary key (founder_id, month)`.

**A month says "Check your investor channel before retrying" forever.** A post stopped after the claim and before the result was recorded. Check the channel. If the update is there, set the row's `status` to `posted`. If it isn't, delete the row and send again.

## 4.4 Chart It in a Sandbox

**Locally, the tool fails with a Vercel Sandbox authentication error.** `agent/sandbox.ts` asks for Vercel Sandbox by name, so eve needs credentials to open one, even on your laptop. Run `vercel link` and `vercel env pull` (or `eve link`), then try again. [Sandbox troubleshooting](https://eve.dev/docs/sandbox/troubleshooting) covers the other causes.

**The chart never appears, and the agent says the chart was created.** The model saw `toModelOutput`'s summary, and the browser didn't get the SVG. Check that the panel matches `part.name === "chartRevenue"` and reads `part.output.svg`.

## 5.1 Trace a Failed Request

**`vercel logs` says the option was removed.** CLI v64 replaced `--request-id` and `--status-code` with `--query`. Some older pages still show the old flags. Update the CLI, or use the replacements from the [`vercel logs` reference](https://vercel.com/docs/cli/logs).

**The search finds nothing for a reference from yesterday.** `--query` searches a short default time window, described in the [`vercel logs` reference](https://vercel.com/docs/cli/logs). Add `--since 1d`. How far back logs go depends on your plan, and the [runtime logs limits](https://vercel.com/docs/logs/runtime#limits) list it.

## 5.2 Protect the AI Route

**Founders get `Access denied` from the page.** The path in `protect` doesn't match the route the page calls. It has to be `/api/draft` with method `POST`, exactly as the panel sends it.

**The Log-mode rule shows no matches.** Check that the rule is published, not only saved: unpublished changes leave a **Review Changes** button in the corner. Then check the path condition matches `/api/draft` exactly, and that your burst sent more requests than the limit inside one window.

**You can't reproduce the block locally.** That's BotID's local behavior: development always returns `isBot: false`. The [local development behavior](https://vercel.com/docs/botid/local-development-behavior) page shows how to simulate a bot if you need to.

## 5.3 Release Behind a Flag

**The flag always returns `classic`.** The adapter couldn't reach Vercel Flags, so it used the default. Run `vercel env pull` again locally, and check that the flag key in the dashboard is exactly `draft-style`.

**Targeting by founder doesn't match.** The dashboard rule has to use the same entity and attribute that `identify` returns: `founder.id`. The [entities docs](https://vercel.com/docs/flags/vercel-flags/dashboard/entities) cover how to declare them.

## 5.4 Launch Review

**The monthly estimate is bigger than the budget.** Good, you found it on paper. Raise the budget, lower the rate limit, move drafting to a cheaper model and rerun the eval, or charge for heavy use.

**A failure has no one to own it.** That's a launch blocker, or an accepted risk with a name and a date on it. Write down which.
