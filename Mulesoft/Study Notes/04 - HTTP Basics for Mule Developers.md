# Chapter 04: HTTP Basics for Mule Developers

> Taught on: 21-02-2022, 22-02-2022, 23-02-2022, 02-03-2022, 04-04-2022

## Why This Matters

HTTP is the protocol underneath almost every integration you'll build in this course. Whether your Mule app is *receiving* data from a source system or *sending* data out to a target system, if either side is API-based, HTTP (or its secure sibling, HTTPS) is how the conversation happens. Getting comfortable with URL structure, the difference between a Listener and a Request, HTTP methods, and status codes isn't optional groundwork you can skip — it's the vocabulary every later chapter (especially Chapter 05, where you build and deploy your first app) assumes you already have.

## Two Connectors, Two Directions

MuleSoft gives you two distinct HTTP-related connectors, and the single most important thing to understand about them is that **they represent opposite directions of data flow**:

- **HTTP Listener** — connects a **source system into** your Mule app. You expose an endpoint (a URL) via the Listener, and hand that endpoint to the source system so it can trigger data *into* your flow. Every flow that starts by receiving a request begins with a Listener.
- **HTTP Request** — connects your Mule app **out to** another (target) system. Whenever your Mule app needs to *send* data onward to another API-based system, you use an HTTP Request connector to make that outbound call.

Both are referred to as **connectors**. A simple rule of thumb that was repeated across sessions: any time Mule talks to an API-based system — in either direction — an HTTP connector is involved. Listener to *receive*, Request to *send*.

Once a request lands on a Listener in an API-based (RAML/APIKit) application, it doesn't go straight to your business logic — it first passes through the **APIKit Router**, which reads the resource name in the request path and routes the message to the correct flow (e.g., a request for the "shipment" resource routes specifically to the shipment flow, while a request for "order" routes to the order flow). You'll see this router in action in Chapter 05.

## URL Structure — The Anatomy of an Endpoint

Every HTTP endpoint you'll configure or call in this course follows the same basic pattern, known as a URL (Universal Resource Locator):

```
protocol://hostname:port/path
```

Broken down:
- **protocol** — `http` or `https`
- **hostname** — the address of the server (e.g., `localhost` for local testing, or a CloudHub domain once deployed)
- **port** — the specific port number the application is listening on
- **path** — the specific resource/route within that application (e.g., `/order`, `/demo`)

Understanding this formation precisely matters because you will be constructing and pasting these URLs into Postman constantly to trigger your flows — a single misplaced character (or, as the course explicitly warned, an accidental extra space) is enough to break a request.

The same structure applies symmetrically on the "sending" side: when you configure an **HTTP Request** connector to call a target system, you fill in the equivalent fields individually — **Host**, **Port**, **Protocol**, **Method**, and **Path** — rather than typing one combined URL string. The Method you configure must match what the target resource actually supports (if the target only accepts POST, your Request connector's method must be POST), and the Path must exactly match the resource path the target application exposes. Getting the path wrong here is one of the most common causes of a `404 Not Found` when calling out to a target system — this exact mistake happened live in the 03-03-2022 session, where a request was sent to `/api/salesforce`, a resource that didn't exist on the target; the fix was correcting the path to the target's actual exposed resource (`/api/order`).

## Local Testing vs. CloudHub Testing

Anypoint Studio's default example Listener configurations point at `localhost` and a local port — this only works when you're testing the application while it's running inside Studio on your own machine (**local testing**).

Once the application is deployed and running on CloudHub, `localhost` and the local port no longer point anywhere useful — you must **replace them with the application's actual CloudHub URL** in whatever tool (Postman) you're using to trigger it. That CloudHub URL can be found under the deployed application's **Settings** in Runtime Manager; copy it and substitute it in place of `localhost:port` in your requests. As with URL structure in general, exact formatting matters — watch for stray spaces when pasting.

## Testing a Flow with Postman

Postman fills the role of the source system throughout this course (in a real production scenario, this role belongs to whatever actual system is triggering your integration — Salesforce, an e-commerce platform, and so on). The basic testing loop looks like this:

1. Open the HTTP Listener in Anypoint Studio and note its configured protocol, host, and port.
2. Open Postman and paste in the resulting URL.
3. Trigger the request.
4. Check the Anypoint Studio console to confirm the data was received by the flow.
5. Check the status code that Postman shows for the response — a **`200`** confirms the request succeeded and the data reached the Mule application.

You can trigger multiple requests from Postman in a row to confirm consistent, repeatable behavior — this is a normal part of verifying a flow before moving on.

## HTTP Methods

The set of HTTP methods came up specifically in the context of comparing REST and SOAP APIs (Chapter 01 covers this comparison in full): REST APIs support all of the standard HTTP methods — **GET, POST, PUT, PATCH, DELETE** — while SOAP APIs support only **POST**. In practical terms in this course, **POST** is by far the most common method used, since most of the exercises involve a source system actively *pushing* data into Mule (rather than Mule *retrieving* data via GET). As one session put it plainly: GET is used specifically when *retrieving* data from a target system, while POST is used when *triggering/pushing* data into a flow.

## HTTP Status Codes — The Full List

Status codes are one of the most important diagnostic tools you have in this course. They're described as fixed, universal codes — they mean the same thing regardless of which technology or platform you're working with, not something specific to Mule. Learning to instantly recognize what each one is telling you will save enormous debugging time later. Here is the complete list referenced across the course, consolidated:

| Code | Meaning | When you'll see it |
|---|---|---|
| **200** | Success (OK) | The request was well-formed, matched the resource/method/media-type restrictions, and was processed successfully. |
| **400** | Bad Request | The request itself is malformed in some way. |
| **401** | Unauthorized | Missing or invalid credentials — e.g., calling a Basic-Auth-secured endpoint without a username/password, or without the correct Client ID/Secret. |
| **403** | Forbidden | The caller is identified but not permitted to access the resource. |
| **404** | Resource Not Found | The path/resource triggered doesn't exist — either because it isn't defined in the RAML, or (when calling a target system) the path configured in the HTTP Request connector doesn't match anything the target exposes. |
| **405** | Method Not Allowed | The wrong HTTP method was used for that resource — e.g., sending a GET to a resource that only accepts POST. |
| **415** | Unsupported Media Type | The wrong data format was sent — e.g., sending XML to a resource whose RAML restricts it to JSON only. (The exact code number wasn't stated in the first session it came up, but was confirmed as 415 in the next session's recap.) |
| **500** | Internal Server Error | A general server-side failure. |
| **502** | Proxy Error (Bad Gateway) | An intermediary/proxy-level failure. |
| **503** | Service Unavailable | The target service is temporarily unable to handle the request. |
| **504** | Timeout | The request took too long and timed out. |

The three you'll personally trigger and observe most often while building and testing your own RAML-restricted APIs are **404** (wrong resource), **405** (wrong method), and **415** (wrong media type) — these three map directly onto the three restrictions a RAML resource defines (path, method, and accepted body type), so seeing one of them is almost always a sign that a specific, identifiable restriction was violated, not a mysterious failure.

## HTTP vs. HTTPS — Mule 3 vs. Mule 4

Switching a Listener's protocol from HTTP to the secure HTTPS is one area where Mule 3 and Mule 4 genuinely differ in how much setup is required:

- **In Mule 3**, switching to HTTPS requires configuring **certificates** — specifically always a pair, a **Truststore** and a **Keystore**, each provided along with a password, and configured under `src/main/resources`.
- **In Mule 4**, this certificate configuration is **not mandatory** — you can change the Listener's protocol directly from HTTP to HTTPS without setting up certificates separately.

Whichever version you're on, once a Listener's protocol is switched to HTTPS, the calling client (Postman, or the real source system) must call using the **HTTPS** URL — triggering the old HTTP URL will simply fail to reach the application. This matters because it's an easy mistake to make right after changing a protocol setting: you change the Listener, but forget to update the URL you're testing against in Postman.

## Common Mistakes / Things Students Got Wrong

- **Confusing which connector goes which direction.** Early on, it's easy to reach for an HTTP Request when you actually need a Listener (or vice versa) — remembering "Listener receives, Request sends" resolves this instantly.
- **Pointing an HTTP Request at a resource path the target doesn't expose**, producing a 404 — demonstrated live in the 03-03-2022 session, where `/api/salesforce` was used instead of the target's actual `/api/order` resource. The fix was checking the failing request's URL in the logs and correcting the path.
- **Forgetting to swap `localhost:port` for the CloudHub URL** after deploying — this was flagged repeatedly across multiple sessions as the single most common reason a request that worked locally suddenly "stops working" once the app is live on CloudHub. It isn't broken; it's simply still being called at the wrong address.
- **Continuing to call the old HTTP URL after switching a Listener to HTTPS** — since the protocol itself changed, the old-style URL will not reach the application anymore.

## Quick Recap

- **HTTP Listener** receives data into your Mule app (source → Mule); **HTTP Request** sends data out (Mule → target). Both are called connectors.
- A URL always follows `protocol://hostname:port/path` — the same shape applies whether you're configuring a Listener or filling in Host/Port/Protocol/Method/Path on a Request connector.
- Local testing uses `localhost:port`; once deployed, you must swap that for the app's actual CloudHub URL (found in Runtime Manager → app Settings).
- Postman acts as your stand-in source system for testing — trigger a request, check the console for received data, and check the response status code (200 = success).
- REST supports all HTTP methods (GET, POST, PUT, PATCH, DELETE); SOAP supports only POST. POST is the most common method used across this course's exercises.
- Full status code list: 200 (Success), 400 (Bad Request), 401 (Unauthorized), 403 (Forbidden), 404 (Not Found), 405 (Method Not Allowed), 415 (Unsupported Media Type), 500 (Internal Server Error), 502 (Bad Gateway), 503 (Service Unavailable), 504 (Timeout). 404/405/415 map directly to path/method/media-type restrictions in your RAML.
- Mule 3 requires a Truststore + Keystore to enable HTTPS on a Listener; Mule 4 lets you switch the protocol directly, with no certificate setup required.
