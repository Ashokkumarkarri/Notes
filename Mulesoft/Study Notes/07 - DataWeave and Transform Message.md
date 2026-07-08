# Chapter 07: DataWeave and Transform Message

> Taught on: 22-02-2022, 03-03-2022, 04-03-2022, 09-03-2022, 16-03-2022, 19-03-2022, 23-03-2022, 08-04-2022, 11-04-2022, 22-04-2022

## Why This Matters

If Chapter 6 was about *what* the different data formats look like, this chapter is about the tool that actually does the work of moving between them: the **Transform Message** component, powered by MuleSoft's own expression language, **DataWeave**. Almost every real integration flow you'll ever build touches Transform Message somewhere — converting XML to JSON, dropping fields the target doesn't need, combining two fields into one, de-duplicating records, or just quietly creating a handful of variables to carry values through the rest of the flow. If you get comfortable with Transform Message and a core set of DataWeave functions, most of the "logic" work in Mule becomes straightforward.

## The Transform Message Component's Role

Transform Message lives in the Mule palette and is the component you drag onto a flow whenever data needs to change shape. The most basic use case, demonstrated early in the course: a source system sends XML, but the target system only accepts JSON, so a Transform Message sits between the Listener and the HTTP Request and converts the incoming XML payload into JSON before it moves on. The reverse direction works exactly the same way — JSON in, XML out — and this pattern (source format ≠ target format → insert a Transform Message) repeats constantly throughout the course's demos (e.g., the Salesforce→Order and Trailer→Shipment flows built on 03-03 and 04-03, and the JDA→Pink integration on 17-03, where JDA sends XML but Pink only accepts JSON).

### Dropping or Reducing Fields

Transform Message isn't only about changing the *format* — it's just as often used to change the *shape*, specifically by dropping fields the target doesn't need. If a source system sends five fields (say, Brand, Model, Price, Pixel, and Color) but the target system only cares about three (Brand, Model, Price), Transform Message can simply not include the unwanted fields in its output mapping. You don't need the real source or target systems connected to demo this — Postman can stand in as the source, sending a full payload, while you watch Transform Message emit only the reduced set of fields.

This "reduce to only what's needed" pattern showed up again later as a full task: a source XML payload with roughly a dozen fields needed to be reduced down to just a single field the target actually required (a Warehouse ID), built out of a nested path in the source data (18-03-2022, reviewed 23-03-2022). The lesson generalized from that exercise is important: **the source system can send any shape of data it wants — what actually gets extracted, renamed, or transformed is dictated entirely by the target's requirement.** Keys can even be renamed to match what the target expects; this mapping flexibility is one of Mule's core strengths.

## Defining Sample Input/Output Metadata

Rather than mapping fields blind against an undefined structure, Transform Message lets you load **concrete sample metadata** for both the input and the output side, so Anypoint Studio can generate the DataWeave mapping against a known schema (19-03-2022).

To do this:
1. In the Transform Message component, click **Define Metadata**.
2. Click **Add**, give the sample a name.
3. Load a sample file representing the structure — for input, this might be a sample XML file saved locally; for output, a sample JSON file showing what the target expects.
4. Repeat on the output side the same way (click **Add**, name it, e.g. "output," load the sample file).

Once both samples are loaded, Studio has a concrete shape to map from and to, which makes building the transformation logic far more reliable than guessing at field names against an undefined structure.

## DataWeave Functions and Operators

Across the course, a set of specific DataWeave functions and operators came up repeatedly in hands-on tasks. These are worth knowing cold, since the instructor flagged DataWeave functions as a commonly asked interview topic (22-04-2022), recommending fresher candidates know **at least 10 functions thoroughly**. Functions explicitly named across the sessions: `map`, `sizeOf`, `splitBy`, `replace`, `pluck`, `flatten`, and various date functions.

### String Concatenation (`++`)

Used to join two values into one. Demonstrated scenario (08-04-2022): a JSON array arrives with separate `firstName` and `lastName` fields, but the target needs a single combined `name` field.

```
name: $.firstName ++ " " ++ $.lastName
```

A useful detail: `++` isn't limited to joining two strings — it works across types. If you concatenate a number and a string, DataWeave automatically converts the number and joins it into the resulting string output.

### `splitBy`

Used to break a string into pieces around a delimiter. Scenario: a source system sends a single `name` field that actually holds a combined value (e.g., a location string), but the target only needs the first word out of it. You split on the space character and then take the first element of the resulting array:

```
$.name splitBy " "
```

...then index into the first element of that array to get just the piece you need (08-04-2022).

### Removing Duplicates

Discussed in the context of employee records where a given "employee" value is supposed to be unique within an organization — if duplicate records slip through, DataWeave has a function to strip duplicate values out of an array/object before it's sent onward (11-04-2022).

### Range Function

Used to generate a sequence of numbers — the example given was generating a range from 0 to 3, useful for cases like auto-generating a sequence of employee IDs (11-04-2022). Worth noting: when a student later asked whether the Range function was the right tool for extracting just the *first digit* of a salary value (as part of the Employee Data task, 23-03-2022), the instructor deliberately did **not** confirm Range as the answer — students were pushed to research and figure out the right DataWeave approach themselves rather than being handed the exact function, as a problem-solving exercise.

### Iterating Arrays with `map`

The Employee Data task (23-03-2022) is a good example of combining several of these ideas at once. Given a JSON array of employee records, for every record where `designation == "Consultant"`, the task required:
- Employee Name → output its **character count** (a `sizeOf`-style function), not the name itself.
- Employee Salary → output only the **first digit** of the value.
- Designation → passed through unchanged.
- Location → output only the **first three characters** as a location code (e.g., Bangalore → BAN).

The suggested approach was to iterate the array using **`map`** inside Transform Message, applying this field-by-field logic to each object as it's processed.

### Reading XML in Java Output Format

As covered in Chapter 6, reading fields directly out of an XML payload with a simple path can trip over an XML namespace issue. The fix lives inside Transform Message: set the **output** to `application/java`, which converts the XML into a navigable Java object, and *then* read fields off that Java object safely (e.g., `payload.phone.brand`) for use elsewhere in the flow, such as inside a Choice router condition (09-03-2022).

## Creating Multiple Variables via One Transform Message

Mule 4 gives you two ways to create a variable: the **Set Variable** component, or a **Transform Message** whose DataWeave output builds one or more variables directly. The rule of thumb taught on 16-03-2022 and reinforced in the 23-03-2022 recap is simple:

- **One variable needed** → use **Set Variable**. It's lighter and executes faster.
- **Multiple variables needed** (the course mentioned needing 4–5 at once, and noted a single Transform Message can technically define upwards of 100 variables) → use a **single Transform Message** with a DataWeave script that assigns them all at once, rather than chaining several separate Set Variable components one after another.

Using one Transform Message for several variables keeps the flow visually shorter and easier to maintain than a long chain of near-identical Set Variable boxes. One practical gotcha: when you first drop a Transform Message onto a flow, its default output target is `payload` — if you're only using it to set variables (not to change the payload itself), you need to remove that default payload output, otherwise you'll be unintentionally overwriting your payload too.

Demoed example: assigning the outbound segment number to a variable called `outboundSegment` and a warehouse ID to a variable called `condition`, both set inside the same Transform Message. To verify they were populated correctly, a breakpoint was added at the Transform Message step and the **Variables** panel was inspected in Debug mode.

## Common Mistakes / Things Students Got Wrong

- **Leaving the default `payload` output in a variable-only Transform Message.** If you only want to set variables, you must explicitly remove the leftover default payload mapping, or you risk silently changing the payload when that wasn't the intent (16-03-2022).
- **Java-format leakage.** As in Chapter 6, forgetting to convert a Java-formatted intermediate payload back into JSON (or whatever the target needs) before sending it onward was a repeat mistake across multiple sessions (14-03, 16-03).
- **Guessing at the wrong function for a job.** When asked how to extract the first digit of a number, the instinct was to reach for the Range function — but that's the wrong tool for the job, and the instructor deliberately left the "correct" function for students to discover on their own (23-03-2022).

## Quick Recap

- **Transform Message** converts data between formats (XML↔JSON↔Java↔etc.) and can also drop/reduce fields the target doesn't need.
- You can load **sample input/output files** as metadata so Studio maps against a real, known schema instead of guessing.
- Core DataWeave tools covered: `++` (concatenation, cross-type), `splitBy` (splitting on a delimiter), a duplicate-removal function, the **Range** function (number sequences), and `map` for iterating arrays record-by-record.
- Reading fields out of XML safely inside DataWeave often requires converting to `application/java` output first.
- **One variable → Set Variable. Several variables → one Transform Message** with a DataWeave script — and remember to strip the default payload output if you're only setting variables.
- Know at least ~10 DataWeave functions well; this is a frequent interview topic.
