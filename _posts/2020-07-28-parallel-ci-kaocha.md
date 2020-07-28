---
layout: post
title: Parallel tests with Kaocha and circleci
subtitle: Using Kaocha hand circleci to improve your tests speed
tags: [clojure, clojurescript, code, kaocha, circleci]
comments: true
---

## How to speed up your build

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

## Kaocha and parallel tests

I won't show examples of options `1` and `2` and dive directly into option `3`, showing how to accomplish
circleci parallelism using [Kaocha](https://github.com/lambdaisland/kaocha) as test runner.

To accomplish that you will need to:
- generate a test plan with `kaocha.repl/test-plan`
- use the `circleci` command (which will be available automatically in CI) to split the tests
- run the tests with `kaocha.repl/run`
- add the circleci configuration setting up the parallelism

Let's look step by step at all the code and configuration you need.

First create a namespace like `your-project.ci-tests`, which will be the entry point for your tests:

```clojure
(ns your-project.ci-tests
  "Run tests in Circleci, splitting by timings with the circleci command"
  (:require [clojure.string :as s]
            [clojure.java.shell :as sh]
            [clojure.walk :as w]
            [kaocha.repl :as kr])
  (:gen-class))

```

Now we need to have a function that extracts the all the namespaces, like:

```clojure
(def found (atom []))

(defn extract-namespaces
  []
  (reset! found [])
  (w/postwalk
   (fn [t]
     (when (= :kaocha.type/ns (:kaocha.testable/type t))
       (swap! found conj t))
     t)
   (kr/test-plan))
  (->> @found
       (map :kaocha.testable/id)
       sort))
```

We need to use `postwalk` and an atom because the test plan is a
deeply nested data structure, (but if you have a better way to extract
the namespaces from the test plan please let me know in the comments
below).

Now we need to use the `circleci` command (which will be automatically available)
to split the tests by timing, which should overall give you the better results.

To do that we just shell out to `circleci` passing the namespaces in the standard input,
and it will return a different list of namespaces to each process automatically.

```clojure
(defn split-tests
  []
  (-> (sh/sh "circleci"
             "tests"
             "split"
             "--split-by=timings"
             :in (s/join "\n" (map name (extract-namespaces))))
      :out
      (s/split #"\n")))
```

 Now the last part is the main function, where we just use `kaocha.repl/run` to
 execute the tests, and exit with a status code = 1 if any test failed.

 ```clojure

(defn -main
  [& _args]
  (let [result (apply kr/run
                      (conj (vec (split-tests))
                            {:plugins                             [:kaocha.plugin/junit-xml]
                             :kaocha.plugin.junit-xml/target-file "test-results/kaocha/results.xml"}))]
    (System/exit (if (or (pos? (:kaocha.result/error result))
                         (pos? (:kaocha.result/fail result)))
                   1
                   0))))

 ```

A correct `System/exit` call is important otherwise test failures will
not trigger a failed build in circleci.
And we also set the `junit-xml` plugin to be able to generate the test summary in case something goes wrong.

Now the last thing you need is the correct `circleci` configuration, where the only different bit is the
`parallelism` setting, in this example set to 4.

```yaml
  run-tests:
    parallelism: 4
    docker:
      - image: <your-image>
    working_directory: ~/<your-project>
    steps:
      - checkout
      - run: lein run -m your-project.ci-tests
      - store_test_results:
          path: test-results
```

And that's all, your tests will now run in 4 separate processes (so 4 times faster in the best possible scenario), while
still generating a single report.
