# Mathrix

**A learning portal for my private math students.** It's live at **[mathrixapp.com](https://mathrixapp.com)** and used by real students.

I've taught high-school math since 2016. Mathrix replaces the WhatsApp groups and shared drives I used before: every student gets materials and homework that match their grade and level, submits work in one place, and I review it from a single queue.

> The source code is in a private repository, because the app serves real students and most of them are minors. This page describes the product and the engineering behind it. I'm happy to walk through the code in an interview.

---

## What it does

**For the teacher**
- Two fully separated divisions: middle school (grades 7–9) and high school (grades 10–12, by unit level). You pick one at a gate and switch from the header.
- A topic tree of any depth per track. Materials (PDF, images, video) can be shared with a whole track, with a group, or with one student.
- Assignments with a review queue: grade a submission, then move to the next one under the same filters.
- Invitation-only students: the teacher creates the record, the student gets an email invitation, and their first sign-in links the account.

**For the student**
- Materials and assignments organized by topic, showing only what was shared with them.
- Homework submissions. A resubmission is a new attempt, never an overwrite.

The whole interface is Hebrew and right-to-left.

## Stack

**Next.js 16** (App Router, Server Components, Server Actions) · **TypeScript** · **PostgreSQL** on Supabase · **Prisma 7** · **Clerk** (auth) · **Supabase Storage** · **Tailwind CSS v4** + shadcn/ui · **Vercel**

## Engineering decisions

**Authorization lives in one place.** Prisma connects with full database privileges, so database-level row security never runs for the app. Instead, every query that touches student data goes through a single module (`lib/authz.ts`) that resolves the current user on the server and checks access. User IDs never come from the URL or a form. Row-level security is still enabled on every table, with no policies, as defense in depth.

**Files are never public.** Uploads go to private storage buckets, and every download or upload uses a short-lived signed URL that is issued only after an authorization check. The file is written first and the database row second, so a failed upload never leaves a row pointing at nothing.

**Migrations are safe to deploy.** Production migrations run automatically during the Vercel build, inside a transaction. If one fails, the build fails and the previous version keeps serving. Schema changes follow expand → contract (add the column with a default, deploy code that writes it explicitly, then drop the default), so code and schema can deploy in either order. Preview deployments never touch the production database.

**Rules are enforced on the server, not just in the UI.** Server actions validate their input and check that all topics, students and groups in a request belong to the same division. Server actions never throw to the client: every error is caught and returned as a readable Hebrew message.

**Measured performance work.** I profiled the hot path and found two problems. Every request was making a call to the auth provider, which I replaced with an indexed lookup by user ID. Database queries were running one after another, so I ran independent ones in parallel. I also learned that `connection_limit` in the connection string does nothing with Prisma's driver adapter, so the pool size is set explicitly.

**Thumbnails are generated in the browser.** PDF pages are rendered with pdf.js and video frames are captured in the browser, then uploaded with the material, so the server never processes files. Safari can't capture a frame from a local video blob, so videos are captured from the signed URL after upload.

## Three environments

| | Local | Preview | Production |
|---|---|---|---|
| Database | dev project | dev project | separate project in a separate org |
| Auth | test instance | test instance | live instance |
| Migrations | manual | never | automatic on deploy |

Every change goes through a branch, a pull request and a Vercel preview before it's merged to production.

---

Built by **Stav Balaish** · [stavbalaish2000@gmail.com](mailto:stavbalaish2000@gmail.com)
