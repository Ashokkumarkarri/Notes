# Chapter 15: Logging, Debugging, and Common Errors

> Taught on: 21-02-2022, 08-03-2022, 09-03-2022, 14-03-2022, 17-03-2022, 18-03-2022, 08-04-2022

## Why This Matters

An integration flow that works perfectly in the demo will, sooner or later, fail somewhere in production — a target system will time out, a source system will send the wrong field, a port will already be in use when you try to restart the app. When that happens, your only real window into what went wrong is what you chose to log, and how quickly you can filter thousands of unrelated log lines down to the one transaction that actually failed. This chapter is about building that discipline: what to log and at what level, how to find one transaction's logs in a sea of others using the Correlation ID, how to step through a flow live in Anypoint Studio, and — because this is the kind of thing you only really learn by hitting it — a reference list of the specific errors students in this course actually ran into, what caused them, and how they were fixed.

## Log Levels: INFO vs. DEBUG

Two logging levels are used consistently throughout the course, and the distinction between them is not just cosmetic — it's a rule about what's safe to log routinely versus what's only meant for occasional deep troubleshooting.

- **INFO level** — always printed by default in runtime logs, with no extra configuration needed. Use this for lightweight, small, traceable fields — a unique ID like an order number, an outbound segment number, a shipment ID. **Never log the entire payload at INFO level** — this was stated explicitly as a rule of thumb, because INFO logs print unconditionally and constantly, and a full payload at that volume wastes storage for little benefit.
- **DEBUG level** — logs the complete payload, which can be large. Crucially, debug-level logs are **not printed or stored unless the logging category is explicitly enabled** in Runtime Manager. This makes DEBUG the right place for full payloads: they're available when you need them, but they don't clutter the logs (or eat storage) the rest of the time.

The workflow this enables: when troubleshooting a specific failed request, you (1) enable debug logging for the relevant category in Runtime Manager, (2) ask the source system to re-trigger the failing request, (3) inspect the full payload that gets logged at debug level, and then (4) **disable debug logging again afterward** once the issue is resolved — leaving debug logging on permanently is not the intended usage pattern.

To enable/view debug logs: go to the deployed application in **Runtime Manager → Settings → Logging**, select the **Debug** level, enter the relevant category name, and click **Apply Changes**. Use the same screen to switch it back off once you're done.

**Logging category naming convention:** `com.<companyName>.<flowName>.<targetName>` (example used: `com.logging.loggingflow.target`). This is flexible and should follow whatever standard the organization has adopted — there's no single universal rule for logging standards; it depends heavily on the organization. One past client mentioned in the course, Unilever, enforced very strict logging standards with architect-level code review — a reminder that "good enough for a demo" logging discipline isn't always good enough for a real client engagement. Note also: a category is not strictly mandatory for INFO-level logs — if none is set, the log still prints by default — but keeping one is still treated as good practice.

### The Recommended Pattern

The pattern reinforced repeatedly across multiple sessions is: store the incoming unique tracking ID (order number, outbound segment number, etc.) into a **flow variable** near the very start of the flow, then reference that variable (e.g., `vars.order` or `var.order`) in every INFO-level logger throughout the flow — including a final "flow completed" log at the very end. This means a single request can be traced end-to-end using nothing but that one ID, regardless of how many transformation or routing steps it passed through. A concrete standard sequence used in one real-project walkthrough (the JDA-to-Pink integration, see below) was:

1. Log the **received payload** (raw, as it arrived from the source) — at **debug** level.
2. Set variables for tracking — e.g., the outbound segment number, and where applicable, an IB flag value or operation code.
3. Log the **tracking ID** (e.g., outbound segment number) — at **info** level.
4. After transforming the payload (e.g., XML → JSON), log the **transformed payload** — again at **debug** level.
5. Log a confirmation message right before and after the outbound call (e.g., "Connection initiated to target" ... "Successfully triggered to target"). If that final success log never appears, that absence itself tells you the target didn't respond, or something failed and execution went into exception handling instead.

A single logger component can print multiple values together (e.g., a variable *and* a payload in the same log line) — you don't need a separate logger for every value.

## Log Retention: Runtime Manager vs. Anypoint Monitoring

There are two places to check logs, and they behave quite differently in terms of how much history they keep:

- **Runtime Manager application logs** — limited storage. Figures cited across different sessions in the course were "roughly 100 MB" and, separately, "roughly 500 MB" (the exact number appears to have been described loosely/differently at different points — treat it as "capped and relatively small," not a precise spec), retained for up to **30 days**.
- **Anypoint Monitoring** — retains log backups for up to **90 days**, with no fixed storage size limit mentioned — better suited for larger volumes of historical data or longer-term investigation.

Both are legitimate places to look, but Anypoint Monitoring is the better tool when you need to go further back in time or expect a large volume of log data.

## Correlation ID: Tracing One Transaction Among Thousands

The **Correlation ID** (also called the event ID) is a unique identifier that Mule automatically generates, in real time, for every single incoming request. It exists purely for **tracking and debugging purposes** — it has no bearing on business logic. This is a Mule 4 concept specifically; Mule 3 does not have a Correlation ID at all (see the message-structure comparison in earlier chapters/notes for the full Mule 3 vs. Mule 4 differences).

The real value of Correlation ID shows up at scale. Imagine an application receiving 1,000 requests in a single day, and a user reports a problem with one specific transaction, giving you only a rough timestamp. Without a way to isolate that one transaction's log lines from the other 999, you'd have to scan everything by hand. The Correlation ID workflow solves this:

1. In Runtime Manager, open the deployed application → **Logs**.
2. Scroll to the reported timestamp and locate the relevant log entry.
3. Copy the **event/correlation ID** shown on that log line.
4. Paste it into the log search bar and search.

Because the *same* correlation ID is attached to every log line generated by that one request — from the payload first being received, through any transformation, through the call to the target, through the target's response — searching by it filters the log view down to just that one transaction's full journey, cleanly separated from every other request in the system. Without this, isolating a single transaction is effectively impossible at any real volume.

## Debugging in Anypoint Studio: Breakpoints and Debug Mode

Running an application normally only shows you the end result — the request came in, something happened, a response went out — it doesn't show what happened at each individual step along the way. **Debug mode** solves that by letting you pause execution and step through a flow component by component.

The steps:
1. Add a **breakpoint** on the relevant component in Anypoint Studio (typically on the **main flow**, not the exception-handling flow, unless you're specifically trying to debug the error path itself).
2. Right-click the project → **Run in Debug Mode**.
3. Trigger a request (from Postman, or by having the app pick up a file, etc.) — execution pauses at the breakpoint.
4. Inspect the **payload** and, separately, the **Variables panel**, to see exactly what data and variable values exist at that point in execution.
5. Use the "resume"/step control (an arrow icon, typically top-right in the debug view) to move execution forward to the next component, one step at a time.

Debug mode runs *exactly* the same logic as a normal run — the only difference is the ability to pause and inspect it. A practical workflow tip from the course: you can start building your Postman request while the app is still finishing deployment in debug mode, so you're ready to trigger the moment it's live, rather than waiting idle.

When live-troubleshooting a student's broken application, the general approach demonstrated was: open **Global Elements** and check the relevant configuration, review the **runtime logs**, and if that's not enough, drop a breakpoint on the main flow and step through in Debug mode to isolate exactly which component is failing.

## Common Errors Reference

This section collects every specific bug or error condition mentioned across the course, what caused it, and how it was diagnosed or fixed. Treat this as a lookup table for "I'm seeing X — what does that usually mean?"

### 404 — Resource Not Found
Occurs when a request targets a resource/path that doesn't exist in the API's RAML definition, or when an HTTP Request in your flow is pointed at a path the target application doesn't actually expose. One concrete case: a student's HTTP Request path was set to `/api/salesforce`, but the target application only exposed two valid resources — `order` and `shipment`. Any other resource name returns 404. Diagnosis approach: check the failing request's exact URL in the console/logs, compare it against the target's actual exposed resources, and correct the path.

### 415 — Unsupported Media Type
Returned when the media type sent (JSON, XML, etc.) doesn't match what the resource is configured to accept in its RAML. Example: sending XML to an `order` resource that's configured to accept only JSON. This is one of the core restriction mechanisms an API-led design gives you for free (see Chapter 12) — it's expected/correct behavior when the wrong format is sent, not a bug in the flow itself, unless the *wrong* media type restriction was configured by mistake.

### "Address Already in Use" (Port Conflict)
Encountered when trying to run or debug a flow. Cause: multiple Mule applications are already running on the same port (a port conflict) — commonly, a previous debug/run session that was never properly stopped. Fix: stop the currently running app(s) and restart Anypoint Studio, then re-run the flow in Debug mode. The same underlying issue can also show up during a debug session start as "port not available" — the fix is the same: find and stop the existing running instance (the red stop icon near the debugger) before starting a new session.

### Correlation ID Generator Bugs ("Too Many Nested Child Contexts")
During one live troubleshooting session, a student's application threw a "too many nested child contexts" error, traced back to a **Correlation ID Generator** configuration sitting in Global Elements. The fix was not to try to repair the existing broken configuration, but to **delete it entirely and create a fresh one** (Global Elements → Create → search "Configuration" → drop it in → Save). This is a good general debugging lesson beyond just this specific error: sometimes a broken global configuration component is faster to recreate from scratch than to diagnose in place.

### Case-Sensitivity Bugs
A recurring, easy-to-miss class of bug: string comparisons in a Choice router (or anywhere else) are exact-match and case-sensitive. One concrete example from the course: a "trailer" flow with a condition checking `color == "black"` wasn't routing correctly even though the data "looked" right. On checking the runtime logs closely, the actual incoming value was `"blac"` — missing the final "k" — a genuine data/typo mismatch rather than a flow bug. Since the condition required an exact match, the request correctly fell through to "condition not satisfied," exactly as designed. The lesson: when a condition that "should" be true isn't firing, check the *exact* string value in the logs character by character before assuming the flow logic itself is wrong. Similarly, demo credentials used across the course (e.g., a target system's Basic Auth password `target@123`, with a capital `T`) were repeatedly called out as case-sensitive — a lowercase `t` would fail authentication even though the rest of the string matches.

### Failed CloudHub Deploys (Missing Platform Properties)
Covered in depth in Chapter 13: an application secured by a platform-level policy (Basic Auth, Client ID Enforcement via API Manager) will fail its first deploy attempt if the `anypoint.platform.client_id` and `anypoint.platform.client_secret` properties haven't been added to the app's Runtime Manager settings yet. A small spelling mistake in either property key name will also silently prevent successful deployment. Fix: fetch the correct client ID/secret from **Access Management** for the target environment, add them under **Settings → Properties** with the exact correct key names, and redeploy.

### Missing/Incorrect Java-Format Error Responses
Not a MuleSoft "error code" in the strict sense, but a recurring practical bug across multiple sessions: when a target system's error response comes back in a Java/internal serialization format (referred to informally in the course as "grizzly" format) instead of JSON, returning it as-is to the source system produces an unreadable response. The fix, seen more than once, was always the same: add a **Transform Message** step in the relevant error-handling block to convert the response into JSON before returning it to the caller.

### 401 — Unauthorized
Returned whenever a security policy (Basic Authentication or Client ID Enforcement) is active and the caller either supplies no credentials or supplies the wrong ones. This is expected, correct behavior confirming the policy is genuinely active — see Chapter 13 for the full authentication picture.

### 405 — Method Not Allowed
Returned when the wrong HTTP method is used against a resource (e.g., GET where only POST is defined in the RAML). See Chapter 12.

## Common Mistakes / Things Students Got Wrong

- Logging the full payload at INFO level instead of DEBUG — this was explicitly flagged as something to avoid, since INFO logs are always on and a large payload wastes log storage unnecessarily.
- Assuming a condition ("color = black") was broken in the flow logic, when the real cause was a case-sensitive exact-match mismatch in the actual data (`"blac"` vs `"black"`) — the fix was checking the logs closely, not rewriting the condition.
- Not stopping a previous debug/run session before starting a new one, leading to "Address already in use" / "port not available" errors that look like flow bugs but are actually just leftover running processes.
- Trying to fix a broken Correlation ID Generator configuration in place rather than deleting and recreating it from Global Elements.
- Forgetting the environment-level `anypoint.platform.client_id`/`client_secret` properties (or misspelling the property keys) before the first deploy of a policy-secured app.

## Quick Recap

- **INFO** level: always printed, use for small trackable values only (never the full payload). **DEBUG** level: full payload, only printed when the category is explicitly enabled in Runtime Manager — enable it to troubleshoot, then disable it again afterward.
- Logging category convention: `com.<company>.<flow>.<target>` — adapt to your organization's standard.
- Standard flow logging pattern: debug-log the raw payload → set a tracking variable → info-log the tracking ID → debug-log the transformed payload → log before/after the outbound call.
- Log retention: Runtime Manager keeps a limited amount for up to 30 days; Anypoint Monitoring keeps up to 90 days with no fixed size limit.
- **Correlation ID** is a unique, auto-generated ID per request (Mule 4 only, no Mule 3 equivalent) — search runtime logs by it to isolate one transaction's full journey out of thousands.
- Debug in Anypoint Studio with breakpoints + Debug mode; inspect payload and the Variables panel; step forward one component at a time.
- Common errors reference: 404 (wrong resource path), 415 (wrong media type), "Address already in use" (leftover running instance, stop and retry), Correlation ID Generator "nested child contexts" bug (delete and recreate the global config), case-sensitivity mismatches (check exact string values in logs), failed CloudHub deploys from missing/misspelled platform client ID/secret properties, non-JSON ("grizzly") error responses needing a Transform Message fix, 401 (missing/wrong credentials), 405 (wrong HTTP method).
