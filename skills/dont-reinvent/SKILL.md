---
name: dont-reinvent
description: Vets whether to build a feature from scratch, use an existing free/open-source solution, or pay for a third-party API/service. Use this whenever the user asks Claude to implement, add, integrate, or write code for a feature or capability, especially one that sounds like it could already be a solved problem (auth, payments, scraping, animation, comment/DM automation, video/image generation, search, notifications, etc.), even if the user doesn't explicitly ask "should I build or buy this." Make sure to trigger this before writing non-trivial implementation code for any feature-level capability, not just when the user asks about libraries directly.
---

# Vet: Build vs. Free vs. Paid

## Why this exists

Left to its own defaults, Claude tends to write code from scratch for things that already exist as mature, free, open-source solutions, burning tokens and iteration cycles reinventing a solved problem. This skill forces a check before that happens.

## Core principle: do the vetting yourself, never delegate it to the user

The target user cannot read a license file, judge code quality, or assess a repo's health themselves. That is the entire reason this skill exists. Never end an answer with "check the license yourself" or "look through the code before adopting." If a claim about license, maintenance, or quality can't be verified, verify it with a tool call before answering. Don't pass the burden back.

## The decision tree (follow in order, do not skip step 0 or step 1)

**Step 0: Check both required connectors before doing anything else.**
This skill depends on two MCP connectors: **GitHub Search MCP** (for real repo search, license/file reads, secret scanning) and **Socket security scan** (for dependency vulnerability checks). Before running Step 1, check whether both are available.

- If **either is missing**, stop and tell the user plainly, in one short message, before proceeding:
  - Name which one is missing.
  - Explain in one sentence what it's for and what's lost without it (e.g. "Without Socket connected, I can't check whether this repo's dependencies have known vulnerabilities. I'd be guessing.").
  - Give the concrete next step to connect it: for a connector already in Anthropic's directory, that's Settings > Connectors > Add and search by name; for one that requires a custom URL (like GitHub's or Socket's official MCP server), give the exact URL and note whether it needs OAuth login or an API token.
  - Ask whether the user wants to connect it now, or proceed anyway with a clearly weaker fallback (general web search instead of GitHub search; no automated security check instead of a real one).
- **Diagnose a failing connector before reporting it. Never assume a cause.**
  A tool call that fails is not evidence that the service is down. Several different conditions produce failures that look identical from the outside, and each has a different fix:
  - **Not authenticated** — the connector is installed but was never signed in. This is the most common cause and the one most often misread as an outage.
  - **Not installed** — the connector was never added to this client.
  - **Genuinely erroring or unreachable** — the service itself is failing.

  Some tools return only a generic message (e.g. `Tool execution failed`) that hides which of these it is. Retrying such a call proves nothing; five identical failures are still one unexplained failure. Before saying anything about the cause:
  1. Call a second, cheaper tool on the same server (for Socket, `organizations`; for GitHub, `get_me`). The clearer error usually surfaces there.
  2. If every tool on that server fails the same way, the problem is the connector, not the tool.
  3. Report only what the errors actually said. Never write "down", "unavailable", or "unreachable" unless a check established that.

- **If the cause is authentication, prompt the user to authenticate.** That is a thirty-second fix on their side; calling it an outage sends them off to wait for something that will never resolve on its own. Name the connector, say plainly that it needs signing in (OAuth login or an API token, whichever applies), and offer to re-run the check as soon as they confirm. Do not proceed to a degraded fallback without asking first.

- **Never silently fall back.** If you proceed without a connector, whether because the user said to or because you have no way to prompt (e.g. an automated/non-interactive run), the final output must say so explicitly (see "Verified / Could not verify" in the output format). Never quietly do a worse job and present it with the same confidence as a fully-checked one.
- Do this check once per conversation, not on every single invocation within the same chat. No need to re-ask if the user already answered earlier in this session.

**Step 1: Search.**
Use the GitHub search MCP tools if connected (real search syntax: `stars:>50 <topic>`, sort by stars). Otherwise use web search with `site:github.com` and similar. Also check package registries relevant to the stack (npm, PyPI) for install/trend data. Cast a slightly wide net and run 2-3 query variations, since the first phrasing often under- or over-shoots (see the Instagram DM example: a narrow query found 1 result, a broader one found 4).

**Step 2: Vet the top 3-5 candidates yourself. Do not stop at metadata.**
For each candidate, actually verify. Don't infer from stars alone:
- **License**: read the LICENSE file (or repo metadata) directly. State the actual license found, not "probably MIT." No LICENSE file is a real finding, not a missing detail. See the "no-license" handling below.
- **Maintenance**: last commit date, open vs. closed issue ratio, whether it looks abandoned mid-build.
- **Code quality / fit**: read the README and, for the top 1-2 candidates, skim actual source (e.g. main entry file, package manifest) to check it does what it claims and doesn't pull in something alarming (e.g. hardcoded credentials, obviously broken auth).
- **Security**: if a dedicated security-scanning MCP tool is available (e.g. secret scanning, known-vulnerability scanning), run it on the top 1-2 candidates before recommending them. Always name the tool that did the check in the output (e.g. "passed GitHub secret scanning"). The user should know a check happened and which third-party service performed it, not just take Claude's word for it. If no such tool is available, say plainly that no automated security check was run, rather than implying one was.
- If something can't be verified (tool access limited, file missing), say so as a real caveat in the output. Don't paper over it with a generic disclaimer.

**Handling a no-license repo**: this is not an automatic disqualification, but it is never a clean "use this" either. Split it into two honest paths in the output: (1) *not usable as a dependency*, since you can't legally redistribute or rely on unlicensed code, but (2) it may still be useful as *reference/inspiration* for a from-scratch build, since reading how someone else solved the problem can cut down the "figuring out the approach" part of build effort even if you can't use their code directly. Say which of these applies, or both. Don't silently discard a well-built no-license repo, and don't recommend it as a dependency either.

**Step 3: Rank with reasoning, not just a single pick.**
Present the top 3 (or fewer, if fewer exist) as a ranked list. For each runner-up, state *specifically* why it ranked below #1 (smaller/less active, unclear license, does less of the job, etc.), not a generic "also found."

**Step 4: Only if no usable free option exists after real vetting, compare Build vs. Paid.**
These are priced in *different currencies*. Don't force them into one number. State both plainly:
- **Build from scratch:** one-time cost, paid in tokens/iteration cycles. Never give a specific token count, since that's not estimable and false precision would be misleading. Instead use a named tier plus the concrete reason behind it:
  - *Low token effort*: simple, well-documented, few edge cases (e.g. a basic CRUD wrapper)
  - *Moderate token effort*: some integration complexity, a handful of edge cases to iterate on (e.g. webhook handling, a documented third-party API)
  - *High token effort*: multiple hard parts likely needing several debugging rounds (e.g. auth flows, async/retry logic, unfamiliar APIs)
  - *Very high token effort*: research-grade or highly stateful problem, likely to need many iterations or may not converge cleanly (e.g. real-time sync, ML inference from scratch)
  Always pair the tier with *why*, naming the specific hard parts driving it, not just the label alone.
- **Paid API/service:** ongoing cost, paid in dollars per month or per unit of usage (find real current pricing).

Give a rough cost framing: "at your expected usage, building pays for itself after about ~X months vs. the paid option." Approximate is fine. Precision isn't the point; honesty about the two currencies is.

## Naming: this is Free vs. Build vs. Buy, three options, not two

Don't call this "build vs. buy." That framing hides the option that matters most: an existing free solution. Always use three explicit labels: **Free** (an existing open-source solution), **Build** (write it from scratch), **Buy** (pay for a third-party API/service). The top-line verdict always names one of these three first, before anything else.

**Always say "open-source" when referring to the Free option, not just "free."** "Free" alone is ambiguous. It could be misread as a free tier of a paid service, which carries its own lock-in and limits. Say "open-source candidate" / "open-source repo" so it's unambiguous this is a self-hosted, no-recurring-cost option, not a freemium plan.

## Output format

```
Recommendation: [FREE, use <Name> / BUILD it yourself / BUY <Service>]
<one sentence why, always naming the token cost explicitly, e.g. "near-zero tokens to 
fork and wire up" for Free, "moderate/high token effort" for Build, "near-zero tokens 
but $X/month" for Buy, plus any real blocker.>


Top candidates:
1. [Name] | [license found] | [maintenance signal] | RECOMMENDED
   Why: <what it does, why it's the pick>
   Security: <one sentence, e.g. "Socket flagged 3 dependencies with known vulnerabilities, 
      update before use" or "Passed GitHub secret scanning, no issues" or "No automated 
      check available for this repo">
2. [Name] | [license] | [maintenance signal]
   Why not #1: <specific reason it ranked lower>
3. [Name] | [license] | [maintenance signal]
   Why not #1: <specific reason it ranked lower>

[If a candidate has no LICENSE file:]
[Name] | NO LICENSE | not usable as a dependency as-is / may still be useful as reference 
   for a from-scratch build (pick whichever applies, or both)

[If no usable free candidate, BUILD vs BUY:]
Build from scratch: [Low/Moderate/High/Very high] token effort. <the specific hard parts driving this>
Paid alternative: [Service], $X/month. <what it covers>. Near-zero tokens to integrate.
Cost comparison: always state build's token cost AND contrast it against the alternative's 
   token cost, not just its dollar cost. Free/fork options are near-zero tokens (mostly 
   reading and wiring up existing code); paid options are also near-zero tokens (an API call) 
   but cost dollars monthly; only Build spends real tokens, and that's the trade-off to 
   name explicitly. E.g.: "Building this yourself is moderate token effort: several rounds 
   of iteration on the Meta API auth, versus near-zero tokens to integrate the paid 
   service, which costs $12-30/month instead. If you have tokens to spare and want to own 
   the code, build. If you'd rather spend dollars than tokens, buy." 
   Never let the free/buy options go unmentioned in token terms. The whole comparison is 
   token-vs-token-vs-dollars, not token-vs-dollars.

Verified: <what you actually checked: license file read, source skimmed, security scan run, etc.>
Could not verify: <anything you couldn't confirm, say this plainly if it applies>
```

**Security specifically must always be one sentence, never a table or a per-dependency score breakdown.** Internally, run the full check (score every relevant dependency, read the detail), but report only the verdict: what's safe, what's flagged, and the one-line reason. If the user asks "why" or "show me the scores," give the detail then, not by default.

## Guardrails

- Don't recommend a paid service just because it's fastest to integrate. Name the recurring cost honestly, every time.
- Don't skip Step 0, Step 1, or Step 2 even if the user seems to already be leaning toward building or paying.
- Never silently degrade. If GitHub Search MCP or Socket security scan isn't connected, say so and offer to help connect it. Don't quietly fall back to web search or skip the security check without flagging it, even once.
- Never tell the user to go verify something themselves. That's this skill's job. If you genuinely can't verify it with available tools, say plainly that it's unverified, as a caveat on the recommendation, not as homework for the user.
- Never assume why a tool call failed. A generic failure is not an outage: probe a second tool on the same server first, and if it turns out to be authentication, prompt the user to sign in rather than reporting the connector as down or quietly degrading.
- Never recommend a specific paid vendor because of any partnership/connector relationship. Recommendations stay neutral, cost-based only.
