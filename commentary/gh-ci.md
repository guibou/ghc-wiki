# GHCi - Implementation and Working details



GHCi currently has two modes of operation


1. Normal mode
1. Remote mode - Using the option -fexternal-interpreter. This was introduced in GHC 8.0.1 by Simon Marlow. See [RemoteGHCi](remote-gh-ci) for more details.


This document provides details of working of GHCi primarily for the normal mode of working, though the details are more or less similar for both. **The main goal here is to answer the question '*How does GHCi support breakpoints?*' **, but it will present more details of the basic working along with this.


## Compilation


### Source code instrumentation


>
>
> `HsTick` are added to annotate the expressions in the de-sugaring phase, which is later converted to `Tick` (part of `CoreSyn`).
>
>


When a source code is loaded in ghci or the user enters an expression, at the front end of the compiler, we annotate the source code with **ticks**, based on the program coverage tool of Andy Gill and Colin Runciman. Ticks are uniquely numbered with respect to a particular module. Ticks are annotations on expressions, so each tick is associated with a source span, which identifies the start and end locations of the ticked expression.



The instrumentation is implemented in [compiler/deSugar/Coverage.hs](/trac/ghc/browser/ghc/compiler/deSugar/Coverage.hs). For details on the heuristics of this instrumentation, see the use of `TickForBreakPoints`. (It would be nice to have this documented properly)



For each module we also allocate an array of breakpoint flags, with one entry for each tick in that module. This array is managed by the GHC storage manager, so it can be garbage collected if the module is re-loaded and re-ticked. We retain this array inside the `ModGuts` data structure, which is defined in [compiler/main/HscTypes.hs](/trac/ghc/browser/ghc/compiler/main/HscTypes.hs). This array is stored inside something called `ModBreaks`, which also stores an association list of source spans and ticks.


### Byte code generation


>
>
> For each `Tick` a special breakpoint instruction `BRK_FUN` is added during byte code generation.
>
>


In the coverage tool the ticks are turned into real code which performs a side effect when evaluated. In the debugger the ticks are purely annotations. They are used to pass information to the byte code generator, which generates special breakpoint instructions for ticked expressions.



The byte code generator turns `CoreSyn` into a bunch of Byte Code Objects (BCOs). BCOs are heap objects which correspond to top-level bindings, and `let` and `case` expressions. Each BCO contains a sequence of byte code instructions (BCIs), which are executed by the byte code interpreter ([rts/Interpreter.c](/trac/ghc/browser/ghc/rts/Interpreter.c)). Each BCO also contains some local data which is needed in the instructions. 



The BCIs for this BCO are generated as usual, and we prefix a new special breakpoint instruction (`BRK_FUN`) on the front. Thus, when the BCO is evaluated, the first thing it will do is interpret the breakpoint instruction, and hence decide whether to break or not. We annotate the BCO with information about the tick, such as its free variables, and the breakpoint number. 


## Runtime


>
>
> The GHCi thread waits on a `statusMVar`, on hitting breakpoint this is filled with `EvalBreak` by the expression thread and it waits on the `breakMVar`. When doing resume `breakMVar` is filled by GHCi thread.
>
>


To understand what happens when a breakpoint is hit, it is necessary to know how GHCi evaluates an expression at the command line.


### Execution of an Expression in GHCi



When the user types in an expression (as a string) it is parsed, type checked, and compiled, and then run. In [compiler/main/InteractiveEval.hs](/trac/ghc/browser/ghc/compiler/main/InteractiveEval.hs) we have the function:


```
-- | Run a statement in the current interactive context.
execStmt
  :: GhcMonad m
  => String             -- ^ a statement (bind or expression)
  -> ExecOptions
  -> m ExecResult
```


The `GhcMonad` carries a `Session` which contains the gobs of environmental information which is important to the compiler. The `String` is what the user typed in, and `ExecResult`, is the answer that you get back if the execution terminates. `ExecResult` is defined like so:
(in [compiler/main/InteractiveEvalTypes.hs](/trac/ghc/browser/ghc/compiler/main/InteractiveEvalTypes.hs)


```
data ExecResult
  = ExecComplete
       { execResult :: Either SomeException [Name]
       , execAllocation :: Word64
       }
  | ExecBreak
       { breakNames :: [Name]
       , breakInfo :: Maybe BreakInfo
       }
```


Normally what happens is that `execStmt` forks a new thread to handle the evaluation of the expression. It calls `evalStmt` ([compiler/ghci/GHCi.hs](/trac/ghc/browser/ghc/compiler/ghci/GHCi.hs) in both remote and normal mode) to create an `EvalStmt` `Message`. This message is processed by the `evalStmt` ([libraries/ghci/GHCi/Run.hs](/trac/ghc/browser/ghc/libraries/ghci/GHCi/Run.hs)  in normal mode). This in turns calls the `sandboxIO` to do `forkIO`. It then blocks on an `MVar` and waits for the thread to finish.



This `MVar` is (now) called `statusMVar`, because it carries the execution status of the computation which is being evaluated. We will discuss its type shortly. When the thread finishes it fills in `statusMVar`, which wakes up `execStmt`, and it returns a `ExecResult`.


### On hitting a breakpoint



To make the discussion comprehensible let us distinguish the two threads: 


1. The thread which runs the GHCi prompt.
1. The thread which is forked to run an expression.


We'll call the first one the *GHCi thread*, and the second the *expression thread*. 



In the debugger, the process of evaluating an expression is made more intricate. The reason is that if the expression thread hits a breakpoint it will want to return *early* to the GHCi thread, so that the user can access the GHCi prompt, issue commands *etcetera*. 



This raises a few questions:


- How do we arrange for the expression thread to stop and return early?
- What information needs to be passed from the expression thread to the GHCi thread, and how do we arrange that flow of information?
- How do we wake up the GHCi thread and return to the prompt?
- How do we continue execution of the expression thread after we have hit a breakpoint?
- What happens if we are running in the GHCi thread after a breakpoint, and we evaluate some other expression which also hits a breakpoint (i.e. what about nested breakpoints?)
- What happens if the expression thread forks more threads?


To arrange the early return of the expression thread when it hits a breakpoint we introduce a second MVar


```
   breakMVar :: MVar ()
```


When the expression thread hits a breakpoint it waits on `breakMVar`. When the user decides to continue execution after a breakpoint, the GHCi thread fills `breakMVar`, which wakes up the expression thread and allows it to continue execution.



Now we must return to `statusMVar` and look at it in more detail. We introduce a new type called `EvalStatus`


```
type EvalStatus a = EvalStatus_ a a

data EvalStatus_ a b
  = EvalComplete Word64 (EvalResult a)
  | EvalBreak Bool
       HValueRef{- AP_STACK -}
       Int {- break index -}
       Int {- uniq of ModuleName -}
       (RemoteRef (ResumeContext b))
       (RemotePtr CostCentreStack) -- Cost centre stack

data EvalResult a
  = EvalException SerializableException
  | EvalSuccess a

statusMVar :: MVar (EvalStatus a)
```


It represents the execution status of the expression thread, which is either `Complete` (with an exception or a value of some type), or `Break`, to indicate that the thread has hit a breakpoint.



`statusMVar` simply contains an `EvalStatus` value:



The two MVars, `statusMVar` and `breakMVar`, are used like so: (This is part of `handleRunStatus`)


- When `evalStmt` begins to execute an expression for the first time it forks the expression thread, and then waits on the `statusMVar`.
- If the expression thread completes execution with an exception or with a final value, it fills in `statusMVar` with the appropriate `EvalStatus` value, which wakes up the GHCi thread. This `EvalStatus` is turned into a `ExecResult` which gets propagated back to the command line as usual.
- If the expression thread does not complete, but hits a breakpoint, it fills in the `statusMVar` with an appropriate `EvalBreak` value, and then waits on the `breakMVar`. The GHCi thread is woken up because of the write to `statusMVar`, and the `ExecResult` is propagated back to the command line (this time it is a `ExecBreak`).
- When the user decides to continue execution after a breakpoint the GHCi thread fills in the `breakMVar`, thus waking up the expression thread, and then the GHCi thread waits on the `statusMVar` again. The whole process continues until eventually the expression thread completes its evaluation.


Now we turn our attention to the `EvalBreak` constructor:


```
  | EvalBreak Bool
       HValueRef{- AP_STACK -}
       Int {- break index -}
       Int {- uniq of ModuleName -}
       (RemoteRef (ResumeContext b))
       (RemotePtr CostCentreStack) -- Cost centre stack
```


The arguments of `EvalBreak` are as follows


1. A heap closure, specifically something which represents a chunk of the Stg stack (an `StgAP_STACK`, to be precise). It is inside this object that we find the values of the local variables of the breakpoint. We use a type variable here for convenience. 
1. The next two Int arguments are stored as part of `BreakInfo`. This is the information about the module and breakpoint number (index of breakpoint in `ModBreaks` array).
1. Remote reference to `ResumeContext` and `CostCentreStack` are stored for use during resume (details below).


The `EvalBreak` get assembled by the I/O action which is executed by a thread when it hits a breakpoint. The code for the I/O action is as follows:


```
type BreakpointCallback = Int# -> Int# -> Bool -> HValue -> IO ()

   onBreak :: BreakpointCallback
   onBreak ix# uniq# is_exception apStack = do
     tid <- myThreadId
     let resume = ResumeContext
           { resumeBreakMVar = breakMVar
           , resumeStatusMVar = statusMVar
           , resumeThreadId = tid }
     resume_r <- mkRemoteRef resume
     apStack_r <- mkRemoteRef apStack
     ccs <- toRemotePtr <$> getCCSOf apStack
     putMVar statusMVar $ EvalBreak is_exception apStack_r (I# ix#) (I# uniq#) resume_r ccs
     takeMVar breakMVar
```


We "pass" the I/O action to the runtime system by way of a global stable pointer, which is called `breakPointIOAction`. 
Note that the I/O action proceeds to write to the `statusMVar`, which wakes up the GHCi thread, and then it waits on the `breakMVar`. 


### Resuming execution after a breakpoint



A resume context is created to store the MVars on the execution thread inside the `onBreak` action.


```
data ResumeContext a = ResumeContext
  { resumeBreakMVar :: MVar ()
  , resumeStatusMVar :: MVar (EvalStatus a)
  , resumeThreadId :: ThreadId
  }
```


On the GHCi side a `Resume` structure is created to store this context (along with a few more details).



When we hit a breakpoint the GHCi client pushes the `Resume` context onto a stack kept in the GHC session.
If the user evaluates a different expression, which hits another breakpoint, its `Resume` context will be pushed on top of the old one. Eventually, when the user enters `:step` or `:continue`, the top of the resume stack is popped, and that is the action which is run next.



If during the execution of an expression multiple breakpoints are hit or user does a `:step`, a history of breakpoints is maintained in the `History` structure. This is used to go `:back` or `:forward` and is useful to inspect the values of free variables at a point of time in history


## Inside the RTS



As mentioned above, we add a new kind of BCI for breakpoints. It is called `bci_BRK_FUN`. It is added for the `Tick` in core, during Byte Code compilation. When the intepreter hits this instruction it does the following things:


1. Check to see if we are returning from a breakpoint (by checking a bit flag in the current TSO). If so, we don't want to stop again (this time), otherwise we'd get into an infinite loop. We record that we are no longer returning from a breakpoint, and then continue to the next BCI.
1. If we aren't returning from a breakpoint, then we check to see if the global single-step flag is set, or if the individual breakpoint flag for the current expression is set. If this is true, we prepare to save the stack, and call the `onBreak` function. If it is not true then we skip to the next BCI.
1. If we are going to stop at this breakpoint, we create a new `AP_STACK` and copy the topmost stack frame into it. Then we push the current BCO onto the stack, and set up the `onBreak` so that when we come back to this thread the action will be executed. We then record that we are now stopping at a breakpoint, and then yield to the scheduler. When the scheduler returns to this thread the `onBreak` will be executed, which will send us back to the GHCi prompt.


Here's how the stack is set up just prior to yielding:


```
    Sp -= 11;
    Sp[10] = (W_)obj;
    Sp[9]  = (W_)&stg_apply_interp_info;
    Sp[8]  = (W_)new_aps;                    /* the AP_STACK */
    Sp[7]  = (W_)False_closure;              /* True <=> an exception */
    Sp[6]  = (W_)&stg_ap_ppv_info;
    Sp[5]  = (W_)BCO_LIT(arg3_module_uniq);  /* 'uniq#' */
    Sp[4]  = (W_)&stg_ap_n_info;
    Sp[3]  = (W_)arg2_array_index;           /* 'ix#' */
    Sp[2]  = (W_)&stg_ap_n_info;
    Sp[1]  = (W_)ioAction;                   /* apply the IO action to its four arguments above */
    Sp[0]  = (W_)&stg_enter_info;            /* get ready to run the IO action */
```

- The first two things are the current BCO and an info table (what do you call these things anyway?). We need these so that when we eventually resume execution from a breakpoint we will start executing the currect BCO again.
- The next eight things correspond to the call to the `onBreak` with its arguments passed first, then the action itself.

  - `new_aps` is the AP\_STACK which saves the topmost stack frame.
  - `False_closure` is the bool for `is_exception`
  - `arg3_module_uniq`  and `arg2_array_index` are `uniq#` and `ix#` respectively. 

## Inspecting values



This is done exactly as it was before in the prototype debugger. See: [GhciDebugger](ghci-debugger). (Need to review this statement and the linked wiki.)


