I would make private GitHub repositories a first-class feature of MainBrella `/try`, using a GitHub App with read-only access. No SSH key setup, no personal access tokens, and no extra work inside the container.

The ideal user experience is:

PROPOSED MAINBRELLA /TRY FLOW

Run any GitHub repository

github.com/acme/private-app

This repository may be private.

Connect once

Connect GitHub to continue

Allow MainBrella to read only the repositories you select. Your code stays private.

Connect GitHub

GitHub handles the permission approval.

Repository access granted

acme/private-app

Private

Repository ready to import

MainBrella can clone your repository into an isolated Linux workspace.

Continue to setup&#x20;

GitHub Apps are designed for this. They let users select individual repositories and grant narrowly scoped permissions using short-lived credentials. GitHub itself recommends Apps over traditional OAuth apps for this type of integration.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs

+1



The important distinction: MainBrella gets permission to read one repository, not permission to control someone's GitHub account.

I would preserve the existing public-repository flow, introduce this connection only when needed, and make reconnecting unnecessary for future launches.

## The complication: ChatGPT and Claude

I looked at the current [/try page](https://mainbrella.com/try/). Your existing workflow has three steps: paste a public repo, prepare setup instructions using ChatGPT or Claude, then run those instructions in MainBrella.

Giving MainBrella access to a private repository does not automatically give ChatGPT or Claude access.

That's the bigger product problem to solve.

For private repositories, I'd support two setup paths:

Automatic setup — preferred

MainBrella clones the repository and examines files such as `Dockerfile`, `package.json`, `Cargo.toml`, `pyproject.toml`, and README instructions. It proposes setup commands, a start command, and the preview port without needing an AI API call.

The user reviews the resulting launch configuration.

Use your own AI — fallback

For complicated repositories, generate a prompt for Codex, Claude Code, or another coding agent that already has access to the repository.

That agent produces MainBrella's launch YAML, which the user pastes into `/try`, just as your current public-repo workflow allows.

This preserves your original requirement: MainBrella shouldn't have to pay for the user's AI inference.

## How I'd implement it

| Component            | Design                                                      |
| -------------------- | ----------------------------------------------------------- |
| GitHub integration   | One MainBrella GitHub App                                   |
| Permissions          | Repository Contents: Read-only; Metadata: Read-only         |
| Repository selection | Only selected repositories recommended                      |
| User verification    | GitHub App user authorization                               |
| Cloning              | HTTPS with short-lived credentials                          |
| Token storage        | Server-side only, never in the workspace                    |
| Preview access       | Private by default                                          |
| AI access            | Never send private code to an AI service without permission |

GitHub App user tokens are particularly useful because their effective repository access is restricted to the intersection of the user's access and the App's permissions. That prevents MainBrella from inadvertently giving one customer access to another customer's repository simply because the App is installed on both.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs

+1



The backend flow would look like this:

GitHub authorization

MainBrella backend validates user access

Generate repo-scoped clone credential

Clone repository through isolated importer

Remove credentials before handing off workspace

Detect setup → review → launch

GitHub installation access tokens expire after one hour and can be restricted to specific repositories. They can authenticate Git-over-HTTPS, so there is no technical need to install permanent SSH credentials in a container.&#x20;

[image](https://www.google.com/s2/favicons?domain=https://docs.github.com\&sz=32)

GitHub Docs

+1



Also, make the private source clone and preview access separate concerns. A private GitHub repository must not accidentally become a publicly accessible development server. Private previews should initially require authentication or an explicitly issued share link.

## What if someone doesn't want to connect GitHub?

I'd offer one secondary option: Upload a ZIP of your repository.

This skips granting persistent GitHub access, but still transfers the source code to MainBrella. It's useful for evaluating the service before trusting it with an ongoing integration.

Longer-term, SSH deploy keys can be an advanced option for GitHub organizations that cannot approve your App, or for other Git hosting providers. MainBrella could generate the deploy key and ask the user to add its public half to their repository. The private key would remain in MainBrella's protected credentials service, not inside the running workload.

I would not make that the default or put it in the initial `/try` experience.

## Important product decisions

There are three details I'd prioritize:

1\. Never describe an inaccessible repository as nonexistent. GitHub may return a 404 for private repositories. Offer "Connect GitHub" or "Check repository URL."

2\. Explain exactly what is stored. Read-only access prevents MainBrella from pushing to GitHub, but MainBrella still copies the code into a cloud machine. Tell users how long that code persists, how to delete it, and that revoking GitHub access does not automatically erase an existing clone.

3\. Don't introduce payment earlier in the flow. Your current page requires a prepaid balance to run a project. I'd let users connect GitHub, inspect their repository, and review the proposed configuration before asking for payment to launch. That gives them confidence the service can run their project before they pay.

## Suggested implementation order

For the first version, I'd build GitHub App installation, authenticated repository import, and automatic setup detection. Keep the existing AI-assisted YAML path for repositories that aren't automatically detected. Then add ZIP upload and advanced deploy-key support if demand justifies them.

The outcome should be that someone pastes a private repository URL, grants read-only access to that one repository, and reaches essentially the same launch experience as with a public repository.

The private-repository experience should feel like one extra GitHub permission screen—not like configuring a server.
