I’d organize Mainbrella as a shared platform with independently deployable products, then build a curated app catalog on top. More standalone apps could be useful, but I’d establish the code and data boundaries before creating more repositories.

After looking through the repos, you’re already partway there:

- Admin and marketing have separate Workers, service bindings, and checks that prevent bundling each other’s implementation.
- Builder has its own frontend build and deployment, while its backend remains in the main API.
- Sandboxes and production machines share the container engine and account controller.
- ai-receptionist is a standalone edition; hosted Voice still runs from backend and web. Its README (/Users/aa/mainbrella/ai-receptionist/README.md:5) explicitly describes the standalone version’s different authentication and billing model.

A product, a deployment, and a Git repository are separate boundaries. Appflare itself demonstrates this: its manager, installer, documentation, and shared packages live together, while catalog applications come from many upstream repositories.

I’d give Mainbrella these ownership boundaries:

| Part | What it should own |
|---|---|
| Platform | Accounts, sessions, API keys, permissions, project identity, wallet, usage receipts, shared audit events |
| Compute | Container provisioning, images, execution, terminals, files, workspaces, network controls |
| Sandbox and production policies | Different lifetime, restart, persistence, and hosting behavior over the shared compute engine |
| Builder | Conversations, agent orchestration, source changes, Git history, build operations; consumes compute and platform services |
| Voice | Receptionist settings, conversations, telephony, transcripts; consumes platform services in its hosted edition |
| Raindrop | Operator workflows over administrative APIs |
| Site and console | Public content and shared account management, with product workspaces deployed separately where useful |

I would keep sandbox and production compute together initially. Your container controller (/Users/aa/mainbrella/backend/containers/container-account-core.js:25) already composes production, lifecycle, workspaces, pools, and billing. Separate policies make sense; maintaining two implementations of execution, storage, and container ownership would create extra work.

The most important extraction is product-neutral platform interfaces. Two current couplings make that clear:

- Voice billing (/Users/aa/mainbrella/backend/worker/lib/voice-billing.ts:1) imports Builder’s pricing helpers.
- The wallet API (/Users/aa/mainbrella/backend/worker/lib/prepaid-billing.ts:65) reaches the CONTAINER_ACCOUNT Durable Object.

That works today, but future email or search services should depend on “reserve and settle usage,” without knowing about Builder or containers. I’d introduce that interface over the existing wallet first, preserving its authoritative state and reconciliation behavior.

Project ownership needs similar attention. Your project model (/Users/aa/mainbrella/backend/docs/projects.md:3) already separates project identity from an optional Builder workspace. However, Builder deletion currently removes the shared project. Once a project contains several products, removing its Builder workspace should preserve its other resources; deleting the whole project should explicitly coordinate their removal.

For repository organization, my preferred destination is one hosted-platform monorepo:

mainbrella/platform/
  apps/
    site/
    console/
    builder/
    admin/                 # Raindrop

  services/
    platform/              # Public API, accounts, projects, billing
    compute/
    builder/
    voice/
    admin/
    marketing/
    ingress/

  packages/
    contracts/
    platform-client/
    ui/
    voice-engine/

These folders describe ownership; they don’t all need new Worker deployments immediately. Existing Worker names, storage identities, URLs, and release histories can stay stable while source moves. Your existing service-binding approach supports independently deployed services behind a consistent API. Cloudflare documents that model directly. (https://developers.cloudflare.com/workers/runtime-apis/bindings/service-bindings/)

I’d keep native clients, the SSH gateway, independent products such as Parsequerade, and upstream forks separate where their release cycles justify it. There’s no need to put everything under ../ into one Git repository.

For standalone products, share an engine between hosted and self-hosted editions. Voice is the first place to prove this:

shared voice engine
  ├── hosted adapter: Mainbrella identity, wallet, permissions
  └── standalone adapter: local authentication, owner’s Cloudflare account

ai-receptionist can remain its own public distribution repository, consuming a versioned engine. The important outcome is that a telephony fix has one implementation and reaches both editions. Extracting further applications by copying hosted source would increase the number of places you have to maintain.

On the app-store question: there’s a credible direction here, with a different customer promise from Appflare. Appflare installs and manages applications inside the customer’s Cloudflare account. That is its stated purpose. (https://github.com/appflare/appflare) Mainbrella could offer applications operated for customers, sharing identity, billing, storage, and other services.

A customer might create one project and enable:

- A website built with Builder.
- A production deployment.
- A receptionist.
- Booking.
- A knowledge base.
- Transactional email.

The useful difference would be how these cooperate: the receptionist reads the project’s knowledge base, creates a booking, and sends a confirmation through its email service.

I’d start that catalog with your own products and a few carefully integrated applications. Each entry needs more than a launch button: provisioning, configuration, permissions, health checks, updates, data retention, removal, and metering. Customer Worker applications and container applications will also need different deployment paths. Workers for Platforms is relevant when you start hosting customer-supplied or generated Workers. Cloudflare describes it as infrastructure for that use case. (https://developers.cloudflare.com/cloudflare-for-platforms/workers-for-platforms/)

You can define Mainbrella’s own catalog format and add Appflare compatibility through an adapter later. Mainbrella Voice could also be distributed through Appflare while you operate its hosted edition.

My next steps would be:

1. Define platform contracts for identity, project access, billing, and audit events, keeping current storage in place.
2. Make hosted and standalone Voice share maintained engine code.
3. Separate Builder’s backend ownership and deployment, matching its existing frontend boundary.
4. Establish table and migration ownership before splitting databases.
5. Add a small catalog once those products have reliable installation and removal lifecycles.

That would let you test the app-store idea using products you already have, while improving the architecture regardless of which business direction wins.
