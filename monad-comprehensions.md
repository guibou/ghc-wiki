# Monad comprehensions



After a long absence, monad comprehensions are back, thanks to George Giorgidze and his colleagues.  With `{-# LANGUAGE MonadComprehensions #-}` the comprehension `[ f x | x <- xs, x>4 ]` is interpreted in an arbitrary monad, rather than being restricted to lists.  Not only that, it also generalises nicely for parallel/zip and SQL-like comprehensions. The aforementioned generalisations can be turned on by enabling the `MonadComprehensions` extension in conjunction with the `ParallelListComp` and `TransformListComp` extensions.



Rebindable syntax is fully supported for standard monad comprehensions with generators and filters. We also plan to allow rebinding of the parallel/zip and SQL-like monad comprehension notations.



For further details and usage examples, see the paper "Bringing back monad comprehensions" [
http://db.inf.uni-tuebingen.de/staticfiles/publications/haskell2011.pdf](http://db.inf.uni-tuebingen.de/staticfiles/publications/haskell2011.pdf) at the 2011 Haskell Symposium.



See ticket [\#4370](http://gitlabghc.nibbler/ghc/ghc/issues/4370).


## Translation rules


```wiki
Variables    : x and y
Expressions  : e, f and g
Patterns     : w
Qualifiers   : p, q and r
```


The main translation rule for monad comprehensions.


```wiki
[ e | q ] = [| q |] >>= (return . (\q_v -> e))
```


`(.)_v` rules. Note that `_v` is a postfix rule application.


```wiki
(w <- e)_v = w
(let w = d)_v = w
(g)_v = ()
(p , q)_v = (p_v,q_v)
(p | v)_v = (p_v,q_v)
(q, then f)_v = q_v
(q, then f by e)_v = q_v
(q, then group by e using f)_v = q_v
(q, then group using f)_v = q_v
```


`[|.|]` rules.


```wiki
[| w <- e |] = e
[| let w = d |] = return d
[| g |] = guard g
[| p, q |] = ([| p |] >>= (return . (\p_v ->  [| q |] >>= (return . (\q_v -> (p_v,q_v)))))) >>= id
[| p | q |] = mzip [| p |] [| q |]
[| q, then f |] = f [| q |]
[| q, then f by e |] = f (\q_v -> e) [| q |]
[| q, then group by e using f |] = (f (\q_v -> e) [| q |]) >>= (return . (unzip q_v))
[| q, then group using f |] = (f [| q |]) >>= (return . (unzip q_v))
```


`unzip (.)` rules. Note that `unzip` is a desugaring rule (i.e., not a function to be included in the generated code).


```wiki
unzip () = id
unzip x  = id
unzip (w1,w2) = \e -> ((unzip w1) (e >>= (return .(\(x,y) -> x))), (unzip w2) (e >>= (return . (\(x,y) -> y))))
```

### Examples



Some translation examples using the do notation to avoid things like pattern matching failures are:


```wiki
[ x+y | x <- Just 1, y <- Just 2 ]

-- translates to:
do x <- Just 1
   y <- Just 2
   return (x+y)
```


Transform statements:


```wiki
[ x | x <- [1..], then take 10 ]

-- translates to:
take 10 (do
  x <- [1..]
  return x)
```


Grouping statements (note the change of types):


```wiki
[ (x :: [Int]) | x <- [1,2,1,2], then group by x ] :: [[Int]]

-- translates to:
do x <- mgroupWith (\x -> x) [1,2,1,2]
   return x
```


Parallel statements:


```wiki
[ x+y | x <- [1,2,3]
      | y <- [4,5,6] ]

-- translates to:
do (x,y) <- mzip [1,2,3] [4,5,6]
   return (x+y)
```


Note that the actual implementation is **not** using the `do`-Notation, it's only used here to give a basic overview about how the translation works.


## Implementation details



Monad comprehensions had to change the `StmtLR` data type in the `hsSyn/HsExpr.lhs` file in order to be able to lookup and store all functions required to desugare monad comprehensions correctly (e.g. `return`, `(>>=)`, `guard` etc). Renaming is done in `rename/RnExpr.lhs` and typechecking in `typecheck/TcMatches.lhs`. The main desugaring is done in `deSugar/DsListComp.lhs`. If you want to start hacking on monad comprehensions I'd look at those files first.



Some things you might want to be aware of:



\[todo\]


