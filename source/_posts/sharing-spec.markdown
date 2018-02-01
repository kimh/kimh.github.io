---
layout: post
title: "Why I prefer using double colon in specs"
date: 2017-12-27 09:20
comments: true
categories:
---

# :: Double Colon way

When you define all specs in one namespace, things are very straightforward. There are two ways to do this. One is using namespaced keywords and the other is double colons.

```clojure
(ns wisky-spec.spec
  (:require [clojure.spec.alpha :as s]))

(s/def ::wisky (s/keys :req [::name ::age])))
```

As you may know, double colong will be auto-resolved the current namespace you are in. So, if you define `::wisky` in `wisky-spec.spec` ns, then it will be resolved to `:wisky-spec.spec/wisky`.

This is annoying to deal with when you want to use it from different namespace. Say, you want to use wisky spec from `liquor-shop` ns, then it looks like this:


```clojure
(ns liquor-shop.core
  (:require [clojure.spec.alpha :as s]
            [wisky-spec.spec]))

(defn wisky? [wisky-map]
  (s/valid? :wisky-spec.spec/wisky wisky-map))
```

Note that you need to write `wisky-spec.spec/wisky` all the time if you want to use wisky spec. This is not handy but Clojure provides a solution to this. You can alias your symbols.

```clojure
(ns liquor-shop.core
  (:require [clojure.spec.alpha :as s]
            [wisky-spec.spec :as ws]))

(defn wisky? [wisky-map]
  (s/valid? :ws/wisky wisky-map))
```

Now you can use `:ws/wisky` instead of long `:wisky-spec.spec/wisky`. Much better. Let's re-write our wisky spec.

```clojure
(ns wisky-spec.spec
  (:require [clojure.spec.alpha :as s]))

(s/def :liquor/wisky (s/keys :req [::name ::age])))
```

# Namespace quaflied keywords

Instead of using `::`, now we appended `liquor` namespace infront of `:wisky` symbol. Let's use the wisky spec.

```clojure
(ns liquor-shop.core
  (:require [clojure.spec.alpha :as s]
            [wisky-spec.spec]))

(defn wisky? [wisky-map]
  (s/valid? :liquor/wisky wisky-map))
```

It depends on your preferences, but using namespace qualified symbol may look better because it gives you a context. In this example, our wisky spec is for selling in liquor shop.

# Problems with namespaced qualfied keywords

## Doesn't tell you where it comes from

However, I see a few problems with namespaced qualified keywords. The first problem is it doesn't tell you where it comes from.

Have you noticed that in the earlier example, `:liquor/wisky` is automatically accessible inside `liquor-shop.core` ? It sounds like you require `wisky-spec.spec` and `:liquor/wisky` speci is globally accessible. This is indeed what happened.

Clojure maintains specs in a global registry. We can confirm this from REPL.

```clojure
(:luquor/wisky (s/registry))
nil

(require 'wisky-spec.spec')
(:luquor/wisky (s/registry))
;; return a spec
```

First time I noticed this, this looked a bit awakward to me. When dealing with Clojure namespace, I always do require other namespace and use it with alias. However in the example of spec, you just need to require the spec namespace and all specs defined are available to you. This reminds me back in the day when I was using Ruby or Javascript.

Because specs are global, soemtimes it's not obvious where the spec comes from. Consider the following example:

```clojure
(ns bar.core
  (:require [distriller.spec
             distributor.spec]))

(defn wisky? [wisky-map]
  (s/valid? :liquor/wisky wisky-map))
```

In this example, you can't tell whether :liquor/wisky comes from `distriller.spec` or `distributor.spec`, so you actually need to go in these namespaces to find out where it's defined.

## Name conflict

Another problem is that it's possible to have name conflict.

Let's say Alice defines the follwing spec.

```clojure
(s/def :liquor/wisky (s/keys :req [::name ::age])))
```

And uses it in `luquor.core` namespace

```clojure
(ns liquor-shop.core
  (:require bar.core))

(defn wisky? [wisky-map]
  (s/valid? :liquor/wisky wisky-map))
```

Later Bob is working in `bar.core` namespace. However, he didn't know that there is already wisky spec exists, so he defins by his own.

```clojure
(ns distributor.core)

(s/def :liquor/wisky (s/keys :req [::name ::age ::strength]))
(defn wisky? [wisky-map]
  (s/valid? :liquor/wisky wisky-map))
```

Because Bob thinks stregnth is a part of important quality of wisky, he requires `::strength` field in wisky map. Because spec is globally defined, the two :liquor/wisky spec conflicts now. When you require `liquor-shop.core` and `distributor.core` namespaces from another namespace, say, `grocery-store.core` and use `:liquro/wisky` spec, which one will be used? It's hard to tell!

## Problems with double collons

For these problems, I prefer to use :: when defining spec. However, double colon isn't problem free, either. When defining a spec fo map, things look a bit weird.

Let's reuse the spec that we've seen earlier.

```clojure
(ns wisky-spec.spec
  (:require [clojure.spec.alpha :as s]))

(s/def ::wisky (s/keys :req [:wisky/name :wisky/age])))
(defn wisky? [wisky-map]
  (s/valid? ::wisky wisky-map))
```

To make a valid wisky, you need to do something liek this.

```
(require '[wisky-spec.spec :as ws])
(wisky? {:wisky/name "laphroig" :wisky/age 15})
```

I feel a bit weird that the consumer of the wisky spee always need to supply `:wisky` infront of each fields. However this can be workaround by using `req-un` when defining spec.

```clojure
(ns wisky-spec.spec
  (:require [clojure.spec.alpha :as s]))

(s/def ::wisky (s/keys :req-un [::name ::age])))
(defn wisky? [wisky-map]
  (s/valid? ::wisky wisky-map))
```

When `req-un` (require unqualified) is used, spec only checkes the right most symbol name and skips namespace. So you now make a valid wisky with general symbols that you get used to.

```
(wisky? {:name "laphroig" :age 15})
```

## Push from Clojure community

We've seen `req-un` is handy but this is not the direction that Clojure community is heading to. Clojure communities encourages the more use of namespace qualifed keywords, not in specs, but everywhere. The argument is that the use of namespaced qualfied keyword gives more context and making maps self-descriptibe.

Sure, that sounds a very good arugment but, in practice, I found it's hard to enforce.

Say Alice builds a new product called "BeaverCI" that does continuous integration and implementing permission things, so I defined the spec like this.

```clojure
(s/def ::permission
    (s/keys :req [:beeaverci/user-id :permission/scope]))
```

In BeaverCI, it's very common to call their product simply as "Beaver" omitting CI part because it's shorter.

Later Bob builds a build system that hooks up permission system made by Alice.

```clojure
(s/def ::user
  (s/keys :req [:beeaver/user-id]))

(defn can-build? [permission user]
  (if (s/valid ::permission)

)
```

And try to use it.

```clojure
(can-build? {:beeaver/user-id: "12345"})
```

This breaks in the middle because permission spec requires `:beeaverci/user-id`, not `:beeaver/user-id`. However, they mean the same things but how to express it will be different among people. However, specs requires the name to be preceise. `:beeaverci/user-id` and `:beeaver/user-id` are completely different thing.

Of course, the problem wont' be solved if you go with simply `user-id` because other person coming from Ruby background may call it `useri_id` and it also breaks. However, by adding namespace, you are going to make this problem harder to deal with becayse you need to care both namespace part and actual symbol name part.
