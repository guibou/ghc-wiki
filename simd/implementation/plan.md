
Please see the [top-level SIMD project page](simd) for further details.


## Introduction



This text documents the implementation stages / components for adding SIMD support to GHC.



Based on that design, the high-level tasks that must be accomplished include the following:


1. Modify autoconf to determine SSE availability and vector size
1. Add new PrimOps to allow Haskell to make use of Vectors
1. Add new MachOps to Cmm to communicate use of Vectors
1. Modify the LLVM Code Generator to translate Cmm to LLVM vector instructions
1. Demonstrate use of PrimOps from Haskell program
1. Modify Vector Library
1. Modify DPH Libraries
1. Arrange that the other Code Generators continue to generate non SIMD code


Introduction of SIMD support to GHC will occur in stages to demonstrate the entire “vertical” stack is functional:


1. Introduce “Float” PrimOps (as necessary to run an example showing SIMD usage in the LLVM)
1. Add appropriate Cmm support for the Float primtype / primop subset
1. Modify the LLVM Code Generator to support the Double vectorization
1. Demonstrate the PrimOps and do limited performance testing to ensure SIMD is functional
1. Modify Vector Libraries to make use of new PrimOps
1. Modify the DPH Libraries
1. Higher level examples using the above libraries
1. Build out the remaining PrimOps
1. Demonstrate full stack
1. Test remaining code generators

## Current Open Questions



These clearly won't be all of the questions I have, there is a substantial amount of work that goes through the entire GHC compiler stack before reaching the LLVM instructions.


## Location of SIMD branch



The SIMD branch of GHC is named, appropriately, `simd`.


## Adding a primtype / primop Outline



When going through this outline, it is helpful to have the [
Compiler Pipeline](http://hackage.haskell.org/trac/ghc/wiki/Commentary/Compiler/HscMain) explanation available as well as the explanation of the source code tree (that includes nested explanations of important directories and files) available [
here](http://hackage.haskell.org/trac/ghc/wiki/Commentary/SourceTree).



Addition of Types for use by Haskell


- ./compiler/types/TyCon.lhs

  - TyCons represent type constructors.  There are \@data declarations, \@type synonyms, \@newtypes and class declarations (\@class).  We will need to modify this to add a proper type constructor.
  - Prelude uses this type constructor in ./compiler/prelude/TysPrim.lhs


Modifications to the compiler to add primtype / primop to Prelude


- ./compiler/prelude/PrelNames.lhs
- ./compiler/prelude/TysPrim.lhs
- ./compiler/prelude/primops.txt.pp

  - Addition of a type here (primtype) to operate on
  - Defines the primops that are associated with the types that are defined
  - For primops defined here that are inline, modify compiler/codeGen/CgPrimOp.hs


Modifications to add support to the compiler for the new types.  Look to individual files for the portion of the compiler that is being modified.  The binaries generated out of this branch are called from the ghc/ binaries.


- ./compiler/codeGen/CgCallConv.hs
- ./compiler/codeGen/CgPrimOp.hs
- ./compiler/codeGen/CgUtils.hs
- ./compiler/codeGen/SMRep.lhs
- ./compiler/codeGen/StgCmmLayout.hs
- ./compiler/codeGen/StgCmmPrim.hs


More modifications to the Cmm portion of the compiler chain.


- ./compiler/cmm contains the code that inputs STG and outputs Cmm
- ./compiler/cmm/CmmExpr.hs
- ./compiler/cmm/CmmType.hs
- ./compiler/cmm/CmmUtils.hs
- ./compiler/cmm/OldCmm.hs
- ./compiler/cmm/StackColor.hs


Modifications to the LLVM code generator


- Generating the human readable LLVM code (.ll) occurs in the compiler/llvmGen code.  It receives Cmm and turns it around to LLVM bytecodes through human readable LLVM code.  A "simple" use of LLVM vector instructions using floats is shown at the [SIMD Vector Example In LLVM](simd-vector-example-in-llvm) page

  - compiler/llvmGen/Llvm/AbsSyn.hs - no changes, describes the abstract structure of an LLVM program
  - compiler/llvmGen/Llvm/PpLlvm.hs - no changes, this is the module that does a Pretty Print of LLVM I(ntermediate) R(epresentation), it still uses generic constructs described in Llvm.AbsSyn (where no changes are necessary), Llvm.Types (where changes are needed)
  - compiler/llvmGen/Llvm/Types.hs - Describes the LLVM Basic Types and Variables.  Changes are necessary to add Vectors.  LMVector Int LlvmType will be added (eventually constraints on the width may need to be added.  "fadd" is already present, as is "add" and other operations.  Most likely, additional operations will have to be added as the code generated is slightly different for, say, a float vector since a series of instructions have to be executed for each "add", even though the basic "fadd" structure remains the same.
  - compiler/llvmGen/LlvmCodeGen/Base.hs, no changes appear necessary here.  These seem to be primarily various Label and Environment handling items ... primarily structural.
  - compiler/llvmGen/LlvmCodeGen/CodeGen.hs, changes will most likely be necessary here for various operators.  This is the primary location where Cmm is converted to LLVM operators ... for example, MO\_F\_Add with a parameter is converted to LM\_MO\_FAdd here:  MO\_F\_Add  \_ -\> genBinMach LM\_MO\_FAdd).  In the event that additional operators are added (MO\_VF\_Add for example), this will definitely have to be modified.
  - compiler/llvmGen/LlvmCodeGen/Data.hs. converts static data types from Cmm (CmmData) to LLVM structures, no changes are likely necessary here
  - compiler/llvmGen/LlvmCodeGen/Ppr.hs, the pretty print helpers for the LLVM Code Generator ... dependent on other files
  - compiler/llvmGen/LlvmCodeGen/Regs.hs, no additional registers necessary (??), no changes


Modifications to the STG Code Generation


- ./includes/stg/MachRegs.h
- ./includes/stg/Regs.h
- ./includes/stg/Types.h


./includes/HaskellConstants.hs
./includes/mkDerivedConstants.c



./includes/rts/storage/FunTypes.h



./utils/genapply/GenApply.hs
./utils/genprimopcode/Main.hs


## Modify autoconf



Determining if a particular hardware architecture has SIMD instructions, the version of the instructions available (SSE, SSE2, SSE3, SSE4 or an iteration of one of those), and consequently the size of vectors that are supported occurs during the configuration step on the architecture that the build will occur on.  This is the same time that the sizes of Ints are calculated, alignment constants, and other pieces that are critical to GHC.



Backing up from the results to the location that the changes are introduced, the current alignment and primitive sizes are available in ./includes/ghcautoconfig.h, here is a sample:


```wiki
...
/* The size of `char', as computed by sizeof. */
#define SIZEOF_CHAR 1

/* The size of `double', as computed by sizeof. */
#define SIZEOF_DOUBLE 8
...
```


These are constructed from mk/config.h\* that are generated by configure.ac and autoheader.  The configure.ac (or a related file) should be able to be sufficiently modified to determine if SSE is available and, consequently, the vector size that can be operated on and (later) how many pieces of data can be operated on in parallel (determined by the operation).  SSE had an MMX register size of 64-bits and all later SSE versions (2 and above) have a register size of 128 bits.  This implies that any type that is 32-bits can have 4 pieces of data calculated against in a single instruction.



There is an example of configure.ac modifications to detect SSE availability available on the web, the primary body of the check is as follows (xmmintrin.h contains the SSE instruction set):


```wiki
AC_MSG_CHECKING(for SSE in current arch/CFLAGS)
AC_LINK_IFELSE([
AC_LANG_PROGRAM([[
#include <xmmintrin.h>
__m128 testfunc(float *a, float *b) {
  return _mm_add_ps(_mm_loadu_ps(a), _mm_loadu_ps(b));
}
]])],
[
has_sse=yes
],
[
has_sse=no
]
)
AC_MSG_RESULT($has_sse)

if test "$has_sse" = yes; then
  AC_DEFINE([_USE_SSE], 1, [Enable SSE support])
  AC_DEFINE([_VECTOR_SIZE], 128, [Default Vector Size for Optimization])
fi
```


Once the mk/config.h is modified with the above, the includes/ghcautoconf.h is modified during the first stage of the GHC build process.  Once includes/ghcautoconf.h is modified, the \_USE\_SSE constant is available in the Cmm definitions (next section).



There are more detailed explanations of how to use cpuid to determine the supported SSE instruction set available on the web as well.  cpuid may be more appropriate but are also much more complex.  Details for using cpuid are available at [
http://software.intel.com/en-us/articles/using-cpuid-to-detect-the-presence-of-sse-41-and-sse-42-instruction-sets/](http://software.intel.com/en-us/articles/using-cpuid-to-detect-the-presence-of-sse-41-and-sse-42-instruction-sets/).



It should be noted, that since the overall goal is to let the LLVM handle the actual assembly code that does vectorization, it's only possible to support vectorization up to the version that the LLVM supports.


## Add new MachOps to Cmm code



It may make more sense to add the MachOps to Cmm prior to implementing the PrimOps (or at least before adding the code to the CgPrimOp.hs file).  There is a useful [
Cmm Wiki Page](http://hackage.haskell.org/trac/ghc/wiki/Commentary/Compiler/CmmType#AdditionsinCmm) available to aid in the definition of the new Cmm operations.



Modify compiler/cmm/CmmType.hs to add new required vector types and such, here is a basic outline of what needs to be done:


```wiki
data CmmType    -- The important one!
  = CmmType CmmCat Width
 
type Multiplicty = Int
 
data CmmCat     -- "Category" (not exported)
   = GcPtrCat   -- GC pointer
   | BitsCat   -- Non-pointer
   | FloatCat   -- Float
   | VBitsCat  Multiplicity   -- Non-pointer
   | VFloatCat Multiplicity  -- Float
   deriving( Eq )
        -- See Note [Signed vs unsigned] at the end
```


Modify compiler/cmm/CmmMachOp.hs, this will add the necessary MachOps for use from the PrimOps modifications to support SIMD.  Here is an example of adding a SIMD version of the MO\_F\_Add MachOp:


```wiki
  -- Integer SIMD arithmetic
  | MO_V_Add  Width Int
  | MO_V_Sub  Width Int
  | MO_V_Neg  Width Int         -- unary -
  | MO_V_Mul  Width Int
  | MO_V_Quot Width Int

  -- Floating point arithmetic
  | MO_VF_Add Width Int   -- MO_VF_Add W64 4   Add 4-vector of 64-bit floats
  ...
```


Some existing Cmm instructions may be able to be reused, but there will have to be additional instructions added to account for vectorization primitives.  This will help keep the SIMD / non-SIMD code generation separate for the time being until we have it working.


## Add new PrimOps



Adding the new PrimOps is relatively straight-forward, but a  substantial number of LOC will be added to achieve it.  Most of this code is cut/paste "like" with minor type modifications.



Background: The following articles can aid in getting the work done:


- [
  Primitive Operations (PrimOps)](http://hackage.haskell.org/trac/ghc/wiki/Commentary/PrimOps)
- [
  Adding new primitive operations to GHC Haskell](http://hackage.haskell.org/trac/ghc/wiki/AddingNewPrimitiveOperations)
- Some guidelines for addition?, at least until I find something on the Wiki


The steps to be undertaken are:


1. Modify ./compiler/prelude/primops.txt.pp (the following instructions may be changed a bit based on the direction)

  1. Add the following vector length constants as Int\# types

    - intVecLen, intVec8Len, intVec16Len, intVec32Len, intVec64Len, wordVecLen, wordVec8Len, wordVec16Len, wordVec32Len, wordVec64Len, floatVecLen, and doubleVecLen, 
  1. Then add the following primtypes:

    - Int : IntVec\#, Int8Vec\#, Int16Vec\#, Int32Vec\#, Int64Vec\#
    - Word : WordVec\#, Word8Vec\#, Word16Vec\#, Word32Vec\#, Word64Vec\#
    - Float : FloatVec\#
    - Double : DoubleVec\#
  1. Add the following primops associated with the above primtypes.   The ops come in groups associated with the types above, for example for IntVec\#’s we get the following family for the “plus” operation alone:

    - plusInt8Vec\# :: Int8Vec\# -\> Int8Vec\# -\> Int8Vec\#
    - plusInt16Vec\# :: Int16Vec\# -\> Int16Vec\# -\> Int16Vec\#
    - plusInt32Vec\# :: Int32Vec\# -\> Int32Vec\# -\> Int32Vec\#
    - plusInt64Vec\# :: Int64Vec\# -\> Int64Vec\# -\> Int64Vec\#
  1. Repeat this for the following set of operations on IntVec\#’s of various lengths, note that the signatures are summarized informally in parentheses behind the operation:        

    - plusIntVec\#, (signature :: Int8Vec\# -\> Int8Vec\# -\> Int8Vec\#)
    - minusIntVec\#, 
    - timesIntVec\#, 
    - quotIntVec\#, 
    - remIntVec\#
    - negateIntVec\# (signature :: IntVec\# -\> IntVec\#)
    - uncheckedIntVecShiftL\# (signature :: IntVec\# -\> Int\# -\> IntVec\#)
    - uncheckedIntVecShiftRA\#, 
    - uncheckedIntVecShiftRL\# 
  1. For the Word vectors we similarly introduce:

    - plusWordVec\#, minusWordVec\#, timesWordVec\#, quotWordVec\#, remWordVec\#, negateWordVec\#, andWordVec\#, orWordVec\#, xorWordVec\#, notWord\#, uncheckedWordVecShiftL\#, uncheckedWordVecShiftRL\#
  1. Float

    - plusFloatVec\#, minusFloatVec\#, timesFloatVec\#, quotFloatVec\#, remFloatVec\#, negateFloatVec\#, expFloatVec\#, logFloatVec\#, sqrtFloatVec\#<sub> sinFloatVec\#, cosFloatVec\#, tanFloatVec\#, asinFloatVec\#, acosFloatVec\#, atanFloatVec\#, sinhFloatVec\#, coshFloatVec\#, tanhFloatVec\#
      </sub>
  1. Double

    - plusDoubleVec\#, minusDoubleVec\#, timesDoubleVec\#, quotDoubleVec\#, remDoubleVec\#, negateDoubleVec\#, expDoubleVec\#, logDoubleVec\#, sqrtDoubleVec\#, sinDoubleVec\#, cosDoubleVec\#, tanDoubleVec\#, asinDoubleVec\#, acosDoubleVec\#, atanDoubleVec\#, sinhDoubleVec\#, coshDoubleVec\#, tanhDoubleVec\#
1. Do NOT Modify ./compiler/prelude/PrimOp.lhs (actually, ./compiler/primop-data-decl.hs-incl) to add the new PrimOps (VIntQuotOp, etc...), this will be generated based on the primops.txt.pp modifications
1. Modify ./compiler/codeGen/CgPrimOp.hs, code for each primop (above) must be added to complete the primop addition.

  1. The code, basically, links the primops to the Cmm MachOps (that, in turn, are read by the code generators)
  1. It looks like some Cmm extensions will have to be added to ensure alignment and pass vectorization information onto the back ends, the necessary MachOps will be determined after the first vertical stack is completed (using the "Double" as a model).  There may be some reuse from the existing MachOps.  There is some discussion to these extensions (or similar ones) on the original [
    Patch 3557 Documentation](http://hackage.haskell.org/trac/ghc/ticket/3557)


Example of modification to ./compiler/prelude/primops.txt.pp to add one of the additional Float operations:


```wiki
------------------------------------------------------------------------
section "SIMDFloat"
	{Float operations that can take advantage of vectorization.}
------------------------------------------------------------------------
primtype FloatVec# a

primop   FloatVectorAddOp   "plusFloatVec#"      Dyadic            
   FloatVec# -> FloatVec# -> FloatVec#
   with can_fail = True

primop   FloatVectorAddOp   "plusFloatVec#"      Dyadic            
   FloatVec# -> FloatVec# -> FloatVec#
   with can_fail = True
```


Here is an example of the update to ./compiler/codeGen/CgPrimOp.hs that generates Machine Ops based on the new PrimOps.


```wiki
-- SIMD Float Ops
translateOp FloatVectorAddOp	= Just (MO_VF_Add W32 4)
translateOp FloatVectorMultOp	= Just (MO_VF_Mult W32 4)
```


The new primtype also needs to be weaved through the code generation path, but it is slightly different then primops.  To complete the primtype definition, the following files need to be modified.



./utils/genprimopcode/Main.hs needs to have an association added between the FloatVec\# type added above and a Type that is used for representation elsewhere:


```wiki
ppType (TyApp "FloatVec#"   []) = "floatVecPrimTy"
```


By adding the floatVecPrimTy, several additional relationships and constructs need to be created as well.



./compiler/prelude/TysPrim.lhs, wires the new type into Prelude:


```wiki
module TysPrim(
....
-- Added
	floatVecPrimTyCon,	floatVecPrimTy,
...

primTyCons 
  = [ addrPrimTyCon
...
-- Added
    , floatVecPrimTyCon

...
floatVecPrimTyConName		  = mkPrimTc (fsLit "FloatVec#") floatVecPrimTyConKey floatVecPrimTyCon

...
-- Add a new subsection for primitive types (others will be added here as well)
%************************************************************************
%*									*
\subsection[TysPrim-SIMDvectors]{The primitive SIMD vector types}
%*									*
%************************************************************************

\begin{code}
floatVecPrimTyCon :: TyCon
floatVecPrimTyCon		  = pcPrimTyCon  floatVecPrimTyConName	       1 PtrRep

floatVecPrimTy :: Type
floatVecPrimTy	    	    = mkTyConTy floatVecPrimTyCon
\end{code}
```


./compiler/prelude/PrelNames.lhs gives keys for each of the primtypes


```wiki
\subsection[Uniques-prelude-TyCons] ...
...
floatVecPrimTyConKey	= mkPreludeTyConUnique 38
```


./compiler/ghci/ByteCodeGen.lhs



./compiler/ghci/RtClosureInspect.hs


```wiki
repPrim t = rep where 
...
-- Added
    | t == floatVecPrimTyCon  = "<floatvec>"
```


The above, after compilation, adds the following to the ./compiler/prelude/PrimOp.lhs file:


```wiki
   | FloatVectorAddOp
   | FloatVectorMultOp
```

## Modify LLVM Code Generator



Take the MachOps in the Cmm definition and translate correctly to the corresponding LLVM instructions.  LLVM code generation is in the /compiler/llvmGen directory.  The following will have to be modified (at a minimum):


- /compiler/llvmGen/Llvm/Types.hs - add the MachOps from Cmm and how they bridge to the LLVM vector operations
- /compiler/llvmGen/LlvmCodeGen/CodeGen.hs - This is the heart of the translation from MachOps to LLVM code.   Possibly significant changes will have to be added.

- Remaining /compiler/llvmGen/\* - Supporting changes


At this point, CodeGen is not modified, though it will likely have to be eventually.  Types.hs has a new LMVector type added to support vectors.  As the operations on vectors are the same as all LLVM types (for float vectors use fadd, etc...), I have not made changes to the operators yet (though I'm guessing I will have to eventually).  Here is the diff of changes to Types.hs:


```wiki
[paul.monday@pg155-n19 Llvm]$ git diff Types.hs
diff --git a/compiler/llvmGen/Llvm/Types.hs b/compiler/llvmGen/Llvm/Types.hs
index 1013426..1133d37 100644
--- a/compiler/llvmGen/Llvm/Types.hs
+++ b/compiler/llvmGen/Llvm/Types.hs
@@ -38,6 +38,7 @@ data LlvmType
   | LMFloat128           -- ^ 128 bit floating point
   | LMPointer LlvmType   -- ^ A pointer to a 'LlvmType'
   | LMArray Int LlvmType -- ^ An array of 'LlvmType'
+  | LMVector Int LlvmType -- ^ A vector of 'LlvmType'
   | LMLabel              -- ^ A 'LlvmVar' can represent a label (address)
   | LMVoid               -- ^ Void type
   | LMStruct [LlvmType]  -- ^ Structure type
@@ -55,6 +56,7 @@ instance Show LlvmType where
   show (LMFloat128    ) = "fp128"
   show (LMPointer x   ) = show x ++ "*"
   show (LMArray nr tp ) = "[" ++ show nr ++ " x " ++ show tp ++ "]"
+  show (LMVector nr tp ) = "<" ++ show nr ++ " x " ++ show tp ++ ">"  
   show (LMLabel       ) = "label"
   show (LMVoid        ) = "void"
   show (LMStruct tys  ) = "<{" ++ (commaCat tys) ++ "}>"
@@ -295,6 +297,7 @@ llvmWidthInBits (LMFloat128)    = 128
 -- it points to. We will go with the former for now.
 llvmWidthInBits (LMPointer _)   = llvmWidthInBits llvmWord
 llvmWidthInBits (LMArray _ _)   = llvmWidthInBits llvmWord
+llvmWidthInBits (LMVector _ _)   = llvmWidthInBits llvmWord
 llvmWidthInBits LMLabel         = 0
 llvmWidthInBits LMVoid          = 0
 llvmWidthInBits (LMStruct tys)  = sum $ map llvmWidthInBits tys
```

## Modify Native Code Generator



Unfortunately, the native code generator will also have to be recompiled.  The GHC compilation depends on a 6.x version of GHC, before native LLVM code generation was built into the HEAD (so simply modifying the mk/build.mk file to go to -fllvm does not work).



For x86 Native Code Generation, locate the ./compiler/nativeGen/X86/CodeGen.hs file and modify it appropriately.  For the example above, simply adding a conversion from MO\_VF\_Add to the equivalent non-vector add is sufficient.


```wiki
	  -- SIMD Vector Instruction in Native Revert to Simple Instructions
      MO_VF_Add w i | sse2      -> trivialFCode_sse2 w ADD  x y
                  | otherwise -> trivialFCode_x87    GADD x y
      MO_VF_Mul w i | sse2      -> trivialFCode_sse2 w MUL x y
                  | otherwise -> trivialFCode_x87    GMUL x y	
```


Changes for the remaining new MachOps may be much larger.


## Example: Demonstrate SIMD Operation



Once the Code Generator, PrimOps and Cmm are modified, we should be able to demonstrate performance scenarios.  The simplest example to use for demonstrating performance is to time vector additions and multiplications using the new vectorized instruction set against a similar addition or multiplication using another PrimOp.



The following two simple programs should demonstrate the difference in performance.  The program using the PrimOps *should* improve performance approaching 2x (Doubles are 64bit and SSE provides two 64bit registers).



Simple usage of the new instructions to add to vectors of doubles:



**Question:**  How does one create one of the new PrimOp types to test prior to testing the vector add operations?  This is going to have to be looked at a little ... the code should basically create a vector and then insertDoubleVec\# repeatedly to populate the vector.  Without the subsequent steps done, this will have to be "hand" done without additional operations defined.  Here is the response from Manuel to expand on this:  I am not quite sure what the best approach is. The intention in LLVM is clearly to populate vectors using the 'insertIntVec\#' etc functions. However, in LLVM you can just use an uninitialised register and insert elements into a vector successively. We could provide a vector "0" value in Haskell and insert into that. Any other ideas?


```wiki
{-# LANGUAGE MagicHash #-}
import GHC.Prim
import GHC.Exts

getPrimFloat :: Float -> Float#
getPrimFloat f = case f of { F# f -> f }

main = do
    numberString <- getLine
    let num = read numberString
    let value = getPrimFloat num
    numberString2 <- getLine
    let num2 = read numberString2
    let value2 = getPrimFloat num2
    
    let packedVector1 = pack4FloatOp# value value value value
    let packedVector2 = pack4FloatOp# value2 value2 value2 value2

    let resultVector = multFloatVec4# packedVector1 packedVector2
    
    let result = extractFloatVec# resultVector 1

    let resultFloat = F# result
    print resultFloat
```


        
Using simple lists to achieve the same operation (note that the below is Integer only, I have to modify it to read floats off the command line otherwise a parse error occurs after the reads).


```wiki
main = do
    numberString <- getLine
    let value = read numberString
    numberString2 <- getLine
    let value2 = read numberString2

    let list1 = [value,value,value,value]
    let list2 = [value2,value2,value2,value2]

    let result = zipWith (*) list1 list2 

    print (result !! 1)
```


The above can be repeated with any of the common operations (multiplication, division, subtraction).  This should be sufficient with large sized vectors / lists to illustrate speedup.



(Note that over time and several generations of the integration, one would hope that the latter path would be “optimized” into SIMD instructions)


## Modify Vector Libraries and Vector Compiler Optimization (Pragmas and such)



Once we've shown there is speed-up for the lower portions of the compiler and have quantified it, the upper half of the stack should be optimized to take advantage of the vectorization code that was added to the PrimOps and Cmm.  There are two primary locations this is handled, in the compiler (compile/vectorize) code that vectorizes modules post-desugar process.  This location handles the VECTORISE pragmas as well as implicit vectorization of code.  The other location that requires modification is the Vector library itself.


1. Modify the Vector library /libraries/vector/Data to make use of PrimOps where possible and adjust VECTORISE pragmas if necessary

  - Modify the existing Vector code
  - We will likely also need vector versions of array read/write/indexing to process Haskell arrays with vector operations (this may need to go into compiler/vectorise)
  - Use the /libraries/vector/benchmarks to test updated code, look for

    - slowdowns - vector operations that cannot benefit from SIMD should not show slowdown
    - speedup - all performance tests that make use of maps for the common operators (+, -, \*, etc..) should benefit from the SIMD speedup
1. Modify the compiler/vectorise code to adjust pragmas and vectorization post-desugar process.  These modifications may not need to be made on the first pass through the code, more evaluation is necessary.

  - /compiler/vectorise/Vectorise.hs
  - /compiler/vectorise/Vectorise/Env.hs
  - /compiler/vectorise/Vectorise/Type/Env.hs


Once the benchmarks show measurable, reproducible behavior, move onto the DPH libraries.  Note that a closer inspection of the benchmarks in the /libraries/vector/benchmarks directory is necessary to ensure they reflect code that will be optimized with the use of SIMD instructions.  If they are not appropriate, add code that demonstrates SIMD speed-up appropriately.


## Modify DPH Libraries



The DPH libraries have heavy dependencies on the previous vectorization modification step (modifying the Vector libraries and the compiler vector options and post-desugar vectorization steps).  The DPH steps should not be undertaken without significant performance improvements illustrated in the previous steps.


1. The primary changes for DPH are in /libraries/dph/dph-common/Data/Array/Parallel/Lifted/\*
1. VECTOR SCALAR is also heavily used in /libraries/dph/dph-common/Data/Array/Parallel/Prelude, these should be inspected for update as well (Double.hs, Float.hs, Int.hs, Word8.hs)
1. Modify pragmas as necessary based on changes made above


**Note to Self:** Determine if the VECTORISE pragmas need adjustment or enhancement (based on previous steps)


## Ensure Remaining Code Generators Function Properly



There are really two options on the remaining code generators:


- Modify each code generator to understand the new Cmm instructions and restore them to non-vectorized instructions
- Add a compiler step that that does a pre-pass and replaces all "length = 1" vectors and operations on them by the corresponding scalar type and operations


The latter makes sense in that it is effective on all code generators, including the LLVM code generator.  Vectors of length = 1 should not be put through SIMD instructions to begin with (as they will incur substantial overhead for no return).



To make this work, a ghc compiler flag must be added that forces all vector lengths to 1 (this will be required in conjunction with any non-LLVM code generator).  A user can also use this option to turn off SIMD optimization for LLVM.


- Add the ghc compiler option: --vector-length=1
- Modify compiler/vectorise to recognize the new option or add this compiler pass as appropriate

## Reference Documentation


- [
  LLVM Vectorization](https://wiki.aalto.fi/display/t1065450/LLVM+vectorization)
- [ LLVM Vector Type](http://llvm.org/docs/LangRef.html#t_vector)

## Reference Discussion Threads


```wiki
From: Manual Chakravarty
Q: Should the existing pure Vector libraries (/libraries/vector/Data/*) be modified to use the vectorized code as a first priority, wait until DPH (/libraries/dph/) is modified, or leave the Vector library as is?

A: The DPH libraries ('dph-*') are based on the 'vector' library — i.e., for DPH to use SIMD instruction, we must modify 'vector' first.

Q: How does one create one of the new Vector Types in a Haskell program (direct PrimOp?, for testing ... let x = ????)

A: I am not quite sure what the best approach is. The intention in LLVM is clearly to populate vectors using the 'insertIntVec#' etc functions. However, in LLVM you can just use an uninitialised register and insert elements into a vector successively. We could provide a vector "0" value in Haskell and insert into that. Any other ideas?

A: I just realised that we need vector version of the array read/write/indexing operations as well to process Haskell arrays with vector operations.

Q: One discussion point was that the "Vector Lengths" should be "Set to 1" for non LLVM code generation, where does this happen? On my first survey of the code, it seems that the code generators are partitioned from the main body of code, implying that each of the code generators will have to be modified to account for the new Cmm MachOps? and properly translate them to non-vectorized instructions.

A: Instead of doing the translation for every native code generator separately, we could have a pre-pass that replaces all length = 1 vectors and operations on them by the corresponding scalar type and operation.  Then, the actual native code generators wouldn't need to be changed.

A: The setting of the vector length to 1 needs to happen in dependence on the command line options passed to GHC — i.e., if a non-LLVM backend is selected.

Q: Can we re-use any of the existing MachOps? when adding to Cmm?

A: I am not sure.
```