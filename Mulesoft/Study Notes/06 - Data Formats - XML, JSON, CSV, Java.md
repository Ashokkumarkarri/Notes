# Chapter 06: Data Formats — XML, JSON, CSV, Java

> Taught on: 22-02-2022, 23-02-2022, 07-03-2022, 09-03-2022, 14-03-2022, 16-03-2022

## Why This Matters

Every system you integrate with speaks its own "language" when it comes to data. Some systems only understand XML, others only JSON, others send flat CSV files, and some older enterprise systems talk in Java object format. As a MuleSoft developer, you rarely get to choose the format — the source system sends what it sends, and the target system demands what it demands. Your job is to sit in the middle and translate.

This is really the heart of what an integration platform does. The same real-world piece of data — say, a phone with a Brand, Model, Price, Pixel count, and Color — can be represented in completely different shapes depending on the format. Mule's real power is that it can read data in one structure, work with it, and then write it back out in a totally different structure, based on whatever the target system expects. Understanding what each format actually looks like, and how Mule represents it internally, is the foundation everything else in this course builds on.

## XML Structure

XML (eXtensible Markup Language) is built around **tags**. Every XML document has one **starting tag** and a matching **ending (closing) tag** that wraps everything else — this outermost wrapping tag is called the **root**.

For example, if you're sending phone data, the whole document would be wrapped like this:

```xml
<Phone>
  <Brand>OnePlus</Brand>
  <Model>Nord</Model>
  <Price>25000</Price>
  <Pixel>64MP</Pixel>
  <Color>Black</Color>
</Phone>
```

Here, `<Phone>...</Phone>` is the root tag, and `Brand`, `Model`, `Price`, `Pixel`, and `Color` are the individual **fields**, each with its own opening and closing tag nested inside the root. The root tag name at the start must exactly match the closing tag name at the end of the document — that's simply how XML is structured, and it's non-negotiable.

If a source system's backend is built on XML, you can count on it always sending data shaped this way: one root tag wrapping a set of child fields. Recognizing the root tag is often the first thing you need to identify when you start mapping a new XML payload.

### A quirk worth knowing: XML and namespaces

You might expect that once you have an XML payload in Mule, you can just reach into it directly — for example, referencing `payload.phone.brand` inside a Set Variable component — the same way you'd reach into a JSON object. In practice, this often breaks with a **namespace issue**: XML documents carry namespace metadata that gets in the way of simple, direct field-path access.

The practical workaround taught in this course: inside a **Transform Message** component, set the **output** to `application/java` first. This converts the XML into a Java object representation, which you can then safely navigate using ordinary field paths (e.g., `payload.phone.brand`) without hitting the namespace problem. This "detour through Java" is a recurring pattern any time you need to read individual fields out of an XML payload for use in a condition or a variable.

## JSON Structure

JSON (JavaScript Object Notation) looks and behaves very differently from XML. Instead of tags, a JSON document always starts and ends with curly braces `{ }`. Data is represented as **key-value pairs** — the key is the field's name, and the value is the field's actual data, separated by a colon.

The same phone data in JSON would look like this:

```json
{
  "Brand": "OnePlus",
  "Model": "Nord",
  "Price": 25000,
  "Pixel": "64MP",
  "Color": "Black"
}
```

The most important structural difference from XML: **JSON does not require a root element**. In XML you must have one enclosing root tag; in JSON, the fields simply live directly inside the outer braces — there's no mandatory wrapping element with its own name. This is a common point of confusion for people coming from an XML background, so it's worth internalizing early: no root tag is needed in JSON, ever.

## CSV, Text, and Java as Data Formats

Beyond XML and JSON, Mule flows can send and receive several other formats:

- **CSV** — comma-separated flat data, commonly used for bulk/file-based data exchange.
- **Text** (`text/plain`) — plain, unstructured text.
- **Java** (`application/java`) — data represented as a Java object. You'll mostly encounter this not as something a source system sends you directly, but as an internal "working format" you deliberately convert into when you need to safely read fields out of an XML payload (see the namespace note above).
- **File** — not really a data format on its own so much as a way data arrives (via File, FTP, SFTP connectors), covered in a later chapter on connectors.

When you're designing an API in RAML, media types are declared using MIME-type syntax rather than plain words:

| Format | MIME type |
|---|---|
| JSON | `application/json` |
| XML | `application/xml` |
| Text | `text/plain` |
| Java | `application/java` |

These are the exact strings you configure in a RAML file's base URI / resource definitions to declare which formats a resource will accept.

## Common Mistakes / Things Students Got Wrong

A recurring bug pattern in this course was **forgetting to convert data back out of Java format before returning it to the caller**. Because reading XML often requires converting to `application/java` internally (to dodge the namespace issue), it's easy to leave that Java-formatted payload as-is and accidentally send it onward. On 14-03-2022 and again on 16-03-2022, students hit exactly this problem: a response came back in "Java" (referred to informally in the session as "grizzly" format) instead of proper JSON, because a Transform Message step further down the flow hadn't been set to convert the payload back to `application/json` before it was returned to the source system or forwarded to the target. The fix each time was the same: add or fix a Transform Message so the final output is explicitly the format the receiving side actually expects — don't assume the format carries over correctly on its own.

## Quick Recap

- **XML**: always has one root tag wrapping child field tags; opening and closing root tags must match exactly.
- **JSON**: starts/ends with `{ }`; data as key-value pairs; **no root element required** (the key structural difference from XML).
- Reading individual fields out of XML directly often breaks due to **namespace issues** — the standard workaround is converting to `application/java` in Transform Message first, then navigating the resulting Java object.
- Other formats you'll meet: CSV, plain text, and Java (`application/java`) — each with its own RAML MIME type (`application/json`, `application/xml`, `text/plain`, `application/java`).
- Always double-check your **final output format** before a response leaves the flow — a stray Java-formatted payload reaching a JSON-expecting caller is a classic, easy-to-miss bug.
