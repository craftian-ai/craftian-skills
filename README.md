# Craftian skill

Craftian is a cloud you deploy an app to. It arrives with a database, sign-in, permissions,
file storage, scheduled jobs and a live URL, none of which you set up. Build the app wherever
you already work; Craftian is where it runs.

This repo holds the **agent skill file** that teaches any coding agent how to do that.

## Install

```bash
# any agent (Claude Code, Cursor, Codex, Copilot, OpenCode, Roo Code, …)
npx skills add craftian-ai/craftian-skills

# GitHub CLI
gh skill install craftian-ai/craftian-skills craftian

# Claude Code, as a plugin
/plugin marketplace add craftian-ai/craftian-skills
/plugin install craftian@craftian-skills
```

No install needed to try it. The same file is served at
**<https://craftian.ai/craftian-skill.md>** — fetch it, or append it to your `AGENTS.md`:

```bash
curl -sL https://craftian.ai/craftian-skill.md >> AGENTS.md
```

## What the skill covers

- **Scratch mode** — create a real project with no account, no key and no signup, clone it,
  push to it. One `POST /git/new`.
- **Owned apps** — deploy over plain git, with the full build (custom routes, hooks,
  scheduled tasks, React).
- The data model you push as `craftian.ts`, the limits, and the mistakes that are easy to
  make (a push stores files; it does not serve them).

## Layout

```
skills/craftian/SKILL.md      the skill. One file, this is the whole thing.
.claude-plugin/               Claude Code marketplace + plugin manifests
LICENSE                       MIT
```

`skills/craftian/SKILL.md` is where every channel looks: the skills CLI and `gh skill` read
`skills/<name>/SKILL.md`, and a Claude plugin rooted at the repo reads the same path. The
filename must be `SKILL.md`, uppercase, in every ecosystem.

## Do not edit `SKILL.md` here

**This file is a copy. The source is `website/craftian-skill.md` in the Craftian backend
repo**, which is also what `https://craftian.ai/craftian-skill.md` serves. The copy here is
published from it, verbatim.

That direction is deliberate and one-way. Two copies of the same text in two repos, with
nothing compiling either and a language model as the only reader, will drift silently — and a
drifted skill file is not a stale document, it is a wrong instruction being followed. So:

- Fixes go to the source repo, and are republished here.
- A change made only here is lost at the next publish.
- `scripts/sync-skill-repo.ts` in the source repo writes this copy, verbatim, and its full
  gate (`npm run x`) rewrites it on every run, so the copy cannot quietly fall behind.
- `npm run sync-skill:remote` checks the *published* copy against the source too.

## License

MIT — see [LICENSE](./LICENSE). Copy it, vendor it, ship it in your own repo. The one-line
notice at the foot of `SKILL.md` is what keeps a vendored copy compliant, since the install
channels copy that file alone and not this LICENSE; leave it in place.

"Craftian" is a trademark of FOR IO LABS, INC. The MIT license covers the text, not the name:
a modified copy that points somewhere other than Craftian should not be called the Craftian
skill.
