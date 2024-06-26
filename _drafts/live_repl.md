---
author: 'anc'
title: 'Live repling to infinity and beyond'
description: 'Everything you wanted (and not wanted) to know about Live REPLs'
---

# Introduction

If you are an experienced Clojure developer, I don't need to convince you of the beauty of REPL Driven Development.
<!-- FIXME: not great sentence -->
This blog article is not about that, I wanted to focus on live REPLs, where a live REPL  is just a REPL connected to a machine that's not your localhost.

<!-- TODO: move the links at the bottom, don't need them all here -->
If you are new to Clojure or REPLs there are lots of great resources, such as:
- [REPL Driven Development, Clojure's
Superpower](https://www.youtube.com/watch?v=gIoadGfm5T8)
- [On repl-driven programming](https://mikelevins.github.io/posts/2020-12-18-repl-driven/)
- [What makes a repl](https://ericnormand.me/podcast/what-makes-a-repl)
- [REPL community resources](https://clojure.org/guides/repl/annex_community_resources)

As part of this blog post, I've also created a small [project](https://github.com/andreacrotti/live-repl) to play around with the different types of REPLs and ways to connect to them, and I will refer to that for examples.

## What's in a REPL

<!-- TODO: is this a dictionary if it's wikipedia? -->
We could not start a blog post about Clojure without a dictionary definition.
From Wikipedia a REPL (*read-eval-print-loop*) is:

> a simple interactive computer programming environment that takes single user inputs, executes them, and returns the result to the user....

Most programming languages nowadays have REPLs, but not all REPLs are created equal.

Every REPL  will allow you to explore a language, defining and evaluating new functions for example.
The big difference between Lisp (or Erlang/Elixir) REPLs and ones found in other programming languages like Python/Ruby however, is that the former allows you to interact with a live system, while the latter doesn't.

For example, when you start iPython you can play around with Python, evaluate code interactively, but, you can't attach to a running Python process in production.
Being able to attach to a running process gives you effectively super powers, because it means that you can both explore the status of the living system and even change its behaviour without redeploying or even restarting the JVM.

For historical reasons there are a few different REPls available in Clojure, I  will just refer to this [comparison](https://nrepl.org/nrepl/1.0/alternatives.html) for that.
It's also interesting to note that even deciding what exactly is a REPL can be controversial, as mentioned in [this discussion](https://groups.google.com/g/clojure-dev/c/Dl3Stw5iRVA/m/D_Kcb-naOx4J).
For our purpose though a REPL is just a way to evaluate code remotely, and we'll just use `nrepl` for the examples, but you can see other examples of repls in [this namespace](https://github.com/AndreaCrotti/live-repl/blob/main/src/system.clj ).

## Simple implementation

`nrepl/nrepl {:mvn/version "1.1.0"}`

```clojure
(defmethod ig/init-key ::nrepl-server [_ {:keys [port]}]
  (println "Starting nrepl server on port " port)
  (nrepl/start-server :port port
                      :bind "0.0.0.0"))

```


## Use cases

###  1. Investigation

The best reason to use a live REPL is (in my opinion) to investigate problems by checking the state of the live system.
For example, a few weeks ago we had a problem in production where the application kept crashing (with OOMs).
From the JMX metrics it was clear that the problem was the that number of threads was continuously increasing, but it wasn't clear why.

All we had to do to find out the root cause of the problem was to get all names of the running threads from the live-repl with something like:

```clojure
(sort (map #(.getName %) (keys (Thread/getAllStackTraces))))
```

From the name of the threads it was clear that the culprit was related to a recent change, and was easy to fix.

### 2. Live experimentation

Sometimes it's quite hard to run a whole infrastructure locally, specially when your application depends on many other services that can't really be run locally.
It's always a good idea to invest as much as possible in your local environment, but when that's not enough a live repl (ideally connected to a dev environment), can help a lot.
Once you are happy with the changes, since most likely your normal IDE is connected to the remote REPL, you can just commit them and follow the normal deployment flow, being already pretty much certain that it will work fine once deployed.

### 3. Maintenance tasks

Most applications require a bunch of maintenance tasks, such as:
- creating new users
- change user permissions
- generate reports

In an ideal world these tasks have an internal UI that talks to the internal API, however sometimes (specially in the beginning of a project) all this plumbing might not be available.
So one easy way to accomplish these tasks is to just have functions you can call from a live repl.
This is not really a novelty idea, it's also very common in the Django/Ruby-on-Rails world for example.

### 4. Hotfix

Last but not least a really important way to use a live-repl is to apply a hot-fix to production.
All the previous use cases don't really need to change the behaviour of the application, however, sometimes you might want to do that.

You might have a bug in a production system, and know exactly how to fix it, but when every minute lost costs a lot of money, you still have this option.

It's also true when you are not 100% sure of the fix, but the system is down anyway, so at least you can try it out very quickly instead of going through the whole flow.
In the simplest possible scenario all you have to do is simply connect your IDE to the production REPL, re-evaluate the offending code with your fix, and everything is fixed.
It's not exactly as simple as that, but we'll look later in more detail at the limitations.
Keep in mind that hot reloading production code can be dangerous and should be done extremely rarely, but it can be a life saver when you really need it.

## Limitations

For a live REPL to work, you need the language to support it.
In many languages, a  REPL was an after thought, and sometimes you can't do everything from a REPL.
While in Clojure the REPL was always a first-class citizen, you still have to be aware of a few things that cause issues with hot reloading, and write your application accordingly.

Thankfully these tips also help with local development, and make your codebase nicer to work with.

### JVM interop

### Compile time

Any operation

### Resources

## Security

If you are allowing REPL access to your applications you need to keep in mind the potential danger, since anyone with REPL access to production can access anything that the production application can access.
From a live repl you could easily:
- break production
- delete data
- leak production data

And by default there would not be any trace of what happened, even changes to the codebase would not be noticed by anyone and only be reverted when the JVM restarts.

So if you do want to allow access to production you (at the very least):
- only allow access from a VPN (ideally the whole infrastructure is only accessible via VPN anyway)
- only allow access from a SSH tunnel (or use [TLS with nrepl for example](https://nrepl.org/nrepl/usage/tls.html))
  Another possible choice is to use [drawbridge](https://github.com/nrepl/drawbridge) and use HTTP Transport for nRepl.
- add auditing to make sure you can record who did what.
  You might need a custom nRepl middleware to store.

## Other tips

- use separate checkouts of the project for local development & live repling (or use [git worktree](https://git-scm.com/docs/git-worktree) to help with that)
- add banners in the live repl to make sure you are aware of the environment you are connected to.
  It's currently not exactly supported but might be in cider when [cider issue about dynamic banners](https://github.com/clojure-emacs/cider/issues/3599) is implemented.
  It's also easy to hack it together yourself by creating some more functions.
- make sure you don't run tests in production (maybe avoid shipping them entirely in the build?)
- disable the \`cider-auto-test-mode\`, you never know you might delete some real data

## Other resources

- [nrepl middleware](https://nrepl.org/nrepl/design/middleware.html)
- [socket prepl cookbook](https://oli.me.uk/clojure-socket-prepl-cookbook/)
- [how to use prepl](https://blog.jakubholy.net/how-to-use-clojure-1.10-prepl/)
- [tools namespace](https://github.com/clojure/tools.namespace)
- [nrepl usage](https://nrepl.org/nrepl/usage/tls.html)
- [clojure workflow reloaded with component](https://cognitect.com/blog/2013/06/04/clojure-workflow-reloaded)
- [live repl sample repository](https://github.com/andreacrotti/live-repl)
- [repl socket Clojure examples](https://en.wikibooks.org/wiki/Clojure_Programming/Examples/REPL_Socket)
- [repl Wikipedia definition](https://en.wikipedia.org/wiki/Read%E2%80%93eval%E2%80%93loop)
- [bad nrepl article](https://metaredux.com/posts/2023/03/26/bad-nrepl.html)
- [nrepl history](https://nrepl.org/nrepl/about/history.html)
- [list of nRepl clients](https://nrepl.org/nrepl/usage/clients.html)
- [git worktree](https://git-scm.com/docs/git-worktree)
- [server REPL architecture definition](https://archive.clojure.org/design-wiki/display/design/Socket%2BServer%2BREPL.html)
- [integrant](https://github.com/weavejester/integrant)
- [integrant-repl](https://github.com/weavejester/integrant-repl)
- [cider issue about dynamic banners](https://github.com/clojure-emacs/cider/issues/3599)
- [nrepl and clojurescript](https://docs.cider.mx/cider/cljs/overview.html#nrepl-and-clojurescript)
- [bencode format (used by nRepl)](https://en.wikipedia.org/wiki/Bencodehttps://en.wikipedia.org/wiki/Bencode)
- [add-lib capability](https://insideclojure.org/2018/05/04/add-lib/)
- [AOT compilation in clojure](https://clojure.org/reference/compilation)
- [enhancing your repl workflow](https://clojure.org/guides/repl/enhancing_your_repl_workflow)
- [reveal](https://vlaaad.github.io/reveal/)
- [flappy bird interactive programming demo](https://www.youtube.com/watch?v=KZjFVdU8VLI )
- [jupyter supported kernels](https://github.com/jupyter/jupyter/wiki/Jupyter-kernels)
