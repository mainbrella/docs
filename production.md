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
