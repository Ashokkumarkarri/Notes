# Chapter 19: File, FTP, SFTP Integration and Scheduling

> Taught on: 04-04-2022, 11-04-2022, 13-04-2022

## Why This Matters

Up to now, every integration has assumed the source or target system is API-based — something you can call over HTTP and poke at with Postman. Plenty of real systems don't work that way. Some systems exchange data by simply dropping a file in a folder and expecting someone else to come and pick it up. This is an older, but still extremely common, integration pattern — especially with legacy systems, banks, and large enterprise back-office systems. This chapter covers how Mule handles file-based integration, and the scheduling mechanics that make "go check that folder periodically" actually work.

## API-Based vs File-Based Integration

**API-based** systems support only the HTTP and HTTPS protocols. A flow that talks to one starts with an HTTP connector — a Listener if the API-based system is the source triggering data *into* Mule, or a Request if Mule is calling *out* to an API-based target. These flows are naturally testable with Postman: you configure a URL, method, and host, and fire requests at it.

**File-based** integration looks different. The requirement typically sounds like: "System X (say, Salesforce) drops a file at some location — read that file and send its contents to a target system." There's no URL to call and no HTTP method to pick. Mule handles this with a different family of connectors:

- **File** — reads files from a local or network filesystem location.
- **FTP** — File Transfer Protocol.
- **SFTP** — Secure Shell File Transfer Protocol. This is fully secure, and the course noted that roughly **99% of organizations** use SFTP for their file-transfer requirements rather than plain FTP.
- **FTPS** — a secure variant of FTP. Unlike File, FTP, and SFTP, the FTPS connector is **not available by default** in the Anypoint Studio palette — you need to import it from **Anypoint Exchange** before you can use it.

Because these are not HTTP-based, **Postman cannot be used to test them.** Instead, you test a file-based flow the low-tech way: place a sample file in the location the connector is watching, and let the flow pick it up naturally.

## A Small Detour: HTTP vs HTTPS Config Differences, Mule 3 vs Mule 4

While on the subject of protocols, it's worth knowing that switching an HTTP listener to HTTPS works differently depending on the Mule version:

- In **Mule 3**, switching a listener from HTTP to HTTPS requires configuring certificates — there are always two of them, a **Truststore** and a **Keystore**, each provided with a password, and configured under `src/main/resources`.
- In **Mule 4**, this certificate setup is not mandatory — you can change the listener's protocol from HTTP straight to HTTPS without configuring certificates separately.
- Either way, once the protocol is changed to HTTPS, the calling client must call the **HTTPS** URL — the old HTTP URL will simply stop working.

## File Connector Demo: Directory and Scheduling Config

The File connector was demoed with a simple "file transfer" project:

1. Drag the **File connector** into the flow.
2. Configure the **Directory** field — the folder path where the connector should look for files (in the demo, a `file` folder on the Desktop containing a file called `sample`).
3. Set a **Scheduling Strategy** on the connector — this determines how often it checks the folder for new files. Two options are available: **Fixed Frequency** or **Cron Expression** (covered in detail below).
4. Once a file is found, its content becomes available in the flow just like any other payload — it can be logged, transformed, and sent on to a target system, with a **Choice** router available if any conditional logic needs to be applied based on the payload.

In the demo, a fixed frequency of `1000` milliseconds (1 second) meant the connector checked the folder every second for a new file. In practice, this interval is set according to the actual business requirement — checking every second, minute, or hour, depending on how time-sensitive the integration is.

**Auto Delete** is an important setting on the File connector:

- **Default: `false`.** The file stays in the folder after being read. This is a trap — if left this way, the *same* file gets picked up and processed again and again on every polling cycle, since nothing removes it. Ten leftover files would each keep getting re-read indefinitely.
- **Set to `true`** to have the connector automatically delete the file once it's been read and delivered to the target system, which prevents this duplicate-processing problem.

## Repeatable vs Non-Repeatable Streaming, and Buffer Sizing

Here's a scenario that trips people up if they don't think about it: what happens when the file being read is *huge* — say, 10 GB?

By default, with no streaming strategy configured, Mule tries to read the **entire file into memory at once**. For a 10 GB file, this will very likely throw an **Out Of Memory exception**, because the memory available to the application is typically much smaller than the file itself.

The fix is the FTP connector's **Advanced** section, which exposes **Streaming Strategies**. There are two types:

- **Non-repeatable** — effectively the default, all-at-once-in-memory behavior described above.
- **Repeatable streaming** — reads the file in **chunks**, based on a configured **buffer size** (the demo used 512 MB as an example). Instead of loading the whole file at once, the connector connects to the file location repeatedly, pulling one buffer-sized chunk at a time until the entire file has been read, piece by piece.

There's also a **maximum buffer size** setting — around 1 GB (1024 MB) was mentioned as the practical ceiling in the demo — similar in spirit to a cap or limit on how large any single chunk is allowed to be.

With repeatable streaming properly configured, Mule can handle files of essentially any size — the example given was a 50 GB file — because it's never trying to hold the whole thing in memory at once. Walking through a concrete number: a 10 GB file read with a 1 GB buffer connects to the file location 10 separate times, sending 1 GB of data per attempt.

## Batch Job vs For Each — Splitting Large Payloads Further

Streaming solves the *reading* problem, but it doesn't automatically solve everything downstream. Suppose streaming successfully reads a 1 GB chunk from the source — but the **target system** can only accept 100 MB per request. Now that already-read chunk has to be split further before it can actually be sent. This is where **Batch Job** and **For Each** come in — two Mule palette components built for processing many records/items.

**For Each** processes records **sequentially**, one at a time (or in a configurable batch size), pushing each one to the target before moving on to the next.

- Default batch size is **1 record per iteration**.
- Example: a 5-record file with batch size 1 → the loop runs 5 times, one record per iteration, fully sequential.
- Example: 100 records with a batch size of 5 → the loop runs 20 times (100 ÷ 5).

**Batch Job**, by contrast, executes in **parallel** rather than sequentially.

- Default batch block size is also **1**, but it's customizable — e.g., set to 100 records per block.
- Batch Job is the recommended choice for very large record counts — the course used the example of **more than 1 crore (10+ million) records, or around 100 GB of data**. At that scale, a sequential For Each loop would require an enormous number of iterations and introduce serious delay before the data ever reaches the target.

**Rule of thumb:** for huge payloads where sequential processing (For Each) would simply be too slow, use **Batch Job** to process in parallel instead. And more generally: rather than reaching for more compute (scaling up vCore) to brute-force a bigger payload through, prefer streaming plus Batch Job/For Each to control how much data is actually sent per request.

**Q&A — "Can't I just configure repeatable streaming with a 100 MB buffer directly, and skip Batch Job/For Each?"**

Yes, that's a perfectly valid option for moderate payload sizes. But for very large payloads (the example given was 100 GB), a small buffer size means the connector has to reconnect to the file location a very large number of times (roughly 100 times, in that example) — and every one of those reconnects is a chance for a connectivity issue. If a connection fails partway through, the whole execution can fail. The recommended approach: test with a smaller payload first (e.g. 100 MB) to confirm the setup works reliably, then scale up carefully — and if repeated connectivity issues show up at scale, switch to Batch Job or For Each instead of continuing to shrink the buffer.

## Scheduling: Fixed Frequency vs Cron Expression

Both the File connector and the dedicated **Scheduler** connector (see below) support two scheduling strategies:

- **Fixed Frequency** — a simple repeating interval, expressed as a duration (e.g. every 1000 ms).
- **Cron Expression** — a more flexible, calendar-aware schedule (e.g. "every Monday at 11:00 AM," or "every weekday after 12 PM").

Writing cron expressions by hand is fiddly, so the course used an online tool called **CronMaker** to generate them:

1. Go to CronMaker.
2. Pick the schedule you want — e.g., day = Monday, time = 11:00 AM.
3. Click **Generate** to produce the cron expression string.
4. Copy that generated expression into the connector's scheduling configuration.

**Timezone matters** when setting a cron expression — IST vs UTC will give you very different actual trigger times, so this needs to be set deliberately according to the business requirement. With a weekly "every Monday at 11 AM" cron schedule, the flow runs exactly once per week, at that time, until the following Monday.

A second cron task later in the course asked for a schedule that runs **only on weekdays** (Monday through Friday, excluding weekends), triggering **once daily after 12 PM** — the same CronMaker tool was used to build and verify that expression.

Scheduling isn't limited to file-watching, either — it applies to API-based systems too. An HTTP Requester call can be scheduled the same way, using GET (to periodically pull/retrieve data from an external system) or POST (to periodically push data to a system).

## Mule 4's Scheduler vs Mule 3's Poll

The dedicated scheduling connector has different names across Mule versions, but works the same way underneath:

- **Mule 4**: called the **Scheduler** connector.
- **Mule 3**: the equivalent was called **Poll** (a "poll job").

Only the naming changed between the two versions — the underlying options (Cron Expression and Fixed Frequency) are the same in both. The Scheduler was noted in the course as the fourth connector covered overall, after HTTP, Anypoint MQ, and the Cloud (CloudHub) Notification connector.

## Quick Recap

- API-based integrations use HTTP/HTTPS connectors and are testable with Postman; file-based integrations use File, FTP, SFTP, or FTPS connectors and are tested by dropping a sample file in the watched location instead.
- SFTP is the dominant choice in real organizations (~99%) for secure file transfer; FTPS must be imported from Exchange since it isn't in the palette by default.
- Mule 3 requires Truststore/Keystore certificates to switch HTTP → HTTPS; Mule 4 lets you switch the protocol directly without that setup.
- The File connector's Directory field sets the watched folder; Auto Delete (default `false`) should usually be set to `true` to avoid reprocessing the same file repeatedly.
- Reading a huge file with no streaming strategy risks an Out Of Memory error; **repeatable streaming** with a configured buffer size (max ~1 GB) reads the file in chunks instead, supporting files of virtually any size.
- Once a chunk is read, if the target's per-request capacity is smaller than the chunk, use **For Each** (sequential, good for smaller volumes) or **Batch Job** (parallel, recommended for very large record counts like 10M+ rows / ~100GB) to split it further before sending.
- Scheduling uses either **Fixed Frequency** (simple repeating interval) or a **Cron Expression** (built easily with the CronMaker tool, with timezone set deliberately); this applies to both file-watching and scheduled API calls.
- Mule 4's **Scheduler** connector is the same concept as Mule 3's **Poll** component — only the name changed.
