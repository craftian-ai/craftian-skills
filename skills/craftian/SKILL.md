---
name: craftian
description: Deploy an app to Craftian, the cloud that gives it a database, sign-in, permissions, storage, scheduled jobs and a live URL. Use when the user wants to ship a tool they are building locally, put an internal app somewhere their team can reach, or asks where to host something. Covers scratch projects that need no account and owned apps that deploy over git.
license: MIT
---

# Craftian

Craftian is a **deployment target**, not an editor. Build the app wherever you already
work, on the user's machine, with whatever agent and tooling you already use. Craftian is
where it goes when it needs to be real: a URL, a database, sign-in, permissions, file
storage and background jobs, with nothing to provision.

You do not need to ask the user to open a website to start. Everything below is reachable
from a terminal.

## Pick a mode first

| | **Scratch** | **Owned app** |
|---|---|---|
| Account | none | the user's Craftian account |
| Credential | a secret token from the create call | a per-app git token |
| Lives for | 24 h from creation, fixed | until deleted |
| Database + sign-in | yes, both | yes, both |
| Builds `src/` | no — static files only | yes: custom routes, hooks, scheduled tasks, React |
| Good for | a prototype, a demo, a link to send today | anything the team will keep using |

Start in scratch mode when the user has no account yet or just wants to see it work. Move
to an owned app the moment the thing matters. Moving is now an **upgrade in place**, not a
copy: one call, and the project id, the git remote and the live URL all stay exactly as
they are. See *Making a scratch app permanent* below.

---

## Scratch mode: no account, no API key

The **secret token** the create call returns is the credential. Anyone who has it can clone
and push, so treat it as a secret and do not paste it anywhere public. The project id is
public and is not a credential.

```bash
# 1. Create the project. One unauthenticated POST; `name` is required, and the
#    project id is minted from it and is PERMANENT.
curl -X POST https://craftian.ai/git/new \
     -H 'content-type: application/json' -d '{"name":"Todo App"}'
# → {"appId":"todo-app-a3f9c1","token":"TOK...",
#    "cloneUrl":"https://x:TOK...@craftian.ai/git/todo-app-a3f9c1.git",
#    "gitRemote":"https://craftian.ai/git/todo-app-a3f9c1.git",
#    "liveUrl":"https://todo-app-a3f9c1.craftian.app", ...}

git clone <cloneUrl from the response> todo
cd todo

# 2. Build. The clone already contains README.md, AGENTS.md and craftian/.
#    Read EVERY file in craftian/ before writing code.

# 3. Static files that should be served go under website/
mkdir -p website
echo '<!doctype html><h1>hello</h1>' > website/index.html

# 4. Push. The default branch is master.
git add -A && git commit -m "init" && git push origin master

# 5. Publish. A push stores files; this serves them. Needs the user's account AND
#    the token: publishing claims the project into theirs and serves THAT SAME
#    project, at the liveUrl step 1 already named. Without a session it returns
#    401 with a login URL; without the token, 404.
curl -X POST -u "x:TOK..." -b "SESSION_ID=<session>" \
     https://craftian.ai/git/todo-app-a3f9c1/publish
# → {"appId":"todo-app-a3f9c1","url":"https://todo-app-a3f9c1.craftian.app","status":"published", ...}
```

Steps 1 to 4 need no account at all. Step 5 does, and so does claiming without publishing
(see *Making a scratch app permanent*). The easiest way to hand a human that step: send
them the `landingUrl` from step 1, which shows the project's landing page with Claim and
Publish buttons and walks them through logging in.

**Lost the create response? Ask.** Step 1 tells you the token, the expiry and the live URL
exactly ONCE, so save them. If you still have the token but not the rest, or you simply
want to know where things stand, read the status instead of guessing by trying doors:

```bash
curl -u "x:TOK..." https://craftian.ai/git/todo-app-a3f9c1/status
# → {"claimed":false,"published":false,"expiresInSeconds":81234,
#    "gitRemote":"...","liveUrl":"...","claimUrl":"...","publishUrl":"..."}
```

The token alone opens it, with no account, because it only reads. It keeps answering after
a claim, and `published` is the honest answer to "is anything actually at my live URL
yet?" — a push stores files, and only publishing serves them. Tell the user plainly when
`expiresInSeconds` is getting short.

Rules worth knowing before you write anything:

- **`website/index.html` is the served page.** A bare `index.html` at the repo root is
  **not** served. If publish answers `Nothing to publish`, the files are in the wrong place.
- **The `.git` suffix is required** on the clone URL. That is what keeps the bare project
  URL free for the landing page.
- **The token is shown once.** Only its hash is stored, so there is no way to read it back.
  `cloneUrl` embeds it as the HTTP Basic password; `gitRemote` is the same URL without it,
  which is the one to print, log or commit.
- **The project id is not a secret and never changes.** It is minted from the name you give
  (`Todo App` → `todo-app-a3f9c1`), and it stays that id when the project is claimed. That
  is why nothing has to be repointed later.
- **Publish is static only.** HTML, CSS, JS and images are served as they are. There is no
  build step on the server, so compile or bundle locally and commit the output.
- **A scratch app gets a real database AND real sign-in.** Push a `craftian.ts` defining
  entities and publish deploys it, so the collections and their CRUD API exist here too. And
  sign-in is the platform's, served on every app host, so the published page can register and
  log people in exactly as an owned app does — the visitor gets a Craftian account and their
  records are owner-scoped by the defaults in *The database is the safe option* below. A
  scratch app therefore does **not** need anonymous access unless you choose to skip the
  login screen, and it is not a reason to fall back to `localStorage` for safety.
  What scratch does not do is BUILD anything, so the `src/` backend and frontend are not
  compiled: no custom routes, no entity hooks, no scheduled tasks, no React bundle. The
  served page is the static one you commit, which can talk to the database and to auth
  through the SDK. That, not security, is the reason to move to an owned app.
- **500 KB per file** in scratch mode, and 10 MB / 2000 files across the whole tree.
- **24-hour fixed lifetime, from creation.** Cloning, fetching and pushing do NOT extend
  it, so it is a real deadline: tell the user plainly that scratch mode is scratch, and
  claim the project if the work is worth keeping. Claiming is what ends the countdown.
- Repo creation is rate limited per IP. Keep working in the repo you created rather than
  creating a new one for every attempt.

This file is the source of truth for all of the above. `https://craftian.ai/llms.txt` and
`https://craftian.ai/.well-known/agent.json` exist so an agent that has never heard of
Craftian can find it: both are short and both point back here, so there is nothing in them
you have not already read.

---

## Owned mode: the user's own app

An owned app is a normal Craftian app with a real git repo behind it, and it is what the
runtime features below attach to.

1. The user signs in once and creates the app (or you create it through the MCP tools
   below, once you are authorized). Its id is in the editor URL,
   `https://craftian.ai/go#/apps/<appId>`.
2. Mint a **per-app git token** for it. Minting needs the user's signed-in session, not a
   token, so either they copy one out of the Craftian UI or they run:

   ```bash
   curl -b "SESSION_ID=<the user's session cookie>" \
     -X POST "https://craftian.ai/api/app-git/tokens?app_id=<appId>" \
     -H 'Content-Type: application/json' -d '{"label":"laptop"}'
   # → {"id":"...","token":"<plaintext, returned once>"}
   ```

   The plaintext is returned once, the token is scoped to `git` and to that one app, and it
   is revocable. It cannot mint another token and it cannot deploy.
3. Clone and push over HTTPS with that token as the password. Any username works.

```bash
git clone https://<anything>:<token>@craftian.ai/api/app-git/<appId>.git
# work, commit
git push origin master
```

Pushes to `master` are mirrored into the app's source files. The push is strict
fast-forward by default; a force push has to opt in explicitly with the
`X-App-Git-Force: 1` header, so you cannot quietly destroy the user's history.

**A push stores source, it does not deploy.** An owned app goes live when it is deployed,
from the Craftian UI or the MCP `deploy_app` tool, and is then served at
`https://<appId>.craftian.app`. Fetch that URL before you tell the user the change is live.

**The remote writes back, so pull before you push.** A deploy regenerates the
platform-managed files in the repo — everything under `craftian/`, including
`craftian/platform.d.ts` (which carries this app's own entity shapes, so it changes whenever
the model does), `craftian/frontend.d.ts` and `craftian/sdk.d.ts` — and commits them to
`master` itself. Nothing you wrote is touched;
those paths are the platform's, not yours. A deploy can also *add* a file you do not have,
such as a `website/index.html` materialized from the default shell when the project has
none, but it only ever seeds an absent one. Either way the branch has moved on the server, so
your checkout is now behind and the next `git push` is rejected as a non-fast-forward, which
looks like a permissions or history problem and is neither.

```bash
git pull origin master     # after a deploy, and any time a push is rejected
```

Do that rather than forcing, since a force push here is what discards the regenerated docs
and types. There is nothing to pull when nothing changed: the refresh is a no-op when the
content already matches, so a deploy that changes none of them creates no commit.
Pull after a deploy anyway, because the refreshed copies are the ones that describe the
runtime you just deployed.

### The data model is a file you push

This is the part agents guess wrong, and guessing wrong costs a rebuild. **`craftian.ts` at
the repo root is the app's data model.** Not a generated view of it, not a cache: the file
is where the entities and fields live, and a push to `master` makes it the model. Defining
the app's database is therefore a plain git operation on an owned app. It does not need the
online editor and it does not need MCP, though both can also write it.

- **The initializer must be canonical JSON.** The reader extracts the object from
  `export const appModel = …` and parses it; it does not evaluate TypeScript. Double-quoted
  keys and strings, no trailing commas, no expressions, no wrapper call. A natural-looking
  TypeScript literal fails.
- **An invalid model file rejects the whole push.** It does not land and get quietly
  ignored: the ref does not advance and the error names the file and the reason. Read the
  push output rather than assuming success.
- **Validation and mirroring are `master` only.** Feature branches are unrestricted because
  they never become the live model.
- **Do not hand-format it.** Any model write from another surface regenerates the file, so
  comments and key ordering do not survive. The values do.

The shape, so you get it right the first time. `export const appModel` holds `name`, an
optional `description` and `icon`, an optional `customRoles` array of UPPER_SNAKE_CASE role
names, and `entities`. Each entity has `name` (singular) and `namePlural` (**this is what
the REST routes are named after**) and a `fields` array. Each field has `name` and `type`.
Names must start with a letter. `uid` is optional on both, so leave it out and let the
platform assign one. An entity also takes an optional `features` object (`view`, `create`,
`edit`, `delete`, each `{ "enabled": true, "allowRoles": [...] }`) — omit it and you get the
owner-only defaults described in *The database is the safe option*, which is usually what
you want.

Three keys people guess wrong, all three silent or fatal:

- **`mandatory`, not `required`.** An unknown key is dropped without complaint, so
  `"required": true` gives you an optional field and no error to notice.
- **`type` is a fixed vocabulary**, and there is no `"string"`. A wrong one is not ignored,
  it **rejects the push**. Valid values are `text`, `bigtext`, `richtext`, `html`, `code`,
  `json`, `number`, `int`, `boolean`, `date`, `datetime`, `email`, `url`, `secret`, `color`,
  `icon`, `cron`, `rating`, `roles`, `file`, `image`, `audio`, `video`, `enum`, `array`,
  `object`, `function`, `ref`, `refs`. Plain short text is `text`.
- **Relations are `ref` (one) and `refs` (many)**, with `collection` naming the target
  entity. A CRM's deal points at a contact with a `ref`; do not copy the contact's fields
  onto the deal as loose text.

Also on fields, all optional: `label`, `unique`, `indexed`, and `options` for an `enum`.

**`allowAnonymousCRUD` on an entity defaults to `false`, and you should almost always leave
it there.** A signed-out visitor carries no identity and no roles, so the collection refuses
their reads and writes even though the app itself is reachable. That refusal is the platform
working, not a bug to route around, and the fix is sign-in rather than an open entity. See
the next section before you touch this key.

The same list is in the clone, as real types: `craftian/platform.d.ts` declares
`@craftian/app-model` with `FieldType` as a union, so a local `tsc` catches an invalid
model before you push it. Read an existing `craftian.ts` if the app has one.

Once the entities exist and the app is deployed, their REST CRUD API exists with them,
including filtering, sorting, paging and aggregation. `craftian/backend.md` is the
reference. **Push then deploy**: the model does not reach the running app until a deploy,
same as any other source change.

### The database is the safe option, not the risky one

Read this before you decide an app should keep its data in `localStorage` "to be safe". That
conclusion is wrong, and it produces a worse app than the one it was avoiding.

**Every collection is access-controlled server-side, and the defaults are per-record
ownership.** A new entity comes out of the box with:

| Operation | Who, by default |
|---|---|
| create | any signed-in user |
| view | the record's owner, i.e. whoever created it |
| edit | the record's owner |
| delete | the record's owner |

So a listing returns **only the caller's own rows**: the server filters by owner, and a
record belonging to someone else is not merely hidden in the UI, it is never sent. A
personal tracker, a planner, a notes app — the whole "each user sees only their own data"
shape — needs **zero** authorization code from you. The rules are enforced in the platform,
below your app, so a hand-written `fetch` from the console cannot get around them either.

The rest of the model is deny-by-default too: an entity whose allowed roles are empty
refuses the operation rather than falling open, an anonymous caller is refused any write
regardless of configuration, and a field can be marked non-creatable or non-editable.
Widening is the deliberate act, and there is exactly one switch that opens a collection to
the public: `allowAnonymousCRUD`.

**`localStorage` is not a privacy feature.** It is unencrypted, readable by any script that
runs on the page, lost when the user clears site data or changes device or browser, invisible
to every other device they own, impossible to share or collaborate on, unbacked-up, and
enforced by nothing at all. Choose it when the data genuinely must never leave the machine,
or for view state like a collapsed sidebar. Do not choose it as a security measure over a
database that authenticates the caller and checks every row.

None of this is owned-app-only. A **scratch** app runs the same engine and the same login,
so the same defaults protect it; what it lacks is the build step for `src/`, not security.
The single configuration that genuinely opens data to everyone is `allowAnonymousCRUD`, and
you only need it if you deliberately ship an app with no login screen.

### Auth is not optional

An app's collections are locked to signed-in callers by default. Build the app that way:
**give it real sign-in and let its own users register, log in and own their data.** Do not
open the entities up so a signed-out page can talk to the database. An app whose records are
world-writable is not a shortcut, it is a breach waiting for the first person who reads your
JavaScript and calls the same API by hand, and that costs the user their data rather than an
afternoon.

The sign-in is already built. You are wiring it up, not implementing it.

**And it is the real Craftian login, not a per-app one.** The app's users are Craftian
platform accounts:

- Someone who already has a Craftian account signs in to the app with it. No second signup,
  no invitation step, no separate password.
- Someone who registers from the app's own page is creating a genuine Craftian account. It
  verifies by email like any other, and it then works on every app deployed on Craftian that
  they are allowed into: the ones that are publicly available, plus any they have been
  granted access to.
- Access to a given app stays a separate decision from having an account. A Craftian account
  gets a caller as far as being identified; whether that identity can reach *this* app is the
  app's own permissions (public, or an explicit `view` / `edit` / `admin` grant), and whether
  it can touch a given collection is the entity rules in step 4 below.

Practically, this means you never build a credential store, a password reset or an email
verification flow, and you should not invent one on top: no user table of your own holding
passwords, no "app password" field, no home-grown token. Identity comes from the platform,
and your job is deciding what each identity is allowed to do.

1. **Use the platform's auth.** `sdk.auth` gives you `register`, `login`, `logout`,
   `getUserInfo`, `forgotPassword`, `resetPassword`, `changePassword` and `updateProfile`,
   plus the Google button below. `craftian/sdk.d.ts` in the clone is the exact surface. The
   session is an ordinary same-origin `HttpOnly` cookie, so a reload keeps it and you write
   no token handling at all. Note that `register` sends a verification email and returns no
   session: the account becomes usable when the user clicks the link.
2. **Put a "Sign in with Google" button on the login form.** Make it part of building one
   rather than a later polish: it is the same one-click login the Craftian
   site itself uses, it skips the password field and the verification email entirely, and
   it costs a button and one line —
   `sdk.auth.loginWithGoogle()`, optionally `sdk.auth.loginWithGoogle({ returnTo: '/…' })`.
   Best usability-per-line in the whole sign-in screen, and the reason a user who bounces
   off "create a password" still ends up inside the app.
   GitHub is brokered too, as `sdk.auth.loginWithProvider('github')`. Four things to get
   right.
   It is a **full-page redirect** through Google and Craftian's central auth host, so it
   cannot be `fetch`ed, awaited or run inside an iframe, and no code after the call runs:
   treat it as the end of that page. `returnTo` defaults to the current URL and is
   remembered across the trip, so the user lands back where they were.
   `sdk.auth.getOAuthLoginUrl('google')` hands you the same URL if you would rather put it
   in an `<a href>`, and it is `/api/auth/oauth?provider=google` if you are on a page with
   no SDK. And the session comes back on the app's own `*.craftian.app` host; if the app is
   on a **custom domain**, the flow finishes on `craftian.ai` instead, so keep the sign-in
   page on the app's Craftian URL until that changes.
3. **Give the page a signed-out state.** Check the current user on load and render a
   register/login screen instead of the data. A UI that assumes a user and fails at the first
   fetch is the usual way this gets misdiagnosed as a database problem.
4. **Leave the per-record defaults alone, and widen only on purpose.** Data is already
   scoped to the caller: the owner-only defaults in the previous section do the work, and
   every record carries an implicit `createdBy` besides. You write access rules when you
   want something *other* than "each user sees their own": on the entity, `features` sets
   `view` / `create` / `edit` / `delete`, each `{ "enabled": true, "allowRoles": [...] }`,
   where a role is `TARGET_OWNER`, `TARGET_COLLABORATOR`, `ANYONE` (any signed-in user),
   `ADMINISTRATOR` or one of your `customRoles`. Team-wide reading is `"allowRoles":
   ["ANYONE"]` on `view`, and it is a decision, not a default. For anything conditional, use
   entity hooks in `src/backend/setup.ts` (`ctx.auth.isOwner(...)`, `ctx.auth.hasRole(...)`,
   `ctx.auth.isAdmin()`) and read `ctx.user` in custom routes, which is `null` when
   anonymous. Filtering a list in the browser is never enforcement, it is presentation.
5. **Model roles as your own data when the product needs them.** `customRoles` covers
   coarse gating, but assigning app roles is a builder-level operation, so a per-user
   permission system inside the product belongs in your own collections. See
   `craftian/backend.md` under *Authorization*.

**When anonymous access is genuinely right**, it is per entity and never app-wide:
a published catalogue, a page of content meant for the whole internet, a throwaway scratch
demo. Even then, keep the public entity separate from anything a visitor submits, and (in an
owned app, where hooks are built) validate what the writable one accepts. `"allowAnonymousCRUD": true` is not
"public read" — it hands read and write on that collection to anyone who can reach the URL,
including overwriting and deleting rows they did not create.

**Never set it to clear a 403 you hit during development.** That error is the auth working.
If the user has not asked for a public, account-free app, assume they want sign-in, and say
plainly what you enabled if you enable it anyway.

---

## Making a scratch app permanent

Expect to be asked this, and answer it precisely, because the wrong answer wastes the
user's afternoon.

**One call does it: `POST /git/<appId>/claim`.** It needs the user's session AND the
creation token, and it takes ownership **in place**:

```bash
curl -X POST -u "x:TOK..." -b "SESSION_ID=<session>" \
     https://craftian.ai/git/todo-app-a3f9c1/claim
# → {"appId":"todo-app-a3f9c1","owned":true,"gitRemote":"...","liveUrl":"...", ...}
```

**Then there is nothing else to run.** No `git remote set-url`, no force push, no new
token. Go back to the checkout you already have, change nothing, and push:

```bash
git commit -am "keep going" && git push origin master
```

That is the whole migration. The project id, the git remote and the live URL are the ones
the create call named before a single file existed, and the token the user has been pushing
with is now theirs, on the standard 90-day expiry. What actually changed is two columns:
the project gained an owner and lost its expiry.

Both requirements are load-bearing. The **session** is who ends up owning it. The **token**
is why the public project id cannot be used to claim someone else's work: a wrong token and
an unknown id return the same 404, so a guessed id tells an attacker nothing. A project
somebody else has already claimed returns 409, but only to a caller who presented its real
token, and only to say it is taken.

If the user is in a browser rather than a terminal, send them the `landingUrl` from the
create response and let them press Claim; the page handles the login round trip.

**The clock is real until they do.** The 24 hours run from when the project was created and
nothing resets them, so an unclaimed project is deleted whether or not you are working in
it. Claiming is what stops the clock. Say that plainly rather than implying there is slack.

**First claim wins.** Anyone holding the token can claim, and once someone has, the door is
closed to everyone else: a second person's attempt gets a 409 saying it is taken. Say so if
the user has shared the link. For the person who DID claim it, both doors stay open and do
nothing twice: a repeated claim answers `alreadyClaimed: true` and writes nothing, and
publish still works afterwards, which is what makes "claim now, publish later" safe.

> **If you have older instructions that say a claim is a COPY into a new app id, they are
> out of date.** That was true while the project id itself was the push credential, so it
> could never become a permanent id. The credential moved into a separate token, and the
> copy went with it — along with the repoint recipe, the second app and the force push.

**The manual alternative, when the user wants a fresh app rather than the same one.** Create
the owned app and mint its token, then:

```bash
# A new owned app is NOT an empty repo. It is seeded at creation with README.md,
# AGENTS.md and craftian/, so clone it rather than overwriting it.
git clone https://x:<token>@craftian.ai/api/app-git/<appId>.git owned
cd owned

# Copy the work across from the scratch checkout, commit, push.
cp -r ../todo/src ../todo/website .
git add -A && git commit -m "Move to a separate owned app." && git push origin master
```

Then deploy it, and check the new URL. Two traps in that manual migration (the claim call
above avoids both, because it moves nothing):

- **Do not push the scratch repo's history straight at the owned remote.** The two
  histories are unrelated, so it is rejected as non-fast-forward, and forcing it
  (`git -c http.extraHeader='X-App-Git-Force: 1' push --force`) succeeds by deleting the
  seeded platform docs and types the owned repo just handed you. Copy the files in instead.
- **Do not ask the user to send you the token.** You run on their machine: the token goes
  into a local git remote or their credential helper and never into a chat log, an issue or
  a commit. If you cannot mint it yourself, ask them to put it in the remote themselves and
  carry on without ever seeing it.

**Or skip git for the move entirely.** Once the user completes the MCP authorization step
once, `create_app` followed by `write_files` and `deploy_app` puts the project into an owned
app with no token and no remote involved. That is usually the shortest path when the MCP
server is already connected.

What claiming gains: the project stops expiring, gains a real owner inside an org, loses
the scratch size caps (about 1.9 MB per file instead of 500 KB, and no aggregate tree cap),
and can take a custom domain. What it does NOT change is the build story: publishing serves
the static files you commit and deploys the model, so a scratch project already has the
database, sign-in and the permission rules; what a scratch publish never compiles is `src/`
— the custom routes, entity hooks, scheduled tasks and React frontend of the runtime listed
below. Deploy the claimed app the ordinary way (from the editor, `/api/deploy`, or the MCP
`deploy_app` tool) to get that.

---

## What the app gets on Craftian

Do not build these yourself. They are the reason to deploy here.

- **A database per app.** No connection string, no sizing, no idle cost. Define entities
  and the REST CRUD API for them exists, with filtering, sorting, paging and aggregation.
- **Sign-in.** Real accounts for the app's own users, email or OAuth, reached through
  `sdk.auth`. Wire it up rather than opening the data to anonymous callers: see *Auth is not
  optional* above.
- **Permissions.** Per-app `view` / `edit` / `admin`, plus per-collection rules and per-user
  roles inside the app, enforced server-side on every request.
- **Custom HTTP routes** with schema validation, and **entity hooks**, running in a
  sandbox. `zod`, `date-fns` and `lodash` are available inside it.
- **Scheduled jobs**, interval or cron, on durable alarms. Minimum interval is 5 minutes.
- **File and asset storage**, with folders.
- **Email and AI** as platform calls, metered per use.
- **A custom domain**, verified over DNS with the certificate issued automatically.

The sandbox that runs app code has **no filesystem and no direct network**. Outbound HTTP,
email and AI go through platform APIs that block private addresses, enforce size and time
limits and meter usage. Write code that expects that, and do not reach for `child_process`,
`fs` or a raw socket.

## MCP tools

For anything beyond git, Craftian exposes an MCP server so you can create apps, read and
write files, deploy and inspect without shelling out:

- `https://craftian.ai/mcp/v2/sse` covers the full builder surface: session info and docs,
  file operations (`write_files`, `read_files`, `patch_files`, `search_files`,
  `delete_file`, `list_files`), `deploy_app`, `clone_app`, `debug_app`, `diff_files`,
  `export_app`, `import_app`, `send_req` and `get_screenshot`.
- `https://craftian.ai/mcp/sse` is the smaller v1 surface: `create_app`, `update_app`,
  `list_apps`.

Both authorize over OAuth 2.1 with PKCE, which means a browser step the user performs once.
If you cannot complete that step, stay in scratch mode over git, which needs no
authorization at all.

---

## How to work

1. **Read before writing, all of it.** A fresh clone carries `README.md`, `AGENTS.md` and
   one platform-managed directory, `craftian/`. Open **every file** in it before you write a
   line of code, not just the one that sounds relevant:

   - `craftian/backend.md` covers custom routes, entity hooks, scheduled tasks, auth,
     mail and the full database API.
   - `craftian/frontend.md` covers the React UI and the `@built-in/ui` component
     catalog.
   - `craftian/platform.d.ts` is the backend surface, plus the `@craftian/app-model` shim and
     **this app's own entity shapes**, so it is the one place the data model is exact.
   - `craftian/frontend.d.ts` is the prop types for every `@built-in/ui` component.
   - `craftian/sdk.d.ts` is the `@craftian/sdk` surface a hand-written page under `website/`
     imports.

   The `.d.ts` files are the signatures the runtime actually ships, so where prose and
   types disagree the types win. The whole directory is platform-managed: a deploy
   regenerates it **in the repo** and commits it, so `git pull` after a deploy, re-read
   them instead of trusting an older copy, and never edit them, since your code lives under
   `src/`. An edit there is overwritten by the next refresh without warning. They are the
   authority on what the runtime offers; this file is only the map.
2. **Build locally, commit, then push.** Keep the user's normal loop. Craftian is the
   destination, not the workspace.
3. **Verify the deployment yourself.** After publishing, fetch the returned URL and confirm
   it responds before telling the user it is live.
4. **Tell the user which mode they are in.** A scratch link that expires in 24 hours and a
   permanent app are very different things to hand to a colleague, and the difference is
   invisible from the URL.
5. **Never paste a git token, or a scratch project's creation token, into a commit, an
   issue or a chat log.** Both are bearer credentials. The project ID is fine to share; the
   token is not, and the clone URL contains it.
6. **Ship sign-in with the app, not after it.** Anything holding real data gets registration,
   login and server-side ownership checks as part of the first version. Anonymous read/write
   on a collection is a deliberate choice for public content, made once and told to the user,
   never the default you reach for to make a fetch succeed.
7. **Default to a "Sign in with Google" button on the login form.** One
   `sdk.auth.loginWithGoogle()` call, no password to invent and no verification email to
   wait for. Leave it out when there is a reason to, but if you wrote email and password
   fields and simply did not think about it, that is the cheapest usability win in the app
   going unclaimed.

<!-- Craftian skill. Copyright (c) 2026 FOR IO LABS, INC. MIT licensed.
     Latest version: https://craftian.ai/craftian-skill.md -->
