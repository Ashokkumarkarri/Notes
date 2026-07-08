# Chapter 21: Git, Bitbucket, and CI/CD Pipelines

> Taught on: 26-04-2022, 27-04-2022

## Why This Matters

Building a working flow in Anypoint Studio is only part of the job in a real organization. That code needs to live somewhere shared, be tracked as it changes, and go through a controlled process before it's trusted to run in production. This is what version control (Git, hosted on **Bitbucket** in this course) and **CI/CD pipelines** are for. None of this is Mule-specific — it's the same discipline applied across every technology an organization uses — but you need to know how it fits around your Mule projects specifically, since this is how your code actually gets from your laptop to a live, running application that a business depends on.

## Pushing an Anypoint Studio Project to Bitbucket

The demo used a simple Anypoint Studio project named "Dummy" to walk through the process of getting a local project into a shared Bitbucket repository.

The first, easy-to-forget step: **you must create an empty repository in Bitbucket before you can push anything into it.** A freshly created, empty Bitbucket repository literally has no files in it yet. For context, a typical Mule project on disk includes a `src/main` folder, a `src/test` folder, and a `pom.xml` (the Maven project file) — but a brand-new empty repo won't contain any of these yet, not even generated files like `artifact.json`, until you actually push your project into it.

## Bitbucket Branching Strategy: master vs develop

Once a repository exists, how you use branches inside it matters a lot. The default branch created with any new repository is called **`master`**.

The standard practice, reinforced across the course as applying to every technology (not just Mule), is:

- **`master` is reserved exclusively for production.** It should always mirror exactly what is currently running in the live production environment.
- **Code that is still being developed or tested should never be pushed directly to `master`.**
- Instead, for ongoing work — like a brand-new project (such as "Dummy") that isn't production-ready yet — you create a **separate branch**, commonly named **`develop`**, and do your testing and development work there.

In the fuller branching model discussed the next day, real organizations typically use three kinds of branches together: **`master`**, **`develop`**, and **`feature`** branches. `master` stays production-only; `develop` holds integrated, in-progress work; `feature` branches are typically where individual pieces of new work happen before being merged upward. This branching approach isn't specific to MuleSoft at all — it's the same pattern an organization would apply across its Java, .NET, or any other codebase.

## Creating and Cloning a Branch

To create a new branch in Bitbucket:

1. Go to the repository's **Branches** section.
2. Click **Create branch** (in the top-right corner).
3. Give the branch a name — e.g., `develop`.

To then get that specific branch onto your local machine using **Git Bash**:

1. Right-click inside the folder where you want the project, and choose **Git Bash Here**.
2. Copy the repository's clone URL from Bitbucket.
3. Run:
   ```
   git clone -b <branch-name> <repository-url>
   ```
   For example: `git clone -b develop <repo-url>`
4. The `-b` flag is what tells Git which branch to check out — if you omit it, Git will clone the default `master` branch instead, which is *not* what you want if your work is happening on `develop`.
5. When prompted, enter your Bitbucket credentials to authenticate the clone.

## CI/CD Pipeline Stages

Once code is pushed to a shared branch, it doesn't just get manually deployed by whoever feels like it — in a real organization, it flows through an automated **CI/CD pipeline** (Continuous Integration / Continuous Deployment) that enforces a set of rules before anything reaches production. The pipeline stages discussed:

**1. Validation stage** — checks the code itself for basic hygiene and security issues before anything else happens:
   - **No commented-out code is allowed** in the codebase. (Leftover commented code is considered clutter/risk, not something that should sit in a shared repo.)
   - **Sensitive information must be encrypted** in property files — you cannot check in plain-text secrets; they need to be secured (see the secure property configuration approach covered elsewhere in the course).

**2. Code coverage stage** — enforces that automated test coverage (via MUnit, covered in the previous chapter) is **above 80%**. If coverage falls short of this bar, **the pipeline will not allow the application to be deployed to runtime at all** — this is a hard stop, not a warning.

**3. Deployment stage** — the final step, where the application is actually deployed to the runtime environment, but only after it has cleared validation and coverage.

If every stage is followed correctly, the pipeline's generated script validates the code end-to-end automatically — nobody has to manually eyeball the code or manually check coverage numbers before every release; the pipeline enforces it mechanically, every time.

The course also mentioned that Jenkins pipeline material was shared alongside this discussion, as one concrete example of a tool used to implement these CI/CD stages in practice.

## How This Connects: Branching Strategy Applies Across Technologies

A point worth sitting with: everything in this chapter — the `master`/`develop`/`feature` branching model, the validation-then-coverage-then-deploy pipeline shape — is **not a MuleSoft-specific pattern**. It's a general software-engineering discipline that gets applied the same way whether the codebase behind it is Mule, Java, .NET, or anything else. As a MuleSoft developer working inside a larger engineering organization, you're expected to plug into this same shared process, not invent a separate one just because your code happens to live in Anypoint Studio.

## Quick Recap

- You must create an empty Bitbucket repository before pushing a local Anypoint Studio project into it — a new repo starts with no files at all.
- `master` is reserved for production only and should always mirror what's live; in-progress work goes on a separate branch, typically `develop`, with individual pieces of work often further split into `feature` branches.
- Create a branch in Bitbucket via Branches → Create branch; clone a specific branch locally with `git clone -b <branch-name> <repository-url>` (omitting `-b` clones `master` by default).
- CI/CD pipelines gate deployment through stages: **validation** (no commented-out code, secrets must be encrypted) → **code coverage** (must exceed 80%, via MUnit) → **deployment** to runtime.
- If coverage or validation fails, the pipeline blocks the deployment outright — this enforcement is automatic and mechanical, not a manual review step.
- This branching and pipeline discipline is a general engineering practice used the same way across every technology in an organization, not something unique to Mule.
