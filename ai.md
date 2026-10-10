# MainBrella Studio — Build it. Run it. Keep it.

I think this could be a major new direction for MainBrella, and potentially a much easier product to sell than infrastructure for AI agents alone.

The idea is to combine three products into one:

1\. Vibe coding

Describe an application in English and watch AI build and modify it.

2\. Instant previews

Run a real application for a few minutes or hours and share the URL.

3\. Production hosting

Turn that same application into a persistent 24/7 service.

The central promise:

> "Describe what you want. MainBrella builds it, runs it, and hosts it. You don't need to figure out the infrastructure."

## 1. The most important architectural decision

Don't build another hosting system. Build a new AI creation layer on top of MainBrella's existing infrastructure.

I reviewed the current `mainbrella/backend`, `web`, and `docs` repositories, including the October 10 commits.

You already have much of the hard infrastructure:

| Capability                                          | Current MainBrella                       |
| --------------------------------------------------- | ---------------------------------------- |
| Isolated Linux environments                         | Implemented                              |
| Start and execute commands                          | Implemented                              |
| Clone and launch GitHub projects                    | Implemented                              |
| Temporary public previews                           | Implemented                              |
| Production lifecycle and automatic restart/recovery | Implemented in backend                   |
| Persistent public project endpoints                 | Implemented in backend                   |
| Custom-domain support                               | Implemented; rollout verification needed |
| Browser terminal and logs                           | Implemented                              |
| Prepaid usage-based compute                         | Implemented                              |

The important limitations are that production recovery starts from the configured image, not an arbitrary unsaved filesystem; HTTP health checks, automatic rollback, and persistent application disks are not yet covered by the first production release. Those become important for generated applications.

Sources: [production lifecycle](https://github.com/mainbrella/backend/blob/main/docs/production.md), [repository launcher](https://github.com/mainbrella/backend/blob/main/docs/repo-launches.md), [project domains](https://github.com/mainbrella/backend/blob/main/docs/project-domains.md).

The opportunity is to add an AI software engineer, a project editor, and a deployment workflow, while reusing the existing runtime, account, networking, and billing foundations.

## 2. Cloudflare Workers AI is a good fit

Cloudflare now has coding-capable models that make this practical without paying OpenAI or Anthropic prices for every generation.

I would start by benchmarking these three:

| Model on Workers AI | Input / 1M tokens | Output / 1M tokens | Role                             |
| ------------------- | ----------------- | ------------------ | -------------------------------- |
| GLM-5.3 Flash       | $0.15             | $0.50              | Default app builder              |
| Kimi K2.7 Code      | $0.95             | $4.00              | Complex coding and repairs       |
| GLM-5.3             | $1.40             | $4.40              | Difficult multi-step engineering |

These are Cloudflare's listed inference prices as of October 10, 2026, before MainBrella's markup. All three support tool calling, and GLM-5.3 Flash and Kimi also support vision.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Workers AI docs

+2



My initial default would be `@cf/zai-org/glm-5.3-flash`. It has a very compelling price and a large context window. But we should test actual completed-app quality rather than assume the cheapest model will give the lowest cost per successful app.

Illustrative inference cost for 100,000 input + 20,000 output tokens

$0$0.06$0.12$0.18$0.24GLM FlashKimi CodeGLM 5.3

Example workload only; excludes repeated agent loops, compute and retries.

The models must work as agents, not simple text generators. A single request asking an LLM to output an entire React app is unlikely to produce the reliability we want.

Instead, the model should have tools to read and edit files, install packages, execute commands, inspect build errors, and test the running application. Cloudflare's function-calling support enables that tool loop.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Workers AI docs



I would also put Cloudflare AI Gateway in front of inference from day one. That gives us model switching, usage analytics, rate limits, and spend controls, plus the option of other providers later without redesigning Studio.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Docs

+1



One operational constraint: paid Workers AI frontier models currently have a default limit of 20 requests/minute per model per account, or 50 using prepaid AI Gateway billing. We need a queue and concurrency limits rather than assuming unlimited simultaneous build agents.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Workers AI docs



## 3. What the user experience should look like

I would introduce a prominent Build an App entry point on MainBrella, leading to `/studio/`.

MainBrella Studio

/ My Projects / Expense Tracker

Preview running

Build

Build a beautiful expense tracker with a dashboard, categories, and charts.

You · 2 minutes ago

Created React application

Installed dependencies

Build passed

What would you like to change?

Live preview

YOUR DASHBOARD

Expense Overview

This month

## $2,458

Transactions

## 42

Preview • Expires in 1h 42m

Temporary preview · 2 hours

Share previewKeep online 24/7&#x20;

Conceptual interface, not an existing Studio screen. The expense data is illustrative.

A successful first session should feel almost effortless:

1. User types an idea.
2. An AI agent creates a real project, edits code, installs dependencies, and starts it in a MainBrella sandbox.
3. The live preview appears alongside the conversation.
4. User asks for changes, or clicks an element and describes what to change.
5. User chooses to share the preview temporarily or deploy permanently.

The important detail is that the generated app is real source code, not a mockup or a special proprietary UI format.

A useful extra feature would be Fix it. When a build or runtime test fails, the user can watch the agent inspect the logs, repair the application, and retry without copying errors manually.

## 4. Two deployment modes, one project

|                  | Temporary preview           | Production application                       |
| ---------------- | --------------------------- | -------------------------------------------- |
| Purpose          | Experiment, demo, share     | Real users and traffic                       |
| Duration         | 15 minutes to several hours | Until explicitly stopped or funding ends     |
| Compute          | Ad Hoc container            | Production lifecycle, or edge static hosting |
| URL              | Temporary shareable link    | Stable project URL or custom domain          |
| Source code      | Saved independently         | Saved independently                          |
| Failure recovery | Restart editing session     | Automatic runtime recovery                   |
| Billing          | Runtime and AI usage        | Hosting resources and AI usage               |

One implementation detail from your existing code matters here: the prepaid usage plan currently has a 30-minute Ad Hoc idle timeout. A promised two-hour demo must remain accessible for the selected duration, even with no visitors. Studio should introduce an explicitly funded preview lease rather than relying on the ordinary idle policy.

For production, there's another critical consideration.

Generated files in a temporary container cannot be the production deployment artifact.

If a container restarts, you must be able to reconstruct exactly the same application. I would use immutable, versioned releases:

`Project source → Tested build → Saved release → Production runtime`

Every generated revision should be saved durably, with Git-compatible history and source exports. R2 is a good home for source archives and build artifacts, with D1 holding the project and revision metadata.

For example, the same project can have revision 17 in a development sandbox while production is still serving the approved revision 15.

Also, I would not force every production app into a 24/7 Linux container. Static React/Vite apps should be eligible for inexpensive Cloudflare edge hosting. Full-stack apps that genuinely need a long-running Node, Python, or other server can use MainBrella's Production containers. Cloudflare supports static asset deployment separately from Containers.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Workers docs



## 5. Proposed architecture

MainBrella Studio UI

Chat · Live preview · Code · Logs · Publish

Studio Agent Controller

Cloudflare Worker + durable orchestration

Workers AI

Generate & reason

Agent tools

Read, write, run, test

MainBrella Linux Build Container

Git · npm · TypeScript · tests · development server

Immutable Release in R2

Source snapshot · build artifact · release metadata

Preview

Time-limited

Production

Continuously available

The AI controller should own only the reasoning and orchestration. All untrusted code execution happens inside the isolated MainBrella container, using existing file and execution APIs.

For long-running build turns, Cloudflare Workflows can checkpoint AI and tool steps, recover from interruptions, and let jobs continue after the browser tab closes. That's preferable to keeping one long HTTP request open.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Workflows docs



For visual quality, add Cloudflare Browser Run. It can capture screenshots and rendered HTML from the preview, giving the agent a way to inspect the result instead of blindly trusting that a successful compilation means the UI looks right.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Browser Run docs



## 6. What I would actually build, in order

I would divide the work into five milestones and avoid tackling every Lovable feature immediately.

| Phase                      | Work                                                                                        | Completion test                                                       |
| -------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------------------- |
| 0. Agent proof of concept  | Connect GLM-5.3 Flash to MainBrella tools                                                   | A prompt creates and repairs a working React app                      |
| 1. Studio MVP              | Chat, code generation, live preview, iterative editing                                      | User can create and modify an app without touching a terminal         |
| 2. One-click deployment    | Durable source storage, temporary preview leases, versioned releases, production deployment | A generated app survives container replacement and can roll back      |
| 3. Full-stack apps         | Database, authentication, secrets, backend templates, persistent data                       | Generated apps support real users and CRUD operations                 |
| 4. Polish and integrations | Visual editing, screenshot feedback, GitHub sync, custom domains, mobile management         | A nontechnical user can manage an app from creation through operation |

For phase 0, start with a single supported stack: React + Vite + TypeScript, using a reliable starter template. Expand to Next.js, Python, Rust, and arbitrary Dockerfiles after the core generation loop works.

The first agent needs only a small set of tools: `list_files`, `read_file`, `write_file`, `apply_patch`, `run_command`, `get_logs`, and `open_preview`. Later, add `take_screenshot`, `run_browser_test`, and `publish_release`.

For full-stack applications, I would initially offer an integration with an external database provider. Native per-project Cloudflare D1 databases are a reasonable later option, but they require a properly isolated application data-access layer rather than exposing MainBrella's D1 credentials to user containers.

### Where this belongs in the existing repositories

| Repository                      | Main changes                                                                |
| ------------------------------- | --------------------------------------------------------------------------- |
| `mainbrella/web`                | `/studio/` UI, chat, editor, preview, publishing                            |
| `mainbrella/backend`            | Studio API, agent orchestration, model integration, deployment coordination |
| `mainbrella/backend` migrations | Project conversations, revisions, release metadata, AI usage                |
| `mainbrella/docs`               | Studio specifications, limits, pricing, supported stacks                    |

I would keep this inside the existing open-source MainBrella repositories rather than create a new standalone service.

## 7. Pricing: AI plus compute, no mysterious credits

This is where MainBrella could have a clear advantage.

Your current business model is already straightforward: customers purchase prepaid compute balance, and the runtime charges $0.02 per compute-unit hour.

I would extend that wallet to pay for AI inference as well, keeping the user-facing distinction between AI building costs and application hosting costs.

Estimated hosting prices at current MainBrella compute rates

| Two-hour Small preview  | $0.24  |
| ----------------------- | ------ |
| 30-day Lite production  | $14.40 |
| 30-day Small production | $86.40 |

Compute only, assuming the full allocated duration. Does not include AI calls, external storage or other services. Static edge hosting would use different pricing.

The product can show the expected price before every deployment.

I would initially charge around 2× the underlying Workers AI token rates, rather than invent a complicated message-credit system. That markup needs testing against retries, screenshot calls, support, and payment fees.

There are two necessary billing changes:

First, the existing compute-only wallet and accounting ledger need to track AI inference as a separate billable resource, with idempotent usage records and per-turn budgets.

Second, the default $5 monthly spending cap must be addressed during production deployment. A user choosing an $86.40/month service needs an explicit cap increase and sufficient funding; otherwise MainBrella will stop the service when the limit is exhausted. Studio should show production runway and offer auto-recharge without promising availability when funding is insufficient.

Cloudflare also charges a 5% credit-purchase fee when using AI Gateway Unified Billing, which should be included in the margin model if you use that funding path.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare AI Gateway docs



## 8. What MainBrella needs to do before claiming production readiness

The new feature must not publish a development server and call that production.

The production deployment gate should verify that the application builds reproducibly, starts from a saved release, answers an HTTP health check, and recovers successfully from a replacement container. Add release rollback, logs, and secrets management.

Generated applications also create a significant security obligation. AI-produced shell commands and dependencies run only in isolated user containers. The agent must never receive platform billing tokens or unrestricted Cloudflare credentials. Enforce per-project file access, resource limits, safe tool authorization, and separate preview origins.

Cloudflare Containers can run indefinitely in principle, but a container instance can still be interrupted for maintenance. The customer promise should be continuous service recovery, not that one Linux process literally stays alive forever.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Containers docs



One particular gap to check: existing temporary previews strip application cookies and Authorization headers. That works for demos, but authenticated generated apps need the separate project ingress behavior that preserves application credentials without exposing MainBrella platform credentials.

## 9. Why someone would choose MainBrella instead of Lovable

This is probably the most important strategic part.

Lovable already supports generating full-stack applications, Git synchronization, hosting, authentication, and code ownership. Simply adding a chat prompt and a preview would make MainBrella another competing app builder, not a clearly differentiated product.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.lovable.dev\&sz=32)

Lovable Documentation



I'd position the offerings like this:

|                        | Lovable                    | MainBrella Studio opportunity                             |
| ---------------------- | -------------------------- | --------------------------------------------------------- |
| Primary experience     | AI application creation    | AI application creation plus Linux infrastructure         |
| Generated code         | Editable and portable      | Editable, portable, open-source platform                  |
| Hosting                | Managed cloud hosting      | Temporary containers, static edge, or 24/7 Linux services |
| Languages and runtimes | Supported app stacks       | Eventually arbitrary Linux runtimes                       |
| Infrastructure access  | Managed integrations       | Terminal, SSH, services, networks, processes              |
| Platform ownership     | Commercial managed service | Fork and self-host MainBrella                             |
| Pricing proposition    | AI credits and hosting     | Transparent inference and resource usage                  |

The strongest differentiator isn't merely that MainBrella can deploy a React app. It's that the user can gradually grow from a simple app into a sophisticated service without leaving the platform or losing access to their infrastructure.

For example:

"Build me a dashboard."

Then:

"Add a Python worker that fetches data every hour."

Then:

"Give it a PostgreSQL database."

Then:

"Run two copies behind a load balancer."

The user should be able to achieve all of that conversationally, while a technical user can open the terminal and take over at any point.

That is an ambitious long-term product direction, but it's a stronger identity than trying to copy Lovable's entire feature list.

## 10. First experiment before investing heavily

Given the difficulty MainBrella has had getting early users, I would validate the core experience before building months of UI and integrations.

Take 10 realistic app prompts. Run each through a GLM-5.3 Flash agent connected to an existing MainBrella container, and measure how many actually produce a working preview.

Track completion rate, number of automatic repair attempts, time to usable app, AI inference cost, and compute cost. Repeat with Kimi K2.7 Code and GLM-5.3 where Flash fails.

For an initial prototype, I'd aim for at least 8 of 10 straightforward apps passing a functional browser smoke test. That is a proposed release gate, not a claimed model performance rate.

If the model can't reliably produce good-looking, functioning apps, improving the model/tool loop is more important than adding another feature to the dashboard.

## My recommended launch proposition

# MainBrella Studio

### From idea to live app, without touching a server.

Describe your app. Watch AI build it on a real Linux machine. Share it for an hour, or keep it running for years. Pay for the AI and infrastructure you actually use.

I would make Phase 0 and Phase 1 the immediate priority: one prompt, one app, one live preview, automatic debugging. Then solve durable deployment before advertising it as a production application builder.

The result would turn MainBrella's existing technical infrastructure into something people can understand in ten seconds. That's potentially a more valuable improvement than adding yet another infrastructure feature.
