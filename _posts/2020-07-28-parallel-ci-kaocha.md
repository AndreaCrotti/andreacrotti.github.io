---
layout: post
title: Parallel tests with Kaocha and circleci
subtitle: Using Kaocha hand circleci to improve your tests speed
tags: [clojure, clojurescript, code]
comments: true
---

Having a faster CI feedback loop is always a good thing, and there at least three ways to make your tests faster:
1. Split them in multiple jobs:
   This definitively makes sense when you have some clear distinctions, like unit/integration/e2e tests that also
   require different configuration or infrastructure to run.
   It requires writing different jobs and most likely one of your jobs will still take a lot longer than the others,
   so the overall time can still be quite long.

2. use a parallel test runner:
   Another option is to use a test runner that supports parallelism directly
   like [Eftest](https://github.com/weavejester/eftest).
   The advantage of this is that you can also use it locally, but the disadvantage
   is that your tests might need to be adapted to be run in parallel, which is sometimes
   tricky specially if you have lots of side effects.

3. use the parallelism features provided by circleci:
   You can also just use [circleci parallelism features](https://circleci.com/docs/2.0/parallelism-faster-jobs/)
   instead, so still keeping a single job but having it split in multiple processes.
   This is probably the easiest option and most likely the one that can get you the biggest speed up.

I won't show examples of options `1` and `2` and dive directly into option `3`, showing how to accomplish
circleci parallelism using [Kaocha](https://github.com/lambdaisland/kaocha) as test runner.

To accomplish that you will need to:
- generate a test plan with `kaocha.repl/test-plan`
- use the `circleci` command (which will be available automatically in CI) to split the tests
- run the tests with `kaocha.repl/run`
- (optionally) add an alias in your `project.clj`/`deps.edn` files
- add the circleci configuration setting up the parallelism
