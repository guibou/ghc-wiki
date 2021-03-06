# Nonlinear type family instances considered dangerous



**Note**: This page is here for historical reasons. The implemented features are discussed on the [NewAxioms](new-axioms) page.



This page discusses problems and solutions that come up when thinking about type family instances with repeated variables on the left-hand side.


## The Problem



Consider the following:


```wiki
type family F a b
type instance F x   x = Int
type instance F [x] x = Bool

type family G
type instance G = [G]
```


Here, `G` is a nullary type family, but its nullariness is just for convenience -- no peculiarity of nullary type families is involved.



These declarations compile just fine in GHC 7.6.3 (with `-XUndecidableInstances`), and on the surface, this seems OK. After all, the two instances of `F` cannot unify. Thus, no usage site of `F` can be ambiguous, right? Wrong. Consider `F G G`. We might simplify this to `Int`, using the first instance, or we might first simplify to `F [G] G` and then to `Bool`. Yuck!



I (Richard/goldfire) have tried to use this inconsistency to cause a seg fault, but after a few hours, I was unable to do so. However, my inability to do so seems more closely related to the fact the type families are strict than anything more fundamental.



It's worth noting that `-XUndecidableInstances` is necessary to exploit this problem. However, we still want `-XUndecidableInstances` programs to be type-safe (as long as GHC terminates).


## General idea for the solution



We need to consider the two instances of `F` to be overlapping and inadmissible. There are a handful of ways to do this:


- (A) **when performing the overlap check between two instances, check a version of the instances where all variables are distinct**


We call the "version of the instance where all variables are distinct" the "linearized form" of the instance.
Using such a check, the two instances for `F` above indeed conflict, because we would compare `(F a b)` against `(F [c] d)`, where a,b,c,d are the fresh distinct variables.



This can break existing code. But, a medium-intensity search did not find *any* uses of nonlinear (i.e. with a repeated variable) family instances in existing code, so I think we should be OK. However, a change needs to be made to be confident that nonlinear axioms and undecidable instances do not introduce a type-soundness hole into GHC.



(Interestingly, proofs of the soundness of the existing system have been published. For example, see [
here](https://www.microsoft.com/en-us/research/wp-content/uploads/2007/01/tldi22-sulzmann-with-appendix.pdf) and [
here](http://www.cis.upenn.edu/~stevez/papers/WVPJZ11.pdf). These proofs are not  incorrect, but they  don't allow both undecidable instances and nonlinear family instances.)



Conor's alternative general idea:


- (B) **when performing the overlap check, allow two types to overlap even if only infinite types would witness the overlap**


The idea here is that our root problem comes from a combination of nonlinearity and infinite types. So, simply make the overlap checker consider the possibility of types that contain themselves (the only possible form of infinite types).
(B) is a bit more permissive than (A).  For example


```wiki
  type instance Good x   x    = blah
  type instance Good Int Bool = boo
```


These would be considered overlapping by (A), but accepted as non-overlapping by (B).  And indeed these two are fine (ie cannot give rise to unsoundness).



Intuition: under (B) the only way that two types don't overlap is because of a definite conflict (eg `Int` \~ `Bool`). 



How to implement (B)? It may be as easy as omitting the occurs check during unification. The way GHC detects overlap is by trying to unify the left-hand sides of two instances. If they can unify, then they overlap. Currently (GHC 7.6.3), unification fails for the two instances for `F` above, so they don't overlap. The unification fails because of the so-called "occurs check": when we are unifying a type variable (say, `x`) against a non-type-variable, (say `[x]`), we check to make sure that the type variable does not appear anywhere within the non-type-variable. (If the variable does appear in the other type, then the substitution has no fixpoint.) We could simply turn this check off, in this particular case. The net effect is to consider the possibility of infinite types.



But, it just might not be so easy. What, exactly, should the unification algorithm do when the occurs check notes an occurrence in this case? Normally, the algorithm builds a substitution, but that would be impossible here. Yet, we need to record *something* about what we've discovered. For example, take this:


```wiki
type family NotSure x y
type instance NotSure x x = Int
type instance NotSure [x] (Maybe x) = Bool
```


It seems that these do not overlap, even in the presence of infinite types. But, a naive omission of the occurs check might say that these types do overlap. Getting this case wrong isn't terrible, but it would be nice to get it right. (Note that we would end up too conservative, which is OK in this case.)


## Problem: coincident overlap



In the official release of GHC, it is permitted to write something like this (see [manual](http://www.haskell.org/ghc/docs/latest/html/users_guide/type-families.html#type-family-overlap)), and the extensive discussion on [\#4259](http://gitlabghc.nibbler/ghc/ghc/issues/4259):


```wiki
type instance F Int = Int
type instance F a   = a
```


These instances surely overlap, but in the case when they do, the right-hand sides coincide. We call this **coincident overlap**.



However, the above proposal of linearizing the LHS before checking overlap (plan (A)) makes a nonsense of exploiting coincident overlap, because when we freshen the LHS we no longer bind the variables in the RHS. **So the proposal abandons support for coincident overlap between standalone type instances**.  (It's worth noting, though, that there is already no support for coincident overlap between branched type instances, or between a singleton type instance and a branched one; it is currently only supported between singleton type instances.)



Plan (B) above has a better relationship with coincident overlap, but not quite a rosy one: in the event that an infinite type is needed to show the overlap, we don't have a well-defined substitution to apply to the RHS. It is conceivable to allow coincident overlap only when the unification algorithm produces a bona fide substitution.


## Concrete Proposal


- Use plan (B) to check for overlap. Thus, our two problematic instances of `F`, at the top, will conflict.

- Continue to allow coincident overlap in standalone instances.

- Allow coincident overlap within branched instances. See [here](new-axioms/coincident-overlap) for more information.

- Optional: Change the syntax for branched family instances, as described [here](new-axioms/closed-type-families).

## Alternative (Non-)Solution



One way we (Simon, Dimitrios, and Richard) considered proceeding was to prohibit nonlinear unbranched instances entirely. Unfortunately, that doesn't work. Consider this:


```wiki
type family H (x :: [k]) :: *
type instance H '[] = Bool
```


Innocent enough, it seems. However, that instance expands to `type instance H k ('[] k) = Bool` internally. And that expansion contains a repeated variable! Yuck. We Thought Hard about this and came up with various proposals to fix it, but we weren't convinced that any of them were any good. So, we concluded to allow nonlinear unbranched instances, but we linearize them when checking overlap. This may surprise some users, but we will put in a helpful error message in this case.


