# Chapter 11: Scaling Mule Applications

> Taught on: 10-03-2022, 11-03-2022, 21-04-2022, 23-03-2022

## Why This Matters

An integration that works fine in testing with a handful of sample requests can fall over completely in production once real volume hits it — either because a single request carries far more data than expected, or because thousands of requests are arriving at once. MuleSoft gives you two independent knobs to deal with this: making each worker "bigger" (vertical scaling), or adding more workers (horizontal scaling). Knowing which one actually solves your specific problem — and what it costs — is a decision every Mule developer eventually has to help make, since the two approaches solve genuinely different problems and cost very differently.

## Vertical Scaling — vCore

**Vertical scaling** means increasing the compute and memory allocated to a **single worker**. This is the right tool when the problem is that **one individual request carries a huge payload** — the worker processing that one request simply needs more headroom to handle it.

- The default allocation is **0.1 vCore**.
- It can be scaled up, with the course citing a ceiling around **32 GB**.
- Cost implication: this is expensive. Larger allocations (16 GB/32 GB range) were described as potentially running into **crores of rupees annually**, since vCore directly consumes MuleSoft platform resources that the client is billed for (10-03-2022).

## Horizontal Scaling — Workers

**Horizontal scaling** means increasing the **number of workers** — where each worker is effectively its own separate server instance, physically located in a different place (the course gave the example of one worker in Ireland and another in the UK or Australia — no two workers run in the same location). This is the right tool when the problem is **volume**: a high number of incoming requests overall, regardless of how large any single one of them is.

- Workers are numbered starting from **0**, with **1 worker by default**.
- Adding more workers spreads the incoming load across multiple servers — the course's example: of 1,000 incoming requests, roughly half might be handled by one worker/server and the rest by another, once a second worker is added.
- The platform's **default worker limit is typically around 8**, though higher limits can be requested from MuleSoft where genuinely needed (11-03-2022).

## The Rule of Thumb

| Problem | Solution |
|---|---|
| One request, huge payload | **Vertical scaling** (vCore) |
| High volume / many concurrent requests | **Horizontal scaling** (Workers) |

This is a genuinely useful mental shortcut: before reaching for either scaling knob, first ask which kind of load problem you actually have. Scaling vCore doesn't help if the real issue is that too many separate requests are arriving at once — you need more workers for that. Conversely, adding more workers doesn't help a single oversized payload that's choking one worker — that needs more vCore.

## Cost Tradeoffs and Who Decides

Both vCore and worker count directly increase cost, so efficient application design means avoiding over-provisioning either one beyond what's actually needed. There's no fixed, universal formula for exactly how much to scale — it depends on the specific client's budget and requirements. In practice, this decision is typically made by whoever is closest to the client relationship — a team leader, project manager, or whoever is directly interfacing with the client — based on the client's stated needs and budget, rather than being a decision any individual developer makes unilaterally (10-03-2022). Teams are still expected to design flexible, resource-efficient applications that work within whatever budget has been set, rather than reflexively over-provisioning "just in case."

## On-Premise vs. Cloud Scaling

Everything above describes scaling on Anypoint Platform's **cloud** deployment model (CloudHub). **On-premise deployments are fundamentally different: there is no scaling capability at all.** On-premise setups rely purely on a fixed "worker" concept and are entirely memory-bound — whatever memory is physically available on that on-premise infrastructure is what you have to work with; there's no equivalent of dynamically adding vCore or spinning up additional cloud workers on demand (21-04-2022).

This is a meaningful factor when a client is choosing (or has already chosen) between on-premise and cloud deployment: cloud gives you flexible, billable vertical and horizontal scaling options that can respond to changing load; on-premise gives you a fixed capacity ceiling determined by the hardware you've already provisioned, with no built-in mechanism to scale beyond it on demand.

## Common Mistakes / Things Students Got Wrong

A student asked directly how to decide the right balance between scaling and budget, given that there's no fixed rule. The honest answer given was that this genuinely varies by organization and by client — it's a judgment call typically made by the team lead or project manager based on the client's specific requirements, not something with a universal formula. The instructor noted that designing a more resource-efficient application (so you need less scaling in the first place) would be covered in a later session — reinforcing that the first lever to pull isn't "scale more," it's "design more efficiently" (10-03-2022).

## Quick Recap

- **Vertical scaling (vCore)**: bigger single worker, for handling one very large payload. Default 0.1 vCore, scalable toward 32 GB, expensive at high allocations.
- **Horizontal scaling (Workers)**: more workers, for handling a high volume of concurrent requests. Default 1 worker (numbered from 0), typical platform default limit ~8.
- Rule of thumb: **big single payload → vCore. Lots of requests → workers.**
- Scaling decisions are cost tradeoffs usually made at the team-lead/PM level based on client budget — avoid over-provisioning either dimension.
- **On-premise deployments have no scaling capability at all** — they're fixed, memory-bound "worker" setups, unlike the flexible vertical/horizontal scaling available in the cloud.
