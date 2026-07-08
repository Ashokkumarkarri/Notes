# Chapter 03: Anypoint Platform Overview

> Taught on: 21-02-2022, 22-02-2022, 23-02-2022, 24-02-2022, 26-02-2022, 04-03-2022, 10-03-2022, 11-03-2022, 18-03-2022, 21-04-2022, 22-04-2022

## Why This Matters

Anypoint Studio is where you *write* Mule applications, but writing them locally is only half the job — a real integration needs to run continuously in the cloud, be manageable by a team, be secured, and be designed collaboratively before a single line of flow logic exists. **Anypoint Platform** is the umbrella name for the whole suite of cloud tools that make that possible. It has several distinct pieces — Design Center, Exchange, API Manager, Runtime Manager, Access Management — and one of the most common sources of early confusion in this course is not knowing which piece does which job. This chapter is your map of that territory.

## The Big Picture: Studio Builds, Platform Runs

Here's the mental model to hold onto as you read the rest of this chapter: **Anypoint Studio is where Mule code and flows are developed. Anypoint Platform (in the cloud, hosted on AWS) is where that developed code is deployed, secured, and managed once it needs to run for real.** Everything below is really just a breakdown of the different rooms inside that "Anypoint Platform" building.

## Design Center — Where APIs Are Designed

**Design Center** is the piece of Anypoint Platform where REST APIs are designed using **RAML** (RESTful API Modeling Language), *before* any actual flow logic is written in Anypoint Studio.

To create a new API in Design Center:
1. Click **New API** and give it a name.
2. The naming convention that was emphasized as a rule (not just a suggestion) in real projects is: **name the API after the source system it's integrating with.** If the source is Salesforce, call it "Salesforce API." If the source is a database, call it "Database API." If the source is a Java-based system, "Java space API." This isn't cosmetic — it makes the platform navigable once dozens of APIs exist side by side.

Once named, Design Center auto-populates a **RAML version and title** for you. A few structural pieces of a RAML file worth knowing by name:

- **Version** (e.g., `v1`, `v2`) — chosen for convenience, but standard practice is to always start a new API at version 1.
- **Base URI** — includes the **protocols** the API will accept from the source system (HTTP, HTTPS) and the **media types** it accepts, written using MIME-type syntax: JSON is `application/json`, XML is `application/xml`, plain text is `text/plain`, Java is `application/java`.
- **Resources** — these represent the API's actual endpoints/paths (e.g., `/order`, `/shipment`). If a caller triggers a path that doesn't exist in the RAML, they get a `404 Not Found`.
- Each resource declares which **HTTP method** it accepts (wrong method → `405 Method Not Allowed`) and which **media type**/body it accepts (wrong media type → `415 Unsupported Media Type`).

Once a RAML is designed, it needs to be **published to Exchange** before it can be imported anywhere else on the platform — publishing is done from inside Design Center, via the API's "Publish to Exchange" option.

There is also an alternative, "manual" way to bring a RAML into Studio without going through Design Center's normal sync: create a `.raml` file directly under `src/main/resources` in your Studio project, paste the RAML content into it, then right-click → **Mule → Generate flows from local REST API**. The trade-off is important to remember: this manual approach is **not linked back to Design Center** — if you later edit the API in Design Center, those changes will not automatically sync into a project built this way.

## Anypoint Exchange — The Shared Catalog

**Exchange** functions as MuleSoft's shared catalog and marketplace, and it does two very different jobs in this course:

**1. It hosts published APIs.** Once you publish a RAML from Design Center, it lives in Exchange, where it can be imported into API Manager (see below) or shared with other developers on your team.

**2. It's where connectors that aren't built into the Studio palette by default get imported from.** For example, the **CloudHub Connector** used for failure-notification emails (a later topic) isn't in the palette out of the box — you search for it in Exchange (search "cloudhub" → Add → Finish) and it gets added to your project. The course noted that MuleSoft has roughly **150–155 connectors** available in total, of which this course focuses on a practical subset of about 7–8 commonly used ones — Exchange is how you'd reach for anything outside that core set.

**3. It's where Client ID / Client Secret credentials get generated for the Client ID Enforcement security policy.** This is a subtlety worth being precise about: applying the Client ID Enforcement policy in API Manager only turns the *policy* on — it does not generate the actual credentials. To get a Client ID and Client Secret, you go into Exchange, open the published API, use the three-dot menu → **Request Access**, and either select an existing client application or create a new one via **Create New Client App**. Opening that application then reveals its auto-generated Client ID and Client Secret, which get copied into Postman (or whatever the real source system is) to authenticate. This is different from Basic Authentication, where the username and password are typed directly into the policy configuration in API Manager — no separate Exchange step needed there.

## API Manager — Where APIs Are Secured

Once a RAML has been published to Exchange, it can be imported into **API Manager**: **Manage API → Manage API from Exchange**, searched by name, then **Import from Exchange → Save**. This creates a managed API instance that API Manager tracks the URL, allowed methods, and media types for.

The main thing API Manager adds on top of a plain RAML definition is **security policies**, under **Policies → Add Policy**. Two policy types came up repeatedly through the course:

- **Basic Authentication** — secures the API with a username and password, entered directly in the policy configuration screen in API Manager. Calling the API without valid credentials returns `401 Unauthorized`.
- **Client ID Enforcement** — secures the API using a Client ID and Client Secret, generated in Exchange (as described above) rather than typed manually. It supports two ways of passing credentials: a **Custom Expression** (client ID/secret sent as separate headers — the default expression is `attributes.headers.clientId`), or via the **HTTP Basic Authentication header** (client ID as username, client secret as password).

**Only one policy can be applied to a given API at a time** — never both Basic Auth and Client ID Enforcement together. Which one an organization uses is a matter of its own security standards, but Client ID Enforcement was described as the more secure of the two: Basic Auth credentials are simple, self-created, and can go unchanged for a long time, whereas Client ID Enforcement credentials are auto-generated, much longer, and — once created — cannot be changed, only regenerated fresh.

A subtlety that trips people up: **a policy can be applied either before or after the application is deployed** — order doesn't matter, and policies can be added to an app that's already live and running.

### Linking a Deployed App to API Manager: Auto Discovery

Having a policy configured in API Manager does nothing on its own until the actual running Mule application is *linked* to that managed API definition. That link is a component called **API Auto Discovery**, configured back in Anypoint Studio's Global Elements:

1. Copy the **API ID** from API Manager.
2. In Studio, create a new **Auto Discovery** global element, paste in the API ID, and select the main flow it should apply to.

Two rules about Auto Discovery are worth memorizing because they explain a lot of confusing behavior you'll otherwise hit:

- **Policies only take effect once the app is deployed via Runtime Manager.** They do **not** apply when you're just running/testing the app locally inside Anypoint Studio. If you want to test your flow logic locally without a policy getting in the way, temporarily remove the Auto Discovery component, do your local testing, then add it back and redeploy.
- Until the required environment credentials are configured (next section), the API sits in an **"Unregistered"** state in API Manager. Once configured correctly, it flips to **"Active"** — only then is the policy genuinely being enforced.

## Access Management — Two Different Kinds of "Client ID and Secret"

This is one of the most confusable pairs of concepts in the whole course, so it deserves its own careful section: there are **two completely different** "Client ID / Client Secret" pairs you'll work with, and mixing them up is a very common source of deployment failures.

**1. The API-level (Client ID Enforcement) policy credentials.** These are the ones generated in Exchange (see above) and passed by the *source system* (Postman, or the real caller) on every single request, so the API knows who's calling it.

**2. The Access Management-level (platform) credentials.** These are used to link a Runtime Manager deployment *itself* back to API Manager, via Auto Discovery, so the API can show as "Active." They're retrieved from **Access Management → Environments → select the environment → copy the Client ID and Secret**, and they get declared as two specific runtime properties on the deployed application:
   - `anypoint.platform.client_id`
   - `anypoint.platform.client_secret`

If an application has a platform policy applied via API Manager but these two properties aren't configured (or are misspelled), the application will fail to deploy or start correctly. This exact failure was hit live in the 04-03-2022 session — the first deploy attempt failed for precisely this reason, and adding the two properties (retrieved from Access Management) fixed it. A small spelling mistake in either property key is enough to break the deployment.

One more distinguishing fact: Client IDs at the environment (Access Management) level always differ between environments (e.g., Sandbox has a different one than Production), whereas the API-level policy credentials from Exchange are tied to the API/client application itself, not the environment.

## Environments and Business Groups

Anypoint Platform organizes deployments into **environments**. By default, a trial account provides environment types like **Design**, **Sandbox**, and **Production**. Despite the default naming, there's no rule that an environment must be called "Sandbox" — real projects commonly use names reflecting their actual purpose, such as UAT, Non-prod, `ecommerce-dev`, or `ecommerce-prod`. On a trial account specifically, some default environments only offer a delete option (no rename), which can feel restrictive compared to a real paid organization account.

A **Business Group** is a container that can hold multiple environments. In real projects, each business group typically maps to one organizational branch or unit, and a single organization can have several business groups, each with its own set of environments underneath it.

There is no built-in "move application between environments" button on a trial account. To migrate an app manually: open it in Runtime Manager → Settings → download its JAR file locally, repeat for every app that needs to move, delete the old environment once everything is downloaded, then upload the JARs into the new environment.

## Mule Runtime Versions — Don't Confuse the IDE with the Engine

A specific and easy-to-make mixup: **the Anypoint Studio (IDE) version is not the same thing as the Mule runtime version.** Anypoint Studio is the application you develop in; the Mule runtime is the actual engine version your deployed application runs on. Over the course, the Mule runtime version in active use progressed from **4.4.0 Enterprise Edition** to **4.4.4**, while the Anypoint Studio IDE version referenced later in the course was **7.11**. When troubleshooting version-specific behavior, always be clear about which of the two you mean.

## Cloud Computing, vCore, and Workers: How Scaling Works

Anypoint Platform (including Runtime Manager) runs in the cloud, hosted on **AWS**. Deployed applications don't run on unlimited, free compute — they consume a configurable amount of resources, and there are two independent dials for controlling that:

- **Vertical scaling (vCore)** — increases the compute/memory given to a *single* worker. Use this when a single request carries a very large payload. The default allocation is **0.1 vCore**, and it can be scaled up toward **32 GB**. Vertical scaling gets expensive fast — large allocations (16 GB/32 GB) were described as potentially running into crores of rupees per year, since you're billed directly for the platform resources consumed.
- **Horizontal scaling (Workers)** — increases the *number* of workers, where each worker is effectively an independent server instance (workers never share a physical location — one might run in Ireland, another in the UK or Australia). Use this when the *volume* of incoming requests is high, regardless of any individual payload's size. Workers are numbered starting from 0, with 1 worker by default; the platform's default worker ceiling is typically 8 (higher limits can be requested). Adding workers spreads incoming load across multiple server instances.

Rule of thumb: **vCore for one huge request, Workers for many concurrent requests.** There's no fixed universal rule for exactly how much of either to provision — that decision is typically made by whoever owns the client relationship (team lead, project manager), balanced against the client's budget, since over-provisioning either dial directly increases cost.

For contrast, **on-premise deployments have no equivalent scaling capability at all** — no vCore, no worker dial. An on-premise Mule installation relies purely on the "worker" concept running on whatever physical/virtual machine it's installed on, and its capacity is entirely memory-bound.

## The POM File and Property Files

Every Mule project managed as a proper software project has a **`pom.xml`** file (the POM file), which stores and manages the project's dependencies — this is the same Maven-style convention used broadly across Java-based build tooling, and Mule projects follow it too.

**Property files** are used to hold configuration values — including sensitive ones — outside of your flow logic, so the same flow logic can point at different environments (test vs. production) just by swapping which property file/config it reads from. Sensitive values (passwords, keys) shouldn't sit in a property file as plain text; they get **encrypted**, and the platform then needs a matching configuration in Global Elements to know how to decrypt them at runtime:

- In **Mule 4**, this Global Elements configuration is called **Secure Property Config**.
- In **Mule 3**, the equivalent was called **Secure Property Placeholder**.

Only the naming changed between versions — the underlying idea (encrypt the sensitive value, then reference it through a secure-properties config) is identical. A project seen in the course used the **AES** algorithm with a 16-digit secure key for this encryption. The syntax to actually *read* these values back differs by context:

- Reading an encrypted property at the connector/config level: `${secure::keyName}`
- Reading an encrypted property inside a Transform Message (DataWeave): `p('secure::keyName')`
- Reading a normal, non-encrypted property: `${keyName}`

If a property value is encrypted but its matching Secure Property Config isn't set up in Global Elements, the application will fail to deploy — this is a very common early misconfiguration to watch for.

## CloudHub Platform Components, at a Glance

Pulling it all together, the pieces of CloudHub/Anypoint Platform referenced across the course are: **Exchange** (shared catalog for APIs and connectors, and where policy credentials are generated), **API Manager** (where policies are applied to secure an API), **Runtime Manager** (where applications are actually deployed, started, stopped, and monitored — covered in depth in Chapter 05), **Object Store** (a platform-level cache for storing and retrieving data, covered briefly as a supporting concept), **Scheduler** (for time/cron-based triggers), and **Anypoint Monitoring**, a separate logging surface from Runtime Manager's own logs that retains up to **90 days** of backup with no fixed storage-size limit — useful for tracing older transactions that Runtime Manager's own shorter-retention logs (roughly 100–500 MB capacity, mentioned inconsistently across sessions) may have already rotated out.

## Common Mistakes / Things Students Got Wrong

- **Confusing the two "Client ID/Secret" pairs.** This came up as a genuine deployment failure in the 04-03-2022 session: the app wouldn't start after a Basic Auth policy was applied, because the *Access Management*-level `anypoint.platform.client_id` / `anypoint.platform.client_secret` properties hadn't been added — students initially expected the API-level policy credentials to be enough.
- **A small spelling mistake in the property keys** (`anypoint.platform.client_id` / `anypoint.platform.client_secret`) silently prevents correct deployment — this was explicitly flagged as an easy, easy-to-miss mistake.
- **Editing an API in Design Center and expecting the already-deployed application to update automatically.** It doesn't — policies and RAML edits made in Design Center are for design/documentation purposes; they don't retroactively touch a running app until you redeploy.
- **A "too many nested child contexts" error** traced back to a broken Correlation ID Generator configuration in Global Elements (18-03-2022) — the fix was to delete the existing configuration entirely and create a fresh one, rather than trying to repair the existing one in place.

## Quick Recap

- **Anypoint Studio** builds Mule apps locally; **Anypoint Platform** (cloud, on AWS) is where they get deployed, secured, and managed.
- **Design Center** — design REST APIs using RAML; name APIs after the source system they integrate with.
- **Exchange** — the shared catalog: hosts published APIs, extra connectors not in the default palette (~150-155 connectors exist total), and is where Client ID Enforcement credentials get generated (via Request Access → Create New Client App).
- **API Manager** — where security **policies** (Basic Authentication or Client ID Enforcement — only one at a time) get applied to a published API.
- **Auto Discovery** (configured in Studio Global Elements) links a deployed app to its managed API definition in API Manager — policies only enforce once the app is deployed, never during local testing.
- **Access Management** provides a *second*, separate Client ID/Secret pair (`anypoint.platform.client_id` / `anypoint.platform.client_secret`) needed just to get the app's status to "Active" in API Manager — don't confuse this with the API-policy credentials from Exchange.
- **Environments** (Sandbox, Production, etc.) live inside **Business Groups**; there's no fixed naming rule beyond convention.
- Mule **runtime version** (e.g., 4.4.4) and the **Anypoint Studio IDE version** (e.g., 7.11) are two different numbers — don't conflate them.
- **Vertical scaling (vCore)** handles single huge payloads; **horizontal scaling (Workers)** handles high request volume; on-premise deployments have neither.
- The **POM file** manages project dependencies; **property files** hold config (encrypted via Secure Property Config in Mule 4 / Secure Property Placeholder in Mule 3) so the same flow can target different environments.
