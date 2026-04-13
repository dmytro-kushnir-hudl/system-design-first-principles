# Claude Code in the Wild: What Actually Works

> **Format:** Talk, not a tutorial. War stories, honest opinions, things I wish I knew earlier. No slides needed — this is a conversation.

---

## 1. THE TALK PLAN

- Useful apps / MCP setup
- MCPs & skills worth building
- Tricks that actually work
- Where AI still can't replace humans
- Cost reality check
- What's happening in the ecosystem

---

## 2. USEFUL APPS / INTEGRATIONS

**gh CLI**
Claude reads/creates PRs, comments, issues. Killer combo with code review. Install it, point Claude at your repo, and stop copy-pasting diffs into chat.

**AWS CLI (read-only mode)**
A read-only agent that can hit your AWS accounts is surprisingly useful — finding instances, reading CloudWatch metrics, exploring infra without risk. The pattern: give Claude AWS credentials scoped to read-only, let it navigate. Blocks writes structurally, not by trust.

**Jira + Slack MCP** (available via claude.ai)
Context from tickets and threads directly in session. Stops you from context-switching to copy-paste. The Slack one is especially useful — paste a thread URL, Claude reads the full discussion.

**Google Drive**
No working MCP found at time of writing. Workaround: install Google Drive desktop sync, point Claude at the local folder. Works in web/desktop chat mode only (not CLI). Good enough for docs and meeting transcripts.

---

## 3. MCPs & SKILLS WORTH BUILDING

The highest-leverage 30-minute investments for any team:

**Compiled code metadata reader**
If you work in a compiled language (.NET, JVM, etc.), Claude only sees source — not what's actually in your binaries or what a library method really does. Build or find an MCP that reads your assembly metadata: types, members, dependencies. Makes Claude actually understand the full picture, not just what's visible in the open tabs.
- Example for .NET: `ilspy-decompile` skill ([davidfowl/dotnet-skillz](https://github.com/davidfowl/dotnet-skillz)) — decompiles NuGet packages on demand. Useful when there's no public source.

**Internal API catalog MCP**
If your org has 50+ microservices, stop pasting Swagger docs manually. Build a lazy-loaded MCP over your OpenAPI specs. Claude can look up any internal contract in context without you doing the legwork. Once built: permanent time saver across every session.

**Log search MCP**
The most underrated one. Connect Claude to your logging platform (Sumo, Datadog, CloudWatch, whatever). Ask "why did this fail in prod yesterday" and get actual log context in the same session, not in a separate tab. Turns post-incident investigation from 20 minutes to 2.

**General principle**
Think about what Claude can't see that you constantly have to look up manually. That's your MCP list.

---

## 4. TRIED, DID NOT LIKE

**LSP for C# (Serena)**
Stale most of the time. More noise than signal in day-to-day use. Might be worth it on a large completely unfamiliar codebase, but for regular work: no. Set `web_dashboard_open_on_launch: false` in `serena_config.yaml` to stop it stealing focus on every launch if you do try it.

**LSP for TypeScript**
Mixed feedback. Reportedly better than the C# variant. YMMV — test it on your codebase before committing.

**Knowledge graph code review tools** (e.g. [code-review-graph](https://github.com/tirth8205/code-review-graph))
Interesting idea: scan repo into a graph, get callers/callees, blast radius estimates, test coverage awareness. Reality: lots of false positives, fails badly on dynamic invocations (which is most of .NET and a lot of JS). Might be useful for very large unfamiliar repos where you need rough orientation. Don't trust the precision.

---

## 5. TRICKS THAT ACTUALLY WORK

### Video / meeting transcription
Zoom and Google Meet recordings already have transcripts. Feed them to Claude for a full summary — great for catching decisions from design reviews you missed or joined late. Pair with the Jira MCP to turn action items into tickets automatically.

### Auto mode
Available in both CLI and desktop apps. Enable it. Trust but verify. You stop babysitting permission prompts, Claude stops asking obvious questions.

⚠️ **Known risk:** Claude has been observed making commits and pushing *while in plan mode*. Not a one-off. Keep eyes on it early in any new session until you trust the setup. Gradually expand autonomy as you build confidence.

### Panel / jury pattern
For hard decisions (architecture, design, root cause analysis): ask Claude to dispatch the problem to multiple "expert personas" working in isolation — each with empty context, no groupthink. Root agent collects opinions and surfaces non-obvious angles. Works especially well when you feel stuck or when the obvious answer seems too obvious.

---

## 6. META-PATTERNS WORTH KNOWING

### Most actionable

**Reflexion / self-critique loop**
Claude generates → critiques its own output → regenerates. Works better than asking for quality upfront. Don't ask "write good code" — ask "write code, then review it as a senior engineer and fix what you'd call out." The plan → review → execute → red-team loop is a named variant of this.

**Confidence gating**
Ask Claude to self-assess confidence (1–10) before acting. If below threshold, it replans or asks rather than hallucinating forward. Useful for anything with real consequences. Some teams build this into their skill definitions as a standard step.

**Persona anchoring**
Assign a role *before* the task: `"You are a principal engineer, 5 years on this codebase, skeptical of over-engineering."` Changes depth and tone significantly vs adding the role mid-prompt or not at all. The specificity matters — "senior engineer" is too generic.

### Also named, worth googling

| Pattern | What it does |
|---|---|
| **Skeleton-of-thought** | Outline first, expand sections in parallel. Less drift on long outputs. |
| **Step-back prompting** | Ask Claude to state the abstract principle before solving. Good for architecture questions. |
| **Least-to-most decomposition** | Solve the easiest subproblem first, chain results upward. Useful when one-shot keeps failing. |
| **ReAct (Reason + Act)** | Interleave reasoning with tool calls. Claude Code does this naturally — the pattern explains *why* it works. |
| **Context seeding** | Give Claude an `INDEX.md` or architecture map, let it pull what it needs. Better than stuffing all files in context upfront. |

### Model selection

**Plan with Opus / execute with Sonnet**
Expensive thinking upfront, cheap execution. Best of both worlds for complex tasks. Worth the setup friction on anything non-trivial.

**1M context vs subagents — genuinely contested**

*Case for 1M:* Cheaper long-term for deep sessions. No compacting = no data loss. Keeps full continuity across a long working session.

*Case for subagents on 200k:* 1M context makes you send more tokens per interaction than you actually need. Subagents + looping on 200k often gives better results at lower cost because each agent gets a focused, lean context.

→ No universal answer yet. Depends on your task profile. Share your findings.

---

## 7. WHERE HUMANS STILL WIN

### Case study: zero-alloc JSON streaming library

Tried Claude (multiple versions), ChatGPT, Gemini. All failed.

- Code was unreadable
- Agents constantly skipped the hard optimisations
- Made up random approaches instead of solving the actual constraint
- Repeated the same mistakes across attempts, across models

**The lesson:** There is still space for human developers. The harder the constraint, the more models hallucinate "clever" workarounds instead of doing the real work. They pattern-match to solutions that *look like* the answer without satisfying the actual requirement.

Use AI to scaffold and accelerate. Don't use it to replace deep algorithmic thinking on genuinely hard problems. The model's confidence inversely tracks how much you should trust it on the hard stuff.

---

## 8. WHAT'S HAPPENING IN THE ECOSYSTEM

### Build an internal sharing platform

If your org has people building Claude tools and workflows independently, the single highest-leverage thing you can do is give them somewhere to share. A simple internal site (even a static HTML page Claude generated) where teams post their tools, scripts, and prompts compounds fast. The value isn't the platform — it's the visibility into what people are actually building.

### Build a shared skills registry

Maintain a shared repo of Claude skills/plugins your org has built. Even a `README.md` with links and one-line descriptions is enough to stop people rebuilding the same tool three times.

### Workflows worth stealing from others

**Sprint planning via diagrams:**
Board tool (Miro/Figma/FigJam) stickies → Claude converts to Jira tickets with correct templates/labels/epics → Claude reviews for gaps, rewrites engineer-heavy tickets as user stories → human final review. Turns a 90-minute meeting artifact into groomed tickets in 10 minutes.

**Compliance automation:**
Playwright + accessibility checker (axe-core etc.) → Claude maps violations to WCAG 2.2 criteria → generates a compliance report document with your branding. AI doing compliance work, not just code. Worth exploring if you have accessibility obligations.

**The localhost proxy trick:**
For browser-based POCs that need Claude access: tiny Node.js proxy on localhost that shells out to `claude -p` (already installed and authed locally). Zero extra infra, zero extra API key management, zero extra cost. Elegant for internal tooling prototypes.

---

## 9. COST REALITY CHECK

Claude usage tends to grow faster than teams expect. Token spend follows a power law — a small number of heavy users drive most of the cost, and autonomous/agent mode amplifies it further.

**What good cost governance looks like:**
- Track spend by team/project (CloudZero or equivalent tagging)
- Set anomaly alerts, not just monthly budgets
- Audit the heavy users — often they're doing something brilliant, sometimes they left a loop running overnight
- Treat token spend like cloud infra spend: visible, attributed, optimized

The cost isn't the problem. Invisible cost is the problem. Once it's visible, teams naturally optimize.

---

## APPENDIX: Prompt patterns quick reference

```
# Reflexion loop
Write [X]. Then review it as a senior engineer and fix anything you'd call out in a PR review.

# Confidence gate
Before acting, rate your confidence 1–10 that this approach is correct.
If below 7, explain what's uncertain and replan.

# Persona anchor
You are a [role], [years] experience with [context], [key trait].
Now: [task]

# Panel / jury
Dispatch this problem to 3 independent expert personas with different backgrounds.
Each works in isolation with no knowledge of the others' opinions.
Then synthesize their views and highlight where they disagree.

# Step-back
Before solving, state the general principle or pattern this problem is an instance of.
Then solve it.

# Context seeding (in CLAUDE.md or session start)
This repo has an INDEX.md at the root describing the architecture.
Read it before exploring code. Pull only the files you need.
```
