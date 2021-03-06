# Notes on the implementation of Supercompilation in GHC



This page is very much a draft and may be incorrect in places. Please fix problems that you spot.


## Current Bugs


- Naivrec: Double runtime

## Shortcomings of the prototype


- Uses eager substitution
- Homeomorphic embedding for types?  Currently all types are regarded as equal (like literals).  Decision: leave it this way for now.
- Msg does not respect alpha-equivalence.  If we match lambda against lambdas, and the binders differ, we say "different".  Decision: deal with alpha-equiv in msg when we have the new alg working.

  - Adding constraint info

    - case (x\>y)of { ....case (x\>y) of ... }
    - Extending this to specialised functions themselves.
- case var subst
- strictness annotations

## Insights


- The whistle is THE bad guy when it comes to performance; we spend 35% of our time on testing. It is trivial to run in parallel. 

- We lambda lift to avoid re-specialising the same local function in two different branches: letrec g = .. in (f1 (g x), f2 (g x))

- Looking for instances, instead of renamings, has implications with the global store. Do we want to fold append (append xs' ys) zs against append (append xs ys) zs (in rho) or append ys zs (in store).

- Applications are not saturated in Core, there's `eta::GHC.Prim.State# GHC.Prim.RealWorld` roughly everywhere.

- The whistle blows on several expressions sometimes. We need to sort them. Example: 

  ```wiki
  case GHC.Num.- @ GHC.Types.Int GHC.Num.$fNumInt (GHC.Num.+ @ GHC.Types.Int GHC.Num.$fNumInt i (Main.check l)) (Main.check r) of _ {
    GHC.Types.I# y [ALWAYS Once Nothing] -> GHC.Types.I# (GHC.Prim.-# (GHC.Prim.+# x 0) y)
  }
  ```

  against both of these:

  ```wiki
  case Main.check r of _ { 
    GHC.Types.I# y [ALWAYS Once Nothing] -> GHC.Types.I# (GHC.Prim.-# (GHC.Prim.+# x 0) y)
  }

  GHC.Num.- @ GHC.Types.Int GHC.Num.$fNumInt (GHC.Num.+ @ GHC.Types.Int GHC.Num.$fNumInt i (Main.check l)) (Main.check r))
  ```

  The latter is better to generalise against. How do we capture this? Perhaps by sorting on the length of free variables in rho.

## Open questions


- The new msg for different types. What should the outcome of:

  ```wiki
  msg (rev @ Int (xs::[Int]) (rev @ Bool (xs::[Bool]))
  ```

  be? We currently fail and split to `[rev @ Int/z]` and `z xs` instead.

- Rank-n-polymorphism. Given: `msg (f e) (f e')`, we return `f (z::typeof e)` and `[e/z]`. I'm not sure if it's possible to create an example where we should have taken z to have the same type as the type signature for the argument of f instead.

- You had a suggestion to convert the entire program to the zipper-representation before supercompiling.

- The supercompiler can after transformation leave deep identity functions in place, that will traverse the entire structure and re-assemble it.  If you supercompile `flip (flip t)` the resulting program will be `deepId t` where 

  ```wiki
  deepId (Leaf d)     = Leaf d
  deepId (Branch l r) = Branch (deepId l) (deepId r)
  ```

  Neil removed these in a post-pass as far as I understood, and he said it was a good idea.

- Using other constraints than equality. This means things like

  ```wiki
   	case (a>b) of ....(case a+1>b of ...) ...
  ```

  (This could save more transformation time than it spends if it manages to eliminate a lot of case-alternatives, and intuitively it doesn't feel prohibitively expensive for many constraint domains).

- Should R contexts include let-statements? Need to worry about name capture even more then.

- Should matching for renamings be modulo permutation of lets? (Performance vs code size)

- Consider `(\x xs. append x xs)`.  Do we inline append, and create a specialised copy?  (Of course, identical to the original definition.)

  - **Yes**.  At provided we don't create *multiple* specialised copies, we are effectively copying library code into the supercompiled program.  Then we can discard all libraries (provided we have all unfoldings).
  - **No**: then need to keep the libraries
    But it's not clear that we can *always* inline *everything*.  For example things with `unsafePerformIO`.

    - (Given no cycle in imports) Perhaps the things we can not inline should be put at the top level in the same module, and the old module discarded?

- Can we improve the homeomorphic embedding so that append xs xs is not embedded in append xs ys? **Done**

- [
  http://hackage.haskell.org/trac/ghc/ticket/2598](http://hackage.haskell.org/trac/ghc/ticket/2598)

## Current status



What next? **Implement the new algorithm.**


- Figure out arity for each top-level (lambda lifted) function, and only inline when it is saturated.  (Write notes in paper, explaining why this might be good.)  NB: linearity becomes simpler, because a variable cannot occur under a lambda.

  - Neil's msg idea; Will not help as much in practice since we are guaranteed to have the same head


Later


- Use idUnfolding and a bitmap for letrec/toplevel things instead of traversing the binds list.
- Using lazy substitutions
- Case-of-case duplication
- Post-pass to identify deepId
- Post-pass to undo redundant specialisation??
- Neil does "evaluation" before specialising, to expose more values to let, and maybe make lets into linear lets.  We don't. Yet.


Done


- Exposing all unfoldings:

  - Flag -fexpose-all-unfoldings (a cousin of -fomit-interface-pragmas) (default is off) to switch on the spit-out-all-unfoldings stuff.
  - Validate with flag off; then push.
- Add IO monad; 
- Faster representation for memo table; a finite map driven by the head function 
- Refined whistle-blowing test
- Write split in the R form.
- Write msg in the R form.  Still with eager substitution
- add logging (one line per specialisation start, and completion)
- Use a record for the memo table contents
- State monad and good logging info; Stole SimplMonad.
- Lambda lifting
- Add the "loop-breaker" info to interface files (and read it back in).
- Export unfoldings for recursive functions; does not validate: 

  - ds060: Overlapping pattern match?
  - ds061: Turns pattern-matches non-exhaustive
  - dsrun015: Foo.x not in scope
  - driver063: Exposes modules that were invisible earlier. 
  - print010: changes output from Integer to GHC.Integer.GMP.Internals.Integer 0 to GHC.Integer.GMP.Internals.S\# 0.
  - break026: No show instance


A substitution-based implementation exists, that transforms append, reverse with accumulating parameters, basic arithmetics and similar things. There are still bugs in the implementation, mainly name capture. It takes 8 seconds to transform the double append example, but there's still plenty of room for improvement with respect to performance.



The typed intermediate representation has caused some trouble, but nothing fundamental. 



Random thoughts about the prettyprinter:


- Let-statements: 

  - LclId is not really useful for me.
  - The empty list on the line below it is also wasting space.
  - The occurence information sometimes pushes the type signature over several lines. 
- Case-statements:

  - The occurence information pushes case branches over several lines.
  - case e of { tp1 tp2 tp3 tp4 -\> tp3 } prettyprints over 5 lines.
- Long types: Is the complete "GHC.Types.Char" necessary?
- Names: 

  - Why is it sometimes $dShow{v a1lm} and sometimes $dShow\_a1lm? The latter is easier to grep for.

## Performance


```wiki
COST CENTRE                    MODULE               %time %alloc  ticks     bytes

isHomemb                       Scp                   19.3   47.2    853 4763146466
peel                           Scp                    9.0    4.0    396 408786710
dive                           Scp                    5.3    8.2    233 832498569
thenSmpl                       SimplMonad             3.1    0.2    135  24371642
thenNat                        NCGMonad               1.9    0.2     85  16580813
>>=_aLo                        RegAlloc.Linear.State   1.8    0.5     79  50778216
mkLitString                    FastString             1.7    1.7     76 171934583
maybeInline                    Scp                    1.6    0.3     69  29715474
thenFC                         CgMonad                1.6    0.6     69  56081175
rhssOfAlts                     CoreSyn                1.5    2.1     68 208655884
inlinePerformIO                FastFunctions          1.4    0.0     64         0
varUnique                      Var                    1.1    1.0     48 100342146
renamings                      Scp                    0.8    1.8     36 185457589
collectArgs                    CoreSyn                0.8    2.3     36 228680316
insert_ele                     UniqFM                 0.7    1.1     30 109012316
```


Full unfoldings on stage2 and libraries:


```wiki
Total bytes: 28541166 bytes vs 55642519 bytes

Largest byte difference: ./libraries/haskeline/dist-install/build/System/Console/Haskeline/Vi.hi, with 887989 bytes
62885 bytes vs 950874 bytes.

Largest % difference: ./ghc/stage2/build/InteractiveUI.hi, with 6031.62723322067% 
10355 bytes vs 624575 bytes
```

## Diffferences from Supero



An attempted list at differences between [
Supero](http://community.haskell.org/~ndm/supero) and [
positive supercompilation for call-by-value](http://www.csee.ltu.se/~pj/papers/scp/popl09-scp.pdf) (abbreviated to pscp below). 



1) Call-by-value vs Call-by-n{ame,eed}. I am not completely clear on whether Supero preserves sharing. (Pscp does, otherwise the improvement theory is not usable and the proof doesn't go through. Previous discussions between Neil and Peter concluded that this is not sufficient - more expressions need to be inlined so we do not want to preserve all the sharing for performance reasons; Simon later pointed out that we must preserve sharing).



2) Termination criterion. Both use the homeomorphic embedding, but there are some small differences.


>
>
> a) Pscp checks against fewer previous expressions compared to Supero. (Pscp only looks at expressions on the form R\<g es\>, where R are evaluation contexts for call-by-name, and g are top/local level definitions. Let-statements, for example, are not bound in our contexts). This could be beneficial for compilation speed, but I don't have any concrete numbers right now. The drawback is that our approach can not perform Distillation; a more powerful transformation that makes the checks even more expensive. This should not be hard to change in an actual implementation though.
>
>

>
>
> b) Generalisation. Pscp currently uses the msg, and Neil has deducted a better way to split expressions. Switching between them should be a matter of changing a couple of lines of code in an actual implementation.
>
>

>
>
> c) simpleTerminate. This roughly corresponds to a mixture between values and what pscp have labeled as "annoying expressions". Neil was spot on in earlier discussions on what annoying expressions are: something where evaluation is stuck (normally because of a free variable in the next position, for example the application "x es"). Simon has an interesting algorithm formulation that avoids annoying expressions altogether and is probably simpler to implement.
>
>


3) Inlining decisions. Neil has a more advanced way of determining when things should be inlined or not. I believe that the solution is some kind of union between the inliner paper, Neil's work and possibly some kind of cost calculus for inlining.



What is not mentioned in either of our works though is that it is a typed intermediate representation in GHC. System Fc has the casts that might get in the way of transformations (so effectively they are some kind of annoying expressions). Rank-n polymorphism is another potential problem, but I am not sure if that will be a problem with System Fc. These are not fundamental problems that will take two months to solve, but I think that implementing a supercompiler for System Fc is more than just two days of intense hacking, even for someone already familiar with GHC internals.


