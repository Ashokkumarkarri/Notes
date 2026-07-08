# Chapter 09: Routers — Choice, Scatter-Gather, Round Robin, APIKit

> Taught on: 23-02-2022, 02-03-2022, 03-03-2022, 08-03-2022, 09-03-2022, 11-03-2022, 23-03-2022

## Why This Matters

Not every flow is a straight line from Listener to target. Real integrations constantly need to make decisions: should this message even be sent? Does it need to go to one target or several? Should those several targets be hit all at once, or one after another in order? MuleSoft has a dedicated **router** component for each of these shapes of problem, and picking the right one is less about memorizing syntax and more about correctly reading the business requirement in front of you. This chapter walks through the four router types covered in the course, side by side, so the differences between them are clear rather than blurry.

A small but important vocabulary point made early in the course: Logger and Transform Message are called **components**, HTTP Listener/Request are called **connectors**, and things like Choice, Scatter-Gather, Round Robin, and APIKit Router are specifically **routers** — they exist to direct the flow of a message, not to transform or transmit it.

## Choice Router — Condition-Based Routing

**Choice Router** is the tool you reach for whenever a message should only proceed down a certain path *if a business condition is true*. It's the direct equivalent of an if/else in general programming.

Typical examples used in the course: only forward data to the target system if the phone's `brand` equals `"OnePlus"`, or only forward it if `color` equals `"black"`. If the condition evaluates true, that specific route executes. If it evaluates false, execution falls into the Choice Router's **default route** — and if that default route has nothing configured in it (no HTTP Request, for instance), the message is simply not sent anywhere; the flow just ends quietly at that point (03-03-2022).

A single Choice Router isn't limited to one condition — it can hold any number of routes, each checking a different value and pointing to different downstream logic. In one demo, a first condition routed OnePlus-branded requests to the target's `order` endpoint; when a Samsung-branded request came in, it initially fell through to the default route because no condition matched it yet — the instructor then added a second condition/route specifically for Samsung, pointing to a different endpoint (`shipment`), after which Samsung requests routed correctly too (03-03-2022).

As covered in Chapter 8, best practice is to avoid writing long, deeply nested payload paths directly inside a Choice condition. Instead, extract the value you need into a variable first (via Set Variable), then reference the short variable name in the condition — e.g., `var.brand == "oneplus"` — which keeps conditions readable even against complex, deeply nested payloads (09-03-2022).

## Scatter-Gather — Parallel Multi-Target Routing

**Scatter-Gather** solves a different shape of problem: you have **one** source/one incoming request, but **multiple target systems all need that same data**. Rather than picking just one path (like Choice does), Scatter-Gather takes the single incoming message, splits it, and sends it to all configured targets **simultaneously, in parallel** (08-03-2022).

Example scenario from the course: a single API resource receives data from the source system, but two separate downstream systems — **Order** and **Shipment** — both need a copy of it. Scatter-Gather is configured with two routes, each containing its own HTTP Request connector pointed at one of the targets, with a Logger after each route printing that target's individual response. Scatter-Gather then combines the responses from every branch into a single combined output. It's not limited to two targets either — you can add more HTTP Request connectors inside additional routes to fan out to three, four, or more targets at once.

One practical wrinkle that came up in the live demo: the two targets didn't necessarily expect the same data format. Order accepted the payload as JSON directly, no conversion needed — but Shipment only accepted XML, and the source data arriving was JSON. The fix was to add a **Transform Message** step *inside that specific branch only*, converting JSON to XML right before that branch's HTTP Request call, while leaving the Order branch untouched. This is an important detail: each branch inside a Scatter-Gather can have its own independent processing logic before it reaches its target — they don't have to be identical.

## Round Robin — Sequential Distribution (vs. Scatter-Gather)

**Round Robin** also splits a message toward multiple targets, but the execution model is the opposite of Scatter-Gather: it runs **sequentially**, not in parallel. The first target (say, Order) is called first, and only once that call completes does execution move on to call the next target (say, Shipment) — one at a time, in order (08-03-2022).

This difference in execution model has a direct, practical consequence for how failures behave, which matters a great deal once you get to error handling:

| | Scatter-Gather | Round Robin |
|---|---|---|
| Execution | Parallel — all targets hit simultaneously | Sequential — one target at a time, in order |
| If one branch fails | The flow still proceeds, since branches are independent of each other | The flow does **not** proceed to the next target — it fails right there |
| Use it when... | Multiple targets need the same data at the same time, independently of one another | Targets need to be hit one after another, in a specific order |

There's no universal "better" router here — the choice depends entirely on the requirement. If the targets genuinely don't depend on each other and speed matters, Scatter-Gather is the natural fit. If there's a meaningful order to respect (or a dependency between calls), Round Robin is the correct choice.

## APIKit Router — Auto-Routing by Resource

**APIKit Router** is a different kind of router altogether — it isn't something you manually drag conditions or branches into. Instead, it's **auto-generated** the moment you import a RAML definition into Anypoint Studio and create flows from it. Each **resource** defined in your RAML (e.g., `/order`, `/shipment`) automatically gets its own dedicated flow, and the APIKit Router sits at the front, inspecting each incoming request's path and routing it to the correct resource's flow — with no manual connector wiring required inside the flow logic itself (23-02-2022).

A practical rule that follows from this: if a flow is already being invoked via the APIKit Router (because it corresponds to a RAML resource), you should **not** also manually configure a Listener on that flow — the router itself is what receives and dispatches the request; adding a redundant Listener doesn't make sense in that setup (09-03-2022).

This is also the router underlying the request-validation behavior discussed in earlier RAML-focused sessions: if an incoming request hits a resource path that doesn't exist in the RAML, the APIKit Router is effectively what returns a `404 Not Found`; if the method or media type doesn't match what that resource declares, it's what enforces the `405` / `415` responses (02-03, 03-03-2022, and see Chapter 6 for the format-restriction side of this).

## Common Mistakes / Things Students Got Wrong

- **Forgetting that different branches inside a Scatter-Gather may need different transformations.** In the first test run of the Scatter-Gather demo, the Order branch worked fine (it accepted JSON as-is), but the Shipment branch failed with an "unsupported media type" error because its target only accepted XML — the source was sending JSON to both branches unchanged. The fix was adding a Transform Message *specific to that one branch* (08-03-2022).
- **Referencing raw, deeply nested payload paths inside Choice conditions** instead of extracting the value into a variable first — technically works, but becomes unreadable and error-prone fast (09-03-2022).
- **Confusing which router to reach for based on failure tolerance.** Since Scatter-Gather continues even if one branch fails, while Round Robin halts on the first failure, picking the wrong one for a requirement that actually needs strict, dependent ordering (or vice versa) leads to subtly wrong behavior that's easy to miss until something actually fails.

## Quick Recap

- **Choice Router** — condition-based routing; if true, go down that route; if false, fall to the default route (which may do nothing at all).
- **Scatter-Gather** — one source, multiple targets, executed **in parallel**; combines all responses; a failure in one branch doesn't stop the others; each branch can have its own independent transform logic.
- **Round Robin** — one source, multiple targets, executed **sequentially**; a failure at one step stops the flow from proceeding to the next.
- **APIKit Router** — automatically generated from a RAML definition; routes incoming requests to the correct resource-specific flow based on the request path, with no manual wiring needed.
- Keep Choice conditions readable by referencing variables instead of long raw payload paths.
