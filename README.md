# renovatebot/renovate#46375

npm: `--before` from `minimumReleaseAge` produces a phantom `ERESOLVE` ("Found: vite@undefined") when a peer is already locked at a version younger than the cooldown.

## Setup

- `package.json`: `@tailwindcss/vite` pinned at `4.3.2`, `vite` at `^8.3.0` (dev dependencies).
- `package-lock.json` (v3) locks `vite 8.3.0` (published 2026-09-10) – i.e. younger than the 30-day cooldown.
- `renovate.json`: `minimumReleaseAge: "30 days"`, nothing else.
- `@tailwindcss/vite 4.3.3` (published 2026-07-16) is older than the cooldown, so Renovate proposes `4.3.2 → 4.3.3`. Its peer range `vite@"^5.2.0 || ^6 || ^7 || ^8"` admits the locked `vite 8.3.0`.

## Current behavior

Renovate 44.116.1 (CLI, npm 11.19.0) opens [#1](https://github.com/Moshe1010/renovate-repro-46375/pull/1) with an "Artifact update problem" and a red `renovate/artifacts` status:

```
npm error code ERESOLVE
npm error ERESOLVE unable to resolve dependency tree
npm error Found: vite@undefined
npm error node_modules/vite
npm error   dev vite@"^8.3.0" from the root project
npm error Could not resolve dependency:
npm error peer vite@"^5.2.0 || ^6 || ^7 || ^8" from @tailwindcss/vite@4.3.3
```

Renovate removes the `node_modules/@tailwindcss/vite` entry from the lockfile and runs `npm install --package-lock-only ... --before=<now - 30 days>`. While re-placing `@tailwindcss/vite`, npm validates its peer edge; the locked `vite 8.3.0` is hidden by the date filter, so its node is versionless (`vite@undefined`) and npm reports a conflict that does not exist.

The existing fallback (retry without `--before`, `beforeFallback` notice) only matches ETARGET stderr (`with a date before`), so it does not fire for this ERESOLVE.

### Same thing with npm alone

```sh
# from a clone of this repo
sed -i.bak 's/"@tailwindcss\/vite": "4.3.2"/"@tailwindcss\/vite": "4.3.3"/' package.json
node -e 'const f="package-lock.json",l=require("./"+f);delete l.packages["node_modules/@tailwindcss/vite"];l.packages[""].devDependencies["@tailwindcss/vite"]="4.3.3";require("fs").writeFileSync(f,JSON.stringify(l,null,2))'
npm install --package-lock-only --no-audit --ignore-scripts --before="$(node -e 'console.log(new Date(Date.now()-30*864e5).toISOString())')"
# -> ERESOLVE ... Found: vite@undefined
npm install --package-lock-only --no-audit --ignore-scripts
# same stripped lockfile, no --before -> succeeds, @tailwindcss/vite 4.3.3, vite stays 8.3.0
```

Tested with npm 11.19.0 / 11.20.0 and 12.1.0. `min-release-age=30` in `.npmrc` fails identically (npm flattens it into the same `before` option).

## Expected behavior

The lockfile update succeeds: `@tailwindcss/vite` becomes `4.3.3`, `vite` stays at the already-locked `8.3.0`. For example, the existing retry-without-`--before` fallback also covers an `ERESOLVE` whose stderr contains `Found: <name>@undefined`, and the existing `beforeFallback` artifact notice is attached.

## Link to the Renovate issue or Discussion

https://github.com/renovatebot/renovate/discussions/46375
