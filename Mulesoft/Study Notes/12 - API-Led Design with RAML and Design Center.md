# Chapter 12: API-Led Design with RAML and Design Center

> Taught on: 22-02-2022, 23-02-2022, 28-02-2022, 04-03-2022, 15-03-2022

## Why This Matters

Up to this point in the course, a Mule flow was just a Listener plus whatever components you dragged onto the canvas — a Logger, a Transform Message, an HTTP Request. That kind of flow works, but it has a serious weakness: it will accept *anything*. Send it JSON, XML, plain text, a GET, a POST, a DELETE — it doesn't care. In real integration work that's a liability, not a feature. A source system might accidentally send the wrong format, or someone might hit your endpoint with the wrong method, and your flow would just try to process it anyway, probably failing in a confusing way deeper in the logic.

The fix is to stop building "just a flow" and start designing an **API** — a formal contract that says exactly what this endpoint accepts and rejects, before a single line of flow logic is even written. That contract is written in a language called RAML, designed in a tool called Design Center, and once it exists, Anypoint Studio can generate most of the flow skeleton for you automatically. This is the foundation of what MuleSoft calls **API-led connectivity**, and it's the difference between a flow that quietly accepts garbage and one that fails loudly and correctly with the right HTTP status code.

## APIs, REST, and SOAP

An **API** (Application Programming Interface) is what lets you place restrictions on a Mule application: which HTTP methods it accepts, which data formats (media types) it will process, and — eventually, once security is layered on top (see Chapter 13) — who is even allowed to call it at all.

Mule supports two styles of API:

- **REST** (Representational State Transfer) — the flexible, modern option. It supports all the common media types (XML, JSON, CSV, Java, text) and all the standard HTTP methods (GET, POST, PUT, PATCH, DELETE).
- **SOAP** (Simple Object Access Protocol) — the older, more rigid option. It only supports the POST method and only accepts XML data.

Most legacy applications you'll encounter in the field were built on SOAP; most modern applications are built as REST APIs, precisely because REST gives you more flexibility in format and method. REST APIs are the ones designed using **RAML**.

## What Is RAML?

**RAML** stands for **RESTful API Modeling Language**. It's the language you use to describe a REST API's shape — its resources, its methods, its accepted data formats — *before* you build the actual Mule flow logic behind it. Think of RAML as a blueprint: you're not writing implementation code, you're writing a specification that both documents the API and drives Anypoint Studio to generate a matching flow skeleton.

RAML-based APIs are built inside **Design Center**, a tool that lives inside Anypoint Platform (the same cloud platform that hosts Runtime Manager, where you deploy your applications).

## Designing an API in Design Center

To start, open Design Center and click **New API**, then give it a name. The naming convention taught in this course is important and is treated as a rule in real projects, not just a suggestion: **name the API after the source system it integrates with.** If Salesforce is the source system, the API is called "Salesforce API." If a database is the source, it's "Database API." A live demo in the course used a source system called "Amazon," so the API was named "Amazon API."

Once the API is created, Design Center auto-populates two fields for you:

- **Version** — RAML has had multiple versions over the years (an older 0.8 was used a few years before this course; by the time of this course, the current version was referred to as 1.0, and elsewhere in the same session as 1.4). You don't need to memorize the version history — just know that a new API in Design Center gets the current version automatically.
- **Title** — generated from the API's name (e.g., "Amazon API" → title "Amazon API").

Separately, you'll also set a **version identifier** for the API itself (e.g., `v1`, `v2`, `v3`) — this is different from the RAML language version. Standard practice is to always start a new API at `v1`.

## The Structure of a RAML File

A RAML file builds up from general (protocol/media type) down to specific (individual resources and their behavior). Here's the structure, piece by piece:

**1. Base URI configuration** — sits near the top of the file and defines:
- **Protocols** — which protocol(s) the API will accept requests over (e.g., HTTP, HTTPS).
- **Media types** — which data formats the API will accept, written using MIME-type syntax:
  - JSON → `application/json`
  - XML → `application/xml`
  - Text → `text/plain`
  - Java → `application/java`

**2. Resources** — below the base configuration, RAML defines **resources**, which represent the API's individual endpoints/paths. Examples used in the course: `/order`, `/demo`, `/dummy`, `/shipment`. If a caller hits a path that isn't declared as a resource in the RAML, the API correctly responds with a **404 Not Found** — the resource simply doesn't exist as far as the API contract is concerned.

**3. Methods per resource** — each resource declares which HTTP method(s) it accepts (for example, a resource might accept only POST). If a caller uses the wrong method — say, a GET where only POST is defined — the response is **405 Method Not Allowed**.

**4. Body/media type per resource** — each resource also declares what body/media type it expects (for example, only JSON). If the wrong media type is sent — say, XML to a JSON-only resource — the response is **415 Unsupported Media Type**.

**5. Status codes** — a well-formed request that matches the right resource, the right method, and the right media type returns **200 OK**.

Put together, this is the complete enforcement chain a RAML-designed API gives you for free, with no manual "if" logic in your flow:

| What went wrong | Status code |
|---|---|
| Resource/path doesn't exist | 404 Not Found |
| Wrong HTTP method used | 405 Method Not Allowed |
| Wrong media type/body sent | 415 Unsupported Media Type |
| Everything correct | 200 OK |

One small but reassuring detail: extra blank lines and spacing inside the RAML file are not mandatory — they exist purely for readability, so don't worry about getting whitespace "wrong."

## Importing RAML into Anypoint Studio

Once your RAML is designed in Design Center, you bring it into Anypoint Studio to actually build the working application. There are two ways to do this:

**Method 1 — the standard project-creation flow:** when creating a new Mule project in Anypoint Studio, there's an option to import/browse and publish the RAML directly from Design Center. This keeps the project properly linked back to the design.

**Method 2 — manual/local import** (useful when you don't want a live link back to Design Center, or Design Center access is unavailable): under `src/main/resources` in your project, create a new file with a `.raml` extension (e.g., `AmazonAPI.raml`), copy the RAML content from Design Center, and paste it into that local file. Then right-click the RAML file and choose **Mule → Generate flows from local REST API**. The important caveat with this manual approach: it is **not linked back to Design Center** — if the RAML is later edited in Design Center, those changes will not automatically sync into your local copy. (A related, more advanced RAML feature called **fragments** was mentioned as a topic for a later stage, once the basic skeleton/structure is solid — it wasn't covered in depth in these sessions.)

Either way, once the RAML is imported, **Anypoint Studio automatically generates the flows matching the RAML's resources and methods** — you don't manually drag and drop a Listener or Logger from the palette to get the basic skeleton. These auto-generated flows are called **APIKit Router flows**, because they're generated from the REST API definition and routed through a component called the **API Kit Router**.

For every resource declared in the RAML, Studio generates its own dedicated flow. For example, an API with `order` and `shipment` resources will produce, at minimum: a main router flow, an "order" flow, and a "shipment" flow — plus an error-handling flow. So a two-resource API typically produces three (or more) generated flows in total.

## How the API Kit Router Works

The **API Kit Router** is the component responsible for directing an incoming request to the correct generated flow, based on the resource name present in the request path. When a request comes in through the HTTP Listener, it's handed to the API Kit Router, which looks at which resource was targeted (e.g., `order` vs `shipment`) and routes execution into that resource's specific flow.

The Listener path pattern typically used with an API Kit setup looks like `/api/*`, where the `*` is a wildcard representing whatever resource name follows (e.g., `/api/order` or `/api/shipment`). A key rule worth remembering for later: **you should not manually configure a separate Listener on a flow that's already being invoked via the API Kit Router** — the router itself is what receives and dispatches the request; adding your own Listener on top of that is redundant and incorrect.

## Testing the API in Postman

With the demo API's `order` resource configured to accept only POST + JSON, four test scenarios show the enforcement in action:

- **GET** instead of POST → **405 Method Not Allowed** (method restriction enforced).
- **POST with XML body** (resource expects JSON) → **415 Unsupported Media Type**.
- **Wrong/incorrect resource path** → resource-not-found error.
- **POST with correct JSON body** → **200 OK**, success response returned.

Postman here is simply standing in for the real source system (in the demo's case, an imaginary system called "Amazon") — there's no actual external integration happening; it's purely a way to fire test requests at the endpoint.

## Adding a Second Resource and Deploying to CloudHub

Adding another resource is straightforward: a second resource, **shipment**, was added to the same API by copying the existing `order` resource's setup and adjusting it. As noted above, each resource gets its own dedicated flow once regenerated in Studio.

To move from local testing to testing the CloudHub-deployed version, the process is the same one used throughout the course: replace `localhost` and the local port in your Postman request with the CloudHub deployment URL (found under the app's Settings in Runtime Manager). After deployment, the same test pattern applies — wrong media type still correctly returns an error, and correct JSON still returns 200 OK with the appropriate success message — confirming the API's restrictions hold in the cloud, not just locally.

## Debugging an API Kit Flow

Running an application normally only shows you the end result — data went in, something happened, a response came back — but it doesn't show what happened at each individual component along the way. **Debug mode** solves this by letting you inspect the flow at a component-to-component level: how data moves from the Listener, into the API Kit Router, and into the specific resource flow it was routed to.

To use it:
1. Add a **breakpoint** on the relevant component or flow in Anypoint Studio.
2. Right-click the project → **Run in Debug Mode**.
3. Send a request (e.g., from Postman) — execution pauses at the breakpoint.
4. Inspect the **payload** (the actual data sent) while paused.
5. Use the "resume" control (an arrow icon, typically in the top-right of the debug view) to step execution forward to the next component.

In the course demo, triggering the `shipment` resource paused execution at the shipment flow's breakpoint with the payload visible; triggering `order` with the wrong media type (XML, when JSON was required) showed the request reaching the API Kit Router but failing once it tried to move forward, because the RAML enforces JSON-only for that resource; triggering `order` with correct JSON routed cleanly through the router into the order flow and completed with a 200 response visible in Postman. Debug mode runs exactly the same logic as a normal run — the only difference is that you can pause and inspect it step by step.

## Manual Flows vs. API-Based Flows: Why the Restriction Matters

It's worth being explicit about the contrast this whole chapter is built on, because it came up again later in the course as a recap point. There are two ways to build an HTTP-based Mule application:

1. **Manual application** — drag and drop a Listener, Logger, and other components directly from the palette. This gives you **no restrictions**: any HTTP method and any media type will be accepted, with nothing stopping a malformed or unexpected request from reaching your business logic.
2. **API-based (RAML/API Kit) application** — flows are auto-generated from a RAML definition. REST APIs *can* technically support all methods and all media types, but RAML lets you explicitly restrict what a given resource accepts (e.g., only POST, only JSON).

The crucial point: **restrictions can only be enforced through the RAML/API Kit approach.** A manually built flow simply has no mechanism to say "reject anything that isn't JSON" — it will happily process whatever arrives. This is precisely why API-led design exists: it moves format and method validation out of your business logic and into a declarative contract that Mule enforces automatically, before your flow logic ever has to think about it.

It's also worth noting what this chapter's restrictions *don't* give you: security. An API designed this way controls format and method, but not *who* is allowed to call it — that's a separate concern, covered in the next chapter, handled through API Manager and policies.

## Common Mistakes / Things Students Got Wrong

- A student's HTTP Request in a later hands-on task was pointed at `/api/salesforce`, but the target application only actually exposed `order` and `shipment` as valid resources — any other resource name returns a 404. The fix was simply to check the failing request's URL in the console/logs and correct the path to match a resource that actually exists in the target's RAML.
- It's easy to assume any REST API automatically restricts formats/methods. It doesn't — a REST API supports *all* formats and methods by default; it's the specific RAML definition for *your* resource that narrows this down. The restriction comes from what you declare, not from REST itself.

## Quick Recap

- APIs let you restrict a Mule application's accepted HTTP methods and media types — something a manually built (drag-and-drop) flow cannot do.
- REST supports all methods/media types and is designed with RAML; SOAP only supports POST + XML.
- RAML = RESTful API Modeling Language; APIs are designed in Design Center, part of Anypoint Platform.
- Name the API after its source system (e.g., "Salesforce API"). Start versioning at `v1`.
- RAML structure, top to bottom: base URI (protocols + media types) → resources (endpoints) → methods per resource → status codes.
- Enforcement chain: wrong resource → 404; wrong method → 405; wrong media type → 415; all correct → 200.
- Import RAML into Anypoint Studio either via the project-creation flow (linked to Design Center) or by manually pasting it into a local `.raml` file under `src/main/resources` (not linked — won't auto-sync).
- Importing RAML auto-generates APIKit Router flows — one flow per resource, plus a router flow and an error-handling flow.
- The API Kit Router directs each incoming request to the correct resource flow based on the request path; don't add a manual Listener on top of an API Kit–routed flow.
- Debug mode lets you step through a flow component by component using breakpoints — same logic as a normal run, just pausable.
- API-based design gives you restrictions, not security — security (who can call the API) is a separate layer, covered next.
