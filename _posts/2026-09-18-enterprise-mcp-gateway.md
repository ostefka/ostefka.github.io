---
layout: default
title: "Connecting Microsoft 365 Copilot, Cowork and Copilot Studio to your line-of-business data"
description: "One MCP server, one gateway, three agent surfaces — and the signed-in user's own permissions applying all the way down to the data. Measured in a live tenant, not taken from documentation."
date: 2026-09-18
permalink: /mcp-gateway/
---

<p style="font-size: 0.85em; color: #606c71; border-left: 3px solid #dce6f0; padding: 0.4em 0 0.4em 0.9em; margin: 0 0 1.8em 0;">
  <strong>Personal blog.</strong> This site reflects my own testing and my own
  opinions. It is not an official Microsoft statement and it is not official
  Microsoft documentation in any way. For authoritative guidance always refer to
  <a href="https://learn.microsoft.com/">Microsoft Learn</a>.
</p>

> **TL;DR** — Copilot, Cowork and Copilot Studio are already capable agentic
> systems, and they already reach your Microsoft 365 estate. What they cannot do
> is know anything about *your* business.
>
> **One MCP server behind one gateway serves all three**, with no change to the
> gateway between them — only three different client-side registrations. The
> signed-in user's own permissions apply all the way down to the data, so a user
> with access gets an answer and a user without one gets a refusal.
>
> The hard parts were not the parts anyone warns you about: the audience claim,
> the `Accept` header, and a maker onboarding path that quietly requires
> directory rights.

> **What this article is.** Built and measured in a live tenant, not taken from
> documentation. None of it is an official Microsoft statement. These were the
> observations in one tenant, last checked on **18 September 2026** — these
> surfaces change fast, so re-verify before relying on any of it.

## Why bother

Copilot, Cowork and Copilot Studio are already capable agentic systems. They plan, they call tools,
they chain steps, they ask for confirmation. There is no reason to rebuild any of that.

They are not blind to your organisation either. Out of the box they already reach your Microsoft 365
estate — mail, files, chats, meetings, calendars — through Microsoft's own work-data layer, and
Copilot Studio ships a large catalogue of prebuilt connectors for common SaaS platforms. If your
question can be answered from Microsoft 365 content or from a system that already has a certified
connector, **use those**; there is nothing to build.

The gap is everything else: the reporting warehouse, the policy-administration system, the
dispatching platform, the contract archive, the thirty-year-old system of record that runs the
business and has never had an API anybody outside the team has used. That is where the value is, and
no catalogue will ever cover it, because it is specific to you.

So the interesting problem is not "build an agent". It is **connect the agents you already have to
the data you already own, without handing them a service account that sees everything.**

This is what that takes: one Model Context Protocol server, one gateway, three agent surfaces, and
the signed-in user's own permissions applying all the way down to the data.

---

## The shape of it

```
Copilot Studio  ─┐
Cowork          ─┼─►  gateway  ─►  MCP server  ─►  your data platform
M365 Copilot    ─┘    validate      validate         as the signed-in user
                      the token     again, then
                      rate limit    exchange it
                      audit         on-behalf-of
```

One endpoint. One application registration. Three client-side registrations, because the three
surfaces are genuinely three different products — more on that below.

---

## 1. What the MCP server itself must do

This is the part that is fully in your control. These server-side choices worked across all three
surfaces we tested; client-side registration, licensing and policy still have to be right too.

**Streamable HTTP, stateless per request.** Build a fresh server instance per POST; hold no session.
We watched two consecutive requests from the same conversation arrive from **different Microsoft
egress IP addresses, twelve seconds apart** — so anything expecting the caller to come back to the
same place breaks between one prompt and the next. Strictly that disproves source-IP affinity rather
than every form of sticky routing, but it is a good reason to want none of it. The protocol has since
gone the same way: MCP `2026-07-28` removes protocol-level sessions and the `Mcp-Session-Id` header
outright, so stateless is no longer a defensive choice but the direction of travel.

**Accept every audience your application answers to — and nothing else.** The most frequent
recurring failure. The `aud` claim is *not* reliably the `api://` identifier URI you configured: a v2
access token carries the API's client id, a v1 token may carry the client id *or* a resource URI, and
registering the server with one surface can add a *third* identifier URI that only that surface ever
presents. Validate against an explicit **list**, in the gateway *and* in the server.

> The list is an allowlist, not a debugging tactic. Every entry must be an identifier *this* API
> owns. Never add an audience because it turned up in a failing request — if another resource's
> audience is arriving, the client is asking for the wrong token, and pasting it into your allowlist
> turns your server into that resource's confused deputy.

> Symptom when you get this wrong: one surface works perfectly while another fails with a generic
> client-side error. It appeared three times in different forms.

**Normalise the `Accept` header.** The transport requires both `application/json` and
`text/event-stream`; some clients send `*/*` and get a 406. If your framework reads raw headers,
rewrite those too, not just the parsed value.

**Annotate every tool honestly.** `readOnlyHint`, `destructiveHint`. In our Cowork tests an
*unannotated* tool was treated as destructive and prompted the user for confirmation on every call;
accurate annotations removed that. They are hints to the host, not enforcement — they do not make a
tool read-only, so enforce that in code and in the backend's permissions.

**Expose a generic `search` / `fetch` pair, not only domain-specific tools.** One surface consumes
MCP as a *retrieval* source rather than an agentic tool caller. With only domain-prefixed tools it
authenticated, listed everything, and then never called anything — no error, just silence. Adding
the generic pair over the same data fixed it. We have not proved those exact names are mandatory —
the published guidance offers `search`, `fetch` and `query` as examples — only that adding them
unblocked a surface that had gone silent. Keep the domain tools too: once invocation is unblocked,
the hosts orchestrate the specific ones well and answer better with them.

**Keep the tool surface small and describe it for an orchestrator.** Around ten tools is where we
landed; it is a curation heuristic, not a limit. Write descriptions that say *when to use this*, not
only *what it returns* — alongside the name and schema they are most of what the orchestrator has to
go on, and on at least one surface they are also what an administrator reads when approving the
tool.

**Return errors as results, not exceptions.** A tool that cannot do its job should return a
structured, actionable message. An HTTP error just makes the agent apologise; a good result lets it
recover or explain.

**Serve OAuth Protected Resource Metadata** (`/.well-known/oauth-protected-resource/...`) and answer
an unauthenticated call with `401` plus a `WWW-Authenticate` challenge pointing at it. Compliant
clients discover where to authenticate instead of being pre-configured.

**Expect one rejected call per connection.** One surface opens with `server/discover`, takes the
error our SDK returns, falls back to `initialize`, and carries on. Don't alert on it — but do not
file it as a vendor quirk either, which is what we did at first. `server/discover` is **required** by
MCP revision `2026-07-28`; the SDK we built on speaks `2025-11-25` and does not implement it. What we
were watching was ordinary version negotiation. Classify that specific fallback separately rather
than suppressing failed requests as a class.

### The parts that are easy to get wrong, in code

TypeScript, against `@modelcontextprotocol/sdk` 1.30.0, `express` 5.2.1, `jose` 6.2.12,
`@azure/msal-node` 5.6.0 and `zod` 4.6.5. That SDK speaks MCP `2025-11-25`; the `2026-07-28` revision
changes the transport substantially, so check it before copying any of this forward.

**These are fragments from a working implementation, not a runnable server.** Imports,
`express.json()` with a bounded body size, the request-scoped identity the tool and the exchange both
read, and the actual data lookup are all omitted. Never hold the current user in a process global —
a fresh server per request is pointless if identity is shared across them.

**Stateless handler.** A fresh server and transport per POST, and teardown on response close —
without the teardown every request leaks a pair and they accumulate for the life of the process. We
ran a version that did, and it degraded; we never measured the exact threshold, so do not read one
into this.

```ts
app.post("/mcp", requireAuth, async (req, res) => {
  const server = buildServer();                                   // cheap; no I/O at construction
  const transport = new StreamableHTTPServerTransport({
    sessionIdGenerator: undefined,                                // stateless — not optional
  });
  res.on("close", () => { void transport.close(); void server.close(); });
  await server.connect(transport);
  await transport.handleRequest(req, res, req.body);              // pass the PARSED body
});
```

**Accept normalisation.** The transport demands both media types and replies `406` otherwise. The
non-obvious half: the SDK reads `req.rawHeaders`, so setting `req.headers.accept` alone does nothing.

Scope this to the case you actually see — a client sending `*/*` or `application/json` alone. Rewriting
every caller's negotiation means a client that deliberately excludes `text/event-stream` gets a stream
it said it could not read, so leave the honest `406` in place for those. Call it before
`handleRequest`, and revisit it on every SDK upgrade; it is a workaround, not a server requirement.

```ts
function normalizeAccept(req: Request): void {
  const want = "application/json, text/event-stream";
  req.headers.accept = want;
  const out: string[] = [];
  let found = false;
  for (let i = 0; i < req.rawHeaders.length; i += 2) {
    const name = req.rawHeaders[i];
    if (name?.toLowerCase() === "accept") { out.push(name, want); found = true; }
    else out.push(name!, req.rawHeaders[i + 1] ?? "");
  }
  if (!found) out.push("Accept", want);
  req.rawHeaders = out;                                           // the line people miss
}
```

**Token validation — audience as a list, both issuer forms.** v1 tokens are issued by
`sts.windows.net`, v2 by `login.microsoftonline.com`; you will see both.

```ts
const jwks = createRemoteJWKSet(
  new URL(`https://login.microsoftonline.com/${TENANT_ID}/discovery/v2.0/keys`));

const { payload } = await jwtVerify(rawToken, jwks, {
  issuer: [
    `https://login.microsoftonline.com/${TENANT_ID}/v2.0`,
    `https://sts.windows.net/${TENANT_ID}/`,
  ],
  audience: [                     // every identifier URI the app answers to — see the warning above
    `api://${APP_ID}`,
    APP_ID,                       // the BARE id: this is what you often actually receive
    `api://auth-${SSO_REGISTRATION_ID}/${APP_ID}`,   // added by one surface's registration
  ],
});
// `payload.scp as string` is a compile-time assertion, not a runtime check — a token with no scp
// (or an array) sails through it. Check the type.
const scp = typeof payload.scp === "string" ? payload.scp : "";
if (!scp.split(" ").includes("access_as_user")) throw new Forbidden();
```

**The On-Behalf-Of exchange — one resource per call.** Mixing resources, or mixing `.default` with
explicit scopes, is rejected outright.

```ts
const cca = new ConfidentialClientApplication({
  auth: { clientId: APP_ID, authority: `https://login.microsoftonline.com/${TENANT_ID}`,
          clientSecret: CLIENT_SECRET },
});
await cca.getTokenCache().getAllAccounts();          // hydrates a PERSISTED cache — see below
const result = await cca.acquireTokenOnBehalfOf({
  oboAssertion: rawToken,                            // the VERIFIED inbound token
  scopes: [`${BACKEND_RESOURCE}/.default`],          // exactly ONE resource
});
```

Map the failures to something the caller can act on. The documented recovery for a downstream
Conditional Access requirement is to return **401 with the claims challenge** so the client can
fetch a token that satisfies it — so preserve that challenge rather than flattening everything into a
403. We never saw a surface complete that recovery in the clients we tested, but discarding the
challenge guarantees none ever will:

```ts
// Anchor the code, don't substring-match it. /AADSTS65001/ also matches AADSTS650053 and
// AADSTS650057 — both REGISTRATION faults, not consent faults. Matching them as consent sends the
// operator to request admin consent for a scope that was never added to the registration.
const code = /AADSTS(\d+)/.exec(msg)?.[1];

if (code === "50076" || code === "50079" || /interaction_required/i.test(msg))
  throw new AuthError(401, "interaction_required",
    "Additional verification is required.", { claims: claimsChallengeFrom(err) });  // pass it on

if (code === "65001")
  throw new AuthError(403, "consent_required",
    "Consent has not been granted for this application to call the backend on your behalf.");

if (code === "65005" || code === "650053" || code === "650057")
  throw new AuthError(500, "misconfigured",
    "This application's own registration is missing the requested permission or resource. " +
    "Consent will not fix it; the registration has to be corrected.");
```

`getAllAccounts()` is the documented way to hydrate a **persisted** cache into a freshly-constructed
client. If you have not wired a cache-persistence plugin — and this fragment has not — a new client
starts empty and the call buys you nothing. Add it when you add the plugin, not before.

For scale-out, give each request its own client instance with a cache partitioned by a **hash of the
assertion** — stricter than per-user, so a re-consent or a step-up cannot reuse a downstream token
issued for a weaker one. It does not make revocation immediate: an access token already issued stays
valid until it expires, whatever you do to the cache.

**Tool definition — annotations and actionable failures.**

```ts
server.registerTool("lob_search", {
  title: "Search <system> records",
  description:
    "Use this FIRST when the user names something in <system> but you do not yet know its exact " +
    "record. Returns ranked hits as { id, type, title, snippet }; pass an id to lob_fetch.",
  inputSchema: { query: z.string().describe("Free-text query, e.g. 'overdue invoices, last quarter'.") },
  annotations: { readOnlyHint: true, destructiveHint: false, idempotentHint: true },
}, async ({ query }) => ({
  content: [{ type: "text", text: JSON.stringify(data, null, 2) }],
  structuredContent: { source: "…provenance the agent can cite…", data },
}));
```

When a tool cannot do its job, return a **result** describing the problem and the way forward — not a
thrown error. Set `isError: true` with it. An English sentence or your own `status` field is not the
protocol's failure signal, and without the flag every client and every dashboard you build will
record a failed tool call as a success. A backend `403` is worth phrasing especially well, because it
*may* be the system working correctly — the user genuinely lacks access — rather than a fault:

```ts
return {
  isError: true,
  content: [{ type: "text", text:
    "You are authenticated but not permitted to read this data. If you expected access, ask the " +
    "owner of that data; this is the source system applying your own permissions." }],
  structuredContent: { status: "forbidden" },
};
```

Keep transport failures at the transport: inbound authentication answers HTTP `401` with a challenge,
and a malformed request answers with a JSON-RPC error. `isError` is for a tool that ran and could not
deliver. And distinguish a `403` that means *this user may not* from one that means *your gateway,
network or policy said no* — they read identically in a log and mean opposite things.

**Discovery.** Serve Protected Resource Metadata and point at it from the challenge, so a compliant
client can find the authorization server without being configured:

```jsonc
// GET /.well-known/oauth-protected-resource/<server>/mcp
{
  "resource": "https://mcp.example.com/<server>/mcp",
  "authorization_servers": ["https://login.microsoftonline.com/<tenant>/v2.0"],
  "bearer_methods_supported": ["header"],
  "scopes_supported": ["api://<app-id>/access_as_user"]
}
```

```
HTTP/1.1 401 Unauthorized
WWW-Authenticate: Bearer resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource/<server>/mcp"
```

Omit `error=` when the request simply carried no credentials; reserve `error="invalid_token"` for a
token that was supplied and rejected. Serve the metadata at your **public** boundary, unauthenticated,
with its `resource` value exactly matching the URL clients call — and make sure the gateway's own 401
advertises it too, not only the workload's. Discovery tells a client where to authenticate; it does
not register it or grant consent, so expect to configure the client side regardless.

---

## 2. What you register on the Microsoft 365 side

**No click-by-click here — this is the fastest-moving part and any such detail would age badly.**
What does not age as quickly is *which components exist* and *how they depend on each other*, and
that is the part which is genuinely hard to discover. As of today:

| Surface | What you register | Reversible? |
|---|---|---|
| Copilot Studio | a connector generated by its own wizard, with OAuth client id and secret | yes |
| Cowork | a plugin package (a zip: manifest, icons, tool list, skills) | **no — only blocked** |
| Microsoft 365 Copilot | a connector record created in the admin centre | yes |

They are **not unified**. A server registered for one does not appear in another. Plan onboarding as
three registrations, and expect the terminology to differ between them for the same idea.

### The component nobody mentions: the Teams Developer Portal

Copilot Studio is self-contained — its wizard takes an OAuth client id and secret and generates its
own connector. **Cowork and Microsoft 365 Copilot are not.** Both consume a third object that lives
in neither Entra nor the admin centre: an **Entra SSO client ID registration**, created in the Teams
Developer Portal. One registration serves both surfaces.

So the full set of components is four, not three:

| # | Component | Where it lives | Used by |
|---|---|---|---|
| 1 | The MCP server's **app registration** (the API) | Entra ID | all three |
| 2 | A **connector** with OAuth client id + secret | Copilot Studio | Studio only |
| 3 | An **Entra SSO client ID registration** | Teams Developer Portal | Cowork + M365 Copilot |
| 4 | A **plugin package** / **connector record** | admin centre | Cowork / M365 Copilot |

**And components 1 and 3 depend on each other, in that order.** This is the part worth knowing
before you start, because the loop is not obvious from either end:

1. Create the Entra app registration first — component 3 asks for its **client id**.
2. Create the SSO registration in the portal. It returns a **registration id**.
3. **Go back to Entra** and add two things derived from that registration id:
   - an additional identifier URI of the form `api://auth-<registration-id>/<app-id>`. The portal
     generates this shape and will not accept the plain `api://<app-id>` you already have. **Add it;
     do not replace** — both must remain.
   - the consent redirect URI the platform returns sign-in to. Note there are two similarly named
     redirect endpoints belonging to two different registration types in the same portal; the
     consent one is what this path needs.
4. The value the Cowork manifest and the Microsoft 365 Copilot connector both ask for — labelled
   *Reference ID* — is **not** the registration id. It is a base64-encoded composite of the tenant id
   and the registration id. Paste the GUID and you get *"the request is malformed or incorrect"*
   with **zero** traffic reaching your server.

> **The trap in step 3.** Adding that identifier URI gives your application a *third* audience.
> Tokens from Cowork and Microsoft 365 Copilot now carry it, while Copilot Studio still presents
> `api://<app-id>`. Until the gateway **and** the workload accept all of them, two surfaces fail with
> `401 audience is not valid` while the third keeps working perfectly — from an Entra edit that looks
> unrelated to the gateway. This is the same audience-list rule from section 1, and this is the
> change most likely to trigger it.

### Two more registration findings

- **One setting in the SSO registration is labelled "for testing only" and is the only one that
  worked.** It governs which Teams application may use the registration. The reassuring-sounding
  alternative — binding it to one specific application id — silently broke tool calls, with zero
  traffic reaching the server. We kept the organisation restriction and left the application
  restriction broad. We have **not** established that this is the supported production
  configuration; settle that before a production rollout rather than concluding from this post that
  the warning label is wrong.
- **Your application does not need to be multi-tenant.** Guides say to make it so, citing an error
  we only ever saw on the shared multi-tenant sign-in endpoint. All three surfaces worked against our
  single-tenant registration. Check which authority your registration path actually uses before
  changing the sign-in audience — but an internal server should stay single-tenant, and a security
  team is right to push back on anything else.

### Makers must never need directory rights

The one piece of M365-side detail worth keeping, because it decides whether this scales.

In Copilot Studio, **every MCP server a maker adds through the wizard becomes its own custom
connector, with its own generated OAuth callback URL.** (They do not appear under Custom connectors;
look in Solutions.) Identity providers match callback URLs exactly, so
without intervention every new server entry needs a change in your identity tenant — which for an
ordinary maker means a support ticket, forever. Adding another *tool* to a server already registered
is a different operation and costs nothing.

Three ways out, in the order a security team will prefer them:

1. **One reusable connector maintained by IT**, its callback registered exactly once.
2. **Exact callbacks registered on request** — an administrator task, with a real queue behind it.
3. **A wildcard callback URL**, registered once at onboarding.

We verified that the third works: two connectors whose callback URLs had never been individually
registered both completed sign-in and reached a connected state. That is a real result and it
dissolves the maker problem entirely. It also proves interoperability rather than suitability — the
namespace being wildcarded belongs to a shared platform redirect service rather than to you, and
Microsoft's own guidance is to avoid wildcard redirect URIs. Treat it as a reviewed exception with a
named owner, not as the default recipe.

> **The principle:** directory work belongs to **onboarding a server** — once, by an administrator.
> It must never appear in the runbook a maker follows to build an agent.

---

## 3. Identity: the part that makes this defensible

Everything above is plumbing. This is the architecture.

The requirement is simple to state: **a user who has access to the data gets an answer; a user who
does not gets a refusal.** Not a service account with broad rights and a filter bolted on top.

The flow: the agent surface sends a token identifying the user and addressed to your MCP server. The
server validates it, then **exchanges** it — On-Behalf-Of — for a token addressed to the data
platform, still representing that user. The data platform decides.

One assumption sits underneath all of that, and it is the one most likely to be wrong in a real
deployment: **OBO preserves the subject of the token it is handed.** It does not make that subject
the person currently chatting. If a surface lets a maker authenticate a shared connection, then every
user of that agent reaches your data as the maker — technically flawless OBO, consistently the wrong
human. Configure end-user authentication on every connection, and verify it in the **published**
agent rather than in the maker's own test pane, which is exactly where that mistake hides.

Two results worth reproducing in your own environment, because both are stronger than a diagram:

**Ask the backend who it thinks you are.** Most data platforms can report the identity they resolved
for the current query. We asked, and it named the signed-in user — not a service principal, not an
item owner. That matters more than it sounds:

> A data platform can authenticate the user correctly and then read the underlying storage **as the
> item owner** — a configuration default on at least one common engine. Token right, identity right,
> data wrong. Silently. The demo looks perfect.

Any server can report the identity it *sent*. Only the backend can report the identity it *applied*.
Build your proof from the backend's answer.

**Check the refusal, not just the success.** We pointed a principal with no access at the same data,
holding an otherwise perfectly valid token. The platform refused it — authenticated, then refused.
The principal we used was the **MCP server's own identity** — the one a careless fallback would reach
for. The platform refused it too.

That proves the application's own identity cannot read the data. It does not, by itself, prove your
code would never use it: that is a separate claim, about your code. Read the exchange path for a
fallback branch, then force the exchange to fail and confirm that **no backend request is made at
all**. A per-user guarantee that quietly degrades to an application identity under error or load is
worse than none, because nothing in the output looks different.

**Classify each backend before writing code.** Two questions decide the design:

1. Does it enforce per-user authorization itself?
2. Does it accept your identity provider's tokens directly?

Yes/yes is the good case — carry identity, let the backend decide. Yes/no needs a per-user
credential exchange. No/yes means **you** now own the authorization decision, so write down that you
own it. No/no means you must supply both the authentication *and* every authorization decision in
your own code, or decline the integration; a shared credential answers only the first half, and
skipping the second is where the leaks come from. Note the converse too: carrying delegated identity
does not settle data-level authorization on its own.

Two refinements we learned the hard way:

- **Classify per surface, not per product.** One platform was "backend decides" for its management
  API and effectively "you decide" for its data plane, because of a default setting. A
  product-level answer is how an integration gets documented as safe when it is not.
- **A single shared credential is not automatically wrong.** A company-wide corpus that every
  employee may read has no per-user decision to make. What must be written down is the claim
  itself — *"this is uniformly readable by all staff"* — because that is what quietly stops being
  true when someone adds a folder.

---

## 4. If you are not on Azure

We built this with a cloud CDN/WAF in front of a managed API gateway, but nothing in the design
requires that. Any reverse proxy or API gateway can play the part — on-premises, another cloud, or
an appliance you already own. **We did not test the on-premises variant**, so treat this as reasoning
from what we did test, not as verified.

**What the gateway layer must do:**

- **Terminate TLS on a publicly reachable HTTPS endpoint.** Every client path we tested reached a
  public HTTPS gateway. We did not test any private-connectivity variant, so assume you need the
  public ingress unless you can show otherwise. Only the *ingress* has to be public: the hop from
  gateway to workload, and from workload to data, can stay entirely private — ours do.
- **Validate the token**: signature against the identity provider's published keys, issuer, expiry,
  tenant, required scope, and **audience as a list**.
- **Not delay or reshape the body.** Inspecting JSON is not itself the problem; buffering a response
  until it completes is. Chunked and event-stream responses must pass through promptly and intact, or
  tool calls hang and truncate. Check timeouts through the whole proxy chain, not just the last hop.
- **Preserve `Authorization`** toward the MCP workload, and leave the negotiated content type alone.
  Never forward that inbound token to a different backend — it is addressed to you, and the exchange
  is what gets you a token addressed to something else.
- **Log every call** — see below.
- **Rate limit per user**, keyed on the identity in the token rather than the source address.

**What the network design must do:**

- **Do not build an allowlist from source addresses you observed.** We saw calls from many different
  egress addresses, changing within a single conversation. Where your platform offers documented
  ranges or service tags, use them as defence in depth — never as the control. The token is the
  control.
- **Do restrict the workload to the gateway.** The MCP server should accept connections only from
  the proxy, so the gateway is provably the only path in.
- **Validate the token at the workload too**, not only at the gateway. It is cheap, and it means a
  misconfigured proxy is not the same as an open door.

**What stays the same regardless of hosting:** the identity provider. Using Entra ID as the token
issuer does not require hosting anything in Azure. The MCP server can run wherever your data is —
which is usually the right place for it anyway.

**Watch latency.** In the surface we measured, the tool budget was around 30 seconds end to end —
measure it for each surface rather than trusting that number. It is generous until a cold start, a
token exchange and a slow query stack up.

---

## 5. Observability: build it before the first integration

The single most practical thing in this whole exercise.

Client-side errors from these surfaces are close to useless: *"Couldn't load MCP tools"*, *"cannot
verify connection"*, *"invalid manifest file"*, and on one occasion an untranslated internal resource
key where the message should have been. None of them name a field or a reason.

One question at the gateway splits every one of those in half:

**Did anything arrive?**

- **No requests at all** → first rule out the dull causes: wrong URL, DNS, TLS, an edge or WAF rule,
  and whether your logging actually covers that path. Once those are clean, the fault is client-side
  registration, and your server, tokens and backend are exonerated without touching them.
- **A 401 with an audience complaint** → a token was issued and you rejected it. Check your audience
  list.
- **A 200, and the client still unhappy** → read the tool's own result, the content type and whether
  the stream completed. A 200 is not proof the exchange was well-formed.

That distinction resolved four separate failures for us where the Microsoft-side UI said nothing
useful. Log per call: timestamp, correlation id, which server and tool, outcome, latency, and the
user and application ids from the token. **Never tool arguments, never tokens.**

One caveat we hit: **all these surfaces present the same generic browser user-agent string**, so you
cannot tell them apart from the request. The nearest thing available is the OAuth client claim in the
token — `appid` in a v1 token, `azp` in a v2 one — and you have to log it deliberately, because
nothing hands it to you. Read it as *which client asked*, not *which product*: surfaces sharing a
token broker share that identity, so it may not separate them at all. Never make an authorization
decision on it.

---

## 6. Before you demo

- Ask the backend which identity it applied — not your own logs.
- Test with **two users with different entitlements**. One user proves nothing. Run them
  concurrently at least once: a token or result cache that is not keyed per user fails only under
  exactly that test.
- Test the **refusal** path explicitly, and make sure it is a clean refusal rather than an empty
  result.
- Revoke one user's access and confirm the next call refuses. An access token already issued stays
  valid until it expires — know how long that window is before anyone asks.
- Confirm a tool call still works after the original token's **actual** expiry. Check the lifetime
  rather than waiting an hour and assuming.
- Check which registrations are **reversible** before publishing anything into production. In the
  path we tested we could block a package but never found a supported way to delete one.
- Remember the client secret is your middle tier's own credential, not a data account: keep it out
  of the repo, rotate it, and look at certificates or client assertions before production.
- Treat tool output as untrusted text. It reaches a model that will act on it, and "the user was
  allowed to read this" is not the same as "this is safe to hand to an agent".
- Debug to a working state somewhere disposable, then publish once. Do not iterate in production.

---

## The shortest version

- The agent platforms are already good. The value you add is **your data**, reached safely.
- One gateway can serve every surface; the gateway does not change between them, only the
  registration does — and there are three of those.
- Carry the user's identity all the way to the data and let the data platform decide. Where it
  cannot, say so explicitly and own the decision in writing.
- Validate every audience your application answers to — and only those — in every layer that
  validates. Registering the server with a new surface is what adds one.
- Two of the three surfaces need a Teams Developer Portal SSO registration, and it forces an edit
  back into the Entra application. Budget for that round trip.
- Keep makers out of the directory. A wildcard callback does it in one step; an IT-owned connector
  does it without wildcarding a namespace you do not own.
- Build the audit trail first. *"Did anything arrive?"* is worth more than every client-side error
  message you will ever read.

*Written from hands-on testing in a live tenant, September 2026. Not official Microsoft guidance.
Verify before relying on it — especially the Microsoft 365 registration steps, which are changing
quickly. Corrections welcome — open an issue on the repository.*
---

<p style="font-size: 0.95em; margin-top: 2em;">
  <a href="{{ '/about' | relative_url }}">About this blog &rarr;</a>
</p>

