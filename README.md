# Laws

*Algebraic laws a la carte!*

**Note: This is not an officially supported Google product.**

Laws is a small library of macros for the Carp programming language that enable
you to define algebraic laws for interfaces and check whether or not
implementations satisfy those laws on demand. To illustrate, let's take a look
at an interface for the classic `fmap` function:

```clojure
(definterface fmap (Fn [&(Fn [a] b) (f a)] (f b)))
```

This interface correctly expresses the typing requirements for valid
implementations of `fmap`, but *lawful* implementations of `fmap` also need to
satisfy two algebraic laws (the functor laws):

- identity: implementations must preserve identity: `fmap id = id`
- composition: implementations must preserve function composition: `fmap (f . g)  ==  fmap f . fmap g`

We can use `laws` to augment our interface with these constraints:

```
(with-laws (definterface fmap (Fn [&(Fn [a] b) (f a)] (f b)))
           [f g functor]
           [identity (= (fmap &id functor) (id functor))
            composition (= (fmap &(compose f g) functor) (fmap &f (fmap &g functor)))])
```

To define laws, we provide an interface, a list of variables we can use in the
algebraic equations associated with laws, and a list of laws, each with a name
and algebraic equation that intermixes variables and plain old Carp symbols.

After our call to `with-laws` our fmap interface will be defined and we'll be
able to check whether or not implementations satisfy the identity and
composition laws using the `lawful?` macro. Here's an example using both a
lawful and unlawful implementation of `fmap`.

```clojure
(defmodule Maybe
  ;; our lawful implementation; an honest to god fmap
  (defn map [f maybe]
   (match maybe
          (Maybe.Nothing) (Maybe.Nothing)
          (Maybe.Just x) (Maybe.Just (~f x))))
   (implements fmap map)
  ;; an fmap by type and appearance only! unlawful!
 (defn fake-map [f maybe]
  (match maybe
         (Maybe.Nothing) (Maybe.Nothing)
         (Maybe.Just x) (do (ignore (~f x))
                            (the (Maybe Int) (Maybe.Nothing)))))
  (implements fmap fake-map)
)

;; let's check which implementations satisfy the fmap laws
(defn main []
  (do
    (if (lawful? fmap Maybe.map [inc dec (Maybe.Just 1)])
        (println* "lawful!")
        (println* "unlawful!")))
    (if (lawful? fmap Maybe.fake-map [inc dec (Maybe.Just 1)])
        (println* "lawful!")
        (println* "unlawful!"))
  )
)
```

The prior code listing will print `lawful!` followed by `unlawful!`. The
`Maybe.map` implementation satisfies both the identity and composition laws,
while the `Maybe.fake-map` implementation fails to satisfy both laws.

This code sample also illustrates some important facets of the library's macros:

1. Law satisfaction checking is "on-demand"

   Laws, unfortunately, doesn't provide any means of performing exhaustive
   checks against implementations. Instead, every check is performed *precisely*
   for the variable substitutions provided at point of use. In other words:

   `(lawful? fmap Maybe.map [inc dec (Maybe.Just 1)])`

   Precisely and *only* checks that `Maybe.map` is a lawful implementation of
   `fmap` when the variables in the equations given for the identity and
   composition laws are replaced with the values `inc`, `dec`, and `Maybe.Just 1`.

   While you cannot use laws for checking that some implementation satisfies
   algebraic laws for *all* possible inputs, this "a la carte" approach makes up
   for its lack of universality with versatility. Enforcing that some
   implementation is lawful is entirely up to you, and you can easily and flexibly
   intermix code that checks for satisfaction of laws with code that doesn't. The
   macros provided by the library won't ever prevent you from declaring unlawful
   implementations or from using them. This enables library authors to define
   sharp boundaries around paths that require extra rigor without hamstringing
   library users.

2. Law satisfaction checking incurs runtime costs

   Another unfortunate aspect of the `lawful?` macro is that its checks are not
   free. Currently, it runs the algebraic equations associated with laws to
   determine their validity under certain inputs. So, there is a runtime cost
   associated with lawfulness checks.

## Usage

```
(load "laws.carp")
```

Further documentation on the `Laws` module macros can be found in the `docs/`
folder. `example.carp` contains a small program that illustrate how to use the
library.
