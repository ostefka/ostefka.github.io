---
layout: default
title: "Every AI can write an app now. Where should it run?"
description: "Copilot, Claude, Codex and Lovable can all produce a working business app in minutes. The open question for a large organisation is where that app runs. A first look at Copilot Managed Runtime, built hands-on with a non-Microsoft coding agent."
date: 2026-10-06
permalink: /managed-runtime/
---

<p style="font-size: 0.85em; color: #606c71; border-left: 3px solid #dce6f0; padding: 0.4em 0 0.4em 0.9em; margin: 0 0 1.8em 0;">
  <strong>Personal blog.</strong> This site reflects my own testing and my own
  opinions. It is not an official Microsoft statement and it is not official
  Microsoft documentation in any way. For authoritative guidance always refer to
  <a href="https://learn.microsoft.com/">Microsoft Learn</a>.
</p>

> **TL;DR** — Writing a business app is no longer the hard part. Any capable AI will produce one in
> minutes. The hard part in a large organisation is the boring question that follows: **where does it
> run, whose identity does it use, what data can it touch, and who can see that it exists?**
>
> **Copilot Managed Runtime** (public preview) is Microsoft's answer: a Microsoft-hosted runtime inside
> the Microsoft 365 tenant boundary where every app — whether a business user built it in the Microsoft
> 365 Copilot app (Code or Cowork), a maker in Copilot Studio, or a developer with *any* coding agent —
> gets Entra sign-in, governed data access, one inventory and one set of admin controls.
>
> It is **largely off by default**: an admin sets up app governance and opens the creation paths in the
> Microsoft 365 admin center, and apps run on Power Apps Premium or Copilot Credits.
>
> I built apps on it both ways: in **Copilot Cowork**, the natural route for business users, and with
> **Claude Code** — not a Microsoft AI — for the bigger ones. That says something about how open it is.

> **What this article is.** An introduction and an opinion, grounded in a few days of hands-on use in a
> lab tenant. It is not a step-by-step guide and not an official Microsoft statement. The product is in
> preview and changes weekly; observations were last checked on **6 October 2026**.

## The problem nobody asked for, and everybody now has

A year ago, "can we build an app for that?" meant a backlog, a budget and a quarter. Today it means a
prompt. Copilot, Claude, Codex, Lovable, Bolt, v0 — pick one, describe a deal tracker or a fee
calculator, and you get a working web app before your coffee is cold.

That is wonderful, and in a large organisation it is also a problem:

- The app needs **somewhere to run**. A developer's laptop, a personal cloud account, a SaaS builder's
  hosting — none of those is where a bank, an insurer or an asset manager wants its client data to go.
- The app needs **an identity**. The fastest route is always a service account or an API key pasted
  into the code — exactly what security teams spend years removing.
- The app needs **data**, and data is the whole point. The moment it reads a CRM table or a SharePoint
  list, someone has to answer "who can see what through this?"
- Somebody has to **know it exists**. Shadow IT used to be spreadsheets and Access databases. With AI it
  is going to be thousands of small, genuinely useful web apps.

Blocking AI app builders doesn't work for long — people find a way, and the useful ideas go underground.
The interesting question is not *whether* people will build apps with AI, but **whether there is a safe,
boring, enterprise-grade place for those apps to land**.

## What Copilot Managed Runtime is

In one sentence: **a Microsoft-operated runtime in your Microsoft 365 tenant where internal web apps run
with your identity, your data policies and your oversight, regardless of what built them.**

```
 Microsoft 365 Copilot app ─┐
   Code · Cowork            │
 Copilot Studio            ─┤                      ┌─► Dataverse, SharePoint, Outlook, Teams,
 ms CLI + SDK with         ─┼─►  Managed Runtime  ─┼─► Work IQ (Microsoft 365 Copilot),
   GitHub Copilot,          │    Entra sign-in     └─► Copilot Studio agents, 1,500+ connectors
   Claude Code, Codex       │    governed data           — always as the signed-in user
 Lovable                   ─┘    one inventory
```

A few things make it feel different from "yet another hosting option":

- **Many front doors, one runtime.** The most visible one is the **Microsoft 365 Copilot app** itself: its
  **Code** mode turns a plain-language description into a reusable app grounded in the user's work context
  (Microsoft describes it as running in a sandbox and hosted in the tenant), and **Cowork** can build simpler
  apps as one of its skills. A maker builds in Copilot Studio; a developer uses the `ms` command line with
  whatever coding agent they like. All of it ends up in the same place, governed the same way.
- **No secrets in the app, by design.** The app is a static web app running in a sandbox. It cannot call
  arbitrary endpoints. Every data call goes through a governed connector, under the signed-in user's own
  identity, through the organisation's data-loss-prevention and connector policies. There is no API key
  to leak because there is nowhere to put one.
- **Sharing an app does not share data.** If a colleague opens your app but has no access to the
  SharePoint list behind it, they get the app and not the data.
- **It is Git all the way down.** Every app has a repository; the platform builds each commit; preview
  and live are separate; going live is an explicit step; rollback is deploying an older commit.
- **One place for admins.** Every app, however it was built, shows up in the Microsoft 365 admin center —
  owner, region, usage, health — and can be blocked with one click.

If you know Power Platform, this will feel familiar underneath: environments, connectors and data
policies are the same machinery that governs Power Apps today. What is new is the code-first, AI-first
front end, the Git-based build pipeline, the admin-center inventory and consumption billing.

## One place to find every app

There is also, finally, **one place where people find their apps**: the portal at
[managedapps.cloud.microsoft](https://managedapps.cloud.microsoft). Everything a user built, everything shared
with them and everything they may open sits in one list with recents and search — whether it came from
the Copilot app, Copilot Studio or a developer's command line. AI-built apps have so far lived wherever the
tool that built them put them; this gives them a single front door.

![The app portal at managedapps.cloud.microsoft: recents, my apps, shared with me, all](/assets/managed-runtime/app-portal.webp)

*The app portal: the same list for apps built in Cowork and apps built with Claude Code.*

## It is (largely) off by default — and that is the right call

The runtime itself is present in every eligible tenant, but the doors into it are mostly closed until an
administrator opens them. The developer command-line path is switched off by default; app building in
Copilot Code and Cowork arrives through Microsoft's early-access Frontier programme; and governance has to
be set up. *(Measured: in the lab tenant, nothing could be built from the command line until an
administrator had done this. Per the documentation, app creation in Copilot Studio is the exception — on by
default during the preview, and controlled from the same place.)* In the Microsoft 365 admin center, **Apps →
Overview** walks an admin through setting up app governance: it creates a governance group with
Microsoft-managed default rules (who gets a personal development environment, which connectors and MCP
servers apps may use, how widely apps may be shared, which content sources the browser may load) and
lets the admin choose **which creation paths** are on — Cowork, Copilot Studio, the command line.

![Microsoft 365 admin center, Apps overview: app governance group created, six managed apps, one environment group, two creation paths turned on](/assets/managed-runtime/mac-apps-overview.webp)

*Apps overview in a lab tenant after setup: the governance group exists, six apps are in the
inventory, two creation paths are on.*

The defaults are conservative in a way security teams will appreciate. Out of the box, apps may use a
curated list of Microsoft first-party connectors, and individual risky actions inside them — "send an
arbitrary HTTP request", "run a script" — are blocked. Opening up more is a deliberate admin decision.

One practical note for Power Platform administrators: the runtime places makers into personal
development environments through environment routing, and once governance is initialised the
catch-all routing rule becomes permanent. Design routing first.

## Who pays, and how

To *run* an app, a user needs either a **Power Apps Premium** licence or access to **Copilot Credits**.
Credits are metered per app launch and per data call — a tenth of a credit per call at the time of
writing — and are granted through the same spending policies that already cover Copilot Cowork and Work IQ.

![Copilot cost management: a spending policy that includes Copilot Managed Runtime among the services that consume Copilot Credits](/assets/managed-runtime/mac-cost-management-policy.webp)

*Copilot → Cost management: "Copilot Managed Runtime" is one of the services a spending policy can cover.*

The interesting consequence is economic. An app that thirty people open a few times a month costs a few
euros on credits, where licensing every user would cost hundreds. Heavy, all-day apps flip the other
way. *(Reasoning, not measurement: the break-even I calculated from the published rates is roughly
twenty thousand data calls per user per month. The runtime is a paid preview; check current pricing
before you plan around it.)*

## What I built, and with what

To see whether the developer route holds up beyond a hello-world, I built four apps with Claude Code for an
investment-management storyline, all with fictional data:

- a **cost calculator** (no data at all),
- an **org explorer** on the Office 365 Users connector (management chain, profiles, photos),
- a **meeting-prep** app on Outlook and Work IQ,
- and **Portfolio Desk**: a deal pipeline in Dataverse, investment-committee requests in a SharePoint
  list that the committee also uses in Teams, and two AI features.

![Portfolio Desk: a monitoring sweep turns Microsoft 365 content into risk signals per portfolio company](/assets/managed-runtime/portfolio-desk-sweep.webp)

The AI part is where the "app versus chat" question gets an answer. A **monitoring sweep** asks Work IQ —
Microsoft 365 Copilot's work-data layer — to read the user's mail, chats, meetings and files about every
portfolio company and return *structured* signals: red, amber or green, with dated evidence and a
suggested action. The app turns them into badges on the pipeline and, with one click, into a tracked
request in the committee's SharePoint list. A second button sends the deal to a **Copilot Studio agent**
that drafts an investment-committee memo against the firm's policy.

Chat would have answered the question once. The app makes the answer **structured, repeatable, shared
and actionable** — and the data never leaves Microsoft 365, because every call runs as the signed-in user.

## Two ways to build: Cowork for business users, Claude Code and its counterparts for technical ones

For **business users** the natural route is **Copilot Cowork** (and Code) in the Microsoft 365 Copilot app:
describe the app in a chat, refine it in a live preview, publish and share. No tooling, no repository to
think about — and the result lands in the same governed runtime as everything else. I used it for the
simpler apps, and it is the part that gets a room full of non-developers leaning forward.

For **more technical users** there is a second route that doesn't involve a Microsoft AI at all. The bigger
apps here were written by **Claude Code**, driving Microsoft's `ms` command line on a Linux server. That is
not a workaround: Microsoft publishes the command line and SDK on npm, and a plugin for coding agents in its
own GitHub repository that works with GitHub Copilot and Claude Code. Its skills turned out to be the best
practical documentation available — how to wire Work IQ, how to bind Dataverse tables, which rules the
sandbox enforces.

**What you need to do the same** (the developer route, on your own machine):

1. An admin has turned on the command-line creation path for you (see above), and you have Power Apps
   Premium or Copilot Credits.
2. Install [Node.js](https://nodejs.org/) 24 LTS, Git and
   [Git Credential Manager](https://github.com/git-ecosystem/git-credential-manager) — the platform's Git
   repositories sign you in through it.
3. Install Microsoft's command line from npm —
   [`@microsoft/managed-apps-cli`](https://www.npmjs.com/package/@microsoft/managed-apps-cli), command `ms` —
   and sign in with `ms auth login`.
4. Give your coding agent Microsoft's plugin. In Claude Code (or the GitHub Copilot CLI):
   `/plugin marketplace add microsoft/Managed-Apps`, then `/plugin install microsoft-managed-apps@Managed-Apps`.
   It adds skills such as *create-app*, *add-dataverse*, *add-sharepoint*, *add-workiq*, *deploy* and *share*,
   described in the [GitHub repository](https://github.com/microsoft/Managed-Apps). *(I had Claude Code read
   those skills and drive the command line directly; installing the plugin is the shortcut.)*
5. Describe the app. The agent creates it (`ms app create`), binds data (`ms app add data-source` generates
   typed TypeScript for each connector or table), runs it locally (`ms app dev`), commits and pushes, and
   deploys when you say so (`ms app deploy`).

Microsoft's own walkthroughs: [quickstart with the command line](https://learn.microsoft.com/microsoft-365/managed-apps/developer/quickstart-managed-apps-cli),
[quickstart with a coding agent](https://learn.microsoft.com/microsoft-365/managed-apps/developer/quickstart-github-copilot)
(GitHub Copilot, and it notes Claude Code works the same way) and the
[command reference](https://learn.microsoft.com/microsoft-365/managed-apps/developer/ms-cli-command-reference).

What it took in practice, beyond those steps:

- **Reading the source as well as the docs.** The SDK's type definitions answered questions the preview
  documentation doesn't cover yet.
- **Sign-in on a headless server.** On a normal laptop the command line signs in through the browser. On a
  server it needs device-code sign-in — and many enterprises block device-code flow with Conditional Access.
  If yours does, build from a machine with a browser, and use the service-principal route that the CI/CD
  guidance describes for pipelines. Getting a token cache and Git credentials to behave on a server without a
  desktop keyring also took more time than any of the apps.
- **Letting the agent follow Microsoft's rules.** The plugin is opinionated in useful ways — connectors only,
  least-privilege action lists for shared connections, never edit generated code, never deploy from an
  unpushed commit. A coding agent that reads those rules produces apps that pass governance by construction.

I tested with Claude Code. Other AI coding tools — Codex, GitHub Copilot, Gemini-based agents, IDE
assistants — will very likely work in a similar way: the path is a command line, an SDK and a React app,
nothing specific to one model. Lovable already publishes into the runtime directly.

The two routes also meet. An app started in Cowork or Copilot Studio is backed by a Git repository, so a
developer with edit rights can [clone it and carry on in code](https://learn.microsoft.com/microsoft-365/managed-apps/developer/collaborate-on-app)
(`ms app clone`) — the business user's prototype becomes the developer's starting point instead of a
throwaway.

That openness is, to me, the most important design decision in the product. It says: **build with
whatever your people like; run it where you can govern it.**

## Can business users and developers work on the same app? What my experiments showed

This is the question every handover raises: if developers build something, can the business keep adjusting it
in Cowork afterwards — and the other way round? The documentation covers one direction (the clone above).
The rest of this section is **what I observed in a lab tenant, a handful of times** — not a documented
contract, and the behaviour may well change.

**An app built in code was not editable in Cowork.** I asked Cowork to change one character in Portfolio Desk,
an app scaffolded by the command line. Cowork found the app, opened it and made the edit, but would not
publish it. Its own explanation, and a look at an app Cowork had created, suggest why: Cowork builds every app
from its own project template and works inside a sandbox with a fixed set of pre-installed packages, a few
protected configuration files and a check that must pass before publishing. An app from a different template
doesn't fit that box.

**An app started in Cowork went back and forth without trouble.**

1. In Cowork I described a fund fee and return explorer; Cowork built and published a first version, but left
   out a fee calculator.
2. With Claude Code I cloned the app and added the calculator in code — changing only application source
   files, using only packages the project already had, and running the same kinds of checks the project defines.
   Pushed, built by the platform, previewed.
3. In a *new* Cowork conversation I asked for a small change ("make the default horizon 15 years"). Cowork picked
   up the code-written calculator, changed exactly that one line, and published.

The app's Git history shows the three steps in a straight line — Cowork, then code, then Cowork — with nothing
overwritten. (A small aside: the model selector in that Cowork session showed Claude Opus, so the business-user
route ran on a Claude model too.)

My working conclusion, to be re-checked before relying on it: **if business users are expected to keep
adjusting an app, start it in Cowork and let developers extend it within that project's conventions; an app
started from the developer template may stay a developer-only app.**

## What to be honest about

It is a public preview, and it behaves like one:

- **Lifecycle is Git-based.** Every app has a repository (platform-managed or GitHub Enterprise Cloud with
  your branch policies and pull requests), preview and live are separate, and you deploy and roll back by
  commit. If your organisation expects multi-environment promotion with different data connections per
  stage, check where the documentation stands before planning around it.
- **Know what is audited.** Creating, building, deploying and launching apps show up in the audit log; check
  that the documented events cover your requirements.
- **Expect preview rough edges** in tooling — the odd command-line or code-generation quirk, and build times
  that vary.
- **Apps don't talk to the internet by default.** An app can't call arbitrary endpoints — AI models
  included — unless an administrator allows them in the content security policy; the intended path is
  connectors such as Work IQ or a Copilot Studio agent. A feature for governance, a constraint for some
  designs.

None of that changes the shape of the idea. Team and departmental apps are a good place to start today; for
business-critical processes, review the preview terms and the current state of the documentation first.

## Takeaways

- **Building apps is no longer the bottleneck. Governing where they run is.** Plan for thousands of small
  AI-built apps, not a handful of projects.
- **Copilot Managed Runtime moves the control point** from "which tools may people use to build" to
  "where do the results run" — one runtime, one identity model, one inventory.
- **It is largely off by default.** Opening it up is an admin decision in the Microsoft 365 admin center,
  with credits or Premium licences behind it. Start with one creation path for one group.
- **It is open.** The Microsoft 365 Copilot app, Copilot Studio, Claude, Codex or Lovable can build; the
  runtime governs.
- **Plan who will maintain the app.** In my tests an app started in Cowork could move between Cowork and
  code; an app started from the developer template could not be edited in Cowork. Observed, not documented.
- **Apps beat chat when the answer must become data.** The best pattern pairs the two: the app owns the
  data and the workflow, Copilot supplies the judgement.

## Further reading

- [What is Copilot Managed Runtime](https://learn.microsoft.com/microsoft-365/managed-apps/) — the three
  creation paths and the app portal
- [Overview and key concepts for admins](https://learn.microsoft.com/microsoft-365/admin/manage/apps/) — governance,
  inventory, monitoring, licensing
- [SDK and CLI overview](https://learn.microsoft.com/microsoft-365/managed-apps/developer/) — the developer route
- [Build solutions with Copilot Code](https://learn.microsoft.com/training/modules/build-solutions-copilot-code/) —
  Microsoft Learn training module
- [Create an app in Copilot Studio](https://learn.microsoft.com/microsoft-copilot-studio/apps-experience/create-app)
- [Build apps with the App skill in Cowork](https://learn.microsoft.com/microsoft-365/copilot/cowork/use-cowork)
- [microsoft/Managed-Apps on GitHub](https://github.com/microsoft/Managed-Apps) — templates, the coding-agent
  plugin, GitHub Actions

*Written from hands-on use in a lab tenant, October 2026. Not official Microsoft guidance. The product
is in preview — verify before relying on any of it. Corrections welcome — open an issue on the
repository.*

---

<p style="font-size: 0.95em; margin-top: 2.5em; color: #606c71;">
  Written by <strong>Ondrej Stefka</strong> &middot;
  <a href="https://www.linkedin.com/in/ondrej-stefka-71a7a821">LinkedIn</a>
</p>

<p style="font-size: 0.95em; margin-top: 2em;">
  <a href="{{ '/about' | relative_url }}">About this blog &rarr;</a>
</p>
