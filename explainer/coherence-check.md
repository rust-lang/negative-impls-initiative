# Coherence check

Coherence works in two parts. The first is the *orphan* check, which is unaffected by this RFC: the orphan check is the one that ensures that you only implement foreign traits for types that are local to your crate.

The second part of the coherence check is the *overlap* check. The principle here is that if you have two impls `Impl1` and `Impl2` for some trait `Trait`, we wish to ensure that `Impl1` and `Impl2` cannot apply to the same set of inputs `P0: Trait<P1...Pn>`.

## Negative of a where clause

We define the negative of a where clause `Negative(WC)` as a partial function that converts `T: Trait` to `T: !Trait`:

* `Negative(P0: Trait<P1...Pn>) = Some(P0: !Trait<P1...Pn>)`
* `Negative(for<'a..'z> P0: Trait<P1...Pn>) = Some(for<'a..'z> P0: !Trait<P1...Pn>)`
* `Negative(WC) = None`, otherwise

## New disjointness rules

The coherence check is extended with a new rule for determining when `Impl1` and `Impl2` are disjoint (the rule is applied twice for any pair `(A, B)` of impl, once with `A` as `Impl1` and once with `B` as `Impl1`). These rules are alternatives to the existing rule, which means that *any program that compiles today continues to compile*, but we will also consider *new* programs as passing the coherence check.

In plain English, the idea is that we can consider `Impl1` and `Impl2` to be disjoint if whenever `Impl1` applies, some where clause in `Impl2` cannot hold (i.e., we can prove the negative of it) (and vice versa). 

* `Disjoint(Impl1, Impl2)` holds if, for some substitution S and some where clause WC on Impl2:
    * `S(TraitRef(Impl2)) = TraitRef(Impl1)` -- there is some instantiation S of the Impl2 generics such that Impl1 and Impl2 apply to the same types
    * `Negative(S(WC)) is provable, given the environment of Impl1` -- we can prove the negative version of some where clase from Impl2

In operational terms:

* `Disjoint(Impl1, Impl2)` if...
    * Instantiate `Impl1` with placeholders
    * Instantiate `Impl2` with inference variables
    * Equate all type parameters from `Impl1` and `Impl2`
    * There exists some where clause `WC` on `Impl2` where `negative(WC)` holds.

