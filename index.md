---
layout: default
title: "Field notes"
description: "Hands-on testing of Microsoft 365 Copilot, Cowork and Copilot Studio in real Azure environments. Measurements, not documentation."
permalink: /
---

<p style="font-size: 0.85em; color: #606c71; border-left: 3px solid #dce6f0; padding: 0.4em 0 0.4em 0.9em; margin: 0 0 1.8em 0;">
  <strong>Personal blog.</strong> This site reflects my own testing and my own
  opinions. It is not an official Microsoft statement and it is not official
  Microsoft documentation in any way. For authoritative guidance always refer to
  <a href="https://learn.microsoft.com/">Microsoft Learn</a>.
</p>

Field notes from building Microsoft 365 Copilot agents and running them in enterprise
Azure environments. Everything here was measured in a lab tenant rather than taken from
documentation. Where a claim is reasoning rather than measurement, it says so.

## Articles

{% for post in site.posts %}
### [{{ post.title }}]({{ post.url | relative_url }})

<span style="font-size: 0.85em; color: #606c71;">{{ post.date | date: "%-d %B %Y" }}</span>

{{ post.description }}

{% endfor %}

---

<p style="font-size: 0.95em; margin-top: 2em;">
  <a href="{{ '/about' | relative_url }}">About this blog &rarr;</a>
</p>
