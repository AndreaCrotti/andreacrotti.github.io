---
layout: post
title: Repl driven e2e tests
subtitle: Use Etaoin to write your e2e tests
tags: [clojure, code, e2e, testing]
comments: true
---

I've been using the great [https://github.com/igrishaev/etaoin](Etaoin library) for a while, but there kare a few things missing from from it to be able to really be productive with it.

When you are testing a full stack Clojure project, these are the things you will need to do before any Etaoin test can run.

- start your jetty server
- compile your CLJS to JS (for example with figwheel-main)
- set up a browser driver, which will be used by the test
- (optionally) compile SASS to CSS
- (optionally) do some database set up

You could have all these steps running in a single big test fixture, but they take a significant amount of time, making it quite painful when running in the REPL and very slow on CI.

So we just need to be a bit smarter, doing the minimum amount of work necessary.

