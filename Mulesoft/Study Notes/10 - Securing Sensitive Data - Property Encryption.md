# Chapter 10: Securing Sensitive Data — Property Encryption

> Taught on: 04-04-2022, 22-04-2022, 27-04-2022

## Why This Matters

Every Mule application ends up needing to store some values that should never be visible in plain text — database passwords, API secrets, third-party credentials, and similar sensitive configuration. If these values sit in a plain property file, anyone with access to the codebase (or the deployed artifact) can read them directly. Encrypting sensitive property values, and configuring Mule to decrypt them safely at runtime, is a basic and non-negotiable security practice — and it's also something enforced automatically by CI/CD pipelines in real organizations, not just a "nice to have."

## Property Files in General

A **property file** is where you keep configuration values a Mule application needs — things like hostnames, ports, and credentials — separated out from the flow logic itself. Keeping configuration in property files (rather than hardcoding values directly into flows) is what lets the same application be pointed at different environments without changing the underlying logic: for example, a global configuration using property placeholders can be switched to point at a test property file versus a production property file, simply by changing which file is loaded, without touching the flow itself (22-04-2022).

## Encrypting Sensitive Values with AES

For values that shouldn't be stored as plain text — passwords, secrets, keys — MuleSoft supports encrypting them directly inside the property file, using an encryption algorithm. The algorithm named specifically in the course is **AES**, and one example configuration used a **16-digit secure key** for the encryption (22-04-2022).

### Reading Property Values — the Syntax

There's a clear syntax distinction between reading a normal (unencrypted) property value and reading an encrypted one, and it differs depending on whether you're inside a config field or inside a DataWeave script:

| Context | Plain property | Encrypted (secure) property |
|---|---|---|
| Config field (e.g. connector settings) | `${keyName}` | `${secure::keyName}` |
| Inside DataWeave / Transform Message | — | `p('secure::keyName')` |

The `secure::` prefix is what tells Mule this particular value needs to be decrypted using the secure properties configuration before it can be used — leaving it off would cause Mule to try to use the still-encrypted string as-is, which won't work.

## Secure Property Config / Secure Property Placeholder

Encrypting the value in the property file is only half the job — Mule also needs a matching configuration in **Global Elements** that knows how to decrypt it (what algorithm was used, what key to decrypt with). Without this configuration in place, **the application will fail to deploy** the moment it tries to read an encrypted property it doesn't know how to decrypt (04-04-2022).

This configuration component is named differently across Mule versions — a naming-only change, not a conceptual one:

- **Mule 4**: called **Secure Property Config**.
- **Mule 3**: called **Secure Property Placeholder**.

In both versions, the underlying concept is identical: encrypt the sensitive value in the property file, then reference a Global Elements configuration that tells Mule how to decrypt values as they're read at runtime. If you ever see a course note or documentation referencing "Secure Property Placeholder," treat it as the Mule 3 name for exactly the same thing Mule 4 calls "Secure Property Config."

## Why This Matters for Credentials Specifically

The whole reason this topic exists is credentials: database usernames/passwords, third-party API keys, client secrets — anything that, if leaked via a plain-text property file or a committed source file, would compromise a live system. Encrypting these values means that even if the property file itself is exposed (e.g., accidentally committed to source control, or read by someone without proper access), the actual sensitive value isn't readable without also having the corresponding secure-properties decryption configuration.

## CI/CD Enforcement — Not Just a Suggestion

This isn't left to individual developer discipline in real organizations — it's checked automatically. As part of the CI/CD pipeline's **validation stage** (27-04-2022), two specific rules are enforced before code is even allowed to proceed toward deployment:

- **No commented-out code** is allowed in the codebase.
- **Sensitive information must be encrypted in the property file** — nothing sensitive is allowed to be checked in as plain text.

If either of these checks fails, the pipeline blocks the code from moving forward — this is on top of (and separate from) the code-coverage gate covered in the MUnit material, which requires coverage above roughly 80% before deployment is permitted. In other words: property encryption isn't just a best practice you're encouraged to follow — in a real pipeline, forgetting to encrypt a sensitive value is treated the same as any other blocking validation failure.

## Common Mistakes / Things Students Got Wrong

- **Encrypting a property value but forgetting to configure Secure Property Config/Placeholder in Global Elements.** The application will simply fail to deploy in this state — Mule has no way to decrypt the value it's being asked to read (04-04-2022).
- **Mixing up the reference syntax.** Using `${keyName}` (plain) instead of `${secure::keyName}` (encrypted) in a config field, or forgetting the `p('secure::keyName')` form required specifically inside DataWeave, will cause the value to be read incorrectly.

## Quick Recap

- Property files hold configuration (including environment-specific values) separately from flow logic, so the same app can point at different environments.
- Sensitive values (passwords, secrets, keys) are encrypted in the property file using **AES**.
- Reference syntax: `${keyName}` for a plain value, `${secure::keyName}` in config fields for an encrypted value, `p('secure::keyName')` inside DataWeave for an encrypted value.
- The decryption configuration lives in Global Elements: called **Secure Property Config** in Mule 4, **Secure Property Placeholder** in Mule 3 — same concept, different name.
- Without this Global Elements configuration, an app using encrypted properties will **fail to deploy**.
- Real CI/CD pipelines enforce this automatically: no commented-out code, and no plain-text sensitive values allowed through the validation stage.
