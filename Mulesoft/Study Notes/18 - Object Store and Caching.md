# Chapter 18: Object Store and Caching

> Taught on: 07-04-2022, 22-04-2022

## Why This Matters

Not every piece of data a flow needs comes from the current request. Sometimes you also need a value from somewhere else entirely — a lookup value from Salesforce, a constant configuration setting, something that barely ever changes. Calling out to that external system on every single incoming request is wasteful, slow, and fragile: if that external system happens to be down at the exact moment your flow needs it, the whole transaction fails, even though the value you needed hasn't changed in months. **Object Store** is Mule's built-in way to hold onto data like this internally, so your flow doesn't have to keep depending on a live call to fetch it.

## Store and Retrieve Operations

At its simplest, Object Store works like a small key-value database sitting alongside your Mule application. It has two core operations:

- **Store** — saves a value under a **key** (think of the key as a variable name). Drag the Store operation into your flow, configure the key, and it will save whatever payload or value you point it at.
- **Retrieve** — reads a value back out, using the *same key* that was used to store it.

Because Store and Retrieve are separate operations, they don't have to happen in the same flow, or even close together in time. In the demo, one flow (triggered by a POST call) stored `car` and `price` values into the Object Store. A completely separate flow (triggered by a GET call with no request body at all) later retrieved those same values using the Retrieve operation and the matching keys.

This separation is genuinely useful: because the data lives in the Object Store rather than being passed directly between flows, **any number of other applications or flows can later read it** just by calling Retrieve with the right key. The flow that stored the data and the flow (or flows) that consume it don't need a direct Flow Reference or a synchronous call linking them together — they're decoupled.

A flow needs some kind of trigger to run its Store or Retrieve logic in the first place — in the demos this was either an **HTTP Listener** (triggered by an incoming request) or a **Scheduler** (triggered on a timer).

## Configuring Retention (TTL) and Size Limits

Object Store configuration includes a couple of settings worth knowing:

- **Max Entries** — how many separate key/value entries the store can hold.
- **Max entry lifetime** (TTL — time to live) — how long a stored value is kept before it's automatically deleted. The **default retention is 30 days**, but this is fully customizable based on the business requirement (you might set it to 1 day, 2 days, 10 days — whatever suits the use case).
- **Maximum payload size** that can be pushed into a single Object Store entry is **10 MB**.
- There is also a **Persistent** checkbox on the Object Store configuration. If it's checked, the stored data survives even if the application goes down or restarts — it won't be lost. If left unchecked, the data is only kept in the application's runtime memory, and it *will* be lost if the app goes down.

## Using Object Store as a Cache for Static/Constant Data

This is the real payoff of Object Store, and the pattern is worth understanding carefully because it comes up constantly in real integrations.

**The problem:** imagine an integration where the incoming request has some data that changes on every call — say, `car` and `price` coming fresh from the source system each time — but the flow *also* needs a value from an external system like Salesforce that almost never changes, for example a constant like `country = India`. If you call Salesforce on every single incoming request just to fetch this unchanging value, two bad things happen:

1. It's wasteful — you're making a network call to fetch a value that hasn't changed since the last time you asked.
2. It's fragile — if Salesforce happens to be down or slow at that moment, your *entire* flow fails, even though the value you needed hasn't actually changed and didn't need to be re-fetched at all.

**The solution:** build a second, separate flow whose only job is to occasionally refresh this static value and cache it locally:

1. A separate flow, triggered on a **Scheduler** with a long interval — the demo used a **Fixed Frequency of 30 days**, roughly once a month — queries Salesforce for the static value.
2. That flow **Stores** the result into the Object Store under a key (e.g. `data`).
3. Your main, business-logic flow — the one that handles every incoming request — never calls Salesforce directly anymore. Instead, for every request, it **Retrieves** the cached static value from the Object Store.
4. It then combines that retrieved static value with the dynamic data from the current request (e.g. `car`, `price`) using **Transform Message**, and sends the combined payload on to the target system.

The net effect: Salesforce gets called only once per scheduling interval (once a month, in this example) instead of once per transaction — which, at real-world volumes, could be the difference between one call a month and tens of thousands of calls a day.

This was demonstrated twice in the same session — once locally, and again after deploying the same setup to a sandbox environment — confirming the same behavior held after deployment: the dynamic values (`car`, `price`) still came straight from the source system on every request, while the static value (`country`) was served entirely from the Object Store cache rather than being re-fetched from Salesforce each time.

## Q&A: When Should You Reach for This Pattern?

**Q: When should you use static data storage (Object Store) in a flow?**

A: For **message reliability** — so that your flow's success doesn't depend on some external system being available and responsive on *every single transaction*, when the data it provides barely changes anyway.

## Quick Recap

- Object Store is a simple key-value store built into Mule: **Store** saves a value under a key, **Retrieve** reads it back using that same key.
- Store and Retrieve don't need to happen in the same flow — this decouples the flow that saves data from the flow(s) that consume it later.
- Configuration includes Max Entries, a max entry lifetime (default TTL = 30 days, customizable), a 10 MB max payload size per entry, and a Persistent checkbox (checked = survives app downtime, unchecked = lost on restart, runs on runtime memory).
- The main real-world use case: caching static/constant data (e.g. a rarely-changing Salesforce lookup value) so it's fetched from the external system only occasionally (via a separately scheduled flow), rather than on every single incoming request.
- This pattern improves both efficiency (fewer redundant external calls) and reliability (the main flow doesn't fail just because the external system is briefly unavailable, since it isn't calling it directly anymore).
