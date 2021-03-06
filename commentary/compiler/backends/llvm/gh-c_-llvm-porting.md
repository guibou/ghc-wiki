# Porting GHC using LLVM backend



This document is kind of short porting roadmap which serves as a high-level overview for porters of GHC who decided to use LLVM instead of implementing new NCG for their target platform. Please have [Design & Implementation](commentary/compiler/backends/llvm/design) at hand since this contains more in-depth information.
The list of steps needed for new GHC/LLVM port is:



**(1)** Make sure GHC unregisterised build is working on your target platform (using the C backend). This guide isn't intended for porting GHC to a completely unsupported platform. If the platform in question doesn't have a GHC unregisterised build then follow the [GHC Porting Guide](building/porting) first.



**(2)** Now try to compile some very simple programs such as 'hello world' or simpler using the GHC you just built. Try with the C backend First to make sure everything is working. Then try with the LLVM backend. If the llvm backend built programs are failing find out why. This is done using a combination of things such as the error message you get when the program fails, [tracing the execution with GDB](debugging/compiled-code) and also just comparing the assembly code produced by the C backend to what LLVM produces. This last method is often the easiest and you can occasionally use techniques like doing doing a 'binary search' for the bug by merging the assembly produced by the C backend and LLVM backend.



**(3)** When the programs you throw at the LLVM backend are running, try running the GHC testsuite. First run it against the C backend to get a baseline, then run it against the LLVM backend. Fix any failures that are LLVM backend specific.



**(4)** If the testsuite is passing, now try to build GHC itself using the LLVM backend. This is a very tough test. When working though its a good proof that the LLVM backend is working well on your platform.



**(5)** Now you have LLVM working in unregistered mode, so the next thing is to implement the GHC calling convention in LLVM that is used by GHC's LLVM backend. This should then allow you to get the LLVM backend working in registered mode but with (TABLES\_NEXT\_TO\_CODE = NO in your build.mk). Majority of this step involves hacking inside the LLVM code. Usually lib/Target/\<your target platform name\> is the best way to start. Also you might study what David Terei did for [
x86 support](http://lists.cs.uiuc.edu/pipermail/llvmdev/2010-March/030031.html) and his [
patch itself](http://lists.cs.uiuc.edu/pipermail/llvmdev/attachments/20100307/714e5c37/attachment-0001.obj) to get an idea what's really needed.



**(6)** Once **(5)** is working you have it all running except TABLES\_NEXT\_TO\_CODE. So change that to Yes in your build.mk and get that working. This will probably involve changing the mangler used by LLVM to work on the platform you are targeting.


## Registerised Mode



Here is an expanded version of what needs to be done in step 5 and 6 to get a registerised port of LLVM working:


1. GHC in registerised mode stores some of its virtual registers in real hardware registers for performance. You will need to decide on a mapping of GHC's virtual registers to hardware registers. So how many registers you want to map and which virtual registers to store and where. GHC's design for this on X86 is basically to use as many hardware registers as it can and to store the more frequently cessed virtual registers like the stack pointer in callee saved registers rather than caller saved registers. You can find the mappings that GHC currently uses for supported architectures in 'includes/stg/MachRegs.h'.

1. You will need to implement a custom calling convention for LLVM for your platform that supports passing arguments using the register map you decided on. You can see the calling convention I have created for X86 in the llvm source file 'lib/Target/X86/X86CallingConvention.td'.

1. Get GHC's build system running on your platform in registerised mode.

1. Add new inline assembly code for your platform to ghc's RTS. See files like 'rts/StgCRun.c' that include assembly code for the architectures GHC supports. This is the main place as its where the boundary between the RTS and haskell code is but I'm sure there are definitely other places that will need to be changed. Just grep the source code to find existing assembly and add code for your platform appropriately.

1. Will need to change a few things in LLVM code gen.


5.1 'compiler/llvmGen/LlvmCodeGen/Ppr.hs' defines a platform specific string that is included in all generated llvm code. Add one for your platform. This string specifies the datalayout parameters for the platform (e.g pointer size, word size..). If you don't include one llvm should still work but wont optimise as aggressively.



5.2 'compiler/llvmGen/LlvmCodeGen/CodeGen.hs' has some platform specific code on how write barriers should be handled.


1. Probably some stuff elsewhere in ghc that needs to be changed (most likely in the main/ subfolder which is where most the compiler driver lives or in codegen/ which is the Cmm code generator).

1. This is just what I know needs to be done, I'm sure there is many small pieces missing although they should all fall into one of the above categories. In the end just trial and error your way to success.
