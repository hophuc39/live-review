# Scope: Client Feedback App

A feedback tool for freelancers. You add a client's website, share a link, the client pins comments right on the live site, and you work through them on a board in your dashboard. Anyone can sign up for their own account; clients never need one.

**Build approach:** Tracer Bullet (prove one real thread through every layer first, then thicken it one strand at a time).
**Workflow:** Beta (after `/develop`: `/check verify`, then `/test`). The project default level of rigor. `/architect` is the recommended first stop for a feature with a real decision, but skippable when you already know the build. Any feature can carry its own tag (like `· GA`) to do more or less.

_These are recommendations to keep your build orderly, not requirements. Skip anything that does not fit: if you already know how to build a feature, use `/develop` and skip `/architect`. You decide when a feature is `done`._

## At a glance

| # | Feature | Phase | Status |
|---|---------|-------|--------|
| 1 | Stack & architecture | Foundation | in-progress |
| 2 | Coding standards & tooling | Foundation | planned |
| 3 | Data model | Foundation | planned |
| 4 | Design system & UI foundation | Foundation | planned |
| 5 | Sign up & sign in | Slice 1 | planned |
| 6 | Projects & share link | Slice 1 | planned |
| 7 | Live preview & pinning | Slice 1 | planned |
| 8 | Comment list in the dashboard | Slice 1 | planned |
| 9 | Status board | Slice 2 | planned |
| 10 | Share link on/off | Slice 3 | planned |
| 11 | Pins across pages | Slice 4 | planned |
| 12 | Image attachments | Slice 5 | planned |
| 13 | Landing page & SEO | Slice 6 | planned |
| 14 | Privacy & terms pages | Slice 6 | planned |

## Foundations

### 1. Stack & architecture
Decide the stack (including hosting and how the app serves both the dashboard and public review pages) and scaffold a runnable project, so every slice builds on real structure.
**Done when:** the stack is recorded in a spec and the empty scaffold boots locally and passes build.
spec [0001](../specs/0001-stack-architecture/index.md)
- [x] Decide the stack (spec): `/architect stack & architecture`
- [ ] Scaffold from the decision: `/develop stack & architecture`

### 2. Coding standards & tooling
Capture conventions from the real scaffolded project, then install lint, format, and pre commit checks.
**Done when:** root `AGENTS.md` reflects the real stack, and lint, format, and pre commit hooks run clean.
- [ ] Capture conventions and tooling choices: `/audit`

### 3. Data model · needs a decision
Core entities every feature builds on: users, projects (with a site URL and share token), comments (text, pin position, page path, status), and attachments.
**Done when:** each user's projects and comments are fully separate from other users', and the entities support the board, link control, multiple pages, and attachments without a breaking migration.
- [ ] Design it (spec): `/architect data model`

### 4. Design system & UI foundation · needs a decision
Visual language, layout, and base components shared by the dashboard, the comment widget on the review page, and the public pages.
**Done when:** `design.md` covers type, color, spacing, and components, and base components handle focus and keyboard use.
- [ ] Design it (spec): `/architect design system & UI foundation`

## Slice 1: First pin, end to end

The thinnest real thread: you sign up, add a site, share the link, a client pins one text comment, you see it. Real auth, real database, real UI; breadth comes later. This slice is the walking skeleton.

### 5. Sign up & sign in · needs a decision · GA
Open signup with one account per person, so your projects and your clients' feedback stay private to you.
**Done when:** a new user can sign up, sign in, sign out, and regain access if they forget their password; dashboard pages are unreachable while signed out.
- [ ] Design it (spec): `/architect sign up & sign in`

### 6. Projects & share link
Create a project for a client site by entering its URL, and get a hard to guess share link for it.
**Done when:** you can create, rename, and delete a project, see your project list, and copy its share link; nobody else sees your projects.
- [ ] Build it: `/develop projects & share link`

### 7. Live preview & pinning · needs a decision
The heart of the product. The client opens your link, sees the live site, clicks anywhere to drop a pin, and types a comment, no account needed. The spec decides how the site is shown and how pins stay attached to the right spot.
**Done when:** an anonymous visitor with the link sees the live site, can pin a text comment that saves with its position and page path, and sees existing pins in the same place after a reload and at other screen widths; review pages are kept out of search engines.
- [ ] Design it (spec): `/architect live preview & pinning`

### 8. Comment list in the dashboard
See every comment on a project in one list, newest first, so feedback reaches you.
**Done when:** a project page lists its comments with text, page path, and time, including an empty state, and opening one shows it on the live preview at its pin.
- [ ] Build it: `/develop comment list in the dashboard`

## Slice 2: Manage feedback

### 9. Status board · needs a decision
Turn the comment list into a board with to do, doing, and done columns, so you can track work through each client's feedback.
**Done when:** new comments land in to do, you can move a comment between columns and the change persists, and done comments no longer clutter the client's view of the site.
- [ ] Design it (spec): `/architect status board`

## Slice 3: Link control

### 10. Share link on/off
Turn a project's review link off when the project wraps up, and back on when you need it.
**Done when:** a disabled link shows a friendly closed message and accepts no new comments, and enabling it again restores it with existing comments intact.
- [ ] Build it: `/develop share link on/off`

## Slice 4: Whole site reviews

### 11. Pins across pages
Clients browse the site inside the review and pin on any page, not just the first one. Relies on the approach decided in the live preview spec.
**Done when:** a client can navigate between pages within the review, each pin appears only on its own page, and the dashboard can filter comments by page.
- [ ] Build it: `/develop pins across pages`

## Slice 5: Richer comments

### 12. Image attachments · needs a decision
Clients can attach images to a comment to show what they mean.
**Done when:** a client can attach one or more images to a comment, you see them on the board and in the preview, and oversized or non image files are rejected with a clear message.
- [ ] Design it (spec): `/architect image attachments`

## Slice 6: Public launch

### 13. Landing page & SEO · needs a decision
A public page that explains the product and leads to signup, so other freelancers can find and try it.
**Done when:** the landing page has a clear pitch and a signup call to action, and ships with page titles, descriptions, social cards, and a sitemap; only public pages are indexable.
- [ ] Design it (spec): `/architect landing page & SEO`

### 14. Privacy & terms pages
A plain privacy note and terms, since anonymous clients submit text and images on your users' projects.
**Done when:** privacy and terms pages exist and are linked from the landing page, signup, and the review page footer.
- [ ] Build it: `/develop privacy & terms pages`

## Deferred
Out of scope for the current build pass, kept so the plan stays honest.
- **Email notifications**: tell you when a client leaves feedback · needs a decision
- **Live dashboard updates**: new pins appear without a refresh · needs a decision
- **Browser and device info**: capture the client's browser and screen size with each pin
- **Screenshot of the pinned spot**: record what the client saw · needs a decision
- **Replies on a comment**: back and forth on one pin · needs a decision
- **Client names**: optional name on comments instead of fully anonymous
- **Password protected links**: extra lock for sensitive client work
- **Review rounds**: archive comments per version of the site · needs a decision
- **Images and PDF reviews**: pin on uploaded mockups, not just live sites · needs a decision
- **Team workspaces**: share projects with teammates · needs a decision · GA
- **Billing & plans**: free and paid tiers · needs a decision · GA
- **Error monitoring**: know when the overlay breaks on a client's site · needs a decision
- **Abuse protection on public review links**: rate limit anonymous comments per share link and per visitor, before links are shared publicly · needs a decision · from spec 0001
- **Accessibility to WCAG AA**: full keyboard and screen reader support beyond the design system basics
- **Product analytics**: measure signups and review activity · needs a decision

## Legend

**The decision box.** Every feature that needs a decision carries one step whose label ends with `(spec)`. Its wording varies (`Design it (spec)` normally, `Decide the stack (spec)` on Stack & architecture), so skills find it by that `(spec)` ending, never by an exact label. Every other box is an execution box, and `/architect` never ticks one.

**Feature lifecycle**: the scope updates as a feature moves; each row is what it shows and who sets it:

| State | Set by | The feature shows |
|---|---|---|
| `planned` · needs a decision | `/scope` | one box: `Design it (spec): /architect <feature>` |
| `in-progress` (designed) | **`/architect` at spec capture** | `Design it` ticked; spec linked; `Build it: /develop <feature>` plus **2 to 5 milestones**; the tier's closing boxes (`Verify it` Alpha and up, `Test it` Beta and up, `Review it` and `Document it` GA); any surfaced follow up enrolled |
| `in-progress` (building) | `/develop` | milestone boxes tick one by one; code pointer filled |
| `in-progress` (verified) | `/check verify` | `Build it` and milestones ticked; `Verify it` ticked |
| `done` | **you, when you decide it is** (any skill sets it when you say so); `/sync` reconciles | boxes you ran ticked, skipped ones marked skipped; the tier's last stage (`Prototype` after `/develop`; `Alpha` after `/check verify`; `Beta`/`GA` after `/test`) is the suggested point to call it done; `/sync` captures conventions |

- **Next step** = the first unticked box (always a command or a tracked milestone).
- **needs a decision** = run `/architect` first; otherwise straight to `/develop` (or `/audit` for standards & tooling). The tag drops once the spec is captured.
- **Atomic build tasks live in the spec's `## Build plan`, not here**: the scope carries only the milestone rollup.
- **Status** `planned` → `in-progress` → `done`, plus `existing` (built before this workflow) and `dropped` (taken out of scope, kept for history).
- **Approach tag** beside a heading (like `· Facade`) overrides the project default for that feature; no tag means it inherits.
- **Workflow tier tag** beside a heading (like `· GA`, `· Prototype`) sets that one feature's rigor above or below the project default; no tag inherits the default. It decides the feature's check boxes and each skill's next suggestion.
- **Workflow** (header line) is the project default, what runs after `/develop`: **Prototype** = nothing (trust develop's own build time self check); **Alpha** = `/check verify`; **Beta** = `/check verify` then `/test`; **GA** = adds a fresh model `/check review` then `/document`. A feature built on an unratified decision (an `Assumed` spec) stays flagged, but that never blocks `done`.
- **Pointer line** (`spec <n> · code in <path>`): the spec link added by `/architect`, the code path by `/develop`.
