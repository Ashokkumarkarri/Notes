# MuleSoft Study Guide — Course Index

This is a **human-readable, topic-organized study guide**, built from the daily lecture logs in [[../sai mulesoft]]. Where those daily notes are terse, date-ordered summaries (good for quick lookup, e.g. by Hermes), the chapters here are full tutorial-style explanations meant to actually be *read and studied* — one topic per chapter, with everything scattered across multiple lecture days pulled together into a single coherent explanation.

Each chapter opens with a `> Taught on:` line pointing back to the original lecture date(s) in `sai mulesoft/`, so you can always trace a concept back to the exact class it came from.

## How to use this guide

Read in order the first time through — later chapters assume you already know the earlier ones (e.g. Routers assumes you understand Variables; API Security assumes you understand RAML). After that, use it as a reference: jump straight to the chapter you need.

## Chapters

### Part 1 — Foundations
1. [01 - Introduction to MuleSoft and Integration Concepts](01%20-%20Introduction%20to%20MuleSoft%20and%20Integration%20Concepts.md) — why integration platforms exist, middleware/ESB, APIs, REST vs SOAP
2. [02 - Environment Setup](02%20-%20Environment%20Setup.md) — Anypoint Studio, Postman, Notepad++, system requirements
3. [03 - Anypoint Platform Overview](03%20-%20Anypoint%20Platform%20Overview.md) — Studio, Runtime Manager, Design Center, Exchange, Access Management, how they fit together
4. [04 - HTTP Basics for Mule Developers](04%20-%20HTTP%20Basics%20for%20Mule%20Developers.md) — Listener vs Request, URLs, status codes, Postman
5. [05 - Building and Deploying Your First Mule Application](05%20-%20Building%20and%20Deploying%20Your%20First%20Mule%20Application.md) — source→Mule→target, local vs CloudHub, naming, common first-deploy errors

### Part 2 — Data, Logic, and Flow Control
6. [06 - Data Formats - XML, JSON, CSV, Java](06%20-%20Data%20Formats%20-%20XML%2C%20JSON%2C%20CSV%2C%20Java.md)
7. [07 - DataWeave and Transform Message](07%20-%20DataWeave%20and%20Transform%20Message.md)
8. [08 - Variables and the Mule Message Model](08%20-%20Variables%20and%20the%20Mule%20Message%20Model.md) — Mule 3 vs Mule 4 message structure
9. [09 - Routers - Choice, Scatter-Gather, Round Robin, APIKit](09%20-%20Routers%20-%20Choice%2C%20Scatter-Gather%2C%20Round%20Robin%2C%20APIKit.md)
10. [10 - Securing Sensitive Data - Property Encryption](10%20-%20Securing%20Sensitive%20Data%20-%20Property%20Encryption.md)
11. [11 - Scaling Mule Applications](11%20-%20Scaling%20Mule%20Applications.md) — vertical vs horizontal, on-prem vs cloud

### Part 3 — API Design and Reliability
12. [12 - API-Led Design with RAML and Design Center](12%20-%20API-Led%20Design%20with%20RAML%20and%20Design%20Center.md)
13. [13 - Securing APIs - Policies and Authentication](13%20-%20Securing%20APIs%20-%20Policies%20and%20Authentication.md) — Basic Auth, Client ID Enforcement, Auto Discovery
14. [14 - Error and Exception Handling in Mule 4](14%20-%20Error%20and%20Exception%20Handling%20in%20Mule%204.md)
15. [15 - Logging, Debugging, and Common Errors](15%20-%20Logging%2C%20Debugging%2C%20and%20Common%20Errors.md)
16. [16 - Circuit Breaker Pattern and Reliability](16%20-%20Circuit%20Breaker%20Pattern%20and%20Reliability.md)

### Part 4 — Integration Patterns and DevOps
17. [17 - Anypoint MQ - Message Queues](17%20-%20Anypoint%20MQ%20-%20Message%20Queues.md)
18. [18 - Object Store and Caching](18%20-%20Object%20Store%20and%20Caching.md)
19. [19 - File, FTP, SFTP Integration and Scheduling](19%20-%20File%2C%20FTP%2C%20SFTP%20Integration%20and%20Scheduling.md)
20. [20 - Testing with MUnit](20%20-%20Testing%20with%20MUnit.md)
21. [21 - Git, Bitbucket, and CI-CD Pipelines](21%20-%20Git%2C%20Bitbucket%2C%20and%20CI-CD%20Pipelines.md)
22. [22 - Real-World Project Walkthrough and Course Recap](22%20-%20Real-World%20Project%20Walkthrough%20and%20Course%20Recap.md) — JDA-to-Pink YMS project + full course recap checklist

## Relationship to the other notes in this vault

- **`sai mulesoft/February|March|April/`** — daily lecture logs, one file per class date, terse and fact-dense. Use these when you (or Hermes) need to check exactly what was said on a specific day.
- **`Study Notes/` (this folder)** — topic-organized, detailed, explained from scratch. Use this to actually *learn* or *review* a concept.
