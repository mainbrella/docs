Yes. This is one of the biggest risks of building your Supabase competitor on Cloudflare—but there's good news: Cloudflare has products specifically designed to run untrusted customer JavaScript securely.

You need to distinguish two completely different risks:

1. Sandbox security: Can malicious customer code steal other customers' data, access your Cloudflare credentials, or compromise your infrastructure?
2. Platform abuse: Can customers use your service for phishing, spam, malware distribution, scraping, or attacks against other websites—leading Cloudflare to suspend your account?

Cloudflare can help considerably with the first. You remain responsible for managing the second.

## 1. You don't actually need isolated containers

Cloudflare's [Workers for Platforms](https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms/?utm_source=chatgpt.com) explicitly supports hosting JavaScript written by untrusted customers.

Cloudflare uses V8 isolates rather than giving each JavaScript application a dedicated Linux container. Different Workers may share an operating-system process while maintaining isolated memory and execution contexts. Additional process-level sandboxing provides defense in depth.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare for Platforms docs

+1



Here's how I'd evaluate your options:

| Execution method                             | Suitable for untrusted customers?      |
| -------------------------------------------- | -------------------------------------- |
| `eval()` inside your main Worker             | &#x20;No                               |
| Import user JS into your shared API Worker   | &#x20;No                               |
| Workers for Platforms, untrusted mode        | &#x20;Yes                              |
| Dynamic Workers, tightly scoped capabilities | &#x20;Yes                              |
| Cloudflare Sandbox containers                | &#x20;Yes, for workloads needing Linux |

Workers for Platforms defaults to untrusted isolation mode. Dynamic Workers can be given only specific capabilities without inheriting your parent Worker's credentials or bindings.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare for Platforms docs

+1



So the absence of a Linux container is not the fundamental problem. Accidentally giving untrusted JavaScript powerful credentials, unrestricted networking, or shared customer resources is.

## 2. Could Cloudflare suspend your entire account?

Yes, although a single abusive customer does not automatically mean your whole service gets shut down.

Cloudflare explicitly says its policy is generally to take targeted action. It even gives an example of a large application platform with a customer violating its hosting policy, explaining that it would not automatically terminate the whole platform.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://blog.cloudflare.com\&sz=32)

Cloudflare Blog



However, its current policy allows hosting services to be suspended or terminated for repeated violations that aren't meaningfully addressed.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://www.cloudflare.com\&sz=32)

Cloudflare



And this is particularly important:

Cloudflare instructs customers to respond to abuse notifications within 24 hours. Failure to respond or address complaints can result in blocked content, disabled applications, or suspension of services associated with the account.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Fundamentals docs



That means your BaaS needs an abuse-response process from day one.

## 3. What malicious customers could actually do

| Attack                                | Risk to your business | Mitigation                                       |
| ------------------------------------- | --------------------- | ------------------------------------------------ |
| Phishing APIs and websites            | High                  | URL/content monitoring, suspend abusive projects |
| Sending spam through your email API   | High                  | Restricted sending and reputation controls       |
| Automated attacks against other sites | High                  | Outbound network restrictions                    |
| Scraping and abusive bots             | High                  | Destination policies and request quotas          |
| Mining or expensive computation       | Medium                | CPU limits and project budgets                   |
| Stealing other customers' data        | Critical              | Tenant-isolated execution and bindings           |
| Uploading malicious files to R2       | High                  | File scanning, access controls and takedowns     |
| Excessive AI or database usage        | Medium                | Quotas and spending limits                       |

Notice that the worst abuse doesn't require escaping a JavaScript sandbox. A customer with perfectly isolated code can still deploy a phishing endpoint or abuse an unrestricted `fetch()` capability.

## 4. How I'd protect your BaaS

Customer API Requests

App users and developers

Trusted Gateway Worker

Authentication · abuse detection · quotas · tenant routing

Managed BaaS APIs

Trusted code written by you

D1 · R2 · Auth · RLS

Untrusted JS Runtime

Isolated customer Workers

Scoped bindings · CPU quotas · restricted egress

Controlled External Access

Outbound gateway · permitted APIs · audit logs

These would be my minimum launch requirements:

| Protection                       | Implementation                                                                          |
| -------------------------------- | --------------------------------------------------------------------------------------- |
| Customer isolation               | Workers for Platforms in untrusted mode, or separate Dynamic Workers                    |
| No shared privileged credentials | Give customer code only project-scoped capabilities                                     |
| Internet access                  | Default deny; explicitly authorize outbound destinations                                |
| Resource limits                  | Per-execution CPU/subrequest limits plus project budgets                                |
| Signup protection                | Verified accounts, anti-bot checks, risk-based billing verification                     |
| Abuse monitoring                 | Per-project logs, suspicious activity alerts, automated suspension                      |
| Incident response                | Monitored abuse mailbox, project kill switch, 24-hour response capability               |
| Account isolation                | Separate production BaaS infrastructure from unrelated business systems where practical |

Cloudflare already supports per-customer CPU and subrequest limits in Workers for Platforms.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare for Platforms docs



For networking, Dynamic Workers support:

```
globalOutbound: null
```

This disables direct outbound networking. You can subsequently expose narrowly scoped HTTP capabilities through your own gateway.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare Dynamic Workers docs



Workers for Platforms alternatively supports Outbound Workers, which intercept customer `fetch()` requests so you can log, allow, or block destinations. One caveat: this interception doesn't cover requests made from Durable Objects, so don't hand untrusted code unrestricted Durable Object networking capabilities.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://developers.cloudflare.com\&sz=32)

Cloudflare for Platforms docs



## 5. I'd make unrestricted JavaScript an advanced feature

There is a useful product distinction here.

Default: Managed Backend

Customers can create databases, define permissions, upload files, use authentication, configure triggers and execute predefined backend operations. They can't run arbitrary code with unrestricted networking.

Lower abuse exposure and easier to secure.

Advanced: Custom JavaScript Functions

Customers can run custom JavaScript in isolated Workers. Access to database operations, outbound HTTP, email and other services requires explicitly granted permissions.

Greater flexibility, with stronger account verification and monitoring.

You could allow isolated JavaScript that only touches its project's database even on a free tier, while requiring a verified paid account before enabling arbitrary outbound HTTP or email sending.

This preserves the key appeal of your Supabase alternative without immediately turning it into an unrestricted, anonymous compute platform.

## 6. One thing I'd do before launching

Contact Cloudflare's platform team, describe the exact BaaS architecture, and get guidance on abuse reporting, acceptable use, and account-level enforcement. I'd also keep this BaaS in a separate Cloudflare account from other important production businesses where feasible; account separation limits collateral damage but doesn't eliminate your responsibility to prevent abuse.

My conclusion: I would not avoid customer-authored JavaScript because it runs in V8 isolates rather than Linux containers. Cloudflare expressly designed these products for that use case.

I would, however, avoid giving anonymous developers unrestricted Internet access, email sending, or public content hosting.

The larger business risk is a malicious customer making your Cloudflare account look like an abuse platform, not someone breaking out of the JavaScript sandbox. And that's something you can substantially reduce through architectural restrictions, verification, monitoring, and rapid takedowns.
