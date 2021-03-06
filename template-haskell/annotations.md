# Problem statement



Currently [wiki:/Annotations](annotations) are only usable from the GHC API.  Our goal is to add support for [TemplateHaskell](template-haskell) too.  Both creation of annotations, reification of annotations and gathering module annotations transitively from our whole import tree should be added.


# Motivation



As a main use-case we will consider the [
http://hackage.haskell.org/package/hflags](http://hackage.haskell.org/package/hflags) library.



The goal of the library is to:


- provide easy to use command line flags,
- that are pure from the programmers point of view (they don't change during one program run),
- that can be defined anywhere, even in a module,
- and the used flags are automatically gathered in the main program for parsing and `--help` generation.


Example:


- `A.hs`:

  ```
  {-# LANGUAGE TemplateHaskell #-}

  module A (verbose) where

  import HFlags

  defineFlag "verbose" True "Whether debug output should be printed."

  verbose :: Bool
  verbose = flags_verbose
  ```
- `ImportExample.hs`

  ```
  {-# LANGUAGE TemplateHaskell #-}

  import HFlags
  import Control.Monad (when)
  import qualified A

  main = do _ <- $initHFlags "Importing example v0.1"
            when A.verbose $ putStrLn "foobar"
  ```


Let's try it out:


```wiki
errge@curry:/tmp/y $ runhaskell ImportExample.hs  --help
Importing example v0.1

  -h  --help, --usage, --version  Display help and version information.
      --undefok                   Whether to fail on unrecognized command line options.
      --verbose[=BOOL]            Whether debug output should be printed. (default: true, from module: A)
errge@curry:/tmp/y $ runhaskell ImportExample.hs  --verbose=True
foobar
errge@curry:/tmp/y $ runhaskell ImportExample.hs  --verbose=False
errge@curry:/tmp/y $
```


What's important to note here is that in the main program we didn't have to specify the list of modules that has to be searched for command line flags, the template haskell `$initHFlags` function automatically found them all.  Even if they are not imported directly, but only indirectly by our main program.  A motivating example for that:


- the TCP module has a connect method that accepts a command-line `tcp_connect_timeout` flag (so the user can change that to 5 seconds from the usual 10 hours :)),
- the HTTP module of course depends on TCP,
- the WGet module depends on HTTP,
- the main program uses WGet to download something from the internet.


In this case HFlags automatically makes it so that the `tcp_connect_timeout` flag is show in `--help` of the main program and can be changed by the user to any value she sees fit.  This is achieved via template haskell, but in exchange the programmer doesn't have to explicitly pass around any kind of values or applicative stuff for every imported module that uses command line flags.



Of course, this whole approach can be debated and maybe we should instead explicitly pass parameters to functions; but let's leave that debate to the getopt authors and focus on TH on this page.


---


# Current implementation with typeclassses



How is this implemented in HFlags currently?  By using typeclasses.



The FlagData ([
https://github.com/errge/hflags/blob/v0.4/HFlags.hs\#L129](https://github.com/errge/hflags/blob/v0.4/HFlags.hs#L129)) datatype contains all the information we need to know about a flag.  Then for every flag we create a new fake datatype that implements the Flag class ([
https://github.com/errge/hflags/blob/v0.4/HFlags.hs\#L149](https://github.com/errge/hflags/blob/v0.4/HFlags.hs#L149)).  In `initHFlags` we simply call template haskell reify on the Flag class.  This gives us our "fake" instances and their `getFlagData` method can be used to query the needed flag data for `--help` generation, parsing, etc.  This can be seen in at [
https://github.com/errge/hflags/blob/v0.4/HFlags.hs\#L397](https://github.com/errge/hflags/blob/v0.4/HFlags.hs#L397).



This is ugly: we are abusing the reification of types and instances to send messages to ourselves between modules.  There should be an explicit way to do that.  This is requested in [\#7867](http://gitlabghc.nibbler/ghc/ghc/issues/7867).


## Aside: with the current GHC, this implementation is not just ugly, but broken



Unfortunately, the current idea is not really working all that nice, because of [\#8426](http://gitlabghc.nibbler/ghc/ghc/issues/8426).



Haskell98 and Haskell prime says that all the instances should be visible that are under the current module in the dependency tree, but this is not the case currently in GHC when using one-shot compilation (`ghc -c`, not `ghc --make`).  This is a optimization, so we can save on the reading of all the module dependencies transitively.  GHC tries to cleverly figure out where to find so called orphan instances.



Template haskell is a corner-case, where this orphan logic is not clever enough and therefore reify doesn't see some of the instances that are under the current module in the dependency tree.  Even more so, if the class instance is in a separate package (and not marked orphan, as is the case in HFlags), then it's not seen either in one-shot or in batch mode.  Therefore HFlags can't gather all the flags in `$initHFlags`.  There is a fix to this as a patch in [\#8426](http://gitlabghc.nibbler/ghc/ghc/issues/8426), but that needs more discussion.



An easier way is to implement [\#1480](http://gitlabghc.nibbler/ghc/ghc/issues/1480), module reification.  If we can get the import list of every module, then HFlags can walk the tree of imports itself and gather all the flags.  The nice in this is that the compiler only needs very basic and simple support, and then the logic of traversal can be implemented in HFlags, not in the compiler.


---


# Design proposal



The proposal is to make it possible to generate annotations from template haskell (when defining a flag) and read them all back via template haskell (in `$initHFlags`).  These module level annotations (in HFlags case) will then contain the info that is needed for flag parsing and `--help` generation.



Specifically, we propose to add the following new function to the `Quasi` class:


```
class Quasi where 
  qReifyAnnotations :: Data a => AnnLookup -> m [a]
  qReifyModule      :: Module -> m ModuleInfo

data AnnLookup = AnnLookupModule Module
               | AnnLookupName Name
               deriving( Show, Eq, Data, Typeable )

data ModuleInfo =
  -- | Contains the import list of the module.
  ModuleInfo [Module]
  deriving( Show, Data, Typeable )

data Module = Module PkgName ModName -- package qualified module name
 deriving (Show,Eq,Ord,Typeable,Data)
```


We also propose to add the new `AnnP` data constructor to `data Pragma`:


```
data Pragma = InlineP         Name Inline RuleMatch Phases
            | SpecialiseP     Name Type (Maybe Inline) Phases
            | SpecialiseInstP Type
            | RuleP           String [RuleBndr] Exp Exp Phases
            | AnnP            AnnTarget Exp

data AnnTarget = ModuleAnnotation
               | TypeAnnotation Name
               | ValueAnnotation Name
              deriving (Show, Eq, Data, Typeable)
```


These functions behave as follows:


- `AnnP` is very similar to the already existing pragma descriptors in `data Pragma`: it contains the target of the annotation and the payload as an `Exp`,
- `qReifyAnnotation` is the dual of `AnnP`, it can be used to reify an annotation to get back the payload,
- `qReifyModule` gives back the list of imports for the named module (in the future, it should reify more: maybe everything that is useful from the interface file).

## Example



Here is a minimalistic implementation showing how we can use these new facilities to implement `defineFlag` and `$initHFlags` in the above example.


- `HFlags.hs`:

  ```
  {-# LANGUAGE TemplateHaskell #-}
  {-# LANGUAGE DeriveDataTypeable #-}

  module HFlags where

  import Control.Applicative
  import Data.Data
  import qualified Data.Set as Set
  import Language.Haskell.TH
  import Language.Haskell.TH.Syntax

  -- in the real world, this is more complex, of course
  data FlagData = FlagData String deriving (Show, Data, Typeable)
  instance Lift FlagData where
    lift (FlagData s) = [| FlagData s |]

  defineFlag :: FlagData -> DecsQ
  defineFlag str = do
    (:[]) <$> pragAnnD ModuleAnnotation (lift str)

  traverseAnnotations :: Q [FlagData]
  traverseAnnotations = do
    ModuleInfo children <- reifyModule =<< thisModule
    go children Set.empty []
    where
      go []     _visited acc = return acc
      go (x:xs) visited  acc | x `Set.member` visited = go xs visited acc
                             | otherwise = do
                               ModuleInfo newMods <- reifyModule x
                               newAnns <- reifyAnnotations $ AnnLookupModule x
                               go (newMods ++ xs) (x `Set.insert` visited) (newAnns ++ acc)

  initHFlags :: ExpQ
  initHFlags = do
    anns <- traverseAnnotations
    [| print anns |] -- in the real world do something here, like generating --help
  ```
- `A.hs`:

  ```
  {-# LANGUAGE TemplateHaskell #-}

  module A where

  import HFlags

  defineFlag (FlagData "A module is here!")
  ```
- `B.hs`:

  ```
  {-# LANGUAGE TemplateHaskell #-}

  module B where

  import A
  import HFlags

  defineFlag (FlagData "B module is here!")
  ```
- `Main.hs`:

  ```
  {-# LANGUAGE TemplateHaskell #-}

  import B
  import HFlags

  main = do
    $initHFlags
  ```
- `build.sh`:

  ```wiki
  #!/bin/sh

  set -e

  rm -f *.o *.hi Main

  GHC="/home/errge/tmp/ghc/inplace/bin/ghc-stage2 -v0 "

  $GHC -c HFlags.hs
  $GHC -c A.hs
  $GHC -c B.hs
  $GHC -c Main.hs
  $GHC --make Main
  ```
- result:

  ```wiki
  errge@curry:~/tmp/sketch $ ./build.sh && ./Main
  [FlagData "A module is here!",FlagData "B module is here!"]
  ```


In spite of only importing B from Main, we see the annotations from A, this was our goal.


---


# Implementation status, options, questions


## Already done: a bugfix, and annotation generation and reification



The already merged [\#3725](http://gitlabghc.nibbler/ghc/ghc/issues/3725) and [\#8340](http://gitlabghc.nibbler/ghc/ghc/issues/8340) makes it possible to generate annotations from TH.  We support all three kind of annotations: annotations on types, values and whole modules.



Annotation reification is implemented and merged in [\#8397](http://gitlabghc.nibbler/ghc/ghc/issues/8397).


## Proposed simplification of annotation data types



See [\#8388](http://gitlabghc.nibbler/ghc/ghc/issues/8388). Easy to do.


## Patch ready, nice to have: typed annotation reification



[\#8460](http://gitlabghc.nibbler/ghc/ghc/issues/8460) provides a very small addition that makes it possible to use annotation reification together with the new typed [TemplateHaskell](template-haskell).



This patch doesn't need to get merged urgently, it's just nice to have.


##
Waiting for review, hopefully to still go into 7.8.1: module reification, [\#1480](http://gitlabghc.nibbler/ghc/ghc/issues/1480)



The only feature that is still not in GHC and needed for HFlags is a way to walk the module dependency tree of the currently being compiled module from TH.  This is made possible by [\#1480](http://gitlabghc.nibbler/ghc/ghc/issues/1480), that just adds minimal module reification (import list).  Once we have that, HFlags can just ask for the imports and than for the imports of the imports, etc. to walk the tree itself.  There is no need to do this from the compiler, as that would obscure the fact that this can be a slow and wasteful operation.



**If you have any opinion about this minimal module reification, please comment on and review [\#1480](http://gitlabghc.nibbler/ghc/ghc/issues/1480).**



Tidy-up ticket: [\#8489](http://gitlabghc.nibbler/ghc/ghc/issues/8489)


# Other related tickets


- [\#8398](http://gitlabghc.nibbler/ghc/ghc/issues/8398): currently on hold, as maybe this can be done as a helper function in `Lib.hs` after [\#1480](http://gitlabghc.nibbler/ghc/ghc/issues/1480) is merged, without touching the inside of GHC.
