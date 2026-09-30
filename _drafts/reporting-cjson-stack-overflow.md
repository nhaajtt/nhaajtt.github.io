---
layout: post
title: "Reporting a stack overflow in cJSON"
tags: [fuzzing, cjson, disclosure]
---

A fuzzing harness against [cJSON](https://github.com/DaveGamble/cJSON) turned up a stack overflow, and this
note is about what happened after finding it, not the finding itself, which is written up in full in
[fuzzing-lab](https://github.com/nhaajtt/fuzzing-lab).

## What the harness found

A JSON value made of 755 consecutive `[` characters, no closing brackets, crashes a build of cJSON compiled
with AddressSanitizer and no optimization. cJSON already ships a recursion-depth guard
(`CJSON_NESTING_LIMIT`, 1000) meant to stop exactly this kind of input. The guard counts recursive calls, not
stack bytes, and under `-O0` plus ASan's redzones each call's stack frame is large enough that the process
stack runs out before the counter reaches 1000.

## Checking it was worth reporting

The first question before writing anything up is whether the bug matters outside the exact build that
found it. The same inputs, at 780, 1000, 1500, 5000 and 50,000 levels of nesting, were run again against an
optimized build with no sanitizers: every one was rejected cleanly, no crash. So a normal release build of
cJSON is not affected. That changes what the report should say. This is not "cJSON has a stack overflow
bug," it is "the documented nesting limit does not hold under the sanitizer builds people use to find bugs
in the first place," which is a narrower and more honest claim.

## Verifying the constant against the live source

Before filing anything, it is worth checking the claim against the actual current code rather than trusting
a local checkout that might be stale. `CJSON_NESTING_LIMIT` is still defined as 1000 in `cJSON.h` on the
`master` branch at the time of writing, so the finding lines up with the real, current library.

## Filing it

cJSON's own contribution guidelines do not treat this class of report as a security disclosure requiring a
private channel, and the finding itself is not exploitable in a normal build, so it went to the public issue
tracker rather than a private security advisory: [DaveGamble/cJSON#1093](https://github.com/DaveGamble/cJSON/issues/1093).
The report includes the minimized input, the ASan output, the root cause, and the release-build test that
shows it not reproducing there, so a maintainer can judge severity without re-deriving any of it.

## What is left

The issue is open with no response yet. If that changes, this note and the fuzzing-lab write-up will be
updated with the outcome. Filing an honestly-scoped report and waiting is the point: the value here is in
telling a maintainer a real, verified gap between documentation and behavior, not in inflating a build
artifact into something it is not.
