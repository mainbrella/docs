# MainBrella: A Repeatable Customer Acquisition System

The guide you shared has an excellent central idea: don't just buy clicks. Build a system that learns which clicks become valuable customers, then use that information to acquire better customers.

I would apply that same eight-step process to MainBrella, with one important adjustment.

The original guide is designed for expensive products that require a salesperson to close a deal. MainBrella is developer infrastructure, where customers can purchase compute and begin using the product without speaking to anyone.

That changes our definition of a successful lead.

THE MAINBRELLA GROWTH LOOP

Attract

An ad shows a useful developer workflow

Capture

Collect a GitHub repo URL and optional contact

Activate

Developer successfully runs a real workload

Qualify

Identify developers likely to use more compute

Retain

Developer returns, funds and runs additional workloads

Improve

Feed quality signals into the next acquisition round

Repeat, measure, and optimize

Our primary goal should be acquiring activated developers, not email addresses or $5 purchases.

I also checked your latest GitHub changes from October 10. MainBrella is moving to prepaid compute with a $5 minimum, no new monthly subscription, and support for promotional discounts up to 100%. That's useful for this strategy: we can offer selected developers free compute credit without requiring a card, while retaining control of the cost. This should be verified in production before we advertise it.

Here's how I'd structure the first 60 days.

## Step 1 — Pick one buyer and work backward from revenue

MainBrella has two potential customer types, but I wouldn't market to both equally.

| Customer                     | What they need                                                                  | Acquisition approach                                     |
| ---------------------------- | ------------------------------------------------------------------------------- | -------------------------------------------------------- |
| Individual developer         | Run a repo, experiment with agents, occasional Linux compute                    | Self-service, demos, free compute promotions             |
| AI startup / agent developer | Sandboxes embedded into a product, repeat workloads, larger compute consumption | Targeted ads, personal onboarding, ongoing relationships |

I would optimize paid acquisition for the second group, while making the first group exceptionally easy to activate.

The economic formula from your guide becomes:

\\[ \text{Max CPL} = C\_{90} \times P \times 0.75 \\]

Where \\(C\_{90}\\) is expected 90-day contribution per paying account, \\(P\\) is the percentage of leads who become paying customers, and 0.75 leaves a 25% safety cushion.

ILLUSTRATIVE ECONOMICS — NOT ACTUAL MAINBRELLA DATA

| Scenario              | Casual developer | Heavy-use developer |
| --------------------- | ---------------- | ------------------- |
| 90-day contribution   | $30              | $300                |
| Lead → paying user    | 10%              | 10%                 |
| Maximum cost per lead | $2.25            | $22.50              |

This matters enormously. If a $5 prepaid customer is unlikely to buy additional compute, paid ads may never be economical.

For prepaid billing, measure contribution from paid compute actually consumed, after infrastructure, payment, and support costs. Don't mistake adding $20 to a wallet for $20 of earned profit.

## Step 2 — Create an offer people actually want

I wouldn't advertise "Linux sandboxes with idempotent APIs." Those features matter, but they don't give most people a reason to click.

Instead, lead with a concrete demonstration:

PROPOSED CAMPAIGN OFFER

## Give us a GitHub repo. We'll help you run it in the cloud.

Paste a public GitHub repository. Use ChatGPT or Claude to prepare its setup, launch a Linux container, and open a live preview when the project supports one.

Selected pilot developers receive $5 in compute credit.

Offer contingent on production verification of the new 100%-discount checkout flow.

Your existing [GitHub repository runner](https://mainbrella.com/run/) is a strong foundation for this. It's more demonstrable than a general-purpose sandbox API.

I would use it as the entry point and then introduce the bigger product: integrate MainBrella into an agent that needs machines repeatedly.

## Step 3 — Build the ad inventory

Following the guide, start with three distinct concepts and two hooks per concept. Use real screen recordings, not elaborate AI-generated marketing art.

Concept A — The GitHub challenge

"I gave Claude a random GitHub repo and asked it to get the project running on a cloud Linux machine. Here's what happened."

20–30 seconds: show the prompt, setup, terminal output, and real preview URL.

Concept B — Your AI agent needs a computer

"Claude can write the code. But where is it going to run it? Give your agent a Linux computer through an API."

Show a short API integration, container creation, command execution, and cleanup.

Concept C — Stop paying for idle compute subscriptions

"Need a Linux sandbox occasionally? Start with $5 prepaid compute. No monthly subscription."

Show a real machine starting, the balance screen, and transparent usage.

Test these as Meta video ads with a single broad ad set, allowing the creative to identify interested developers.

But I would treat Meta as a hypothesis, not the inevitable winning channel. Developers researching sandbox providers may be easier to reach through Google Search and Reddit. A 2026 developer-tool advertising case study also supports testing narrow, intent-based Reddit acquisition rather than generic broad social ads.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://promoted.com\&sz=32)

Promoted



## Step 4 — Give cold traffic a purpose-built landing page

Currently, MainBrella's homepage presents an SDK snippet, platform features, and a purchase CTA. That's useful for developers already evaluating the product, but it's not the landing experience I would give someone from a 20-second video.

I would build `/try/` around one action.

mainbrella

Landing page concept

# Run your GitHub project in a cloud Linux machine.

Paste a public repository. Get setup instructions and launch it in your own sandbox.

Public GitHub URL

Watch a real 45-second walkthrough

GitHub URL → AI-generated setup → Linux container → live preview

Pilot offer: $5 compute credit

No card needed with an eligible fully discounted checkout.

Illustrative layout; input intentionally nonfunctional in this concept.

The experience should be:

1. Visitor pastes a real repo URL.
2. MainBrella recognizes the repo and shows what will happen.
3. Visitor receives the prompt to run through their ChatGPT or Claude account.
4. Email and account creation happen when they want to execute in MainBrella, not before they understand the value.
5. Successfully opening a preview or executing a meaningful workload completes activation.

The lead capture event should require an actual contact plus intent, such as an email and repository URL. Anonymous repo submissions are useful engagement events, but shouldn't be reported as leads.

This is also where I'd invest engineering time in clearer setup errors. If visitors get stuck importing AI-generated configuration, paying for more visitors won't help.

## Step 5 — Build the data spine

The biggest transferable lesson from the document is that advertising data should connect directly to product usage.

I would create a small analytics layer in the existing Cloudflare backend with a lead record, associated account, attribution fields, and append-only product events.

| Event                   | Meaning                                                     | Purpose                    |
| ----------------------- | ----------------------------------------------------------- | -------------------------- |
| `repo.submitted`        | Valid public repo entered                                   | Intent                     |
| `lead.captured`         | Email plus intent saved                                     | Initial ad conversion      |
| `user.created`          | Account registered                                          | Signup                     |
| `workspace.started`     | Container running                                           | Progress                   |
| `workload.activated`    | Real repository-specific command or workload succeeds       | Primary product activation |
| `preview.opened`        | User views running web app                                  | Strong engagement          |
| `developer.qualified`   | Repeat real workload or other verified high-intent behavior | Quality feedback           |
| `wallet.funded_paid`    | Real-money top-up completed                                 | Monetization               |
| `compute.consumed_paid` | Purchased balance consumed                                  | Revenue analysis           |

Store UTM campaign data, page variant, ad click identifiers when permitted, and an immutable reference connecting the initial lead to the eventual account.

For each failed launch, capture the actual failure stage: AI configuration, dependency installation, checkout, container startup, or preview networking.

That gives us something especially useful: we'll know whether our marketing failed or the product failed after bringing in an interested person.

## Step 6 — Build the activation and follow-up process

I wouldn't call every $5 developer like the original guide suggests for expensive human-closed sales.

Instead, automate basic assistance and personally engage with the developers who show evidence of potentially substantial usage.

| Trigger                              | Follow-up                                                          |
| ------------------------------------ | ------------------------------------------------------------------ |
| Submitted a repo but didn't register | Email the personalized next step, if they opted in                 |
| Signed up but never launched         | Help finish setup                                                  |
| Launch failed                        | Offer specific debugging help based on the error                   |
| First successful workload            | Show how to integrate the same workflow into an AI agent           |
| Second successful workload           | Ask what they're building and what's missing                       |
| Meaningful recurring usage           | Offer founder-level support for their production or agent workflow |

Initially, I would personally work with about 20 pilot developers, including watching some complete onboarding.

Their objections become the next ad hooks, exactly as the source guide recommends.&#x20;

Pasted text.txt



## Step 7 — Close the feedback loop with Meta

Once the product events are reliable, connect Meta Pixel and server-side Conversions API.

The first ad optimization event should be `Lead`, fired only after your backend successfully accepts a real contact and developer intent. Match the browser and server event IDs to avoid duplicates, and respect tracking consent.

Separately send an `ActivatedDeveloper` or `QualifiedDeveloper` event when the person demonstrates real usage.

Do not immediately switch Meta's primary goal to that deeper event. First collect enough volume to see whether the label predicts retention and purchases. A common planning benchmark is roughly 50 optimization events per ad set per week, which a brand-new developer tool with a modest ad budget may not reach.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://adseditor.app\&sz=32)

adseditor.app



Only after sufficient conversions should we test optimizing for qualified developers instead of ordinary leads.

This is the mechanism that eventually makes the system smarter rather than merely bigger.

## Step 8 — Run a 60-day experiment with gates

I would approve a maximum initial advertising budget of $1,000, released in stages rather than committed all at once.

| Timeline   | Work                                                                            | Go/no-go evidence                                             |
| ---------- | ------------------------------------------------------------------------------- | ------------------------------------------------------------- |
| Days 1–7   | Landing page, lead endpoint, attribution, activation events, promo verification | Complete end-to-end tracking test                             |
| Days 8–14  | Personally recruit 20 developers and give controlled credit                     | Target: 10 complete real workloads, with friction documented  |
| Days 15–28 | One Meta campaign, six creatives, roughly $35/day                               | Measure actual cost per captured lead and activated developer |
| Days 29–42 | Fix biggest funnel leak; continue Meta or test high-intent Search/Reddit        | Evidence of repeat use and first real paid top-ups            |
| Days 43–60 | Improve successful ads; send verified quality events back                       | Acquisition cost plausibly supported by expected contribution |

The $35/day Meta experiment is approximately $490 over two weeks. At that level, we'd be testing positioning and onboarding, not claiming Meta has fully learned the audience.

The most important stop rule: if the first 20 carefully recruited developers mostly can't complete a real workload without your help, don't spend $490 sending hundreds more people into that process.

## The scoreboard

I'd build one internal dashboard, probably in Raindrop or the MainBrella admin system, and inspect it every Monday.

MainBrella acquisition dashboard

Illustrative values only — not actual campaign data

# $490

Ad spend

# 75

Leads captured

# 24

Activated developers

# 9

7-day repeat users

# 5

Self-funded users

# $20.42

Cost per activation

Example conversion funnel

Landing visitors

600

Leads

75

Activated

24

Repeat users

9

Paid users

5

Report both marketing spend and infrastructure credit costs, plus 7-day retention and 90-day contribution by acquisition cohort. Separate real paid top-ups from promotional credit.

## What I would prioritize first

Given MainBrella's difficulty acquiring its initial customers, I wouldn't begin by building the entire sophisticated Meta feedback system.

I'd begin with three concrete things:

- A single compelling offer: run a real GitHub project in a MainBrella sandbox using your existing ChatGPT or Claude account.
- A controlled first-use promotion: distribute $5 of prepaid compute to 20 carefully selected developers, with no card required after verifying the production checkout flow.
- A complete activation funnel: measure exactly where those 20 developers succeed, get stuck, return, or decide the product isn't useful.

The reason is that the original guide assumes the underlying offer already converts. MainBrella hasn't established that yet. Advertising algorithms can find more people resembling good customers, but they can't manufacture product demand that hasn't been demonstrated.

Our first milestone should be 10 developers who successfully use MainBrella, followed by 5 who independently come back and use it again. Not impressions, not Reddit reactions, not even 100 signups.

If we can demonstrate that behavior, we'll finally have something concrete to build the repeatable paid acquisition machine around.&#x20;
