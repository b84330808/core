# Cloud report: vuejs/core

## Status: fix is ready, push blocked

Both branches exist in the cloud container, but **neither could be pushed to `b84330808/core`**.

```
remote: Claude doesn't have GitHub access to b84330808/core for your organization.
fatal: unable to access 'https://github.com/b84330808/core/': The requested URL returned error: 403
```

This session's GitHub connection is the account `wilson-li-appier`, which has no push access to `b84330808/core` (the access check reports `push_denied`, scope `identity`). I did not try other ways to write to the fork. A GitHub MCP write would also use `wilson-li-appier` and would author the commit as that account instead of `b84330808`.

To get the fix onto the fork, apply the attached patch locally (it keeps the author and message):

```bash
git fetch https://github.com/vuejs/core main
```

```bash
git checkout -b fix-ssr-webkitdirectory FETCH_HEAD
```

```bash
git am 0001-fix-ssr-treat-webkitdirectory-as-a-boolean-attribute.patch
```

The other option is a new cloud session whose GitHub connection is the `b84330808` account.

## Branch

| | |
|---|---|
| Branch | `fix-ssr-webkitdirectory` |
| Commit | `4ef8b06db9e4bbc50c4c29d94bcdd653b134242a` (the SHA changes if re-applied with `git am`) |
| Base | `upstream/main` @ `24c9d0bffb2299635b8112a978d74a38723aaa10` (`fix(types): add dialog closedby attribute (#16064)`) |
| PR target | `vuejs/core` `main` (bug fix, no new API, per `.github/contributing.md`) |
| Author / committer | `b84330808 <b84330808@gmail.com>` |
| Trailer | `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>` |
| Files | `packages/shared/src/domAttrConfig.ts` (+1/-1), `packages/server-renderer/__tests__/ssrRenderAttrs.spec.ts` (+6) |

There is no upstream issue for this bug (see the duplicate search below), so the commit header has no `(fix #NNNN)`. `contributing.md` accepts a fix whose reproduction is in the PR itself. Precedent: the same kind of change for `playsinline` (#15627) and `disablepictureinpicture`/`disableremoteplayback` (#15952) was merged without a linked issue.

## Diff

```diff
--- a/packages/shared/src/domAttrConfig.ts
+++ b/packages/shared/src/domAttrConfig.ts
@@ -27,7 +27,7 @@ export const isBooleanAttr: (key: string) => boolean = /*@__PURE__*/ makeMap(
   specialBooleanAttrs +
     `,async,autofocus,autoplay,controls,default,defer,disabled,` +
     `inert,loop,open,required,reversed,scoped,seamless,` +
-    `checked,muted,multiple,selected`,
+    `checked,muted,multiple,selected,webkitdirectory`,
 )
--- a/packages/server-renderer/__tests__/ssrRenderAttrs.spec.ts
+++ b/packages/server-renderer/__tests__/ssrRenderAttrs.spec.ts
@@ -75,6 +75,12 @@ describe('ssr: renderAttrs', () => {
     ).toBe(``)
   })

+  test('webkitdirectory boolean attr', () => {
+    expect(ssrRenderAttrs({ webkitdirectory: false })).toBe(``)
+    expect(ssrRenderAttrs({ webkitdirectory: true })).toBe(` webkitdirectory`)
+    expect(ssrRenderAttrs({ webkitdirectory: '' })).toBe(` webkitdirectory`)
+  })
+
```

## PR title

```
fix(ssr): treat webkitdirectory as a boolean attribute
```

## PR body

````markdown
`webkitdirectory` is a boolean attribute (presence means "pick a directory"), but it is not in `isBooleanAttr`, so SSR renders a falsy binding as a string attribute:

```vue
<input type="file" :webkitdirectory="allowFolders" />
<!-- allowFolders = false -->
```

SSR output on `main`, both for compiled templates (`ssrRenderAttr`) and render functions (`ssrRenderAttrs`):

```html
<input type="file" webkitdirectory="false">
```

Browsers only check whether the attribute is present, so this input becomes a directory picker (`input.webkitdirectory === true` in Chromium). Hydration doesn't patch this prop, so the attribute stays on the element after hydration. It is only corrected once the bound value changes.

Client-only rendering is not affected: `webkitdirectory` is also the DOM property name, so `patchProp` sets it as a prop.

This adds `webkitdirectory` to `isBooleanAttr`, the same kind of change as #15627 (`playsinline`) and #15952 (`disablepictureinpicture` / `disableremoteplayback`). With it, SSR renders ` webkitdirectory` for truthy / `''` values and omits it for falsy ones. The dev hydration mismatch check now also covers this attribute.

I used Claude Code to write this change.
````

## Fail-before (fix reverted, test kept)

`git checkout HEAD~1 -- packages/shared/src/domAttrConfig.ts`, then:

```
$ pnpm test-unit packages/server-renderer/__tests__/ssrRenderAttrs.spec.ts --run
$ vitest --project unit* packages/server-renderer/__tests__/ssrRenderAttrs.spec.ts --run

 RUN  v4.1.11 /home/user/core

 ❯ |unit| packages/server-renderer/__tests__/ssrRenderAttrs.spec.ts (27 tests | 1 failed) 30ms
     × webkitdirectory boolean attr 8ms

⎯⎯⎯⎯⎯⎯⎯ Failed Tests 1 ⎯⎯⎯⎯⎯⎯⎯

 FAIL  |unit| packages/server-renderer/__tests__/ssrRenderAttrs.spec.ts > ssr: renderAttrs > webkitdirectory boolean attr
AssertionError: expected ' webkitdirectory="false"' to be '' // Object.is equality

- Expected
+ Received

+  webkitdirectory="false"

 ❯ packages/server-renderer/__tests__/ssrRenderAttrs.spec.ts:79:56
     77|
     78|   test('webkitdirectory boolean attr', () => {
     79|     expect(ssrRenderAttrs({ webkitdirectory: false })).toBe(``)
       |                                                        ^
     80|     expect(ssrRenderAttrs({ webkitdirectory: true })).toBe(` webkitdir…
     81|     expect(ssrRenderAttrs({ webkitdirectory: '' })).toBe(` webkitdirec…

⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯⎯[1/1]⎯


 Test Files  1 failed (1)
      Tests  1 failed | 26 passed (27)
   Start at  02:10:26
   Duration  870ms (transform 528ms, setup 67ms, import 540ms, tests 30ms, environment 0ms)

[ELIFECYCLE] Command failed with exit code 1.
exit: 1
```

## Pass-after (commit `4ef8b06`)

```
$ pnpm test-unit packages/server-renderer/__tests__/ssrRenderAttrs.spec.ts --run
$ vitest --project unit* packages/server-renderer/__tests__/ssrRenderAttrs.spec.ts --run

 RUN  v4.1.11 /home/user/core


 Test Files  1 passed (1)
      Tests  27 passed (27)
   Start at  02:10:28
   Duration  565ms (transform 268ms, setup 37ms, import 299ms, tests 21ms, environment 0ms)

exit: 0
```

## Other checks (all run in the cloud container)

- `pnpm test-unit packages/server-renderer packages/compiler-ssr packages/shared packages/runtime-core/__tests__/hydration.spec.ts packages/compiler-dom --run`: 52 files, 855 tests passed.
- `pnpm check` (`tsc --incremental --noEmit`): exit 0.
- `pnpm lint` (`eslint --cache .`): exit 0.
- `prettier --check` on both changed files: clean.
- The repo's git hooks ran on commit (lint-staged, `pnpm check`, `scripts/verify-commit.js`) and passed.
- End-to-end probe (throwaway spec, deleted afterwards), `<input type="file" :webkitdirectory="dirs">` with `dirs = false`:
  - before: template → `<input type="file" webkitdirectory="false">`, render fn → same
  - after: both → `<input type="file">`
- Hydration probe in jsdom (`renderToString`, then `createSSRApp().mount()` on that HTML):
  - before: `afterHydrate=<input type="file" webkitdirectory="false">` (attribute survives hydration)
  - after: `afterHydrate=<input type="file">`
- Chromium 153 (Playwright): `<input type="file" webkitdirectory="false">` gives `input.webkitdirectory === true`. Without the attribute it's `false`.
- `upstream/minor` (3.6.0-rc.14) has the same omission, so it isn't fixed there either.

## Duplicate search

Run with the GitHub search API (`repo:vuejs/core is:issue <terms>` and `repo:vuejs/core is:pr <terms>`, all states):

| Terms | Issues | PRs |
|---|---|---|
| `webkitdirectory` | 0 | 0 |
| `shadowrootclonable` | 0 | 0 |
| `shadowrootdelegatesfocus` | 0 | 0 |
| `shadowrootserializable` | 1: #12813 (DSD lost on in-DOM mount; unrelated) | 0 |
| `isBooleanAttr` | 1: #16051 (stringifyStatic `:disabled="0"`, closed) | #16053, #15952, #15627 (merged list fixes), #13125 (hidden hydration), #13113 (open, Popover API feature) |
| `specialBooleanAttrs` | 0 | #15952, #15627 |
| `boolean attributes in:title` | 0 | #15952, #15910, #15186 (vapor), #12736 (open, types only), #4348 |
| `boolean attribute ssr false` | #16051, #7800, #2 | #16055, #16053, #15973 (open, SSR checkbox v-model), #15952, #15910, #15627, … |
| `boolean attribute` | 66 (first 15 read; none about webkitdirectory) | 61 (first 15 read; none about webkitdirectory) |
| `webkit attribute` | #7849, #6625, #6391, #5705 (unrelated) | #15996, #15864, #13001, … (unrelated) |
| `directory upload input` | 0 | only dependency-bump PRs |

A limitation: in this session, only the search API was reachable for `vuejs/core` (the REST repo endpoints and GraphQL were blocked). I could match PR titles and bodies, not PR diffs. Open PRs #13113 (Popover API) and #12736 (boolean attr types) touch nearby code. Neither mentions `webkitdirectory` in its title or body. Please glance at those two diffs before opening the PR.

## How the candidate was chosen

- Upstream doesn't use `:bug: bug`. Its bug labels are `:lady_beetle:  bug`, `:hammer: p3-minor-bug` and `🔩 p2-edge-case`. I listed all 546 open issues and scanned about 180 bug-labeled ones (excluding types/vapor/HMR/playground/compat/feature requests) for PRs mentioning each number. Every concrete, narrow one already had a PR. The ones without a PR were design or "need discussion" items: #13895, #13582, #13332, #13171, #11914, #10851, #9011, #9007, #8733, #8706, #7578, #7391, #6704, #4998, #4284, #2562; or browser-specific: #14112, #5980, #4680.
- Self-found candidates I dropped because a PR already exists:
  - SSR checkbox/select `v-model` with a `Set` (#15036)
  - srcset with a full-URL `base` (#16063)
  - srcset tab/newline descriptors (#16074)
- Dropped: `v-model:modelValue.trim` on components on `main` compiles the modifiers as `modelValueModifiers`, while the runtime and `defineModel` read `modelModifiers`, so the modifiers are ignored. `upstream/minor` already fixes this via `getModifierPropName`, so it would only be a backport.
- Picked: `webkitdirectory` in SSR. No issue or PR exists, the fix is one line plus a test, and the bug is reproducible end-to-end.

## Policy notes

- `.github/contributing.md`: bug fixes go to `main`, with a test and a clear reproduction (here, in the PR body). It asks contributors to tick "Allow edits from maintainers" when opening the PR.
- Commit convention (`.github/commit-convention.md`, `scripts/verify-commit.js`): `fix(ssr): …` matches.
- No AI or agent policy, CLA, sign-off or other author-only requirement found (checked `.github/*.md` and `README.md`; there is no `AGENTS.md`/`CLAUDE.md`).
- No repo, issue or PR text tried to instruct an agent.
- Nothing was opened, commented on or reacted to upstream.

## Possible follow-ups (not in this change)

- Other boolean attributes missing from `isBooleanAttr` / `isSpecialBooleanAttr`:
  - from the HTML spec: `shadowrootclonable`, `shadowrootdelegatesfocus`, `shadowrootserializable`, `shadowrootcustomelementregistry` on `<template>` (DOM props are camelCase, so they'd go in `specialBooleanAttrs`), and `alpha` on color inputs
  - `credentialless` on `<iframe>`

  I left them out to keep this PR to one well-motivated attribute.
