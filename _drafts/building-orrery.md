---
layout: post
title: "Building Orrery: a GitHub profile drawn as a solar system"
tags: [tools, github-api, javascript]
---

Orrery turns any GitHub profile into a small solar system. Public repositories are planets, recent pushes pull them closer to the sun, and a second username puts two systems side by side. It ships as a website and as a GitHub Action that renders the same idea as an SVG for a profile README. This note covers the decisions that were not obvious.

The code is at [github.com/nhaajtt/orrery](https://github.com/nhaajtt/orrery) and the site is at [orbit-gh.vercel.app](https://orbit-gh.vercel.app).

## The data is smaller than it looks

Everything comes from three public REST endpoints: the user, the user's repositories and the user's public events. There is no private data anywhere in the path, and I checked that against a profile with private repositories: the payload contains only what `/users/:login/repos` returns for a stranger.

Two limits shaped the design.

**Rate limits.** Unauthenticated calls get 60 requests an hour per IP. The site calls a small serverless function that adds an optional token with no scopes (5,000 an hour) and caches responses at the edge for five minutes. If the function is missing, the browser falls back to calling the API directly, so the same code works on plain static hosting.

**The events window.** `/users/:login/events/public` returns at most the latest 300 events. For a busy account that can cover only a few days, so a "last 90 days" chart quietly lies: the days before the window look like days with no activity. The fix is to treat them as unknown instead of zero:

```js
const capped = events.length >= 300;
const windowStart = capped ? dayKey(oldest) : null;
days.push({ d, n: windowStart && d < windowStart ? null : counts.get(d) || 0 });
```

Unknown days are drawn with a dashed outline, and when two profiles are compared both are measured over the shorter window. Small detail, but it is the difference between a chart and a decoration.

## Languages are symbols, not colors

The first version colored planets by language. It looked like every other dashboard. The redesign borrowed from technical drawings, where materials are shown with hatching rather than color, so each language gets a fill symbol: solid, ring, hatch in either direction, cross, lines, dots, half, target. TypeScript is solid, JavaScript is hatched up, Python is dotted, and so on, with a stable hash for languages that are not in the table.

The mapping lives in one small module that three renderers share: the canvas on the site, the SVG in the Action and the legends in the page. That is what keeps a planet and its legend entry identical without copying code.

## The Action is a Node script and a YAML file

The Action is a composite action that runs one script:

```yaml
runs:
  using: composite
  steps:
    - id: render
      shell: bash
      run: node "${{ github.action_path }}/action/render.mjs"
```

The script builds the SVG as a string. Orbits are ellipses and the planets move with `animateMotion` along an elliptical path, with a negative `begin` to give each planet its own starting phase. That keeps the image animated inside a README, where JavaScript is not available. Colors switch with `prefers-color-scheme` inside the SVG.

One limitation I did not expect: GitHub does not load web fonts inside an image, so the SVG uses a system fallback stack and looks slightly different from the site.

## Testing in a real browser

I drove headless Chrome over the DevTools protocol and took screenshots at five sizes in light and dark, measuring horizontal overflow at each. That found two real bugs that reading the code would not have. First, a `display: grid` rule was overriding the `hidden` attribute, so a form that should have disappeared stayed on the page. Second, the pinned scroll tour zoomed to a repository that was not in the list of planets, so the camera did nothing for popular repos that had not been pushed recently.

## What I would change

The site reads only what GitHub exposes publicly, so a profile with mostly private work looks empty. That is correct, but a version that lets an owner opt in with their own token could show much more. For now the honest picture is the useful one.
