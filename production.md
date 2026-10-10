# Current billing decision — October 2026

Mainbrella uses prepaid compute credit with a $5 minimum one-time purchase.
There is no new monthly subscription, monthly minimum, or postpaid usage
invoice. Purchased credit carries forward until consumed. The historical
proposal below is retained for context; this decision supersedes its billing
recommendations.

Compute remains $0.02 per compute-unit hour, weighted by machine size. The
account wallet stores exact lifetime weighted milliseconds, separate from the
UTC monthly spending cap. A month boundary resets the cap's usage window,
never the wallet's consumption or outstanding reservations.

Every runtime lease reserves existing paid credit before provisioning or
extending its deadline. Concurrent starts share one durable account ledger.
Only elapsed runtime is consumed; unused reservations are released after a
confirmed stop. Unreadable machines retain their reservations. New production
servers require enough funds and spending-cap room for 24 hours of the entire
desired fleet, including services waiting for recovery and the requested
server. Ad hoc reservations also reduce available funding. Existing production
services renew short funded leases and stop when funds or the cap run out.

The account menu and billing page show current balance. The billing page also
shows available and reserved credit, UTC monthly consumption, and production
runway. Optional automatic recharge requires explicit consent, a purchase
amount, and an independent monthly recharge limit. A recharge funds runtime
only after Stripe confirms a successful captured payment. Pending or failed
payments do not extend deadlines. Durable charge identifiers prevent duplicate
charges after ambiguous responses or eviction.

Stripe uses a one-time Mainbrella Compute Credit Product and a $5 USD one-time
reference Price (`STRIPE_PREPAID_PRICE_ID`). Larger purchases use the same
Product with the selected purchase amount. Development and production have
separate test/live Price IDs. Verified completion and signed payment, refund,
and dispute webhooks update the wallet idempotently. Refunds and disputes
remove funding and fence affected running or pending leases; delayed success
events cannot restore reversed credit.

All purchases receive credit dollar for dollar, with the same account limits
and compute rate. Thirty-six $5 purchases equal one $180 purchase; two hundred
$5 purchases equal one $1,000 purchase. There are no volume bonuses or purchase
tiers. Each start reserves paid runtime, including startup and idle time, with
no separate start fee. The existing 10,000 new-starts-per-UTC-month limit remains
an abuse guard; idempotent creation retries do not count as additional starts.

## Historical proposal

Yes. I think MainBrella should move toward usage-based pricing now, before you add more infrastructure services. But I wouldn't copy AWS's complicated pricing model. I'd copy the simplicity of Railway or Fly.io.

There's something interesting about your existing pricing that makes this transition particularly natural.

## 1. Your three plans already charge almost exactly the same rate

| Plan    | Monthly price | Compute-unit hours | Effective rate   |
| ------- | ------------- | ------------------ | ---------------- |
| Builder | $5            | 250                | $0.0200/CU-hour  |
| Pro     | $180          | 9,000              | $0.0200/CU-hour  |
| Scale   | $999          | 50,000             | $0.01998/CU-hour |

You're essentially selling the same compute at the same price, but making customers buy increasingly large buckets of it upfront.

A customer who wants $40 worth of MainBrella has to choose between a $5 plan that's too small and a $180 plan that's too expensive.

That's the biggest pricing problem I'd fix.

## 2. What successful infrastructure platforms do

Railway

A $5 monthly minimum that includes $5 of usage. If a customer consumes $18 of resources, the total is $18, not $23. This is very close to what I would implement for MainBrella.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.railway.com\&sz=32)

Railway Docs



Fly.io

Pay-as-you-go compute, storage, network traffic, and dedicated IPs, with published unit prices. A straightforward model for infrastructure that can grow into many services.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://fly.io\&sz=32)

Fly



Render

Combines workspace subscriptions with separately metered compute, bandwidth, and custom-domain usage. This shows how platform features and resource charges can coexist.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://render.com\&sz=32)

Render Changelog



## 3. The MainBrella model I'd implement

# MainBrella

## $5/month minimum

Includes $5 of usage. Pay only for additional resources consumed.

Example monthly resource rates

| Lite container       | $0.02/hour       |
| -------------------- | ---------------- |
| Small container      | $0.12/hour       |
| Medium container     | $0.20/hour       |
| Persistent storage   | Per GB-month     |
| Dedicated IP address | Per IP-month     |
| Email sending        | Per 1,000 emails |
| Network egress       | Per GB           |

Compute prices reuse your current implied rates. Other service rates would need to be set after supplier-cost analysis.

Explore a monthly estimate

Temporary Lite sandbox hours

100h

One always-on production machine

Small

Estimated monthly bill

# $88.40

Assumes a 30-day month, no other billable resources, and a $5 monthly minimum credited toward usage.

This solves your original problem automatically: the user can spend $90 on an always-on server, $90 on temporary agents, or any combination.

## 4. What happens to $180 and $999?

I'd eventually turn these into optional monthly spending commitments, rather than required plans for larger workloads.

| Offering        | How it works                                                                          |
| --------------- | ------------------------------------------------------------------------------------- |
| Standard        | $5 minimum, credited toward any resource usage                                        |
| $180 commitment | At least $180/month in usage, with optional volume benefits                           |
| $999 commitment | At least $999/month in usage, with optional volume benefits and higher service limits |

There's no reason to push people into the higher commitments today when your effective compute rates are already identical.

The higher commitments only become compelling once you can provide tangible advantages, such as volume discounts, more concurrency, stronger support, or contractual guarantees.

## 5. The important safeguards

Pure usage-based billing introduces risks that your current fixed plans avoid. I'd implement these before enabling unlimited overages:

- Spend limits: Customers choose a maximum monthly bill, with alerts at 50%, 80%, and 100%.
- Cost estimates: Before starting an always-on machine, show its estimated monthly cost, as well as costs for IPs and storage.
- Production protection: Warn users prominently if a spending cap could take a production service offline. Don't silently stop a critical application.
- Abuse protection: Require a payment method or small prepaid balance before allowing significant resource consumption, with limits on provisioning rates and concurrency.

Railway already offers usage limits and alerts, which is useful precedent, though exceeding its hard limit can shut down workloads.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.railway.com\&sz=32)

Railway Docs



## 6. Don't become AWS just because you can

I would distinguish between building an infrastructure platform and exposing every infrastructure primitive.

MainBrella's advantage should be that somebody can say:

> Run this GitHub repository, give it a permanent URL, add a database, and let me know what it costs.

They shouldn't have to understand the mechanics of provisioning VMs, gateways, networks, domains, certificates, and storage.

Likewise, an email service doesn't necessarily mean operating your own mail-delivery infrastructure. You could integrate an established provider and meter usage through MainBrella.

My recommendation: Keep the $5 monthly minimum, replace the fixed compute quotas with metered billing, and let the same account balance cover all infrastructure services. De-emphasize $180 and $999 subscriptions until customers have a demonstrated reason to commit at those levels.

This gives you a pricing system that can grow from sandbox agents into broader cloud hosting without redesigning your plans each time you add a new service.


## MainBrella: Can sandboxes run 24/7 as production servers?

Short answer: Cloudflare's underlying infrastructure can support long-running containers, but MainBrella's current implementation is designed for temporary workloads.

The distinction matters: you don't need to restrict MainBrella to AI agents, and there's an opportunity to support permanent application hosting. But changing the maximum runtime alone won't make it production-ready.

I checked the current source, including the latest commits to [mainbrella/backend](https://github.com/mainbrella/backend) and [mainbrella/web](https://github.com/mainbrella/web).

## 1. Current MainBrella limits

The authoritative configuration is [`containers/plan-policy.js`](https://github.com/mainbrella/backend/blob/main/containers/plan-policy.js).

|                      | Builder | Pro      | Scale    |
| -------------------- | ------- | -------- | -------- |
| Maximum session      | 1 hour  | 24 hours | 72 hours |
| Idle shutdown        | 10 min  | 30 min   | 60 min   |
| Max containers       | 5       | 100      | 500      |
| Persistent disk      | No      | No       | No       |
| Filesystem snapshots | No      | No       | No       |

That means even the Scale plan cannot currently keep a particular container generation running indefinitely.

More importantly, the idle timeout means that an otherwise healthy application could be shut down after an hour of inactivity, even though the customer intends to keep it deployed.

These are MainBrella product restrictions, not maximum runtime limits imposed by Cloudflare.

## 2. Saved workspaces are already supported

There's an important qualification to the snapshot row above: I found a newer, separate workspace-persistence implementation that the plan-policy feature summary doesn't reflect.

The production configuration sets `WORKSPACE_PERSISTENCE_ENABLED=true`, and [`docs/workspaces.md`](https://github.com/mainbrella/backend/blob/main/docs/workspaces.md) records successful production qualification on October 6.

So MainBrella does support manually saving and restoring a container's filesystem. Retention is 7 days on Builder, 14 on Pro, and 29 on Scale.

However, saving is explicit, not automatic. Restoring creates a new machine generation and does not restore running processes, RAM, or existing network connections. A container that crashes unexpectedly can still lose unsaved data.

That's useful for agents, development environments, and reproducible deployments. It isn't equivalent to a persistent production disk.

## 3. The other production blockers I found

Hard deadlines are enforced in code.

[`user-container-core.js`](https://github.com/mainbrella/backend/blob/main/containers/user-container-core.js) sets both expiration and idle deadlines and schedules an alarm that destroys expired containers. Simply leaving an application process running won't bypass them.

There's no automatic application resurrection.

When the machine stops, the controller records that state. It doesn't automatically start a replacement container and restart the customer's application. In fact, [`executions.js`](https://github.com/mainbrella/backend/blob/main/containers/executions.js) deliberately stops a generation when recovering interrupted managed execution.

Stable project domains don't mean permanent hosting.

Your newest backend work adds default application endpoints and Cloudflare custom-domain configuration. But [`project-domains.md`](https://github.com/mainbrella/backend/blob/main/docs/project-domains.md) explicitly says those endpoints do not start machines or extend leases, and replacement generations must be republished.

Continuous stateful services need external storage.

Hosting an application using D1, external Postgres, R2, or another durable database is viable. Depending on a container's writable local filesystem for production database data is not.

## 4. Cloudflare itself does support long-running workloads

Cloudflare's documentation is quite clear:

> Cloudflare does not stop a container instance after a fixed maximum runtime.&#x20;
>
> [image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)
>
> Cloudflare Containers docs
>
>

However, Cloudflare also says instances can be interrupted for host maintenance, and it does not guarantee uninterrupted execution for any fixed duration. A stopped container can restart on different hardware.

Cloudflare's native inactivity timeout supports up to six hours. Beyond that, you can use Durable Object alarms and application activity to maintain a service, but you still need to recover from infrastructure interruptions.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Sandboxes docs

+1



So yes, MainBrella can evolve into a production application host without abandoning Cloudflare Containers. The platform doesn't force an agent-only usage model.

## 5. The economics of 24/7 hosting

Your existing monthly compute-unit budgets are another obstacle.

Here is the cost of one container running continuously for a 30-day month.

| Size   | MainBrella compute-unit hours | Cloudflare RAM + disk baseline |
| ------ | ----------------------------- | ------------------------------ |
| Lite   | 720                           | $1.98                          |
| Small  | 4,320                         | $27.37                         |
| Medium | 7,200                         | $41.06                         |
| Large  | 11,520                        | $54.74                         |
| XL     | 20,160                        | $81.39                         |

Cloudflare estimates are gross provisioned-memory and disk costs for 720 hours, before included account allowances, active CPU charges, network egress, Workers, Durable Objects, and other costs. Based on Cloudflare's October 5, 2026 rates.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Containers docs

Your Builder plan includes just 250 compute-unit hours/month, so even one continuously running Lite machine exceeds its allowance. Pro includes 9,000, enough in principle for one continuously running Medium machine, but not a Large, assuming no other usage.

Production hosting needs a different usage model, including automatic monthly budget renewal, rather than fixed-length reservations.

## 6. What I would build

I would preserve the current sandbox product and introduce a separate deployment mode.

PROPOSED MAINBRELLA ARCHITECTURE

## One runtime, two lifecycle policies

Sandbox

Temporary development and agent tasks

Time-limited

Idle shutdown

Explicit snapshots

Disposable processes

Deployed App

Continuously available applications

No session deadline

Automatic recovery

External durable data

Stable HTTPS endpoint

Deploy sandbox as app

Capture a deployable image or build, define a startup command, and publish a stable service

The changes I would prioritize are:

1. Persistent desired state. Store whether an app should be running, its image, startup command, environment, size, and routing configuration in Durable Object storage.
2. Self-healing supervision. If the container exits, is killed, or becomes unhealthy, automatically provision a replacement using bounded retries and backoff. Check actual HTTP readiness.
3. Stable service identity. Keep the project's domain attached to the logical application, not one container generation. Automatically update its backend binding after replacement.
4. External state. Use D1, R2, or managed databases for durable application data. Existing snapshots can support deployment and recovery, but not live transactional persistence.
5. Production billing. Offer continuously allocated resources with metered or suitably priced monthly usage, rather than hard session limits.
6. Availability controls. Support deploy rollbacks, application logs, failure alerts, and eventually multiple replicas for higher availability.

For the first release, I'd target stateless HTTP applications with external databases. I wouldn't promise conventional VPS behavior, guaranteed uninterrupted Linux processes, or high availability yet.

## My recommendation

Build production app hosting on top of your existing sandboxes, but keep the products separate.

The interesting user experience is:

Clone a GitHub repo → build and test in a sandbox → click Deploy → get a permanent HTTPS application URL.

Your latest commits are already moving in this direction: [backend ](https://github.com/mainbrella/backend/commit/1d23bf43bbdae3cfae7eab1ff061f950c6772684)[`1d23bf4`](https://github.com/mainbrella/backend/commit/1d23bf43bbdae3cfae7eab1ff061f950c6772684) adds Cloudflare for SaaS configuration, and [web ](https://github.com/mainbrella/web/commit/738e28d77a4721fe5b989e65bacb1fcca877322b)[`738e28d`](https://github.com/mainbrella/web/commit/738e28d77a4721fe5b989e65bacb1fcca877322b) surfaces project preview URLs.

The missing piece is not fundamentally compute or networking. It's the lifecycle supervisor that turns an ephemeral machine into a continuously available service.

That is a tractable extension to your existing architecture, and it would give MainBrella a much broader use case than ad-hoc agent workspaces.
