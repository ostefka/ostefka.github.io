---
layout: default
title: "Does OAuth survive a reverse proxy?"
description: "We put a Microsoft 365 Copilot custom engine agent behind a reverse proxy and measured what survives. The tokens were never the fragile part — the HTTP plumbing was."
date: 2026-08-27
permalink: /copilot-reverse-proxy/
redirect_from:
  - /copilot-custom-agent-reverse-proxy/
---

<p style="font-size: 0.85em; color: #606c71; border-left: 3px solid #dce6f0; padding: 0.4em 0 0.4em 0.9em; margin: 0 0 1.8em 0;">
  <strong>Personal blog.</strong> This site reflects my own testing and my own
  opinions. It is not an official Microsoft statement and it is not official
  Microsoft documentation in any way. For authoritative guidance always refer to
  <a href="https://learn.microsoft.com/">Microsoft Learn</a>.
</p>

> **TL;DR** — Enterprises rarely let a cloud container answer the internet
> directly. There is always a gateway, a load balancer, or an application
> delivery controller in the way, and the container lives in a private segment
> behind it. So: does Microsoft 365 Copilot single sign-on still work when you
> put a reverse proxy in front of a custom engine agent?
>
> **Yes.** We built a minimal echo agent that reports the caller's identity,
> deployed it to Azure Container Apps, put a proxy in front, and the user's
> identity arrived byte-for-byte identical to the direct path — including a
> Microsoft Graph call executed as that user.
>
> The interesting part is what *did* break. It was never the tokens. It was
> SNI, and the `Host` header, and both fail **silently** — with nothing
> whatsoever in the application logs.

> **What this article is.** Measurements, not documentation. Every number
> below came from a live lab tenant on **26–27 August 2026**. Where something
> is reasoning rather than measurement, it says so explicitly.

## Why this matters

The reference architecture for a Copilot custom engine agent is charmingly
simple: Copilot talks to Azure Bot Service, Bot Service posts activities to
your HTTPS endpoint, your code answers.

Now deploy that inside a real enterprise. The container is not allowed a public
IP. There is a gateway terminating TLS at the perimeter, quite possibly
inspecting the traffic, and forwarding into a private subnet. Suddenly the
architecture is:

```
Copilot -> Azure Bot Service -> your gateway -> private container
```

And the question everybody asks, usually a week before go-live, is whether the
authentication still works through that hop — or whether the whole design has
to be rethought.

Nobody could tell us. So we measured it.

## The three token flows nobody separates

This is the single most useful thing in this article, and it reframes the
entire problem.

People say "OAuth goes through the proxy" as though it were one thing. It is
three, and **only one of them touches the proxy at all**:

| Exchange | Path it travels | Proxy in the path? |
|---|---|---|
| Bot Service authenticating its call to your agent | Bot Service → **proxy** → app, in an HTTP header | **Yes — the only one** |
| Copilot's silent SSO token exchange | Inside the activity **body**, not a header | No |
| On-behalf-of exchange, and your reply | **Outbound** from the container to Microsoft | No |

The user's SSO token arrives as a `tokenExchange` invoke activity in the JSON
body. No amount of header mangling touches it. The on-behalf-of exchange goes
from your container directly to `login.microsoftonline.com` — the proxy is not
even on that route. Same for the response, which goes out to the Bot Service
connector.

So the entire risk surface is **one HTTP header on one inbound call**. That is
a much smaller problem than "will OAuth survive", and it is why the answer
turns out to be yes.

## The test rig

A deliberately stupid agent. No search, no LLM, no RAG. It answers with the
identity of whoever is talking to it, at three levels, plus a dump of how the
HTTP request looked on arrival:

1. **What Bot Framework asserts** — `from.name`, `aadObjectId`. Unauthenticated.
2. **The SSO token** — decoded claims from the token Entra issued for this app.
3. **Microsoft Graph `/me`** — reached via an on-behalf-of exchange.
4. **The inbound request** — `Host`, whether `Authorization` survived, and every
   header a proxy might have added.
5. **Independent verification of the Bot Framework token** — using only the
   public signing keys.

Level 1 exists purely to make a point, and it made it immediately:

```
from.name    :
aadObjectId  : 99999999-8888-7777-6666-555555555555
```

**The display name came back empty.** The naive version of this agent — the one
that echoes `activity.from.name` and calls it identity — would have returned a
blank string. Unauthenticated fields are not evidence. They are not even
reliably populated.

The SSO token, meanwhile:

```
name  : Example User
upn   : user@contoso.onmicrosoft.com
aud   : 11111111-2222-3333-4444-555555555555   <- the bot app, not Graph
scp   : access_as_user
iss   : https://login.microsoftonline.com/<tenant>/v2.0
```

That is a real Entra-issued token, silently exchanged, audience pinned to our
own application. That is evidence.

## The result: it survives

With the proxy in the path, sections 2 and 3 were **identical** to the direct
run. Same user, same audience, same successful Graph call. Section 4 showed the
Bot Framework token arriving intact:

```
authorization: Bearer (present)
  audience   : 11111111-2222-3333-4444-555555555555
  issuer     : https://api.botframework.com
x-forwarded-for: 52.112.86.172, 100.100.0.114, 100.100.0.103
x-original-host: agent-proxy.example.azurecontainerapps.io
x-proxy-hop    : nginx
```

Three hops in `x-forwarded-for`, `Authorization` untouched. Done.

Incidentally, if you deploy to Azure Container Apps you already have a reverse
proxy in the path — the platform ingress is Envoy. Your tokens have survived
one L7 proxy before you add any of your own.

## What actually breaks

Here is the part worth printing out. Each row is a real test, driven with curl
and read back from the application's own logs.

| Proxy behaviour | Reaches app? | Identity intact? | Symptom | In app logs? |
|---|---|---|---|---|
| Correct pass-through | yes | **yes** | works | — |
| TLS re-encrypt **without SNI** | **no** | — | `502` | **no** |
| Rewrites `Host` to its own name | **no** | — | `404` | **no** |
| Strips `Authorization` | yes | **no** | `401` | yes |
| Path prefix, stripped correctly | yes | **yes** | works | — |
| Path prefix, **not** stripped | yes, wrong path | — | `404` | yes |

Note the pattern: **the two failures that reach the application are easy. The
two that do not reach it are the ones that will cost you a day.**

### The one that costs you a day: SNI

Our first attempt through the proxy returned `502`:

```
peer closed connection in SSL handshake (104: Connection reset by peer)
while SSL handshaking to upstream
```

Azure Container Apps ingress selects the target application from the **TLS SNI
value**. nginx does not send SNI on upstream TLS connections unless you tell
it to. Two lines:

```nginx
proxy_ssl_server_name on;
proxy_ssl_name backend.internal.example.azurecontainerapps.io;
```

Any gateway that re-encrypts TLS to an Azure PaaS origin with shared ingress
needs the equivalent. Nothing reaches the application, so nothing appears in
its logs, and every other check you can think of passes.

### The silent killer: the `Host` header

Azure routes to your container by `Host`. If the proxy helpfully replaces it
with its own virtual server name — which many do by default — Azure has no
application matching that name and returns `404`.

Everything looks healthy. DNS resolves. TCP connects. TLS completes. The
certificate validates. The endpoint responds. And your application is **never
invoked**, so there is not a single line anywhere to tell you why.

We saw this by accident before we saw it on purpose. When we switched the
container from public to internal ingress, its old public URL immediately
started returning `404` — DNS still resolved, the platform ingress still
answered, but no application matched that `Host` any more. That is exactly the
failure a `Host`-rewriting proxy produces.

**If your integration fails while the application looks healthy and silent,
check the upstream `Host` header first.**

## "But isn't that an open endpoint?"

The moment a security reviewer sees a public virtual server forwarding into a
private segment, this question arrives. It deserves a precise answer.

**The endpoint is not anonymous.** Every call must carry a valid Bot Service
token whose audience is your specific application ID. Unauthenticated requests
get `401` and never reach application logic. A valid token issued for somebody
else's bot is rejected too.

But be careful about what that token proves. We decoded a live one and listed
**every** claim it carries. There are five:

```
all claims present : aud, exp, iss, nbf, serviceurl
user-identifying   : NONE - this token names no human being
```

| Proves | Does not prove |
|---|---|
| The caller is genuinely Azure Bot Service | Anything about the end user |
| The message is addressed to *this* bot | Which person is asking |
| Which channel it came from | Any group, role or entitlement |

So a perimeter device can establish *"this is a genuine call to our bot"*. It
can never establish *"this is Jan Novák"*. User identity lives in the SSO token
in the body, usable only by the application after the exchange. **User-level
authorisation belongs in your code, not on your gateway.**

## The IP allowlist trap

The obvious next thought is to restrict source IPs. And there is an obvious
candidate — Azure publishes an `AzureBotService` service tag:

```bash
az network list-service-tags --location westeurope \
  --query "values[?name=='AzureBotService'].properties.addressPrefixes" -o json
```

125 IPv4 prefixes, 674 addresses, mostly `/30`s. Small enough to paste into an
allowlist. Perfect.

**Except every source address we actually observed was outside it.**

| Observed caller | In `AzureBotService`? | Published under |
|---|---|---|
| `52.112.113.208` | no | `AzureCloud.centralus` |
| `52.123.185.237` | no | `AzureCloud.centralus` |
| `52.112.86.172` | no | `AzureCloud.westus` |

Three out of three, from **two different Azure regions**. An allowlist built
from the obvious service tag would have blocked live production traffic — and
pinning to one region would not have helped either.

The addresses do sit inside the broad `AzureCloud` ranges, but allowing those
means allowing every virtual machine in the region, including one an attacker
can rent for the price of a coffee. It looks like a control in the config. It
is not one.

**Caveat, stated plainly:** three observations, one tenant, one lab. This is not
proof that the tag is never correct, and the traffic is plainly arriving from
channel infrastructure rather than the Bot Service control plane, which may be
exactly what the tag is scoped to. What it *is* sufficient to conclude: never
enforce an IP allowlist here without first running it in log-only mode against
real traffic. If your results differ, please tell us.

## Validate the token at the gateway instead

Here is the good news, and it is the practical answer to the security review.

The Bot Service token can be validated using **only public metadata** — no
client secret, no access to your app registration. Which means the gateway can
do it, and reject forgeries before anything enters the private network.

| Setting | Value |
|---|---|
| OpenID configuration | `https://login.botframework.com/v1/.well-known/openidconfiguration` |
| Signing keys | `https://login.botframework.com/v1/.well-known/keys` |
| Algorithm | RS256 |
| Issuer | `https://api.botframework.com` |
| Audience | your bot's application (client) ID |

We did not just read the metadata — we ran the full validation against a live
production token, in about sixty lines of dependency-free Node:

```js
import { createPublicKey, createVerify } from "node:crypto";

const cfg  = await (await fetch("https://login.botframework.com/v1/.well-known/openidconfiguration")).json();
const jwks = await (await fetch(cfg.jwks_uri)).json();

const [h, p, sig] = token.split(".");
const header = JSON.parse(Buffer.from(h, "base64url").toString());
const claims = JSON.parse(Buffer.from(p, "base64url").toString());

const jwk = jwks.keys.find(k => k.kid === header.kid);
const ok  = createVerify("RSA-SHA256")
  .update(`${h}.${p}`)
  .verify(createPublicKey({ key: jwk, format: "jwk" }), Buffer.from(sig, "base64url"));

// then check: claims.iss === "https://api.botframework.com"
//             claims.aud === YOUR_APP_ID
//             claims.exp * 1000 > Date.now()
```

Result against a real inbound call:

```
signature (RS256)  : PASS
issuer             : PASS  (https://api.botframework.com)
audience           : PASS  (11111111-2222-3333-4444-555555555555)
not expired        : PASS
signing key id     : NEVoKQB_b2SXtMQCJ9XZP11gH2s
key endorsed for   : m365extensions, skype, outlook, omnichannel, acschat, msteams
```

That last line is a nice detail. The key set publishes **271 signing keys**, and
each carries channel *endorsements*. The SDK checks the activity's channel
against them, so a token minted for a Slack bot cannot be replayed against your
Teams endpoint. That check is already running in your container.

If your gateway can validate JWTs — most commercial ones can, and NGINX Plus
does it natively with `auth_jwt` — this is a far stronger control than any IP
list, and it does not break when Microsoft renumbers.

## A working pass-through config

For reference, the nginx that survived every test:

```nginx
location / {
    proxy_pass https://backend.internal.example.azurecontainerapps.io;
    proxy_http_version 1.1;

    # Azure routes on Host. Send the ORIGIN's hostname, not your own.
    proxy_set_header Host backend.internal.example.azurecontainerapps.io;

    # Azure ingress selects the app by SNI. Without these, the handshake dies.
    proxy_ssl_server_name on;
    proxy_ssl_name backend.internal.example.azurecontainerapps.io;

    # Authorization passes through untouched — do not "clean" it.
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    proxy_read_timeout 120s;
    client_max_body_size 10m;
}
```

Three lines carry all the risk: the two SNI directives and the `Host` header.

## What Microsoft could do better

- **A service tag that reflects inbound bot traffic.** `AzureBotService` exists
  and is the natural thing to reach for, but our measured callers were not in
  it. Either a tag that matches reality, or clear documentation that inbound
  channel traffic is out of scope for it.
- **Documented guidance for PaaS behind a gateway.** The SNI requirement of
  shared ingress is discoverable only by hitting a `502` and reading nginx
  error logs. It belongs in the custom engine agent documentation.
- **A published list of what the Bot Service token contains.** We had to decode
  one to establish that it carries no user identity. That is a common security
  review question and deserves a documented answer.

## Try it yourself

The whole rig is about 300 lines of TypeScript. The essentials:

```ts
// The SSO handler. In SDK 1.7.x the option is azureBotOAuthConnectionName —
// older samples say `name`, which no longer compiles.
super({
  storage: new MemoryStorage(),
  authorization: {
    SSO: { azureBotOAuthConnectionName: "SsoConnection", title: "Sign in" },
  },
});

// The ["SSO"] guard: only runs once a token is available.
this.onActivity(ActivityTypes.Message, this.handleMessage, ["SSO"]);
```

Two things that will cost you time if nobody warns you:

1. **The SDK reads lowercase environment variables** — `clientId`,
   `clientSecret`, `tenantId`. Use `MicrosoftAppId` and it crashes at startup
   with *"ClientId required in production"*.
2. **The Bot Service OAuth connection scope must be your own app's scope**,
   `api://botid-{appId}/access_as_user` — *not* `User.Read`. Get this wrong and
   the SSO token comes back with a Graph audience and every on-behalf-of
   exchange fails with `AADSTS50013`.

And put your request diagnostics **before** the SDK's JWT middleware. That way
you log what arrived even for requests that get rejected — which is the only
reason we could test the failure modes with curl instead of sending a hundred
Copilot messages by hand.

---

<p style="font-size: 0.95em; margin-top: 2em;">
  <a href="{{ '/about' | relative_url }}">About this blog &rarr;</a>
</p>
